## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).

## Arquitetura v3.3 — Persistência e Topologia
Persistência do Pluriverso = SQLite embutida via `better-sqlite3` (JSON1 + FTS5, WAL), arquivo único externo
ao container via `SQLITE_DB_PATH` (default `/data/pluriverso.sqlite`). Ref.: ADR-008.
Pluriverso é instanciável (não singleton): associação pode rodar instância própria, escopada aos seus
membros, sem hierarquia entre instâncias. Ref.: ADR-009.
Ref. gerais: Arquitetura-BioCultural/docs/architecture-decisions/ADR-005, ADR-008, ADR-009.
