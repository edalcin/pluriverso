# Graph Report - pluriverso  (2026-07-31)

## Corpus Check
- 17 files · ~34,507 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 278 nodes · 316 edges · 19 communities (18 shown, 1 thin omitted)
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `740f9249`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- Pluriverso
- ADR-002: API Pública REST
- §5.1 — Pública, sem autenticação
- CLAUDE.md
- Governança
- Arquitetura do Pluriverso (C4)
- ADR-003: Autenticação do Comitê
- ADR-001: Stack e Framework
- ADR-004: Expansão Semântica SKOS — Regras de Travessia por Predicado
- ADR-005: Perfil Mínimo de Publicação
- Demais Controles de Segurança
- Busca Semântica
- Modelo de Dados
- §5.2 — Comitê Federado, autenticada
- Contrato de Harvest
- Roadmap
- Operação
- Princípios do desenvolvimento

## God Nodes (most connected - your core abstractions)
1. `Modelo de Dados` - 13 edges
2. `§5.2 — Comitê Federado, autenticada` - 12 edges
3. `Pluriverso` - 11 edges
4. `ADR-001: Stack e Framework` - 9 edges
5. `ADR-002: API Pública REST` - 9 edges
6. `ADR-003: Autenticação do Comitê` - 9 edges
7. `ADR-004: Expansão Semântica SKOS — Regras de Travessia por Predicado` - 9 edges
8. `ADR-005: Perfil Mínimo de Publicação` - 9 edges
9. `Demais Controles de Segurança` - 9 edges
10. `Roadmap` - 9 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Communities (19 total, 1 thin omitted)

### Community 0 - "Pluriverso"
Cohesion: 0.09
Nodes (22): 1. Harvest Periódico, 2. Índice Central, 3. Camada de Mapeamento Semântico, 4. API Pública Unificada, 5. Interface de Governança, Agradecimentos, CARE na Prática, Contato (+14 more)

### Community 1 - "ADR-002: API Pública REST"
Cohesion: 0.08
Nodes (24): ADR-002: API Pública REST, Consequências, Contexto, Códigos de erro (`error.code`) — lista fechada do Pluriverso, Códigos de status HTTP (herdados, com uso no contexto do Pluriverso), Data de Revisão, Decisão, Envelope de coleção (herdado verbatim) (+16 more)

### Community 2 - "§5.1 — Pública, sem autenticação"
Cohesion: 0.11
Nodes (19): §5.1 — Pública, sem autenticação, API do Pluriverso, Autenticação, Convenções, `GET /api/v1/concepts/{uri}/mappings`, `GET /api/v1/members`, `GET /api/v1/records/{federated_id}`, `GET /api/v1/search` (+11 more)

### Community 4 - "Governança"
Cohesion: 0.11
Nodes (19): 10. `MembershipService`, 11. `ProbeService`, 12. `MappingService`, 13. `PurgeService`, 14. `AuditService`, 1. `PublicApi`, 2. `CommitteeApi`, 3. `WebUi` (+11 more)

### Community 5 - "Arquitetura do Pluriverso (C4)"
Cohesion: 0.11
Nodes (19): 1. Aplicação Node.js/Express (porta `3100`), 1. Pesquisador / Aplicação, 2. Arquivo SQLite (volume externo), 2. Membro do Comitê Federado, 3. Solicitante de Adesão, 4. Instância de Membro, Arquitetura do Pluriverso (C4), Atores Detalhados (+11 more)

### Community 6 - "ADR-003: Autenticação do Comitê"
Cohesion: 0.11
Nodes (18): ADR-003: Autenticação do Comitê, Consequências, Contexto, Data de Revisão, Decisão, Funcionais, Mitigações, Negativas (+10 more)

### Community 7 - "ADR-001: Stack e Framework"
Cohesion: 0.12
Nodes (17): ADR-001: Stack e Framework, Consequências, Contexto, Data de Revisão, Decisão, Funcionais, Mitigações, Negativas (+9 more)

### Community 8 - "ADR-004: Expansão Semântica SKOS — Regras de Travessia por Predicado"
Cohesion: 0.12
Nodes (17): ADR-004: Expansão Semântica SKOS — Regras de Travessia por Predicado, Consequências, Contexto, Data de Revisão, Decisão, Funcionais, Mitigações, Negativas (+9 more)

### Community 9 - "ADR-005: Perfil Mínimo de Publicação"
Cohesion: 0.12
Nodes (17): ADR-005: Perfil Mínimo de Publicação, Consequências, Contexto, Data de Revisão, Decisão, Funcionais, Mitigações, Negativas (+9 more)

