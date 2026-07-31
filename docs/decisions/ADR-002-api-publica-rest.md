# ADR-002: API Pública REST

## Status
Aceito

## Contexto

O Pluriverso expõe uma API pública para busca federada, consulta de membros, consulta de mapeamentos semânticos e estatísticas, além de um canal de cadastro self-service de novos membros (`POST /api/federation/membership-requests`, [ADR-006/E1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)) e um canal de coleta periódica dos membros ativos (`GET /api/federation/records`, [ADR-004/D6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md)).

A Arquitetura-BioCultural já discutiu padrões de API em [ADR-002 "Padrões de API e Integração"](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-002-api-standards.md), cujo status é **`Proposto`** — não `Aceito`. Este documento local trata esse ADR como **referência**, não como norma vinculante: nada nele obriga o Pluriverso. O ADR-002-arquitetura prescreve uma abordagem híbrida (REST para aquisição/curadoria, **GraphQL** para o contexto de apresentação/API pública), autenticação **JWT + refresh token**, **API Keys com tiers**, e base URL fixa `https://api.etnoknowledge.org/v1` — decisões pensadas para a topologia centralizada pré-federação (API Gateway, múltiplos serviços), incompatíveis com o container único e SQLite embutida do Pluriverso ([ADR-008/DB1-DB6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-008-pluriverso-database-engine.md)) e com a exigência de simplicidade e imagem Docker mínima de `principios.md`.

Este ADR decide o que o Pluriverso **aproveita** desse ADR de referência e o que **rejeita**, para que a API pública tenha uma base técnica única e documentada, sem reabrir a discussão a cada endpoint novo.

## Requisitos

### Funcionais
- Suportar busca federada com filtros estruturados (membro, tipo de fonte, comunidade, espécie, região) e busca textual (`search`).
- Suportar paginação previsível em toda resposta de coleção.
- Retornar erros em formato consistente, com código de erro máquina-legível e detalhamento por campo.
- Distinguir claramente, na estrutura de caminho, endpoints de API pública versionada dos endpoints de protocolo de federação (contratos fixados por outros ADRs, não versionáveis livremente).

### Não-Funcionais
- Superfície mínima de manutenção: um único estilo de API (REST), sem camada de resolvers nem schema paralelo.
- Compatibilidade com container único, sem gateway, sem serviço de autenticação externo, sem cache distribuído.
- Retrocompatibilidade: mudança de contrato de uma versão não pode quebrar clientes de uma versão anterior sem aviso.

## Opções Consideradas

### Opção 1: REST-only, herdando apenas envelope/erro/filtros/status HTTP do ADR-002-arquitetura
**Prós:**
- Um único estilo de API para todo o Pluriverso — mesma stack Express/EJS já decidida em ADR-001 local, sem dependência nova.
- Envelope de paginação, envelope de erro, operadores de filtro e tabela de status HTTP do ADR-002-arquitetura já são adequados a um caso de busca com filtros fixos; não há motivo para reinventar esse vocabulário.
- Endpoints de federação (`/api/federation/...`) ficam naturalmente fora do versionamento de caminho, preservando os contratos já fixados por outros ADRs aceitos.

**Contras:**
- Sem GraphQL, clientes que precisem de composição arbitrária de campos relacionados fazem múltiplas chamadas REST em vez de uma consulta única.

### Opção 2: Híbrida REST + GraphQL, replicando o ADR-002-arquitetura
**Prós:**
- Clientes avançados podem compor exatamente os campos que precisam numa única requisição.

**Contras:**
- Dobra a superfície de manutenção (dois estilos de API, dois conjuntos de testes, duas formas de documentar contrato).
- Exige camada de resolvers e introspection de schema para um caso de uso que é, na prática, busca com filtros fixos — complexidade desproporcional ao problema.
- Incompatível com o Docker mínimo exigido por `principios.md`: aumenta dependências e superfície de imagem sem necessidade comprovada.

### Opção 3: REST com autenticação JWT + API Keys com tiers, como no ADR-002-arquitetura
**Prós:**
- Modelo pronto, já desenhado no ADR de referência, com política de expiração e refresh definida.

**Contras:**
- Não há usuário final autenticado na API pública do Pluriverso — o único ator autenticado é o Comitê Federado, e o mecanismo de autenticação do Comitê é decisão de outro documento (ADR-003 local), não deste.
- Não há contrato comercial nem quota diferenciada por cliente que justifique API Keys com tiers; o controle de abuso da API pública é rate limit por IP, tratado em `docs/governanca-e-seguranca.md`.

