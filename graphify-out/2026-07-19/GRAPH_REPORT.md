# Graph Report - pluriverso  (2026-07-15)

## Corpus Check
- 3 files · ~1,351 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 24 nodes · 22 edges · 4 communities (3 shown, 1 thin omitted)
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `b14bb066`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- Community 0
- Community 1
- Community 2
- CLAUDE.md

## God Nodes (most connected - your core abstractions)
1. `Pluriverso` - 11 edges
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

## Communities (4 total, 1 thin omitted)

### Community 0 - "Community 0"
Cohesion: 0.20
Nodes (9): Agradecimentos, Contato, Documentação da Arquitetura, Licença, Necessidades de Implementação (v3.1), O que é o Pluriverso?, Pluriverso, Posição na Arquitetura Federada (+1 more)

### Community 1 - "Community 1"
Cohesion: 0.33
Nodes (6): 1. Harvest Periódico, 2. Índice Central, 3. Camada de Mapeamento Semântico, 4. API Pública Unificada, 5. Interface de Governança, Responsabilidades

### Community 2 - "Community 2"
Cohesion: 0.40
Nodes (5): CARE na Prática, Princípios de Design, Remoção Imediata, Soberania dos Membros, Transparência de Origem

## Knowledge Gaps
- **19 isolated node(s):** `O que é o Pluriverso?`, `Posição na Arquitetura Federada`, `1. Harvest Periódico`, `2. Índice Central`, `3. Camada de Mapeamento Semântico` (+14 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **1 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Pluriverso` connect `Community 0` to `Community 1`, `Community 2`?**
  _High betweenness centrality (0.652) - this node is a cross-community bridge._
- **Why does `Responsabilidades` connect `Community 1` to `Community 0`?**
  _High betweenness centrality (0.336) - this node is a cross-community bridge._
- **Why does `Princípios de Design` connect `Community 2` to `Community 0`?**
  _High betweenness centrality (0.277) - this node is a cross-community bridge._
- **What connects `O que é o Pluriverso?`, `Posição na Arquitetura Federada`, `1. Harvest Periódico` to the rest of the system?**
  _19 weakly-connected nodes found - possible documentation gaps or missing edges._