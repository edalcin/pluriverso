# Operação

Este documento cobre a operação do Pluriverso em produção: variáveis de ambiente, empacotamento Docker, integração contínua, implantação no UNRAID, backup/restauração do índice SQLite e observabilidade mínima. Nenhum arquivo de código, `Dockerfile` ou workflow de CI é criado por este documento — o conteúdo de cada um é **descrito** aqui, em bloco de código, para que a implementação futura o materialize sem ambiguidade. A stack de runtime (Node.js/Express, `better-sqlite3`) está fixada em [`docs/decisions/ADR-001-stack-e-framework.md`](decisions/ADR-001-stack-e-framework.md); os nomes de componente citados (`HarvestScheduler` etc.) estão fixados em [`docs/arquitetura.md`](arquitetura.md).

---

## Variáveis de Ambiente

Conjunto **fechado** — nenhuma outra variável é lida pela aplicação. `SQLITE_DB_PATH` materializa o `DB_PATH` exigido por `docs/principios.md` §Dados; os defaults de harvest e de expansão semântica reproduzem, verbatim, os valores já fixados em [`docs/contrato-harvest.md`](contrato-harvest.md) e [`docs/busca-semantica.md`](busca-semantica.md) — este documento não os redecide, só os expõe como configuração.

| Variável | Obrigatória | Default | Descrição |
|---|---|---|---|
| `SQLITE_DB_PATH` | Opcional | `/data/pluriverso.sqlite` | Caminho do arquivo SQLite, externo ao container, em volume persistente ([ADR-008/DB2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)). |
| `PORT` | Opcional | `3100` | Porta HTTP escutada pela aplicação Node.js/Express ([`docs/arquitetura.md`](arquitetura.md), Nível 2). |
| `NODE_ENV` | Opcional | `production` | Modo de execução do Node/Express; controla verbosidade de erro e cache de views EJS. |
| `INSTANCE_NAME` | **Obrigatória** | — | Identificador da instância, exibido em `GET /health` e no rodapé da `WebUi` — sem default sensato num contexto multi-instância ([ADR-009/MI5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md)). |
| `PUBLIC_BASE_URL` | **Obrigatória** | — | URL pública desta instância; base dos `links` de paginação e da variável de servidor `{baseUrl}` de [`docs/api/openapi.yaml`](api/openapi.yaml). |
| `COMMITTEE_USERS` | **Obrigatória** | — | JSON com as contas nomeadas do Comitê Federado (`[{"username":...,"passwordHash":...}]`, bcrypt custo 12) — sem isso, nenhum endpoint de [`docs/api.md`](api.md) §5.2 é acessível ([`docs/decisions/ADR-003-autenticacao-do-comite.md`](decisions/ADR-003-autenticacao-do-comite.md)). |
| `HARVEST_CRON_INCREMENTAL` | Opcional | `0 3 * * *` | Expressão cron do harvest incremental default, sobrescrevível por membro ([`docs/contrato-harvest.md`](contrato-harvest.md) §2). |
| `HARVEST_CRON_FULL` | Opcional | `0 4 * * 0` | Expressão cron do harvest completo default, sobrescrevível por membro ([`docs/contrato-harvest.md`](contrato-harvest.md) §2). |
| `HARVEST_PAGE_SIZE` | Opcional | `100` | Tamanho de página que o `HarvestClient` solicita (`size`), nunca o máximo de 500 do contrato ([`docs/contrato-harvest.md`](contrato-harvest.md) §1). |
| `HARVEST_MAX_PAGES` | Opcional | `1000` | Guarda-corpo de páginas por execução de harvest; excedido → run `partial` ([`docs/contrato-harvest.md`](contrato-harvest.md) §1). |
| `HARVEST_TIMEOUT_MS` | Opcional | `10000` | Timeout total por página de harvest (conexão + resposta completa) ([`docs/contrato-harvest.md`](contrato-harvest.md) §3). |
| `HARVEST_MAX_CONCURRENT` | Opcional | `3` | Número máximo de runs de harvest simultâneas, entre membros diferentes ([`docs/contrato-harvest.md`](contrato-harvest.md) §3). |
| `HARVEST_MAX_FAILURES` | Opcional | `5` | Runs `failed` consecutivas que desativam `harvest_enabled` de um membro ([`docs/contrato-harvest.md`](contrato-harvest.md) §3). |
| `SKOS_EXPANSION_MAX_DEPTH` | Opcional | `3` | Profundidade máxima de `skos:narrowMatch`/`skos:broadMatch` na expansão semântica ([`docs/busca-semantica.md`](busca-semantica.md)). |
| `RATE_LIMIT_PUBLIC_PER_MIN` | Opcional | `60` | Requisições/minuto por IP para `/api/v1/*` ([`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md)). |
| `RATE_LIMIT_MEMBERSHIP_PER_HOUR` | Opcional | `5` | Pedidos/hora por IP para `POST /api/federation/membership-requests` ([`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md)). |
| `LOG_LEVEL` | Opcional | `info` | Nível mínimo de log do winston (`error`\|`warn`\|`info`\|`debug`). |

