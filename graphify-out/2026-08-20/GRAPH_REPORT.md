# Graph Report - D:\git\pluriverso  (2026-08-20)

## Corpus Check
- cluster-only mode — file stats not available

## Summary
- 38 nodes · 28 edges · 15 communities (6 shown, 9 thin omitted)
- Extraction: 82% EXTRACTED · 18% INFERRED · 0% AMBIGUOUS · INFERRED: 5 edges (avg confidence: 0.88)
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `218663c3`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- ADR-001: Stack e Framework
- Arquitetura do Pluriverso (C4)
- README.md
- HarvestClient
- SearchService
- ADR-004: Expansão Semântica SKOS
- AuditService
- Contrato de Harvest
- Governança e Segurança
- Operação
- Princípios do desenvolvimento
- Roadmap
- MappingService
- WebUi

## God Nodes (most connected - your core abstractions)
1. `Arquitetura do Pluriverso (C4)` - 4 edges
2. `SearchService` - 4 edges
3. `PublicApi` - 3 edges
4. `CommitteeApi` - 3 edges
5. `ADR-001: Stack e Framework` - 3 edges
6. `ADR-002: API Pública REST` - 3 edges
7. `API do Pluriverso` - 2 edges
8. `Busca Semântica` - 2 edges
9. `HarvestScheduler` - 2 edges
10. `HarvestClient` - 2 edges

## Surprising Connections (you probably didn't know these)
- `API do Pluriverso` --references--> `CommitteeApi`  [INFERRED]
  docs/api.md → docs/arquitetura.md
- `API do Pluriverso` --references--> `PublicApi`  [INFERRED]
  docs/api.md → docs/arquitetura.md
- `RecordIndexer` --references--> `SQLite (better-sqlite3)`  [INFERRED]
  docs/arquitetura.md → docs/modelo-de-dados.md
- `ADR-005: Perfil Mínimo de Publicação` --conceptually_related_to--> `OpenAPI Specification`  [INFERRED]
  docs/decisions/ADR-005-perfil-minimo-de-publicacao.md → docs/api/openapi.yaml
- `Busca Semântica` --references--> `SearchService`  [EXTRACTED]
  docs/busca-semantica.md → docs/arquitetura.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Pluriverso Core Services** — pluriverso_search_service, pluriverso_membership_service, pluriverso_mapping_service, pluriverso_harvest_scheduler [EXTRACTED 0.90]
- **Pluriverso Persistence Layer** — sqlite_db, pluriverso_record_indexer, pluriverso_audit_service, pluriverso_purge_service [INFERRED 0.80]
- **Pluriverso Core Architecture** — docs_decisions_adr_001_stack_e_framework, docs_decisions_adr_002_api_publica_rest, docs_decisions_adr_003_autenticacao_do_comite [EXTRACTED 1.00]
- **Semantic Search Flow** — docs_decisions_adr_004_expansao_semantica_skos, docs_decisions_adr_005_perfil_minimo_de_publicacao, skos_semantic_expansion [INFERRED 0.90]

## Communities (15 total, 9 thin omitted)

### Community 0 - "ADR-001: Stack e Framework"
Cohesion: 0.25
Nodes (8): better-sqlite3, requireCommittee Middleware, OpenAPI Specification, ADR-001: Stack e Framework, ADR-002: API Pública REST, ADR-003: Autenticação do Comitê, ADR-005: Perfil Mínimo de Publicação, Express.js

### Community 1 - "Arquitetura do Pluriverso (C4)"
Cohesion: 0.33
Nodes (7): API do Pluriverso, Arquitetura do Pluriverso (C4), CommitteeApi, HarvestScheduler, MembershipService, ProbeService, PublicApi

### Community 2 - "README.md"
Cohesion: 0.50
Nodes (3): Arquitetura BioCultural, BioCultDB, BioCultTermos

### Community 3 - "HarvestClient"
Cohesion: 0.50
Nodes (4): Modelo de Dados, HarvestClient, RecordIndexer, SQLite (better-sqlite3)

### Community 4 - "SearchService"
Cohesion: 1.00
Nodes (3): Busca Semântica, SearchService, SemanticExpander

## Knowledge Gaps
- **20 isolated node(s):** `Contrato de Harvest`, `Governança e Segurança`, `Modelo de Dados`, `Operação`, `Roadmap` (+15 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **9 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Arquitetura do Pluriverso (C4)` connect `Arquitetura do Pluriverso (C4)` to `SearchService`?**
  _High betweenness centrality (0.076) - this node is a cross-community bridge._
- **Why does `HarvestScheduler` connect `Arquitetura do Pluriverso (C4)` to `HarvestClient`?**
  _High betweenness centrality (0.054) - this node is a cross-community bridge._
- **Why does `HarvestClient` connect `HarvestClient` to `Arquitetura do Pluriverso (C4)`?**
  _High betweenness centrality (0.045) - this node is a cross-community bridge._
- **What connects `Contrato de Harvest`, `Governança e Segurança`, `Modelo de Dados` to the rest of the system?**
  _20 weakly-connected nodes found - possible documentation gaps or missing edges._