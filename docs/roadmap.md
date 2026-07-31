# Roadmap

Sete fases de implementação, cada uma com uma entrega observável e um critério de aceitação que não depende de leitura de código — só de exercitar o sistema. A ordem é sequencial: cada fase pressupõe que a anterior está aceita. Para cada fase, a seção "Documentos-fonte" lista **só** o que é preciso ler para implementá-la — é o "teste da mesa vazia": ler apenas esses documentos deve bastar, sem decisão de arquitetura pendente.

| Fase | Entrega | Critério de Aceitação |
|---|---|---|
| 0 | Esqueleto: Express, migrações, `/health`, Docker, CI | A imagem publica em `ghcr.io/edalcin/pluriverso` e `GET /health` responde `200` |
| 1 | Membership: `POST`/`GET`/`PATCH` + probe + auth do Comitê | Um pedido entra `pending`, o probe grava `technical_check`, e o Comitê aprova e recebe um `member_id` gerado |
| 2 | Harvest + índice: client, indexer, FTS, `harvest_runs` | Contra um membro simulado, uma run completa indexa N registros e `GET /api/v1/search?q=` os devolve |
| 3 | API pública + UI de busca com rolagem infinita | Busca textual com filtros funciona, e toda resposta carrega atribuição de origem |
| 4 | SKOS: conceitos, mapeamentos, expansão, painel de curadoria | Busca por termo de um membro devolve registro de outro, com `matched_via` |
| 5 | Purge + auditoria + painel do Comitê completo | `purge` remove registros e mapeamentos, `audit_log` registra a ação, e `VACUUM` reduz o tamanho do arquivo |
| 6 | Detecção de remoção (modo completo) + desativação por falha | Um registro removido no membro desaparece do índice depois de uma run completa |

---

## Fase 0 — Esqueleto

**Entrega:** projeto Express inicializado, mecanismo de migração de esquema funcionando, `GET /health` respondendo, `Dockerfile` publicando imagem via CI.

**Critério de aceitação:** a imagem publica em `ghcr.io/edalcin/pluriverso` e `GET /health` responde `200` — sem depender de nenhum outro subsistema (harvest, busca, membership) já implementado.

