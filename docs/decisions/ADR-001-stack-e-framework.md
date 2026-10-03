# ADR-001: Stack e Framework

## Status
Aceito

## Contexto
O Pluriverso precisa de uma stack de implementação (runtime, framework HTTP, camada de views, empacotamento de front-end) definida antes de qualquer código ser escrito. Nenhum repositório do Pluriverso contém essa decisão hoje: `docs/principios.md` delega a escolha ("Encontre o melhor framework...") mas fixa uma restrição concreta em §Frontend — "mesmo look and feel de ferramentas já implementadas, como BioCultDB". [ADR-008/DB6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md) menciona "SvelteKit + shadcn-svelte" como stack de front-end, mas essa menção cita um arquivo `Desenvolvimento.md` ("Parâmetros de Desenvolvimento") que **não existe** em nenhum repositório da federação — é contexto não-normativo, órfão de fonte. O usuário liberou explicitamente, nesta sessão, a escolha de stack para o Pluriverso, com a orientação de replicar o padrão já em produção na federação.

Esta decisão fixa a stack de runtime e framework. A API pública (REST vs. GraphQL) é decidida em separado em [ADR-002 local](ADR-002-api-publica-rest.md); esta ADR só registra, por referência, que a ausência de GraphQL aqui é consistente com aquela decisão.

## Requisitos
### Funcionais
- Servir uma API HTTP REST paginada (contrato de harvest e API pública) e uma UI web server-rendered com rolagem infinita (`docs/principios.md`).
- Persistir dados em SQLite embutida no mesmo container, com engine em JS/Node sem dependência de servidor de banco externo ([ADR-008/DB1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)).
- Agendar rotinas periódicas de harvest no próprio processo, sem fila ou worker externo.
- Servir busca textual sem depender de motor de busca externo.

### Não-Funcionais
- Imagem Docker mínima (`docs/principios.md` §Empacotamento); evitar etapas de build de SPA e dependências pesadas.
- Mesmo look and feel das ferramentas já implementadas da federação, para não introduzir uma terceira convenção visual/técnica.
- Superfície de segurança pequena: sem serviços auxiliares (cache, fila, motor de busca) que ampliem o perímetro de ataque.
- Reuso de padrões já validados em produção (BioCultDB, BioCultTermos) sobre soluções novas sem histórico na federação.

## Opções Consideradas

### Opção 1: Stack BioCultDB (escolhida)
Replicar exatamente a stack já em produção no BioCultDB, com componentes de BioCultTermos onde o BioCultDB não cobre a necessidade (agendamento, headers de segurança, hash de senha, log, compressão).

|Camada|Escolha|Versão|Origem|
|---|---|---|---|
|Runtime|Node.js|`>=20.0.0` (imagem `node:20-alpine`)|BioCultDB `engines`|
|HTTP|Express|`^4.18.2`|BioCultDB|
|Views|EJS|`^3.1.9`|BioCultDB|
|CSS|Tailwind CSS|`^3.4.1` (devDependency)|BioCultDB|
|Interatividade|Alpine.js `3.13.3` + HTMX `1.9.10`|assets locais (sem CDN)|BioCultDB|
|SQLite|better-sqlite3|`^11.3.0`|[ADR-008/DB6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md) + BioCultDB|
|Agendamento|node-cron|`^4.6.0`|BioCultTermos|
|Headers de segurança|helmet|`^7.1.0`|BioCultTermos|
|Hash de senha|bcrypt|`^6.0.0`|BioCultTermos|
|Log|winston|`^3.11.0`|BioCultTermos|
|Compressão|compression|`^1.7.4`|BioCultTermos|
|Config|dotenv|`^16.3.1`|BioCultDB|
|Ícones|Boxicons|pacote local, sem CDN|`docs/principios.md`|

**Prós:**
- Mesmo look and feel do BioCultDB, satisfazendo `docs/principios.md` §Frontend diretamente, sem interpretação.
- Toda a stack já roda em produção na federação — zero risco de descoberta tardia de incompatibilidade.
- Sem etapa de build de SPA: EJS é renderizado no servidor, Tailwind compila para CSS estático, Alpine.js/HTMX são scripts locais sem bundler.
- Imagem Docker pequena: `node:20-alpine` + dependências puramente Node, sem toolchain de front-end além do Tailwind CLI.
- `better-sqlite3` é síncrono e already-validated para o padrão de tabela `doc JSON` + colunas geradas de [ADR-005/DA2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-005-sqlite-json-persistence.md).

**Contras:**
- EJS + Alpine/HTMX é menos expressivo que um framework de componentes reativos para interações client-side complexas.
- Nenhuma dessas bibliotecas foi escolhida pensando especificamente no Pluriverso — é reuso, não uma escolha otimizada para o caso de uso.

### Opção 2: SvelteKit + shadcn-svelte (rejeitada)
A stack mencionada em [ADR-008](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md), citando um `Desenvolvimento.md` que não existe em nenhum repositório da federação.

**Prós:**
- Componentes reativos, DX moderna, SSR nativo.

**Contras:**
- Adiciona etapa de build de SPA (Vite + adapter-node), aumentando a imagem Docker e o tempo de build no CI.
- Divergiria do look-and-feel exigido por `docs/principios.md` §Frontend — nenhuma outra ferramenta da federação usa Svelte.
- A fonte da recomendação (`Desenvolvimento.md`) não existe; não há especificação de versões, componentes shadcn-svelte a usar, nem padrão de projeto a seguir — adotá-la exigiria decidir tudo do zero mesmo assim.