### Community 10 - "Demais Controles de Segurança"
Cohesion: 0.13
Nodes (15): CI — Dependabot + Trivy, Demais Controles de Segurança, Endurecimento do container, Escape de consulta FTS5, Governança, Governança e Segurança, Headers HTTP (helmet), LGPD — dado pessoal (+7 more)

### Community 12 - "Busca Semântica"
Cohesion: 0.15
Nodes (13): Busca Semântica, CTE recursiva completa, Etapa 1 — Normalizar `q`, Etapa 2 — Resolver `q` a conceitos-semente, Etapa 3 — Expandir as sementes pela CTE recursiva, Etapa 4 — Recuperar candidatos, Etapa 5 — Aplicar filtros estruturais, Etapa 6 — Ordenar (+5 more)

### Community 13 - "Modelo de Dados"
Cohesion: 0.15
Nodes (13): 1. `members`, 2. `membership_requests`, 3. `records`, 4. `record_terms`, 5. `records_fts`, 6. `concepts`, 7. `concept_mappings`, 8. `harvest_runs` (+5 more)

### Community 14 - "§5.2 — Comitê Federado, autenticada"
Cohesion: 0.17
Nodes (12): §5.2 — Comitê Federado, autenticada, `GET /api/federation/audit-log?action=&target_id=`, `GET /api/federation/harvest-runs?member_id=&status=`, `GET /api/federation/membership-requests?status=`, `GET /api/v1/mappings` e `POST /api/v1/mappings`, `PATCH /api/federation/members/{member_id}`, `PATCH /api/federation/membership-requests/{id}`, `PATCH /api/v1/mappings/{id}` (+4 more)

### Community 15 - "Contrato de Harvest"
Cohesion: 0.17
Nodes (12): §1 — Contrato de registros (cliente), §2 — Modos de coleta e detecção de remoção, §3 — Resiliência, §4 — Perfil Mínimo de Publicação, §5 — Contrato de conceitos, Completo (semanal, e sempre na primeira coleta de um membro), Comportamento do coletor, Configuração por membro (+4 more)

### Community 16 - "Roadmap"
Cohesion: 0.22
Nodes (9): Fase 0 — Esqueleto, Fase 1 — Membership, Fase 2 — Harvest e índice, Fase 3 — API pública e UI, Fase 4 — SKOS, Fase 5 — Purge e auditoria, Fase 6 — Detecção de remoção, Referências (+1 more)

### Community 17 - "Operação"
Cohesion: 0.25
Nodes (8): Backup e Restauração, `.env.example`, Imagem Docker, Implantação no UNRAID, Integração Contínua (CI), Observabilidade, Operação, Variáveis de Ambiente

### Community 18 - "Princípios do desenvolvimento"
Cohesion: 0.29
Nodes (6): Dados, Empacotamento e Implantação (Docker), Frontend, Princípios do desenvolvimento, Segurança (desde o início), Versionamento (Git)

## Knowledge Gaps
- **207 isolated node(s):** `O que é o Pluriverso?`, `Posição na Arquitetura Federada`, `1. Harvest Periódico`, `2. Índice Central`, `3. Camada de Mapeamento Semântico` (+202 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **1 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Arquitetura do Pluriverso (C4)` connect `Arquitetura do Pluriverso (C4)` to `roadmap.md`?**
  _High betweenness centrality (0.233) - this node is a cross-community bridge._
- **Why does `API do Pluriverso` connect `§5.1 — Pública, sem autenticação` to `roadmap.md`, `§5.2 — Comitê Federado, autenticada`?**
  _High betweenness centrality (0.195) - this node is a cross-community bridge._
- **Why does `ADR-002: API Pública REST` connect `ADR-002: API Pública REST` to `roadmap.md`?**
  _High betweenness centrality (0.152) - this node is a cross-community bridge._
- **What connects `O que é o Pluriverso?`, `Posição na Arquitetura Federada`, `1. Harvest Periódico` to the rest of the system?**
  _207 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Pluriverso` be split into smaller, more focused modules?**
  _Cohesion score 0.09090909090909091 - nodes in this community are weakly interconnected._
- **Should `ADR-002: API Pública REST` be split into smaller, more focused modules?**
  _Cohesion score 0.08333333333333333 - nodes in this community are weakly interconnected._
- **Should `§5.1 — Pública, sem autenticação` be split into smaller, more focused modules?**
  _Cohesion score 0.10526315789473684 - nodes in this community are weakly interconnected._