## Decisão

A API pública do Pluriverso é **REST-only**. Do [ADR-002 "Padrões de API e Integração"](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-002-api-standards.md) (Arquitetura-BioCultural, status `Proposto`), o Pluriverso aproveita **apenas quatro elementos**, reproduzidos abaixo verbatim: o envelope de paginação, o envelope de erro, os operadores de filtro e a tabela de códigos de status HTTP. Todo o restante daquele ADR — GraphQL, JWT + refresh token, API Keys com tiers, base URL `api.etnoknowledge.org` — é **rejeitado** para o Pluriverso, conforme justificado nas Opções Consideradas.

### Envelope de coleção (herdado verbatim)

```json
{
  "data": [],
  "pagination": { "page": 1, "limit": 20, "totalPages": 50, "totalItems": 1000, "hasNext": true, "hasPrev": false },
  "links": { "self": "...", "first": "...", "prev": null, "next": "...", "last": "..." }
}
```

### Envelope de erro (herdado verbatim)

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Falha na validação dos dados",
    "details": [{ "field": "member_type", "message": "Valor inválido", "code": "INVALID_ENUM" }]
  },
  "meta": { "requestId": "req_abc124", "timestamp": "2026-08-01T10:35:00Z" }
}
```

### Operadores de filtro (herdados verbatim)

- Igualdade: `?campo=valor`
- Maior que: `?campo[gt]=`
- Menor que: `?campo[lt]=`
- Contém: `?campo[contains]=`
- Pertence a: `?campo[in]=a,b`
- Busca full-text: `?search=`

### Códigos de erro (`error.code`) — lista fechada do Pluriverso

Literais válidos, sem extensão fora deste conjunto sem revisar este ADR:

`VALIDATION_ERROR`, `INVALID_ENUM`, `REQUIRED_FIELD`, `NOT_FOUND`, `UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMIT_EXCEEDED`, `MEMBER_NOT_ACTIVE`, `MAPPING_CONFLICT`, `PROBE_FAILED`, `INTERNAL_ERROR`.

### Códigos de status HTTP (herdados, com uso no contexto do Pluriverso)

| Código | Uso no Pluriverso |
|---|---|
| `200 OK` | Resposta bem-sucedida a `GET`/`PUT`/`PATCH` — busca, consulta de registro/membro/mapeamento, decisão de pedido de adesão, decisão de mapeamento |
| `201 Created` | Recurso criado — pedido de adesão registrado (`POST /api/federation/membership-requests`), mapeamento proposto (`POST /api/v1/mappings`), conceito cadastrado manualmente (`POST /api/v1/concepts`) |
| `204 No Content` | Operação bem-sucedida sem corpo de retorno — nenhum endpoint público remove recurso sem resposta; reservado para uso futuro |
| `400 Bad Request` | Payload ou parâmetro de query mal formado — `error.code: VALIDATION_ERROR` ou `REQUIRED_FIELD` |
| `401 Unauthorized` | Credencial ausente ou inválida em endpoint do Comitê Federado — `error.code: UNAUTHORIZED` |
| `403 Forbidden` | Credencial válida sem permissão para a ação — `error.code: FORBIDDEN` |
| `404 Not Found` | Registro, membro, conceito ou pedido inexistente — `error.code: NOT_FOUND` |
| `409 Conflict` | Estado do recurso impede a operação — pedido de adesão já decidido, `url_base` já pertence a membro ativo, confirmação de purge divergente — `error.code: MAPPING_CONFLICT` ou `VALIDATION_ERROR` conforme o caso |
| `422 Unprocessable Entity` | Payload sintaticamente válido mas com valor de enum inválido (ex.: `member_type` fora da lista fechada) — `error.code: INVALID_ENUM` |
| `429 Too Many Requests` | Rate limit por IP excedido (`docs/governanca-e-seguranca.md`) — `error.code: RATE_LIMIT_EXCEEDED` |
| `500 Internal Server Error` | Falha não tratada no servidor — `error.code: INTERNAL_ERROR` |
| `503 Service Unavailable` | Banco indisponível ou instância em manutenção |

Os códigos `MEMBER_NOT_ACTIVE` (ação de federação recusada porque o membro alvo não está `active`) e `PROBE_FAILED` (probe técnico de adesão falhou — sinal, nunca gate, conforme ADR-006/E3) tipicamente acompanham `400` ou `409`, a depender do endpoint.

### Rejeitados (do ADR-002-arquitetura)

- **GraphQL**: o ADR-002-arquitetura o escolheu para o contexto de "Apresentação". Dobra a superfície de manutenção, exige camada de resolvers e introspection de schema para um caso de uso que é busca com filtros fixos, e é incompatível com o Docker mínimo exigido por `principios.md`.
- **JWT + refresh token**: não há usuário final autenticado na API pública do Pluriverso; o único ator autenticado é o Comitê Federado. O mecanismo de autenticação do Comitê é decisão de outro documento (ADR-003 local) — este ADR apenas registra que JWT não é esse mecanismo.
- **API Keys com tiers**: não há contrato comercial nem quota por cliente distinta do rate limit por IP; introduzir tiers seria complexidade sem consumidor.
- **Base URL fixa `api.etnoknowledge.org`**: nomenclatura pré-federação. No Pluriverso a URL base é por instância ([ADR-009/MI2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md)), configurada via `PUBLIC_BASE_URL`, nunca um domínio fixo.

### Versionamento

Estratégia herdada do ADR-002-arquitetura: **URL Path Versioning**. Toda rota da API pública do Pluriverso leva o prefixo `/api/v1/` (ex.: `GET /api/v1/search`, `GET /api/v1/members`).

Os endpoints de **federação** — `POST /api/federation/membership-requests` ([ADR-006/E1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)), `GET /api/federation/records` ([ADR-004/D6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md)) e `GET /api/federation/concepts` — **não** levam `/v1`. São contratos fixados verbatim por ADRs aceitos da Arquitetura-BioCultural (ou pela extensão mínima descrita em `docs/contrato-harvest.md` §3.5, no caso de `concepts`); versioná-los junto com a API pública os exporia a uma política de depreciação que não é deles e quebraria o contrato já publicado para os membros da federação. Endpoints administrativos do Comitê sob `/api/federation/...` (ex.: `PATCH /api/federation/membership-requests/{id}`) seguem a mesma regra por consistência de namespace, mesmo sem serem contratos externos ao membro.

## Consequências

### Positivas
- Um único estilo de API (REST) simplifica implementação, testes e documentação (`docs/api.md` + `docs/api/openapi.yaml`).
- Vocabulário de erro e paginação já testado em produção pela família de sistemas da federação, sem reinvenção.
- Separação clara entre API pública versionável (`/api/v1/`) e contratos de federação não-versionáveis (`/api/federation/`) evita que uma mudança de versão da API pública quebre o harvest entre instâncias.

### Negativas
- Clientes que precisem compor várias entidades relacionadas numa única chamada fazem múltiplas requisições REST em vez de uma consulta GraphQL.
- A lista fechada de `error.code` exige revisão deste ADR sempre que um novo tipo de erro de negócio surgir.

### Mitigações
- `docs/api/openapi.yaml` documenta declarativamente cada endpoint, reduzindo o custo de integração que o GraphQL resolveria por introspection.
- Novos códigos de erro entram por atualização deste ADR, não por invenção ad-hoc em endpoint individual — mantém a lista fechada auditável.

## Referências
- [ADR-002 — Padrões de API e Integração (Arquitetura-BioCultural, status `Proposto`)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-002-api-standards.md)
- [ADR-004 — Arquitetura Federada, D6 (Contrato de Publicação de Registros)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md)
- [ADR-006 — Protocolo de Adesão à Federação, E1 (Modelo de Dados Mínimo)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)
- [ADR-008 — Motor de Banco de Dados do Pluriverso, DB1–DB6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-008-pluriverso-database-engine.md)
- [ADR-009 — Topologia Multi-Instância do Pluriverso, MI2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md)
- `docs/principios.md` (Pluriverso) — exigência de simplicidade e imagem Docker mínima
- `docs/decisions/ADR-001-stack-e-framework.md` (Pluriverso) — stack Express/EJS que a API REST implementa
- `docs/api.md` e `docs/api/openapi.yaml` (Pluriverso) — especificação completa de cada endpoint

## Data de Revisão
Revisitar se o ADR-002 "Padrões de API e Integração" da Arquitetura-BioCultural mudar de status (`Proposto` → `Aceito` ou reescrita), pois qualquer alteração no envelope, no vocabulário de erro ou na tabela de status HTTP herdados verbatim exige reavaliar este documento.
