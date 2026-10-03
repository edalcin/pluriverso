# ADR-004: Expansão Semântica SKOS — Regras de Travessia por Predicado

## Status
Aceito

## Contexto

O `SemanticExpander` (componente canônico definido em [`arquitetura.md`](../arquitetura.md)) é a peça que transforma uma busca por um termo de um membro em resultados de outros membros que usam um termo diferente para o mesmo conceito — o caso de uso central do Pluriverso, descrito no `README.md` do projeto com o exemplo "mandioca" ↔ "macaxeira" ↔ "Maniaka".

[ADR-008/DB5 (Arquitetura-BioCultural)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md) já decide **como** essa expansão é implementada: uma CTE recursiva (`WITH RECURSIVE`) sobre `concept_mappings`, dentro do SQLite único do Pluriverso, "sem banco de grafo nem vetorial". O que o ADR-008 **não** decide é a semântica de travessia por predicado — e essa omissão importa, porque `concept_mappings.predicate` aceita quatro literais SKOS distintos ([ADR-008/DB5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md): `skos:exactMatch` \| `skos:closeMatch` \| `skos:broadMatch` \| `skos:narrowMatch`), e a [SKOS Reference (W3C)](https://www.w3.org/TR/skos-reference/skos-xl.html) atribui a cada um deles propriedades formais diferentes de simetria e transitividade. Tratar os quatro do mesmo jeito numa única perna de CTE — a implementação "óbvia" e mais simples — produziria resultados que nenhum Comitê Federado aprovou explicitamente, e o efeito não é um bug isolado: **muda o conjunto de resultados de toda busca federada**, porque a expansão semântica roda em toda consulta com `expand=skos` (default).

Este ADR fixa, uma única vez, a regra de travessia de cada predicado, para que `docs/busca-semantica.md` (pipeline e SQL de referência), `docs/arquitetura.md` (`SemanticExpander`) e a implementação futura nunca precisem decidir isso de novo nem divirjam entre si.

## Requisitos

### Funcionais
- Expandir `skos:exactMatch` de forma **simétrica e transitiva**, sem limite de salto semântico — apenas um guarda-corpo técnico de profundidade, como defesa contra dado patológico, nunca como regra de negócio.
- Expandir `skos:closeMatch` de forma **simétrica, não-transitiva** — exatamente 1 salto, e somente a partir de conceitos já alcançados pelo fechamento transitivo de `exactMatch` da semente (incluindo a própria semente).
- Expandir `skos:narrowMatch` de forma **direcional, descendente** (do conceito mais amplo para o mais específico), até `SKOS_EXPANSION_MAX_DEPTH` saltos (default `3`).
- **Não** expandir `skos:broadMatch` por padrão. Disponibilizar apenas por opt-in explícito do cliente da API (`expand_broader=true`), com a mesma disciplina de profundidade de `narrowMatch`.
- Detectar e interromper ciclos de mapeamento — dois membros podem propor `A exactMatch B` e `B exactMatch A` de forma independente e igualmente legítima, e a travessia não pode entrar em recursão infinita por causa disso.
- Considerar, na expansão, **somente** mapeamentos com `status='approved'` — mapeamentos `proposed` (ver §"Fluxo de curadoria" em `docs/busca-semantica.md`) nunca influenciam um resultado de busca.

### Não-Funcionais
- Implementável como uma única consulta SQL (`WITH RECURSIVE`) sobre SQLite, sem grafo externo, sem serviço adicional, sem material armazenado fora de `concept_mappings` ([ADR-008/DB5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)).
- Determinística e auditável: dado o mesmo conjunto de mapeamentos `approved`, a mesma busca sempre expande para o mesmo conjunto de conceitos, na mesma ordem de prioridade de predicado.
- Estável frente ao crescimento do grafo de mapeamentos: aprovar um novo mapeamento nunca exige recalibrar a regra de travessia — só amplia o conjunto de arestas que ela percorre.

## Opções Consideradas

### Opção 1: Regras de travessia diferenciadas por predicado (escolhida)
**Prós:**
- Cada predicado se comporta conforme sua semântica formal SKOS ([SKOS Reference, W3C](https://www.w3.org/TR/skos-reference/skos-xl.html)): equivalência exata propaga livremente, relação próxima não encadeia, relação hierárquica não some resultados numa árvore inteira sem pedido explícito.
- `exactMatch` transitivo é exatamente o mecanismo que resolve o caso de uso do `README.md`: três membros com três rótulos distintos para o mesmo táxon, unidos por uma cadeia de equivalências aprovadas pelo Comitê.
- `closeMatch` limitado a 1 salto evita o problema concreto que motivou este ADR: encadear `closeMatch` sucessivos arrasta a busca para conceitos cada vez mais distantes do termo original, sem que nenhuma aprovação individual do Comitê tenha decidido essa cadeia completa.
- `broadMatch` desligado por padrão preserva precisão: subir a hierarquia devolveria o gênero taxonômico inteiro (todas as espécies do gênero) para uma busca por uma espécie — praticamente sempre indesejado.

**Contras:**
- Quatro pernas de travessia distintas na CTE (mais o parâmetro de opt-in de `broadMatch`), em vez de uma única perna homogênea — mais superfície para revisar e testar.

### Opção 2: Tratar todos os predicados como transitivos e simétricos
**Prós:**
- Uma única perna de CTE, sem distinção de predicado — a implementação mais simples possível sobre `WITH RECURSIVE`.

**Contras:**
- Viola a semântica formal do SKOS: `skos:closeMatch` é explicitamente **não-transitivo** e `skos:narrowMatch`/`skos:broadMatch` são **direcionais**, não simétricos ([SKOS Reference, W3C](https://www.w3.org/TR/skos-reference/skos-xl.html)); tratá-los como `exactMatch` finge uma equivalência que ninguém no Comitê aprovou.
- Produz falsos positivos exatamente no cenário citado pelo pedido que originou este ADR: "mandioca" alcançaria, em três ou quatro saltos de `closeMatch` encadeado, um conceito sem relação real alguma — destruindo a confiança na busca federada.
- Deixaria `narrowMatch`/`broadMatch` sem limite de profundidade, arriscando trazer uma subárvore taxonômica inteira como resultado de uma busca por um único termo.

### Opção 3: Sem expansão automática, só busca textual (FTS)
**Prós:**
- Zero risco de falso positivo por expansão — toda contradição deste ADR desaparece porque não há travessia nenhuma.
- Implementação trivial: `records_fts MATCH` e nada mais.

**Contras:**
- Destrói o caso de uso central do Pluriverso: buscar "mandioca" não encontraria o registro que só menciona "macaxeira" ou "Manihot esculenta" — exatamente o cenário que o `README.md` do projeto usa para justificar a existência da federação.
- Torna todo o trabalho de harvest de conceitos ([`contrato-harvest.md` §5](../contrato-harvest.md)) e de curadoria de mapeamentos (`concept_mappings`, `docs/governanca-e-seguranca.md`) inútil em tempo de busca — os dados existiriam no índice sem nunca serem usados para o que foram coletados.

## Decisão

A expansão semântica do Pluriverso segue **quatro regras de travessia distintas**, uma por predicado, todas aplicadas por uma única CTE recursiva sobre `concept_mappings` filtrada a `status='approved'` (a consulta de referência completa está em [`docs/busca-semantica.md`](../busca-semantica.md)):

| Predicado | Simetria | Transitividade | Limite de salto | Ativação |
|---|---|---|---|---|
| `skos:exactMatch` | Simétrico | **Transitivo** | Nenhum limite semântico — só o guarda-corpo técnico de profundidade (defesa contra dado patológico, não regra de negócio) | Sempre (default) |
| `skos:closeMatch` | Simétrico | **Não-transitivo** | Exatamente 1 salto, e só a partir de um conceito no fechamento transitivo de `exactMatch` da semente (a própria semente conta) | Sempre (default) |
| `skos:narrowMatch` | Direcional (descendente) | Transitivo dentro do limite | `SKOS_EXPANSION_MAX_DEPTH` (default `3`) saltos | Sempre (default) |
| `skos:broadMatch` | Direcional (ascendente) | Transitivo dentro do limite | `SKOS_EXPANSION_MAX_DEPTH` (default `3`) saltos | **Opt-in** — só com `expand_broader=true` na consulta |

Regras adicionais, válidas para as quatro pernas:

1. **Só `status='approved'`.** Um mapeamento `proposed` (sugestão automática ou submissão do Comitê ainda não decidida) e um mapeamento `rejected` nunca entram na travessia. A CTE filtra `concept_mappings.status = 'approved'` em toda perna, sem exceção.
2. **Detecção de ciclo por caminho acumulado.** Cada linha da CTE carrega o caminho de URIs já visitado (concatenado, delimitado); antes de seguir uma aresta, a URI de destino é conferida contra esse caminho, e a aresta é descartada se o destino já apareceu. Isso é necessário e suficiente para impedir recursão infinita quando o Comitê aprova, por exemplo, `A exactMatch B` e, separadamente, `B exactMatch A` — duas propostas independentes, ambas legítimas, que sem detecção de ciclo formariam um laço.
3. **Um nó alcançado via `closeMatch` não expande mais.** Como `closeMatch` não é transitivo, a aresta de `closeMatch` é sempre a última de um caminho — nenhuma perna (nem outro `closeMatch`, nem `narrowMatch`, nem `broadMatch`) continua a partir de um nó cuja última aresta usada foi `closeMatch`. Isso é o que torna "exatamente 1 salto" uma garantia estrutural da consulta, não uma convenção que a aplicação precisa lembrar de respeitar.
4. **`broadMatch` nunca entra na expansão default.** A perna de `broadMatch` só é incluída na consulta quando o cliente da API pública passa `expand_broader=true` explicitamente (ver `docs/api.md` §`GET /api/v1/search`). Sem esse parâmetro, a CTE nem contém essa perna — não é um filtro pós-consulta, é ausência de trecho de SQL.
5. **O guarda-corpo técnico de `exactMatch` é distinto de `SKOS_EXPANSION_MAX_DEPTH`.** `SKOS_EXPANSION_MAX_DEPTH` é um parâmetro de **negócio**, que define até onde descer/subir numa hierarquia antes que o resultado deixe de ser preciso (`narrowMatch`/`broadMatch`). O guarda-corpo de `exactMatch` é puramente **defensivo** — protege contra uma cadeia patologicamente longa de equivalências que a detecção de ciclo, sozinha, não limitaria em tamanho (só em repetição). Na prática, cadeias de `exactMatch` curadas pelo Comitê são curtas (poucos sinônimos por conceito); o guarda-corpo existe para o caso degenerado, não para o caso comum.

## Consequências

### Positivas
- A busca federada encontra sinônimos entre membros respeitando exatamente a força de relação que o Comitê aprovou: equivalência plena propaga livremente (`exactMatch`), relação próxima soma um resultado adicional sem encadear (`closeMatch`), hierarquia desce sob controle de profundidade (`narrowMatch`) e nunca sobe sem pedido explícito (`broadMatch`).
- Regra única e citável: `docs/busca-semantica.md`, `docs/arquitetura.md` (`SemanticExpander`) e a implementação futura referenciam este ADR em vez de redefinir a semântica cada um a seu modo.
- Ciclo entre mapeamentos aprovados independentemente por diferentes decisões do Comitê nunca trava a busca nem exige coordenação prévia entre os curadores que propuseram cada lado.

### Negativas
- A consulta de expansão é mais complexa que uma CTE homogênea de perna única — quatro pernas distintas em `UNION ALL`, mais o parâmetro de opt-in de `broadMatch`, exigem teste e manutenção maiores.
- Uma relação "de segunda ordem" — dois conceitos que só se conectam por dois saltos de `closeMatch` encadeados — nunca é alcançada automaticamente, mesmo quando o domínio justificaria. O sistema prefere um falso negativo (deixar de mostrar uma relação plausível) a um falso positivo (mostrar uma relação que ninguém aprovou explicitamente).

### Mitigações
- O fator de pontuação por predicado (`docs/busca-semantica.md`, etapa 6 do pipeline de busca) é heurística ajustável, não invariante — recalibrar pesos de ranking não exige revisar este ADR.
- Se o Comitê julgar que dois conceitos hoje ligados só por um `closeMatch` merecem equivalência mais forte, o caminho é propor um `exactMatch` direto entre eles (ou uma cadeia de `exactMatch`) — nunca afrouxar a regra de travessia global para todos os mapeamentos da federação.
- O opt-in `expand_broader=true` mantém `broadMatch` disponível para o cliente que sabe que quer o gênero inteiro (ex.: uma pesquisa exploratória), sem penalizar a precisão do caso default.

## Referências
- [ADR-004 — Arquitetura Federada, D2 (Camada de Mapeamento Semântico)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)
- [ADR-008 — Motor de Banco de Dados do Pluriverso, DB5 (CTE Recursiva para Expansão SKOS)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)
- [SKOS Simple Knowledge Organization System — Reference / SKOS-XL Namespace Document (W3C)](https://www.w3.org/TR/skos-reference/skos-xl.html)
- [SKOS Reference (W3C) — vocabulário de mapeamento (`skos:exactMatch`, `skos:closeMatch`, `skos:broadMatch`, `skos:narrowMatch`)](https://www.w3.org/TR/skos-reference/)
- `docs/busca-semantica.md` (Pluriverso) — pipeline de busca, SQL de referência da CTE completa e exemplo ponta a ponta
- `docs/arquitetura.md` (Pluriverso) — componente `SemanticExpander`
- `docs/governanca-e-seguranca.md` (Pluriverso) — workflow de aprovação de mapeamentos pelo Comitê Federado
- `docs/modelo-de-dados.md` (Pluriverso) — DDL de `concepts` e `concept_mappings`

## Data de Revisão
Revisitar se o volume de mapeamentos aprovados pelo Comitê tornar a ausência de encadeamento de `closeMatch` (ou a ausência de expansão default de `broadMatch`) uma reclamação recorrente de precisão de busca; ou se o ADR-008 da Arquitetura-BioCultural for revisado e substituir a orientação de "CTE recursiva" por outro mecanismo de expansão.
