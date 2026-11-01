# Graph Report - D:\git\pluriverso  (2026-06-24)

## Corpus Check
- cluster-only mode — file stats not available

## Summary
- 12 nodes · 9 edges · 4 communities (1 shown, 3 thin omitted)
- Extraction: 89% EXTRACTED · 11% INFERRED · 0% AMBIGUOUS · INFERRED: 1 edges (avg confidence: 0.9)
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `a78a18a1`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- [[_COMMUNITY_Community 0|Community 0]]
- [[_COMMUNITY_Community 1|Community 1]]
- [[_COMMUNITY_Community 2|Community 2]]
- [[_COMMUNITY_Community 3|Community 3]]

## God Nodes (most connected - your core abstractions)
1. `Pluriverso` - 8 edges
2. `etnoDB` - 2 edges
3. `EtnoArquitetura` - 1 edges
4. `etnopapers` - 1 edges
5. `etnoRelatos` - 1 edges
6. `etnoTermos` - 1 edges
7. `Harvest Scheduler` - 1 edges
8. `Índice Central` - 1 edges
9. `Camada de Mapeamento Semântico (SKOS)` - 1 edges
10. `Princípios CARE` - 1 edges

## Surprising Connections (you probably didn't know these)
- `Pluriverso` --references--> `etnoDB`  [EXTRACTED]
  README.md → README.md  _Bridges community 0 → community 1_

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Ecossistema de Federação CTA** — readme_pluriverso, readme_etnodb, readme_etnorelatos, readme_etnotermos [EXTRACTED 1.00]
- **Componentes Internos do Pluriverso** — readme_harvest_scheduler, readme_central_index, readme_skos_mapping_layer, readme_public_api, readme_governance_interface [EXTRACTED 1.00]

## Communities (4 total, 3 thin omitted)

### Community 0 - "Community 0"
Cohesion: 0.25
Nodes (8): Princípios CARE, Índice Central, EtnoArquitetura, etnoRelatos, etnoTermos, Harvest Scheduler, Pluriverso, Camada de Mapeamento Semântico (SKOS)

## Knowledge Gaps
- **10 isolated node(s):** `EtnoArquitetura`, `etnopapers`, `etnoRelatos`, `etnoTermos`, `Harvest Scheduler` (+5 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **3 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Pluriverso` connect `Community 0` to `Community 1`?**
  _High betweenness centrality (0.636) - this node is a cross-community bridge._
- **Why does `etnoDB` connect `Community 1` to `Community 0`?**
  _High betweenness centrality (0.145) - this node is a cross-community bridge._
- **What connects `EtnoArquitetura`, `etnopapers`, `etnoRelatos` to the rest of the system?**
  _10 weakly-connected nodes found - possible documentation gaps or missing edges._