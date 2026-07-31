# API do Pluriverso

Superfície HTTP completa do Pluriverso: busca federada, consulta de membros e conceitos, cadastro self-service de novos membros, e a superfície administrativa do Comitê Federado (fila de adesão, harvest manual, mapeamentos SKOS, purge, auditoria).

Este documento é a leitura humana da API — propósito de cada endpoint, exemplos e regras de negócio. **A fonte normativa de tipos, enums e schemas é [`docs/api/openapi.yaml`](api/openapi.yaml)** (OpenAPI 3.1); este arquivo não repete definição de tipo, apenas referencia o `operationId` correspondente. Onde os dois divergirem, `openapi.yaml` prevalece.

A base técnica de envelope de coleção, envelope de erro, operadores de filtro e códigos de status HTTP é herdada verbatim do ADR de referência da Arquitetura-BioCultural, conforme já decidido em [`docs/decisions/ADR-002-api-publica-rest.md`](decisions/ADR-002-api-publica-rest.md) (ADR-002 local) — este documento **não** repete essa decisão, só a aplica endpoint a endpoint.

A URL base de cada instância é configurável via `PUBLIC_BASE_URL` — não existe domínio fixo. É a materialização de [ADR-009/MI2 — Topologia Multi-Instância do Pluriverso](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md), que trata cada instância do Pluriverso como implantação independente, com seu próprio `member_id`-space e sua própria base URL. `openapi.yaml` usa a variável de servidor `{baseUrl}` pelo mesmo motivo.

## Sumário