### Opção 3: Fastify + Preact SSR (rejeitada)
Combinação alternativa mais leve que Express, com Preact para SSR de componentes.

**Prós:**
- Fastify tem overhead de request menor que Express em benchmarks sintéticos.

**Contras:**
- Nenhum ganho mensurável sobre Express no volume esperado do Pluriverso (federação com dezenas de membros, não milhares de req/s).
- Zero reuso de padrão já validado na federação — nem BioCultDB nem BioCultTermos usam Fastify ou Preact; seria uma quarta convenção técnica introduzida sem necessidade.
- Exigiria retreinar decisões de middleware (auth, rate limit, error handling) já resolvidas no ecossistema Express usado pelos demais membros.

## Decisão
Adotar a stack da **Opção 1**, replicando exatamente o BioCultDB (com componentes de BioCultTermos onde aplicável), conforme a tabela acima.

Registrar explicitamente as ausências deliberadas:

- **Sem GraphQL** — a API pública do Pluriverso é REST-only (decisão em [ADR-002 local](ADR-002-api-publica-rest.md)); GraphQL dobraria a superfície de manutenção (resolvers, introspection) para um caso de busca com filtros fixos.
- **Sem Redis** — não há necessidade de cache distribuído com um único processo servindo um único arquivo SQLite; cache em memória do processo, quando necessário, é suficiente.
- **Sem Elasticsearch** — FTS5 do próprio SQLite cobre a busca textual ([ADR-008/DB4](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)), sem exigir um serviço externo nem sincronização de índice entre dois bancos.
- **Sem TipTap** — o Pluriverso não tem autoria de texto rico; é middleware de federação (harvest, indexação, busca), não ferramenta de aquisição de conteúdo.
- **Sem servidor de banco** — a engine SQLite é embutida no processo Node via `better-sqlite3` ([ADR-008/DB1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)), sem processo de banco separado a operar ou expor.

**Rolagem infinita** (exigida por `docs/principios.md`) é resolvida inteiramente na UI: HTMX com `hx-trigger="revealed"` num elemento sentinela dispara a próxima página contra a mesma API REST paginada — não é necessária nenhuma biblioteca nova. A **API permanece paginada** (contrato REST convencional, com `page`/`limit` ou `page`/`size` conforme o endpoint); apenas a camada de apresentação HTML encadeia as páginas visualmente. Paginação de API e rolagem infinita de UI não são requisitos conflitantes — são camadas diferentes.

## Consequências
### Positivas
- Qualquer pessoa familiarizada com o BioCultDB ou o BioCultTermos consegue orientar-se no código do Pluriverso sem curva de aprendizado adicional de framework.
- Imagem Docker pequena e build simples, sem toolchain de SPA — alinhado a `docs/principios.md` §Empacotamento.
- Todas as dependências já têm histórico de produção conhecido dentro da federação, reduzindo risco de descoberta tardia de bug ou vulnerabilidade não mapeada.
- Nenhum serviço auxiliar (cache, fila, motor de busca, servidor de banco) amplia o perímetro de segurança ou a lista de pontos de falha.

### Negativas
- A stack não foi escolhida pensando em otimizar especificamente o caso de uso do Pluriverso — é herdada por consistência, não por análise de requisito único.
- EJS + Alpine/HTMX oferece menos expressividade client-side que um framework reativo, caso a UI do Pluriverso cresça em complexidade de interação no futuro.

### Mitigações
- Se a UI do Pluriverso exigir interações client-side significativamente mais ricas que rolagem infinita e filtros simples, reavaliar apenas a camada de interatividade (Alpine.js/HTMX), sem tocar runtime, HTTP ou persistência — a stack foi escolhida em camadas independentes justamente para permitir essa troca isolada.
- Se `Desenvolvimento.md` vier a existir formalmente e fixar uma decisão de stack distinta desta, esta ADR é revisitada (ver Data de Revisão).

## Referências
- [ADR-008: Pluriverso Database Engine](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md) — DB1 (engine embutida), DB4 (FTS5), DB6 (`better-sqlite3`); origem da menção não-normativa a SvelteKit+shadcn-svelte via `Desenvolvimento.md` inexistente.
- [ADR-006: Federation Membership Protocol](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-006-federation-membership-protocol.md) — E1, contexto de convenções da federação que esta stack deve honrar em superfície (formulários, validação server-side).
- `docs/principios.md` §Frontend, §Empacotamento, §Segurança — requisito de look-and-feel equivalente ao BioCultDB, rolagem infinita, imagem Docker mínima.
- [ADR-002 local: API Pública REST](ADR-002-api-publica-rest.md) — decisão irmã que fixa REST-only, referenciada pela ausência de GraphQL nesta ADR.
- `package.json` do BioCultDB e do BioCultTermos (repositórios da federação) — origem das versões registradas na tabela de stack.

## Data de Revisão
Revisitar se `Desenvolvimento.md` vier a existir formalmente no repositório da Arquitetura-BioCultural com uma decisão de stack de front-end distinta desta (por exemplo, adoção formal de SvelteKit + shadcn-svelte com versionamento e convenções especificadas). Até lá, esta decisão permanece válida sem prazo fixo de revisão.