### `.env.example`

Conteúdo do arquivo `.env.example` a versionar no repositório — com placeholders genéricos, nunca segredo real (`docs/principios.md` §Versionamento). **Este documento mostra o conteúdo; não cria o arquivo.**

```
# Persistência
SQLITE_DB_PATH=/data/pluriverso.sqlite

# Servidor
PORT=3100
NODE_ENV=production
INSTANCE_NAME=pluriverso-instancia-exemplo
PUBLIC_BASE_URL=https://pluriverso.example.org

# Autenticação do Comitê Federado (ADR-003 local)
# Gerar hash bcrypt (custo 12) offline por conta; nunca commitar hash real.
COMMITTEE_USERS=[{"username":"comite-exemplo","passwordHash":"YOUR_BCRYPT_HASH"}]

# Harvest
HARVEST_CRON_INCREMENTAL=0 3 * * *
HARVEST_CRON_FULL=0 4 * * 0
HARVEST_PAGE_SIZE=100
HARVEST_MAX_PAGES=1000
HARVEST_TIMEOUT_MS=10000
HARVEST_MAX_CONCURRENT=3
HARVEST_MAX_FAILURES=5

# Busca semântica
SKOS_EXPANSION_MAX_DEPTH=3

# Rate limiting
RATE_LIMIT_PUBLIC_PER_MIN=60
RATE_LIMIT_MEMBERSHIP_PER_HOUR=5

# Log
LOG_LEVEL=info
```

`.env` (com valores reais) permanece no `.gitignore`, nunca commitado.

---

## Imagem Docker

Build **multi-stage** sobre `node:20-alpine`, alvo de tamanho **< 200 MB** (`docs/principios.md` §Empacotamento). O toolchain nativo de compilação existe **só** no estágio de build — a imagem final não carrega compilador algum, mitigação explícita registrada em [ADR-008 — Motor de Banco de Dados do Pluriverso, seção Mitigações](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md), para o binário nativo de `better-sqlite3` não obrigar a imagem final a carregar `python3`/`make`/`g++`.

Conteúdo de referência do `Dockerfile` (não criado como arquivo por este documento):

```dockerfile
# ---- estágio de build ----
FROM node:20-alpine AS build
WORKDIR /app

# Toolchain nativo só neste estágio: compila o binding nativo de
# better-sqlite3 (ADR-008/DB6) e nunca chega à imagem final.
RUN apk add --no-cache python3 make g++

COPY package.json package-lock.json ./
RUN npm ci --omit=dev=false

COPY . .
RUN npm run build:css   # compila Tailwind (docs/decisions/ADR-001-stack-e-framework.md) para CSS estático
RUN npm prune --omit=dev

# ---- estágio final ----
FROM node:20-alpine
WORKDIR /app

RUN addgroup -g 1001 nodejs && adduser -D -u 1001 -G nodejs nodejs \
    && mkdir -p /data && chown nodejs:nodejs /data \
    && apk add --no-cache dumb-init

COPY --from=build --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nodejs:nodejs /app/public ./public
COPY --from=build --chown=nodejs:nodejs /app/views ./views
COPY --from=build --chown=nodejs:nodejs /app/src ./src
COPY --from=build --chown=nodejs:nodejs /app/migrations ./migrations
COPY --from=build --chown=nodejs:nodejs /app/package.json ./package.json

USER nodejs
EXPOSE 3100
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3100/health || exit 1

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "src/index.js"]
```

Pontos fixados:

- **`node:20-alpine`** nos dois estágios — mesma imagem base de [`docs/decisions/ADR-001-stack-e-framework.md`](decisions/ADR-001-stack-e-framework.md).
- **Usuário `nodejs:1001`, não-root** — `docs/principios.md` §Segurança e [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md) §Endurecimento do container.
- **`/data` criado e `chown` para `nodejs`** antes do `USER nodejs` — é onde `SQLITE_DB_PATH` grava por padrão.
- **`dumb-init` como entrypoint** — encaminha sinais (`SIGTERM`) corretamente ao processo Node, para desligamento gracioso em vez de matar o container sem drenar conexões em curso.
- **`HEALTHCHECK` contra `GET /health`** ([`docs/api.md`](api.md)) — o orquestrador (UNRAID, `docker compose`, etc.) usa esse status para saber se a instância está pronta.
- **`EXPOSE 3100`** — mesma porta do `PORT` default.

---

## Integração Contínua (CI)

Workflow `.github/workflows/docker-publish.yml`, **descrito** aqui — não criado como arquivo:

```yaml
name: docker-publish

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/edalcin/pluriverso
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha
            type=semver,pattern={{version}}

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/edalcin/pluriverso:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

Publica em `ghcr.io/edalcin/pluriverso` (`docs/principios.md` §Empacotamento) com tags `latest` (só em `main`), `type=sha` (rastreabilidade por commit) e `type=semver` (releases marcadas por tag `v*.*.*`). O passo de **Trivy** falha o build em vulnerabilidade `CRITICAL`/`HIGH` na imagem publicada, atendendo `docs/principios.md` §Segurança e o controle já registrado em [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md) §CI — Dependabot + Trivy. **Dependabot** (`.github/dependabot.yml`, também descrito e não criado) mantém `npm` e `github-actions` atualizados, com PR automático em nova versão de dependência.

---

## Implantação no UNRAID

Via **Docker → Add Container** na interface gráfica do UNRAID (`docs/principios.md` §Empacotamento):

| Campo UNRAID | Valor |
|---|---|
| Repository | `ghcr.io/edalcin/pluriverso:latest` |
| Network Type | `Bridge` |
| Port: Container Port | `3100` |
| Port: Host Port | `3100` (ou outra porta livre, se já ocupada) |
| Path: Container Path | `/data` |
| Path: Host Path | `/mnt/user/appdata/pluriverso` (ou equivalente) |
| Path: Access Mode | `Read/Write` |

Variáveis de ambiente, uma entrada **Variable** por linha da tabela acima (`Config Type: Variable`), com `Key`/`Value` correspondentes — em particular `SQLITE_DB_PATH=/data/pluriverso.sqlite` (deve apontar para dentro do volume mapeado), `INSTANCE_NAME`, `PUBLIC_BASE_URL` e `COMMITTEE_USERS` preenchidos com valores reais da instância (nunca os placeholders do `.env.example`).

---

## Backup e Restauração

**Backup consistente, com a aplicação no ar.** Em modo WAL (`PRAGMA journal_mode = WAL`, [ADR-005/DA1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-005-sqlite-json-persistence.md)), copiar o arquivo `.sqlite` diretamente com `cp` é inseguro — o conteúdo confirmado pode estar parcialmente no arquivo `-wal`, ainda não incorporado ao arquivo principal. O procedimento correto usa a própria engine SQLite para produzir uma cópia consistente sem parar o container:

```sql
VACUUM INTO '/backup/pluriverso-2026-07-31.sqlite';
```

Executado como uma instrução SQL contra a conexão já aberta pela aplicação (ou por uma ferramenta `sqlite3` externa apontando para o mesmo arquivo) — `VACUUM INTO` gera um arquivo novo, compacto e transacionalmente consistente, sem exigir *downtime*.

**Cópia bruta (alternativa, exige parar o container):** parar o container, copiar os três arquivos juntos — `pluriverso.sqlite`, `pluriverso.sqlite-wal`, `pluriverso.sqlite-shm` — e só então reiniciar. Copiar só o `.sqlite` sem os arquivos `-wal`/`-shm` associados perde as transações ainda não sincronizadas ao arquivo principal.

**Reindexação total do FTS.** Se `records_fts` ([`docs/modelo-de-dados.md` — 5. `records_fts`](modelo-de-dados.md)) corromper ou dessincronizar de `records` (por exemplo, após uma recuperação de crash malsucedida), o índice de busca é inteiramente reconstruível a partir de `records`, sem nenhum novo harvest contra os membros: `TRUNCATE` (via `DELETE FROM records_fts`) seguido de um `INSERT` para cada linha de `records`, recompondo `titulo`/`corpo` pelo mesmo algoritmo de achatamento já especificado. O `data` íntegro de cada registro permanece armazenado — a reindexação é uma operação puramente local, o mesmo raciocínio que já vale para o `profile` derivado (`docs/contrato-harvest.md` §4).

---

## Observabilidade

**Log estruturado.** Toda linha de log é JSON, via `winston` ([`docs/decisions/ADR-001-stack-e-framework.md`](decisions/ADR-001-stack-e-framework.md)), nível controlado por `LOG_LEVEL`. Cada execução de harvest — incremental ou completa, bem-sucedida ou não — produz **um** registro de log correlacionado ao seu `harvest_runs.id` ([`docs/modelo-de-dados.md` — 8. `harvest_runs`](modelo-de-dados.md)), com `member_id`, `mode`, `status`, `pages_fetched`, `upserted`, `removed`, `rejected` e a duração total — os mesmos campos já persistidos na tabela, espelhados no log para agregação externa (ex.: um coletor de log do UNRAID) sem exigir consulta ao SQLite.

**`/health` com estado por membro.** `GET /health` ([`docs/api.md`](api.md)) devolve `status`, `version`, `instance_name`, `db` e `last_harvest_at` — um mapa `member_id → timestamp da última run concluída`, para o operador identificar rapidamente um membro cujo harvest parou de rodar, sem abrir o painel do Comitê.

**Sem Prometheus, sem OpenTelemetry.** O volume esperado da federação (dezenas de membros, tráfego de busca modesto) não paga a dependência e a superfície operacional adicional de um coletor de métricas ou de um backend de tracing dedicado — log estruturado por run de harvest e um `/health` informativo já cobrem o que o operador de uma instância precisa monitorar no dia a dia.