- [Convenções](#convenções)
- [Autenticação](#autenticação)
- [§5.1 — Pública, sem autenticação](#51--pública-sem-autenticação)
- [§5.2 — Comitê Federado, autenticada](#52--comitê-federado-autenticada)
- [Regras de conflito e auditoria](#regras-de-conflito-e-auditoria)
- [Referências](#referências)

## Convenções

### Regra de envelope: coleção vs. recurso único

O ADR-002 local define um **envelope de coleção** (`data[]` + `pagination` + `links`). Ele se aplica **apenas** a endpoints que retornam uma lista. Endpoints que retornam um recurso único — consulta por id, criação, atualização, ou um agregado como `/api/v1/stats` — retornam o **objeto diretamente**, sem `pagination` nem `links`. O envelope de erro (`ErrorEnvelope`, também herdado verbatim do ADR-002 local) é usado em **toda** resposta de erro, independentemente do tipo de recurso. Essa distinção é uma decisão de implementação deste documento, não uma reabertura do ADR: o ADR-002 local nomeia o wrapper "envelope de **coleção**", o que já implica que recursos únicos não o usam.

### Paginação padrão

Salvo indicação em contrário no endpoint, toda listagem usa `page` (inteiro, default `1`) e `limit` (inteiro, default `20`, **máximo 100**) — o mesmo par usado por `GET /api/v1/search`. `limit` acima de 100 é rejeitado com `400 VALIDATION_ERROR`, não truncado silenciosamente.

### Operadores de filtro

Os operadores herdados do ADR-002 local (`?campo=valor`, `?campo[gt]=`, `?campo[lt]=`, `?campo[contains]=`, `?campo[in]=a,b`, `?search=`) valem em qualquer parâmetro de filtro estruturado desta API, salvo onde o parâmetro já é inerentemente um valor único de igualdade (ex.: `federated_id` na rota, `uri` na rota). Exemplos de uso: `member_type[in]=comunidade_tradicional,acervos_historicos`, `species[contains]=mandioca`, `member_updated_at[gt]=2026-01-01T00:00:00Z`.

### Regra de `error.code` para `409 Conflict`

A lista fechada de códigos (ADR-002 local) reserva `MAPPING_CONFLICT` ao recurso de mapeamentos semânticos (`/api/v1/mappings/{id}`) — é o único caso em que o nome do código corresponde 1:1 ao recurso em conflito. Todo **outro** `409` desta API (pedido de adesão já decidido, `url_base` duplicada, disparo de harvest concorrente) usa `VALIDATION_ERROR`, com `error.details[].field` identificando o campo/estado em conflito. Essa distinção está registrada aqui porque o ADR-002 local deixa ambos os códigos disponíveis "conforme o caso" (ver sua tabela de status HTTP) sem fixar qual código vai em qual endpoint — esta seção fixa isso, endpoint a endpoint, nas seções abaixo.

### `MEMBER_NOT_ACTIVE` e `PROBE_FAILED`

- **`MEMBER_NOT_ACTIVE`** — usado quando uma ação de federação é recusada porque `members.doc.harvest_enabled = false` para o `member_id` alvo (desativação automática por falhas consecutivas, [`contrato-harvest.md` §3](contrato-harvest.md), ou desativação manual pelo Comitê). Só se aplica a `POST /api/federation/members/{member_id}/harvest`: reativar exige `PATCH /api/federation/members/{member_id}` explícito primeiro — disparar harvest não reativa implicitamente um membro desativado.
- **`PROBE_FAILED`** — reservado ao caso em que o probe **não pôde sequer ser tentado** (ex.: `url_base` armazenado não é uma URL sintaticamente válida). Um probe que roda e **encontra** um problema de rede, DNS bloqueado ou schema incorreto **não** é erro HTTP — é resultado de negócio, registrado em `technical_check.ok = false` numa resposta `200 OK` (a decisão de adesão nunca é bloqueada pelo resultado do probe, [ADR-006/E3](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)). O algoritmo do probe em si (faixas de IP bloqueadas, resolução DNS, timeout) é especificado em [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md), não repetido aqui.

## Autenticação

- **§5.1 (pública):** nenhuma autenticação. Sujeita a rate limit por IP (`docs/governanca-e-seguranca.md`).
- **§5.2 (Comitê Federado):** HTTP Basic Auth, uma conta nomeada por integrante do Comitê. O esquema de autenticação (`securitySchemes.committeeBasic` em `openapi.yaml`: `type: http`, `scheme: basic`) está fixado em [`docs/decisions/ADR-003-autenticacao-do-comite.md`](decisions/ADR-003-autenticacao-do-comite.md) (ADR-003 local) — este documento só referencia o esquema pelo nome; a decisão de mecanismo, o formato de `COMMITTEE_USERS` e o custo de hash não são repetidos aqui.
- Credencial ausente ou inválida em qualquer endpoint de §5.2 → `401 UNAUTHORIZED`. Credencial válida sem permissão para a ação específica → `403 FORBIDDEN` (reservado para eventual segmentação futura de papéis dentro do Comitê; hoje toda conta do Comitê tem as mesmas permissões).

---

## §5.1 — Pública, sem autenticação

### `GET /api/v1/search`

`operationId: searchRecords`. Busca federada — o endpoint central da API pública, combinando filtros estruturados, busca textual (FTS5) e expansão semântica opcional via mapeamentos SKOS aprovados. O pipeline de resolução (normalização, expansão de conceitos, ranking) está especificado em [`docs/busca-semantica.md`](busca-semantica.md); este endpoint só documenta a superfície HTTP.

**Parâmetros de query:**

| Parâmetro | Tipo | Default | Observação |
|---|---|---|---|
| `q` | string | — | Consulta textual, casada contra `records_fts` (FTS5) e contra rótulos de conceito quando `expand=skos` |
| `expand` | enum `skos`\|`none` | `skos` | `skos` expande a busca por mapeamentos `approved` ([`busca-semantica.md`](busca-semantica.md)); `none` restringe a FTS direto |
| `member_id` | string | — | Igualdade; aceita `[in]=id1,id2` |
| `member_type` | enum (4 valores, ver abaixo) | — | Igualdade; aceita `[in]=` |
| `source_type` | enum `primary`\|`secondary` | — | Igualdade |
| `community` | string | — | Igualdade ou `[contains]=` |
| `species` | string | — | Igualdade ou `[contains]=` (casa contra `profile.scientific_name`) |
| `region` | string | — | Igualdade ou `[contains]=` |
| `country` | string | — | Igualdade (ISO ou nome livre, conforme declarado pelo membro) |
| `state` | string | — | Igualdade |
| `use_category` | string | — | Igualdade; um registro pode ter várias categorias (`record_terms`), portanto o filtro casa se **qualquer** uma bater |
| `updated_since` | string (ISO-8601) | — | Filtra por `member_updated_at >=` |
| `page` | integer | `1` | — |
| `limit` | integer | `20` | **Máximo 100** |
| `sort` | enum `-relevance`\|`-member_updated_at`\|`scientific_name` | `-relevance` | `-relevance` exige `bm25`/score de expansão; nos demais casos ordena por coluna |

`member_type` usa os quatro literais fixados em [ADR-006/E1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md): `fontes_secundarias`, `comunidade_tradicional`, `acervos_historicos`, `obras_naturalistas` — em português, verbatim, sem tradução.

**Corpo de requisição:** nenhum. **Autenticação:** nenhuma.

**Resposta de sucesso — `200 OK`** (envelope de coleção; cada item de `data[]` segue o schema `RecordItem` de `openapi.yaml`):

```json
{
  "data": [
    {
      "federated_id": "0192f.../record_001",
      "member_id": "0192f...",
      "member_name": "Iniciativa USEFLORA",
      "member_type": "fontes_secundarias",
      "member_url_base": "https://useflora.example.org",
      "license": "CC BY-NC-SA 4.0",
      "attribution": "Comunidade Baniwa do Rio Içana, 2024",
      "member_updated_at": "2026-06-01T00:00:00Z",
      "harvested_at": "2026-06-02T03:00:12Z",
      "profile": { "scientific_name": "Manihot esculenta", "vernacular_names": ["mandioca"] },
      "data": { "species": { "scientificName": "Manihot esculenta" } },
      "matched_via": [
        { "concept_uri": "https://termos.baniwa.example.org/concept/0192a...", "predicate": "skos:exactMatch", "member_id": "0191c..." }
      ]
    }
  ],
  "pagination": { "page": 1, "limit": 20, "totalPages": 3, "totalItems": 47, "hasNext": true, "hasPrev": false },
  "links": { "self": "/api/v1/search?q=mandioca&page=1", "first": "/api/v1/search?q=mandioca&page=1", "prev": null, "next": "/api/v1/search?q=mandioca&page=2", "last": "/api/v1/search?q=mandioca&page=3" }
}
```

Quando o registro entrou no resultado por FTS direto (não por expansão de mapeamento), `matched_via` **está ausente** do item — não é `[]`, é omitido. `license`/`attribution` ausentes no membro nunca são preenchidos com um default inventado; ver a seção "Atribuição de origem" abaixo.

**Erros:**

- `400 VALIDATION_ERROR` — `limit` acima de 100, ou `updated_since` fora do formato ISO-8601.
  ```json
  { "error": { "code": "VALIDATION_ERROR", "message": "Falha na validação dos dados", "details": [{ "field": "limit", "message": "Máximo permitido é 100", "code": "VALIDATION_ERROR" }] }, "meta": { "requestId": "req_a1b2c3", "timestamp": "2026-07-31T12:00:00Z" } }
  ```
- `422 INVALID_ENUM` — `member_type`, `source_type`, `expand` ou `sort` fora da lista fechada.
  ```json
  { "error": { "code": "INVALID_ENUM", "message": "Falha na validação dos dados", "details": [{ "field": "member_type", "message": "Valor inválido", "code": "INVALID_ENUM" }] }, "meta": { "requestId": "req_d4e5f6", "timestamp": "2026-07-31T12:00:01Z" } }
  ```
- `429 RATE_LIMIT_EXCEEDED` — acima de `RATE_LIMIT_PUBLIC_PER_MIN` (default `60`) requisições/minuto por IP.

**Atribuição de origem.** `license`/`attribution` ausentes no membro (campo `permissions.license`/`permissions.attribution` não preenchido em `data`, [`contrato-harvest.md` §4](contrato-harvest.md)) resultam em `"license": null, "attribution": null` **explícitos**, mais `"license_notice": "não declarada pelo membro; consultar {member_url_base}"` no item. Nunca um valor inventado — é o Pluriverso agindo como "tradutor, não ditador taxonômico" também em matéria de proveniência.

### `GET /api/v1/records/{federated_id}`

`operationId: getRecord`. Consulta um registro único pelo identificador federado. `federated_id` é o par `{member_id}/{record_id}` fixado em [ADR-004/D6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md) — como contém `/`, o segmento de path deve ser percent-encoded (`0192f...%2Frecord_001`).

**Corpo de requisição:** nenhum. **Autenticação:** nenhuma.

**Resposta de sucesso — `200 OK`** (objeto `RecordItem` direto, sem envelope de coleção):

```json
{
  "federated_id": "0192f.../record_001",
  "member_id": "0192f...",
  "member_name": "Iniciativa USEFLORA",
  "member_type": "fontes_secundarias",
  "member_url_base": "https://useflora.example.org",
  "license": "CC BY-NC-SA 4.0",
  "attribution": "Comunidade Baniwa do Rio Içana, 2024",
  "member_updated_at": "2026-06-01T00:00:00Z",
  "harvested_at": "2026-06-02T03:00:12Z",
  "profile": { "scientific_name": "Manihot esculenta" },
  "data": { "species": { "scientificName": "Manihot esculenta" } }
}
```

**Erros:**

- `404 NOT_FOUND` — nenhum registro com esse `federated_id` no índice.
  ```json
  { "error": { "code": "NOT_FOUND", "message": "Registro não encontrado", "details": [] }, "meta": { "requestId": "req_g7h8i9", "timestamp": "2026-07-31T12:00:02Z" } }
  ```

### `GET /api/v1/members`

`operationId: listMembers`. Lista os membros **ativos** da federação (a única existência que a tabela `members` registra — não há membro "inativo" listado aqui; ver [ADR-006/E2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md) para o ciclo de vida completo do pedido de adesão).

**Parâmetros de query:** `page`, `limit` (padrão geral); `member_type` (igualdade ou `[in]=`); `sort` (enum `member_name`\|`-joined_at`\|`-record_count`, default `member_name`).

**Corpo de requisição:** nenhum. **Autenticação:** nenhuma.

**Resposta de sucesso — `200 OK`** (envelope de coleção; itens no schema `MemberSummary`):

```json
{
  "data": [
    { "member_id": "0192f...", "member_name": "Iniciativa USEFLORA", "member_type": "fontes_secundarias", "url_base": "https://useflora.example.org", "joined_at": "2026-03-10T00:00:00Z", "record_count": 1523 }
  ],
  "pagination": { "page": 1, "limit": 20, "totalPages": 1, "totalItems": 12, "hasNext": false, "hasPrev": false },
  "links": { "self": "/api/v1/members?page=1", "first": "/api/v1/members?page=1", "prev": null, "next": null, "last": "/api/v1/members?page=1" }
}
```

**Erros:** `422 INVALID_ENUM` se `member_type` ou `sort` fora da lista fechada; `400 VALIDATION_ERROR` se `limit` > 100.

### `GET /api/v1/concepts/{uri}/mappings`

`operationId: getConceptMappings`. Lista os mapeamentos semânticos com `status: approved` de um conceito — os únicos que a `SemanticExpander` usa em busca ([`busca-semantica.md`](busca-semantica.md)) e os únicos que fazem sentido expor publicamente (mapeamentos `proposed`/`rejected` são material de curadoria interna do Comitê, expostos só em `GET /api/v1/mappings`, §5.2). `uri` é a URI absoluta do conceito, percent-encoded no path (ex.: `https%3A%2F%2Ftermos.baniwa.example.org%2Fconcept%2F0192a...`).

**Parâmetros de query:** `page`, `limit` (padrão geral). Não aceita filtro de `status` — sempre `approved`.

**Corpo de requisição:** nenhum. **Autenticação:** nenhuma.

**Resposta de sucesso — `200 OK`** (envelope de coleção; itens no schema `ConceptMapping`):

```json
{
  "data": [
    {
      "id": "0193a...",
      "source_uri": "https://termos.baniwa.example.org/concept/0192a...",
      "source_member_id": "0191c...",
      "target_uri": "https://termos.useflora.example.org/concept/0088b...",
      "target_member_id": "0192f...",
      "predicate": "skos:exactMatch",
      "status": "approved",
      "proposed_by": "comite-baniwa",
      "proposed_at": "2026-05-01T00:00:00Z",
      "decided_by": "comite-useflora",
      "decided_at": "2026-05-03T00:00:00Z",
      "note": "Confirmado: mesmo taxon, grafias regionais distintas"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "totalPages": 1, "totalItems": 1, "hasNext": false, "hasPrev": false },
  "links": { "self": "...", "first": "...", "prev": null, "next": null, "last": "..." }
}
```

Se o conceito existir mas não tiver nenhum mapeamento `approved`, a resposta é `200 OK` com `data: []` — ausência de mapeamento não é erro.

**Erros:**

- `404 NOT_FOUND` — `uri` não corresponde a nenhum conceito conhecido em `concepts`.

### `GET /api/v1/stats`

`operationId: getStats`. Totais agregados do índice — membro, tipo de fonte, país. Usado pela UI para os painéis de visão geral. Não é uma coleção paginada (é um único objeto agregado); não aceita `page`/`limit`.

**Corpo de requisição:** nenhum. **Autenticação:** nenhuma.

**Resposta de sucesso — `200 OK`** (objeto direto, sem envelope de coleção):

```json
{
  "total_records": 15230,
  "total_members": 12,
  "generated_at": "2026-07-31T12:00:00Z",
  "by_member": [{ "member_id": "0192f...", "member_name": "Iniciativa USEFLORA", "record_count": 1523 }],
  "by_source_type": [{ "source_type": "secondary", "record_count": 9000 }, { "source_type": "primary", "record_count": 6230 }],
  "by_country": [{ "country": "BR", "record_count": 14210 }, { "country": "PE", "record_count": 1020 }]
}
```

`by_member`/`by_source_type`/`by_country` só contam registros cujo campo de origem (`profile.source_type`, `profile.country`) está preenchido — best-effort, igual à extração do perfil ([`contrato-harvest.md` §4](contrato-harvest.md)); registros sem o campo não aparecem no agrupamento correspondente, mas contam em `total_records`.

**Erros:** `500 INTERNAL_ERROR` — nenhum erro de validação de entrada é possível (não há parâmetros).

### `POST /api/federation/membership-requests`

`operationId: createMembershipRequest`. Cadastro self-service de um novo membro, conforme [ADR-006/E1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md). Não leva `/v1` — é contrato de protocolo de federação, não API pública versionada (ver [ADR-002 local](decisions/ADR-002-api-publica-rest.md) §Versionamento).

**Corpo de requisição** (campos de [ADR-006/E1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md), todos obrigatórios):

```json
{
  "member_name": "Comunidade Baniwa do Rio Içana",
  "member_type": "comunidade_tradicional",
  "url_base": "https://termos.baniwa.example.org",
  "contact_email": "contato@baniwa.example.org",
  "care_declaration": "Declaramos adesão aos princípios C.A.R.E. de governança de dados indígenas..."
}
```

**Autenticação:** nenhuma. Sujeito a rate limit dedicado (`RATE_LIMIT_MEMBERSHIP_PER_HOUR`, default `5`/hora por IP — [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md)).

Ao receber o pedido, o `ProbeService` executa o probe técnico de [ADR-006/E3](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md) **de forma síncrona**, contra `{url_base}/api/federation/records?page=1&size=1`, e grava o resultado em `technical_check` **antes** de responder — mas o resultado, seja qual for, **nunca** impede a criação do pedido (`status` sempre nasce `pending`).

**Resposta de sucesso — `201 Created`** (objeto `MembershipRequest` direto):

```json
{
  "id": "req_0192f...",
  "member_name": "Comunidade Baniwa do Rio Içana",
  "member_type": "comunidade_tradicional",
  "url_base": "https://termos.baniwa.example.org",
  "contact_email": "contato@baniwa.example.org",
  "care_declaration": "Declaramos adesão aos princípios C.A.R.E. de governança de dados indígenas...",
  "status": "pending",
  "technical_check": { "ok": false, "checked_at": "2026-07-31T12:00:00Z", "checks": { "https": true, "dns_safe": true, "http_200": false }, "failure_reason": "connection_refused", "http_status": null, "elapsed_ms": 5002 },
  "member_id": null,
  "created_at": "2026-07-31T12:00:00Z",
  "decided_at": null,
  "decided_by": null,
  "rejection_reason": null
}
```

**Erros:**

- `400 VALIDATION_ERROR` / `REQUIRED_FIELD` — campo obrigatório ausente.
  ```json
  { "error": { "code": "REQUIRED_FIELD", "message": "Falha na validação dos dados", "details": [{ "field": "care_declaration", "message": "Campo obrigatório", "code": "REQUIRED_FIELD" }] }, "meta": { "requestId": "req_j1k2l3", "timestamp": "2026-07-31T12:00:03Z" } }
  ```
- `422 INVALID_ENUM` — `member_type` fora dos quatro literais fechados.
- `429 RATE_LIMIT_EXCEEDED` — acima do limite por IP/hora.

### `GET /health`

`operationId: getHealth`. Sem prefixo `/api` — endpoint de infraestrutura, consumido por orquestrador/`HEALTHCHECK` do Docker, não por cliente de API. Sem autenticação.

**Resposta de sucesso — `200 OK`:**

```json
{
  "status": "ok",
  "version": "1.4.0",
  "instance_name": "pluriverso-etnobotanica-br",
  "db": "ok",
  "last_harvest_at": {
    "0192f...": "2026-07-31T03:00:12Z",
    "0191c...": "2026-07-30T03:01:05Z"
  }
}
```

`status` ∈ `ok`\|`degraded` — `degraded` quando o banco responde mas há membros com `harvest_enabled=false` por falhas consecutivas, ou o `HarvestScheduler` acumula runs `failed`. `db` ∈ `ok`\|`unreachable`. `last_harvest_at` é um mapa `member_id → timestamp ISO-8601` da última run **bem-sucedida** (`status: ok` ou `partial`) por membro; membro nunca coletado não aparece na chave.

**Erros:** se o banco SQLite estiver inacessível (arquivo bloqueado, disco cheio), `503 Service Unavailable` com `db: "unreachable"` no corpo — não um `ErrorEnvelope`, para que orquestradores simples de `HEALTHCHECK` leiam o status HTTP sem parsear JSON de erro.

---

## §5.2 — Comitê Federado, autenticada

Toda ação de escrita desta seção grava uma entrada em `audit_log` com `actor` = usuário autenticado (o `username` do Basic Auth), **na mesma transação SQLite** da mudança de estado — nunca em duas escritas separadas, para não haver estado possível em que a mudança existe sem o registro de auditoria correspondente. Ver [`docs/modelo-de-dados.md`](modelo-de-dados.md) para o esquema de `audit_log` e a lista fechada de `action`.

### `GET /api/federation/membership-requests?status=`

`operationId: listMembershipRequests`. Fila de pedidos de adesão ([ADR-006/E2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)).

**Parâmetros de query:** `status` (enum `pending`\|`active`\|`rejected`, opcional — omitido lista todos); `page`, `limit` (padrão geral); `sort` (default `-created_at`).

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (envelope de coleção; itens no schema `MembershipRequest`, mesmo shape do exemplo de `createMembershipRequest` acima).

**Erros:** `401 UNAUTHORIZED` (sem credencial); `422 INVALID_ENUM` (`status` fora da lista fechada).

### `PATCH /api/federation/membership-requests/{id}`

`operationId: decideMembershipRequest`. Decisão humana sobre um pedido ([ADR-006/E2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md) — só decisão humana move o estado; o resultado do probe nunca decide sozinho).

**Corpo de requisição:**

```json
{ "status": "active", "rejection_reason": null }
```

`rejection_reason` é **obrigatório** quando `status: "rejected"` (senão `400 REQUIRED_FIELD`). Ao decidir `status: "active"`: gera `member_id` (UUIDv7, nunca reciclado — [ADR-006/E5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)), cria a linha em `members`, e agenda a primeira coleta (sempre modo `full`, [`contrato-harvest.md` §2](contrato-harvest.md)).

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (objeto `MembershipRequest` atualizado, com `member_id`, `decided_at`, `decided_by` preenchidos):

```json
{
  "id": "req_0192f...",
  "member_name": "Comunidade Baniwa do Rio Içana",
  "member_type": "comunidade_tradicional",
  "url_base": "https://termos.baniwa.example.org",
  "contact_email": "contato@baniwa.example.org",
  "care_declaration": "Declaramos adesão...",
  "status": "active",
  "technical_check": { "ok": false, "checked_at": "2026-07-31T12:00:00Z", "checks": {}, "failure_reason": "connection_refused", "http_status": null, "elapsed_ms": 5002 },
  "member_id": "0193c...",
  "created_at": "2026-07-31T12:00:00Z",
  "decided_at": "2026-08-01T09:15:00Z",
  "decided_by": "comite-alice",
  "rejection_reason": null
}
```

**Erros:**

- `404 NOT_FOUND` — `id` inexistente.
- `409 VALIDATION_ERROR` — pedido já decidido (`status` != `pending`); `error.details[].field = "status"`.
  ```json
  { "error": { "code": "VALIDATION_ERROR", "message": "Pedido já decidido", "details": [{ "field": "status", "message": "Pedido já está com status 'active'", "code": "VALIDATION_ERROR" }] }, "meta": { "requestId": "req_m4n5o6", "timestamp": "2026-08-01T09:14:00Z" } }
  ```
- `409 VALIDATION_ERROR` — aprovar (`status: "active"`) quando `url_base` já pertence a um membro `active` existente; `error.details[].field = "url_base"`.
- `400 REQUIRED_FIELD` — `rejection_reason` ausente ao rejeitar.
- `422 INVALID_ENUM` — `status` fora de `active`\|`rejected`.

### `POST /api/federation/membership-requests/{id}/probe`

`operationId: reprobeMembershipRequest`. Re-executa o probe técnico contra o `url_base` do pedido ([ADR-006/E3](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)) — útil quando o membro corrige a implementação depois de um `technical_check` falho. O algoritmo (validação anti-SSRF, timeouts, faixas de IP bloqueadas) está especificado em [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md); este endpoint só documenta o contrato HTTP.

**Corpo de requisição:** nenhum. **Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (objeto `MembershipRequest` atualizado, com novo `technical_check` — a resposta é `200` **mesmo quando `technical_check.ok = false`**, porque o probe foi executado com sucesso; o resultado do probe é dado, não erro).

**Erros:**

- `404 NOT_FOUND` — `id` inexistente.
- `409 VALIDATION_ERROR` — pedido não está `pending` (reprobe só faz sentido antes da decisão).
- `400 PROBE_FAILED` — `url_base` armazenado não é sintaticamente uma URL válida, portanto o probe não pôde nem ser tentado (caso raro — o cadastro já valida formato na entrada).

### `POST /api/federation/members/{member_id}/purge`

`operationId: purgeMember`. Remoção imediata e completa de um membro, conforme [ADR-004/D4](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md) — registros, `record_terms` em cascata, `concept_mappings` em qualquer direção, `concepts`, `harvest_runs`, e a linha de `members`, numa única transação. A sequência exata de exclusão está especificada em [`docs/modelo-de-dados.md`](modelo-de-dados.md) §4.2 (`purge_by_member`); este endpoint só documenta o contrato HTTP.

**Corpo de requisição:**

```json
{ "confirm_member_name": "Comunidade Baniwa do Rio Içana" }
```

`confirm_member_name` deve corresponder **exatamente** (case-sensitive, sem trim adicional) ao `member_name` gravado — é o gesto de confirmação por digitação exigido para uma operação irreversível.

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (resumo da purga, não um recurso persistente):

```json
{
  "member_id": "0192f...",
  "member_name": "Comunidade Baniwa do Rio Içana",
  "purged_at": "2026-08-01T10:00:00Z",
  "counts": { "records": 1523, "concept_mappings": 8, "concepts": 214, "harvest_runs": 47 }
}
```

**Erros:**

- `404 NOT_FOUND` — `member_id` não existe em `members` (nunca existiu, ou já foi purgado antes).
- `400 VALIDATION_ERROR` — `confirm_member_name` não corresponde ao `member_name` real.
  ```json
  { "error": { "code": "VALIDATION_ERROR", "message": "Falha na validação dos dados", "details": [{ "field": "confirm_member_name", "message": "Não corresponde ao nome do membro", "code": "VALIDATION_ERROR" }] }, "meta": { "requestId": "req_p7q8r9", "timestamp": "2026-08-01T10:00:00Z" } }
  ```

### `PATCH /api/federation/members/{member_id}`

`operationId: updateMember`. Atualização administrativa de um membro ativo — reativar/desativar harvest, ou sobrescrever o cron por membro ([`contrato-harvest.md` §2](contrato-harvest.md)).

**Corpo de requisição** (todos os campos opcionais, ao menos um obrigatório):

```json
{ "harvest_enabled": true, "harvest_cron_incremental": "0 5 * * *", "harvest_cron_full": "0 6 * * 0" }
```

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (objeto `Member` atualizado — extensão de `MemberSummary` com os campos administrativos):

```json
{ "member_id": "0192f...", "member_name": "Iniciativa USEFLORA", "member_type": "fontes_secundarias", "url_base": "https://useflora.example.org", "joined_at": "2026-03-10T00:00:00Z", "record_count": 1523, "harvest_enabled": true, "harvest_cron_incremental": "0 5 * * *", "harvest_cron_full": "0 6 * * 0", "consecutive_failures": 0 }
```

**Erros:**

- `404 NOT_FOUND` — `member_id` inexistente.
- `400 VALIDATION_ERROR` — corpo vazio, ou expressão cron malformada.

### `POST /api/federation/members/{member_id}/harvest`

`operationId: triggerHarvest`. Dispara uma run de harvest manual, fora do agendamento cron — usado para forçar recoleta imediata após um membro corrigir um bug, por exemplo.

**Corpo de requisição:**

```json
{ "mode": "incremental" }
```

`mode` ∈ `incremental`\|`full`, conforme [`contrato-harvest.md` §2](contrato-harvest.md). **Autenticação:** `committeeBasic`.

**Resposta de sucesso — `201 Created`** (objeto `HarvestRun` recém-criado, `status: "running"`):

```json
{ "id": "0194a...", "member_id": "0192f...", "mode": "incremental", "status": "running", "started_at": "2026-08-01T11:00:00Z", "finished_at": null, "pages_fetched": 0, "upserted": 0, "removed": 0, "rejected": 0, "http_status_last": null, "error_message": null }
```

**Erros:**

- `404 NOT_FOUND` — `member_id` inexistente.
- `409 MEMBER_NOT_ACTIVE` — `harvest_enabled = false` para este membro (desativado por falhas consecutivas ou pelo Comitê); reative primeiro via `PATCH /api/federation/members/{member_id}`.
- `409 VALIDATION_ERROR` — já existe uma run em andamento (`status: "running"`) para este `member_id` — no máximo 1 run simultânea por membro ([`contrato-harvest.md` §3](contrato-harvest.md)); o disparo é **rejeitado**, não enfileirado.
- `422 INVALID_ENUM` — `mode` fora de `incremental`\|`full`.

### `GET /api/federation/harvest-runs?member_id=&status=`

`operationId: listHarvestRuns`. Histórico de runs de harvest, para auditoria e diagnóstico.

**Parâmetros de query:** `member_id` (igualdade, opcional); `status` (enum `running`\|`ok`\|`partial`\|`failed`, opcional); `page`, `limit` (padrão geral); `sort` (default `-started_at`).

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (envelope de coleção; itens no schema `HarvestRun`, mesmo shape do exemplo de `triggerHarvest` acima, com `finished_at`/contagens preenchidos para runs concluídas).

**Erros:** `422 INVALID_ENUM` (`status` fora da lista fechada).

### `GET /api/v1/mappings` e `POST /api/v1/mappings`

`operationId: listMappings` (GET) / `proposeMapping` (POST). Apesar do prefixo `/api/v1/` (é API pública **versionada** na forma, mas restrita ao Comitê no acesso — o mapeamento é dado de curadoria interna até aprovado, quando passa a ser exposto publicamente via `GET /api/v1/concepts/{uri}/mappings`).

**`GET` — parâmetros de query:** `status` (enum `proposed`\|`approved`\|`rejected`, opcional); `source_member_id`, `target_member_id`, `predicate` (enum, ver abaixo); `page`, `limit` (padrão geral); `sort` (default `-proposed_at`).

**`GET` — resposta de sucesso `200 OK`:** envelope de coleção de `ConceptMapping`, incluindo mapeamentos em qualquer `status` (diferente do endpoint público §5.1, que só mostra `approved`).

**`POST` — corpo de requisição:**

```json
{ "source_uri": "https://termos.baniwa.example.org/concept/0192a...", "target_uri": "https://termos.useflora.example.org/concept/0088b...", "predicate": "skos:closeMatch", "note": "Sugerido por similaridade de rótulo" }
```

`predicate` ∈ `skos:exactMatch`\|`skos:closeMatch`\|`skos:broadMatch`\|`skos:narrowMatch` ([ADR-008/DB5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-008-pluriverso-database-engine.md)). O servidor resolve `source_member_id`/`target_member_id` a partir do `member_id` dono de cada `uri` em `concepts`; `proposed_by` é preenchido com o usuário autenticado; `status` nasce sempre `proposed` — nunca `approved` diretamente, mesmo quando a proposta vem de sugestão automática por similaridade ([ADR-004 "Mitigações"](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md)); só decisão humana em `PATCH .../mappings/{id}` aprova.

**`POST` — resposta de sucesso `201 Created`** (objeto `ConceptMapping`, `status: "proposed"`).

**Autenticação (ambos os métodos):** `committeeBasic`.

**Erros (`POST`):**

- `404 NOT_FOUND` — `source_uri` ou `target_uri` não corresponde a nenhum conceito conhecido.
- `400 VALIDATION_ERROR` — `source_member_id == target_member_id` (mapeamento é sempre **entre** membros distintos, ver `docs/modelo-de-dados.md` §invariantes de `concept_mappings`).
- `422 INVALID_ENUM` — `predicate` fora dos quatro literais SKOS.

**Erros (`GET`):** `422 INVALID_ENUM` (`status`/`predicate` fora da lista fechada).

### `PATCH /api/v1/mappings/{id}`

`operationId: decideMapping`. Decisão do Comitê sobre uma proposta de mapeamento.

**Corpo de requisição:**

```json
{ "status": "approved", "note": "Confirmado por curador do membro alvo" }
```

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (objeto `ConceptMapping` atualizado, `decided_by`/`decided_at` preenchidos).

**Erros:**

- `404 NOT_FOUND` — `id` inexistente.
- `409 MAPPING_CONFLICT` — mapeamento já decidido (`status` != `proposed`) — **este é o único endpoint que usa `MAPPING_CONFLICT`**, por corresponder ao próprio recurso do nome do código.
  ```json
  { "error": { "code": "MAPPING_CONFLICT", "message": "Mapeamento já decidido", "details": [{ "field": "status", "message": "Mapeamento já está com status 'approved'", "code": "MAPPING_CONFLICT" }] }, "meta": { "requestId": "req_s1t2u3", "timestamp": "2026-08-01T13:00:00Z" } }
  ```
- `422 INVALID_ENUM` — `status` fora de `approved`\|`rejected`.

### `POST /api/v1/concepts`

`operationId: createConcept`. Cadastro manual de conceito — o caminho alternativo ao harvest automático de `/api/federation/concepts` ([`contrato-harvest.md` §5](contrato-harvest.md)), usado quando o membro não implementa (ainda) o endpoint de conceitos, hoje não-normativo.

**Corpo de requisição:**

```json
{
  "uri": "https://termos.baniwa.example.org/concept/0192a...",
  "member_id": "0191c...",
  "scheme_uri": "https://termos.baniwa.example.org/scheme/etnobotanica",
  "pref_labels": [{ "value": "mandioca", "language": "pt-BR" }],
  "alt_labels": [{ "value": "macaxeira", "language": "pt-BR" }],
  "broader": [],
  "narrower": []
}
```

O servidor grava `origin: "manual"` (distinto de `origin: "harvest"`, ver `docs/modelo-de-dados.md`); `updated_at` é gerado no momento do cadastro.

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `201 Created`** (objeto `Concept`, `origin: "manual"`).

**Erros:**

- `404 NOT_FOUND` — `member_id` referenciado não existe em `members`.
- `400 REQUIRED_FIELD` — `uri`, `member_id` ou `pref_labels` ausente.
- `409 VALIDATION_ERROR` — `uri` já cadastrado (violação da restrição de unicidade em `concepts.uri`, `docs/modelo-de-dados.md`).

### `GET /api/federation/audit-log?action=&target_id=`

`operationId: listAuditLog`. Consulta somente-leitura do log de governança — append-only, sem endpoint de escrita direta (toda entrada é gerada como efeito colateral das ações desta seção, ver "Regras de conflito e auditoria" abaixo).

**Parâmetros de query:** `action` (enum fechado — `membership_approved`, `membership_rejected`, `member_purged`, `harvest_disabled`, `harvest_reenabled`, `mapping_approved`, `mapping_rejected`, `mapping_removed_by_purge`, `concept_registered`; ver `docs/modelo-de-dados.md`); `target_id` (igualdade); `page`, `limit` (padrão geral); `sort` (default `-at`).

**Autenticação:** `committeeBasic`.

**Resposta de sucesso — `200 OK`** (envelope de coleção; itens no schema `AuditEntry`):

```json
{
  "data": [
    { "id": "0195b...", "action": "member_purged", "actor": "comite-alice", "target_type": "member", "target_id": "0192f...", "reason": null, "before": { "records": 1523, "concept_mappings": 8 }, "after": null, "at": "2026-08-01T10:00:00Z" }
  ],
  "pagination": { "page": 1, "limit": 20, "totalPages": 1, "totalItems": 1, "hasNext": false, "hasPrev": false },
  "links": { "self": "...", "first": "...", "prev": null, "next": null, "last": "..." }
}
```

`audit_log` nunca guarda conteúdo de registro do membro — só contagens e identificadores (`before`/`after`), suficiente para auditabilidade sem reter dado do membro que saiu.

**Erros:** `422 INVALID_ENUM` (`action` fora da lista fechada).

---

## Regras de conflito e auditoria

Resumo transversal a toda §5.2, consolidado aqui para não repetir em cada endpoint:

| Situação | Status | `error.code` |
|---|---|---|
| `PATCH` numa `membership-request` já decidida | `409` | `VALIDATION_ERROR` |
| Aprovar `membership-request` com `url_base` já ativo em outro membro | `409` | `VALIDATION_ERROR` |
| `PATCH` num `mapping` já decidido | `409` | `MAPPING_CONFLICT` |
| `purge` com `confirm_member_name` divergente | `400` | `VALIDATION_ERROR` |
| Disparo de harvest manual com run já em andamento para o membro | `409` | `VALIDATION_ERROR` |
| Disparo de harvest manual com `harvest_enabled=false` | `409` | `MEMBER_NOT_ACTIVE` |

Toda linha acima corresponde a uma escrita rejeitada **antes** de qualquer alteração de estado — nenhuma delas produz um `audit_log` parcial. Toda escrita bem-sucedida desta seção (aprovação/rejeição de pedido, purge, ativação/desativação de harvest, decisão de mapeamento, cadastro manual de conceito) grava exatamente uma entrada em `audit_log` (ou mais de uma, no caso de `purge`, que gera uma entrada `member_purged` mais uma `mapping_removed_by_purge` por mapeamento removido — `docs/modelo-de-dados.md` §4.2), sempre na mesma transação SQLite da mudança.

## Referências

- [ADR-002 local — API Pública REST](decisions/ADR-002-api-publica-rest.md) — envelope, erros, filtros, status HTTP, versionamento (fonte deste documento para tudo que não é específico de endpoint)
- [ADR-003 local — Autenticação do Comitê](decisions/ADR-003-autenticacao-do-comite.md) — esquema `committeeBasic`
- [ADR-004 — Arquitetura Federada, D4 (purge) e D6 (contrato de registros)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md)
- [ADR-006 — Protocolo de Adesão à Federação, E1–E5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)
- [ADR-008 — Motor de Banco de Dados do Pluriverso, DB5 (predicados SKOS)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-008-pluriverso-database-engine.md)
- [ADR-009 — Topologia Multi-Instância do Pluriverso, MI2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md)
- [`docs/contrato-harvest.md`](contrato-harvest.md) — comportamento do coletor, perfil mínimo, contrato de conceitos
- [`docs/modelo-de-dados.md`](modelo-de-dados.md) — DDL, invariantes, sequência de `purge_by_member`
- [`docs/busca-semantica.md`](busca-semantica.md) — pipeline de busca e regras de expansão SKOS
- [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md) — algoritmo do probe, rate limit, headers de segurança
- [`docs/api/openapi.yaml`](api/openapi.yaml) — especificação normativa de tipos e schemas
