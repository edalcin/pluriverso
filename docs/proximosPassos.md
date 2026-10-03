# Próximos Passos — Pluriverso

> **Documento de estado deste componente.** Registra onde o Pluriverso está e o que falta fazer. Ponto de entrada de qualquer nova sessão de trabalho — humana ou assistida por IA.
>
> Pendência de arquitetura da federação **não** mora aqui: mora em [`Arquitetura-BioCultural/docs/tecnico/proximosPassos.md`](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/proximosPassos.md), que é a referência única do projeto. Aqui ficam só as pendências deste componente.
>
> **Regras de manutenção:** ao final de cada sessão, atualizar a data, o estado e a lista de pendências. Pendência resolvida não é apagada: é marcada como feita, com o `onde`. Caminhos são relativos à raiz deste repositório.

**Estado em:** 2026-08-30

---

## 1. Estado

**Só documentação; nenhuma linha de implementação.** A especificação está completa: arquitetura C4 em três níveis (14 componentes lógicos em quatro camadas), contrato de harvest, modelo de dados, princípios, governança e segurança, roadmap de sete fases (0–6) com critério de aceitação observável, e a stack fixada na ADR-001 — Node.js ≥20, Express, EJS, Tailwind, Alpine.js, HTMX, better-sqlite3, node-cron, sem GraphQL, Redis, Elasticsearch ou servidor de banco.

É o **middleware de federação**: coleta de cada membro apenas o que ele publicou, indexa em SQLite e expõe API unificada com busca semântica sobre SKOS. Instanciável em múltiplos escopos (ADR-009 da arquitetura): uma associação de comunidades pode federar-se sem depender do índice público global.

## 2. Pendências

Ordenadas pelo roadmap (`docs/roadmap.md`).

| Fase | Pendência | Bloqueio |
|---|---|---|
| **0** | Esqueleto Express, migrações SQL, `/health`, Docker e CI publicando `ghcr.io/edalcin/pluriverso` | — |
| **1** | API de *membership* (POST/GET/PATCH), probe anti-SSRF, autenticação HTTP Basic do Comitê, geração de `member_id` | 0 |
| **2** | `HarvestClient` paginado, `RecordIndexer`, FTS5, tabela `harvest_runs` — indexar e servir registros de um membro simulado | 1 |
| **3** | API pública REST e UI de busca com rolagem infinita | 2 |
| **4** | Camada SKOS: conceitos, mapeamentos, expansão de consulta | 3 |
| **5** | *Purge* e auditoria | 2 |
| **6** | Detecção automática de remoção — registro que deixou de ser publicado sai do índice na coleta seguinte | 2 |

### Dependências externas a este repositório

- **Contrato de harvest**: a versão normativa é a da arquitetura (`docs/contrato-harvest.md` + ADR-016, estado *Proposto*). O `docs/contrato-harvest.md` deste repositório é a visão do implementador e precisa ser conferido contra ela antes da Fase 2.
- **Nenhum provedor real existe ainda**: só o BioCultDB está em produção, e sem endpoint de harvest. A Fase 2 termina em membro **simulado**, por necessidade.
- **H-Q1 (`sacred` ≡ `private`)** segue aberta e vai à reunião com as lideranças. Regra interina no índice: trata-se como `private` — nunca atravessa.

## 3. Onde está cada coisa

| Artefato | Caminho |
|---|---|
| Roadmap de sete fases, com critério de aceitação | `docs/roadmap.md` |
| Arquitetura C4 (contexto, contêiner, componentes) | `docs/arquitetura.md` |
| Contrato de harvest (visão do implementador) | `docs/contrato-harvest.md` |
| Stack e framework | `docs/decisions/ADR-001-stack-e-framework.md` |
| Governança e segurança | `docs/governanca-e-seguranca.md` |
| Referência única do projeto | <https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/proximosPassos.md> |
