# Graph Report - pluriverso  (2026-07-05)

## Corpus Check
- 3 files · ~1,214 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 20 nodes · 19 edges · 3 communities
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `6425087d`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- [[_COMMUNITY_Community 0|Community 0]]
- [[_COMMUNITY_Community 1|Community 1]]
- [[_COMMUNITY_Community 2|Community 2]]

## God Nodes (most connected - your core abstractions)
1. `Pluriverso` - 10 edges
2. `Responsabilidades` - 6 edges
3. `Princípios de Design` - 5 edges
4. `O que é o Pluriverso?` - 1 edges
5. `Posição na Arquitetura Federada` - 1 edges
6. `1. Harvest Periódico` - 1 edges
7. `2. Índice Central` - 1 edges
8. `3. Camada de Mapeamento Semântico` - 1 edges
9. `4. API Pública Unificada` - 1 edges
10. `5. Interface de Governança` - 1 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Ecossistema de Federação CTA** — readme_pluriverso, readme_etnodb, readme_etnorelatos, readme_etnotermos [EXTRACTED 1.00]
- **Componentes Internos do Pluriverso** — readme_harvest_scheduler, readme_central_index, readme_skos_mapping_layer, readme_public_api, readme_governance_interface [EXTRACTED 1.00]

## Communities (3 total, 0 thin omitted)

### Community 0 - "Community 0"
Cohesion: 0.22
Nodes (8): Contato, Documentação da Arquitetura, Licença, Necessidades de Implementação (v3.0), O que é o Pluriverso?, Pluriverso, Posição na Arquitetura Federada, Relação com os Demais Componentes

### Community 1 - "Community 1"
Cohesion: 0.33
Nodes (6): 1. Harvest Periódico, 2. Índice Central, 3. Camada de Mapeamento Semântico, 4. API Pública Unificada, 5. Interface de Governança, Responsabilidades

### Community 2 - "Community 2"
Cohesion: 0.40
Nodes (5): CARE na Prática, Princípios de Design, Remoção Imediata, Soberania dos Membros, Transparência de Origem

## Knowledge Gaps
- **16 isolated node(s):** `O que é o Pluriverso?`, `Posição na Arquitetura Federada`, `1. Harvest Periódico`, `2. Índice Central`, `3. Camada de Mapeamento Semântico` (+11 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Pluriverso` connect `Community 0` to `Community 1`, `Community 2`?**
  _High betweenness centrality (0.854) - this node is a cross-community bridge._
- **Why does `Responsabilidades` connect `Community 1` to `Community 0`?**
  _High betweenness centrality (0.468) - this node is a cross-community bridge._
- **Why does `Princípios de Design` connect `Community 2` to `Community 0`?**
  _High betweenness centrality (0.386) - this node is a cross-community bridge._
- **What connects `O que é o Pluriverso?`, `Posição na Arquitetura Federada`, `1. Harvest Periódico` to the rest of the system?**
  _16 weakly-connected nodes found - possible documentation gaps or missing edges._