**Documentos-fonte:**
- [`docs/decisions/ADR-001-stack-e-framework.md`](decisions/ADR-001-stack-e-framework.md) — runtime, framework HTTP, versões exatas de dependência.
- [`docs/modelo-de-dados.md`](modelo-de-dados.md#migrações) — mecanismo de `schema_migrations` e convenção de arquivo `migrations/NNN-descricao.sql`.
- [`docs/operacao.md`](operacao.md) — variáveis de ambiente, `Dockerfile` de referência, workflow de CI.

---

## Fase 1 — Membership

**Entrega:** `POST`/`GET`/`PATCH /api/federation/membership-requests`, probe técnico anti-SSRF, autenticação HTTP Basic do Comitê Federado.

**Critério de aceitação:** um pedido de adesão submetido via `POST` entra com `status: pending`; o `ProbeService` grava `technical_check` no próprio pedido; e uma conta do Comitê, autenticada, aprova o pedido via `PATCH` — o pedido passa a `active` e um `member_id` novo é gerado.

**Documentos-fonte:**
- [ADR-006 — Protocolo de Inscrição na Federação](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md) (Arquitetura-BioCultural) — contrato de campos, máquina de estados, E1–E5.
- [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md) — máquina de estados em Mermaid, painel do Comitê, algoritmo completo do probe anti-SSRF.
- [`docs/decisions/ADR-003-autenticacao-do-comite.md`](decisions/ADR-003-autenticacao-do-comite.md) — mecanismo de autenticação (Basic Auth + bcrypt, `COMMITTEE_USERS`).
- [`docs/api.md`](api.md) §5.1 e §5.2 — superfície HTTP exata dos três endpoints, corpo, respostas e códigos de erro.

---

## Fase 2 — Harvest e índice

**Entrega:** `HarvestClient` (coleta paginada), `RecordIndexer` (upsert + perfil + FTS), tabela `harvest_runs`.

**Critério de aceitação:** contra um membro simulado (endpoint de harvest de teste), uma run em modo completo indexa N registros, e `GET /api/v1/search?q=<termo presente nos registros>` os devolve.

**Documentos-fonte:**
- [`docs/contrato-harvest.md`](contrato-harvest.md) — contrato de registros, comportamento do coletor, modos de coleta, resiliência, Perfil Mínimo de Publicação.
- [`docs/modelo-de-dados.md`](modelo-de-dados.md) — DDL de `records`, `record_terms`, `records_fts`, `harvest_runs`.

---

## Fase 3 — API pública e UI

**Entrega:** superfície pública completa (`§5.1` de `api.md`), UI de busca com rolagem infinita.

**Critério de aceitação:** busca textual com filtros estruturados (membro, tipo de fonte, comunidade, espécie, região) funciona ponta a ponta, e toda resposta de item de busca carrega a atribuição de origem (`member_id`, `license`, `attribution`) obrigatória.

**Documentos-fonte:**
- [`docs/api.md`](api.md) — parâmetros, envelopes, exemplos e regras de negócio de cada endpoint público.
- [`docs/api/openapi.yaml`](api/openapi.yaml) — schemas e tipos normativos.
- [`docs/decisions/ADR-001-stack-e-framework.md`](decisions/ADR-001-stack-e-framework.md) — rolagem infinita resolvida via HTMX (`hx-trigger="revealed"`) sobre a API já paginada.

---

## Fase 4 — SKOS

**Entrega:** `ConceptHarvester`, `concepts`/`concept_mappings`, `SemanticExpander`, painel de curadoria de mapeamentos.

**Critério de aceitação:** uma busca por um termo usado só por um membro (ex.: "mandioca") devolve também um registro publicado por outro membro sob um termo diferente (ex.: "macaxeira"), com o item de resposta trazendo `matched_via` explicitando o mapeamento SKOS que ligou os dois.

**Documentos-fonte:**
- [`docs/busca-semantica.md`](busca-semantica.md) — pipeline de busca completo, CTE recursiva de expansão.
- [`docs/decisions/ADR-004-expansao-semantica-skos.md`](decisions/ADR-004-expansao-semantica-skos.md) — regras de transitividade por predicado.
- [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md) — fila de mapeamentos propostos, workflow de aprovação do Comitê.

---

## Fase 5 — Purge e auditoria

**Entrega:** `PurgeService` (`purge_by_member`), `AuditService`, painel do Comitê completo (fila de adesão, mapeamentos, histórico de harvest, `audit_log`, purge).

**Critério de aceitação:** disparar `purge` sobre um membro de teste remove seus registros e os mapeamentos SKOS que o envolvem; `audit_log` registra a ação (`member_purged` e um `mapping_removed_by_purge` por mapeamento); e o arquivo `.sqlite` reduz de tamanho após o `VACUUM` que fecha a operação.

**Documentos-fonte:**
- [`docs/modelo-de-dados.md`](modelo-de-dados.md#purge_by_membermember_id) — sequência transacional exata de `purge_by_member(member_id)`.
- [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md) — tela de purge com confirmação por digitação do nome do membro, leitura do `audit_log`.

---

## Fase 6 — Detecção de remoção

**Entrega:** remoção de registros ausentes em modo completo (`RecordIndexer`), desativação automática de `harvest_enabled` após falhas consecutivas.

**Critério de aceitação:** um registro removido do lado do membro desaparece do índice do Pluriverso depois que a próxima run **completa** (não incremental) daquele membro é executada com sucesso.

**Documentos-fonte:**
- [`docs/contrato-harvest.md`](contrato-harvest.md#2--modos-de-coleta-e-detecção-de-remoção) — §2, modos de coleta, regra de segurança da remoção, e §3, desativação por `HARVEST_MAX_FAILURES` consecutivas.

---

## Referências

- [`docs/arquitetura.md`](arquitetura.md) — C4 completo; nomes canônicos de componente citados nas entregas acima.
- [`docs/operacao.md`](operacao.md) — variáveis de ambiente, Docker, CI, backup, observabilidade que sustentam todas as fases em produção.
- [`README.md`](../README.md) — seção "Necessidades de Implementação (v3.3)", cada item apontando para a documentação que a fecha.
