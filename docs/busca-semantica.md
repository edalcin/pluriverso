# Busca Semântica

Este documento especifica o pipeline de busca federada do Pluriverso — a lógica implementada pelo `SearchService`, com apoio do `SemanticExpander` (ambos componentes canônicos definidos em [`arquitetura.md`](arquitetura.md)) — desde a normalização da consulta do usuário até a lista ordenada de registros de múltiplos membros retornada por `GET /api/v1/search` ([`api.md`](api.md)).

A regra de travessia semântica por predicado SKOS que este pipeline usa está **decidida e justificada** em [`docs/decisions/ADR-004-expansao-semantica-skos.md`](decisions/ADR-004-expansao-semantica-skos.md); este documento não repete a análise de opções, só resume a regra e mostra o SQL de referência que a implementa.

---

## Regras de travessia por predicado (resumo)

Decisão completa: [ADR-004 local](decisions/ADR-004-expansao-semantica-skos.md). Base normativa herdada: [ADR-004/D2 (Arquitetura-BioCultural)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) decide que a camada de mapeamento semântico vive no Pluriverso, com predicados `skos:exactMatch`/`skos:closeMatch`/`skos:broadMatch` entre `ConceptScheme` de membros distintos; [ADR-008/DB5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md) fixa os quatro literais de `concept_mappings.predicate` e manda implementar a expansão como CTE recursiva sobre SQLite. A semântica formal de cada predicado vem da [SKOS Reference (W3C)](https://www.w3.org/TR/skos-reference/skos-xl.html).

| Predicado | Comportamento | Ativação |
|---|---|---|
| `skos:exactMatch` | Simétrico e **transitivo**. Expande recursivamente em ambas as direções, sem limite de salto semântico — só um guarda-corpo técnico de profundidade, como defesa contra dado patológico. | Sempre |
| `skos:closeMatch` | Simétrico, **não-transitivo**. Expande exatamente 1 salto, só a partir de um conceito no fechamento transitivo de `exactMatch` da semente. Nó alcançado por `closeMatch` não expande mais. | Sempre |
| `skos:narrowMatch` | Direcional, descendente (mais amplo → mais específico). Expande até `SKOS_EXPANSION_MAX_DEPTH` saltos (default `3`). | Sempre |
| `skos:broadMatch` | Direcional, ascendente. Mesma disciplina de profundidade de `narrowMatch`. Subir a hierarquia por padrão devolveria o gênero inteiro e destruiria a precisão. | **Opt-in** — só com `expand_broader=true` na query |

Duas regras valem para as quatro pernas, sem exceção: só mapeamentos com `concept_mappings.status = 'approved'` entram na expansão (um mapeamento `proposed` — ver [§Fluxo de curadoria](#fluxo-de-curadoria-de-mapeamento) — nunca influencia um resultado de busca); e a travessia detecta ciclo por caminho acumulado, porque `A exactMatch B` e `B exactMatch A` podem ser duas aprovações independentes e igualmente legítimas do Comitê Federado.

---

## Pipeline de busca

Seis etapas, executadas nesta ordem por `SearchService` a cada chamada de `GET /api/v1/search?q=...`.

### Etapa 1 — Normalizar `q`

`trim` (remove espaço nas bordas), normalização Unicode NFC, e minúsculas para o casamento de rótulo da Etapa 2. SQLite não tem normalização Unicode nativa (NFC/NFD), então essa etapa roda em JavaScript, antes de qualquer consulta; o resultado (`q_norm`) é o parâmetro ligado (bind) em todas as etapas seguintes. A versão **não normalizada** de `q` (só com espaços nas bordas removidos e aspas escapadas) segue em paralelo para a Etapa 4, porque `records_fts` já faz seu próprio *case-folding*/*diacritic-folding* via `tokenize='unicode61 remove_diacritics 2'` ([ADR-005/DA4](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)) e não precisa de `q` já minúsculo.

```sql
-- Referência do que a normalização produz (a normalização NFC em si roda em
-- JS: `q.trim().normalize('NFC').toLowerCase()`; o SQL abaixo é só o
-- equivalente do trim+lowercase, para os casos em que basta ASCII simples).
SELECT lower(trim(:q)) AS q_norm;
```

### Etapa 2 — Resolver `q` a conceitos-semente

Casamento em `concepts` por `pref_labels`/`alt_labels` (arrays JSON de `{value, language}`, [`modelo-de-dados.md`](modelo-de-dados.md)). Primeiro tenta casamento **exato**; só cai para **prefixo** se o exato não encontrar nenhuma semente — um termo que já existe exatamente como rótulo de um conceito nunca deve perder para um casamento de prefixo mais frouxo de outro conceito.

```sql
-- Prioridade 1: casamento exato de rótulo (pref_label OU alt_label)
SELECT DISTINCT c.uri, c.member_id
FROM concepts c, json_each(c.doc, '$.pref_labels') pl
WHERE lower(json_extract(pl.value, '$.value')) = :q_norm
UNION
SELECT DISTINCT c.uri, c.member_id
FROM concepts c, json_each(c.doc, '$.alt_labels') al
WHERE lower(json_extract(al.value, '$.value')) = :q_norm;

-- Prioridade 2 (só roda se a consulta acima não retornar nenhuma linha):
-- casamento por prefixo de rótulo
SELECT DISTINCT c.uri, c.member_id
FROM concepts c, json_each(c.doc, '$.pref_labels') pl
WHERE lower(json_extract(pl.value, '$.value')) LIKE :q_norm || '%'
UNION
SELECT DISTINCT c.uri, c.member_id
FROM concepts c, json_each(c.doc, '$.alt_labels') al
WHERE lower(json_extract(al.value, '$.value')) LIKE :q_norm || '%';
```

O conjunto de `uri` resultante é o conjunto de **conceitos-semente** — ponto de partida da Etapa 3. Cada semente carrega uma **pontuação de rótulo** (`label_score`): `1.0` se veio do casamento exato, `0.5` se veio só do casamento por prefixo — usada na Etapa 6. `q` sem nenhuma semente (nenhum conceito de nenhum membro usa esse rótulo) não é erro: o pipeline segue para a Etapa 4 só com o ramo de FTS.

### Etapa 3 — Expandir as sementes pela CTE recursiva

Aplica as regras de travessia da seção anterior sobre `concept_mappings`, partindo do conjunto de sementes da Etapa 2. A consulta completa, comentada, está na próxima seção (["CTE recursiva completa"](#cte-recursiva-completa)); a assinatura de entrada/saída é:

```sql
-- Entrada: conjunto de conceitos-semente (Etapa 2) + parâmetros
--   :max_narrow_depth = SKOS_EXPANSION_MAX_DEPTH (default 3)
--   :expand_broader    = 0 ou 1 (opt-in da requisição, default 0)
-- Saída: um URI de conceito por linha, com o predicado que o alcançou
--   (NULL para a própria semente) — ver bloco completo abaixo.
SELECT uri, predicate_used FROM expansao;
```

### Etapa 4 — Recuperar candidatos

União de dois ramos independentes, sem duplicar lógica de filtro entre eles: registros ligados a algum conceito do conjunto expandido (via `record_terms`), e registros que casam diretamente com `q` por busca textual — um registro pode ser candidato pelos dois caminhos ao mesmo tempo (ex.: o próprio registro do membro dono da semente).

```sql
-- Ramo A: candidatos via mapeamento semântico — o registro está ligado
-- (record_terms.concept_uri) a algum conceito da Etapa 3, incluindo a
-- própria semente (predicate_used IS NULL nesse caso).
SELECT DISTINCT
  rt.record_id,
  'concept' AS via,
  rt.concept_uri,
  ex.predicate_used
FROM record_terms rt
JOIN expansao ex ON ex.uri = rt.concept_uri
WHERE rt.term_type = 'concept_uri'

UNION

-- Ramo B: candidatos via busca textual direta sobre o texto achatado do
-- registro (records_fts, ver modelo-de-dados.md) — não depende de nenhum
-- mapeamento semântico existir.
SELECT
  r.id AS record_id,
  'fts' AS via,
  NULL AS concept_uri,
  NULL AS predicate_used
FROM records_fts f
JOIN records r ON r.federated_id = f.federated_id
WHERE records_fts MATCH :q_fts_escaped;
```

`:q_fts_escaped` é `q` com cada token envolto em aspas duplas (e aspas internas duplicadas), para que o texto do usuário nunca seja interpretado como sintaxe de consulta FTS5 (`NEAR`, `*`, `OR`, `:`) — a regra de escape completa está em [`governanca-e-seguranca.md`](governanca-e-seguranca.md), que trata isso como injeção real, não teórica.

### Etapa 5 — Aplicar filtros estruturais

Sobre o conjunto de candidatos da Etapa 4, usando as colunas geradas de `records` para os filtros de valor único e `record_terms` (via `LEFT JOIN`) para os multivalorados (`use_category`, [`modelo-de-dados.md`](modelo-de-dados.md)). Todo filtro é opcional — `NULL` no parâmetro significa "sem filtro nesse campo".

```sql
SELECT DISTINCT r.id
FROM records r
JOIN (
  -- resultado combinado da Etapa 4
  SELECT record_id FROM candidatos
) cand ON cand.record_id = r.id
LEFT JOIN record_terms ut
  ON ut.record_id = r.id AND ut.term_type = 'use_category'
WHERE (:member_id     IS NULL OR r.member_id       = :member_id)
  AND (:source_type   IS NULL OR r.source_type     = :source_type)
  AND (:country       IS NULL OR r.country         = :country)
  AND (:state         IS NULL OR r.state           = :state)
  AND (:region        IS NULL OR r.region          = :region)
  AND (:community     IS NULL OR r.community_name  = :community)
  AND (:species       IS NULL OR r.scientific_name = :species)
  AND (:use_category  IS NULL OR ut.term_value     = :use_category)
  AND (:updated_since IS NULL OR r.member_updated_at >= :updated_since);
```

### Etapa 6 — Ordenar

Dois esquemas de pontuação convivem no mesmo `ORDER BY`, escolhidos por linha conforme a origem do candidato (`via`, da Etapa 4). Quando um registro chega pelos dois ramos ao mesmo tempo (`fts` e `concept`), usa-se a maior das duas pontuações (`MAX`).

- **Via FTS** (`via = 'fts'`): `bm25(records_fts, 10.0, 1.0)` — o `10.0` pesa o campo `titulo` dez vezes mais que `corpo` ([`modelo-de-dados.md`](modelo-de-dados.md)).
- **Via expansão semântica** (`via = 'concept'`, nenhum casamento textual direto no próprio registro): `label_score` do melhor rótulo do conceito-semente que originou a cadeia (Etapa 2: `1.0` exato / `0.5` prefixo) multiplicado pelo **fator do predicado** que alcançou o conceito ligado ao registro (`predicate_used`, Etapa 3).

```sql
-- Fator por predicado — HEURÍSTICA AJUSTÁVEL, não invariante (ADR-004
-- local, "Mitigações"): recalibrar estes pesos não exige revisar o ADR.
--   skos:exactMatch  -> 1.0  (equivalência plena — mesma confiança do FTS)
--   skos:closeMatch  -> 0.7  (relação próxima, não plena)
--   skos:narrowMatch -> 0.5  (descida hierárquica — resultado mais amplo)
--   skos:broadMatch  -> 0.5  (subida hierárquica, só quando expand_broader=true)
SELECT
  cand.record_id,
  MAX(
    CASE
      WHEN cand.via = 'fts' THEN bm25(records_fts, 10.0, 1.0)
      WHEN cand.via = 'concept' THEN cand.label_score * (
        CASE cand.predicate_used
          WHEN 'skos:exactMatch'  THEN 1.0
          WHEN 'skos:closeMatch'  THEN 0.7
          WHEN 'skos:narrowMatch' THEN 0.5
          WHEN 'skos:broadMatch'  THEN 0.5
          ELSE 1.0 -- própria semente (predicate_used IS NULL): confiança plena
        END
      )
    END
  ) AS score
FROM candidatos_filtrados cand
JOIN records_fts f ON f.federated_id = (SELECT federated_id FROM records WHERE id = cand.record_id)
GROUP BY cand.record_id
ORDER BY score DESC, cand.record_id
LIMIT :limit OFFSET :offset;
```

`concepts` não tem tabela FTS5 própria — "o melhor rótulo do conceito" (`label_score`) vem inteiramente da Etapa 2, não de um `bm25` calculado sobre `concepts`. O nome "fator por predicado" é literal; a fórmula inteira (`label_score × fator`) é uma aproximação deliberadamente simples de relevância, não uma pontuação estatisticamente calibrada — mudá-la é uma decisão de produto, não uma migração de esquema.

---

## CTE recursiva completa

Bloco de referência da Etapa 3, com cada regra do [ADR-004 local](decisions/ADR-004-expansao-semantica-skos.md) comentada na perna de `UNION ALL` que a implementa. `:seeds` representa o conjunto de `uri` resolvido na Etapa 2 (na prática, uma tabela temporária ou um `VALUES (...)` com uma linha por semente).

```sql
-- Parâmetros:
--   :seeds             conjunto de URIs-semente (Etapa 2)
--   :max_narrow_depth  SKOS_EXPANSION_MAX_DEPTH (default 3) — teto de
--                       negócio para narrowMatch/broadMatch
--   :expand_broader     0 ou 1 — opt-in de broadMatch (default 0)
--   :max_technical_hops teto técnico de defesa para exactMatch (ex.: 20) —
--                       nunca alcançado em uso normal; existe só contra
--                       cadeia patologicamente longa, não é regra de negócio
--                       e não é o mesmo parâmetro que SKOS_EXPANSION_MAX_DEPTH

WITH RECURSIVE expansao(uri, predicate_used, narrow_depth, close_hops, path) AS (

  -- Caso base: cada conceito-semente, profundidade zero, nenhum predicado
  -- usado ainda (predicate_used IS NULL identifica "é a própria semente"
  -- em toda consulta a jusante, inclusive na Etapa 6).
  SELECT s.uri, NULL, 0, 0, '|' || s.uri || '|'
  FROM (SELECT uri FROM :seeds) s

  UNION ALL

  -- Perna 1 — skos:exactMatch: SIMÉTRICO e TRANSITIVO (ADR-004 local).
  -- Segue em ambas as direções da aresta (source->target ou target->source,
  -- o mapeamento não distingue lado), sem incrementar narrow_depth — o
  -- único guarda-corpo é o técnico (:max_technical_hops), não um teto de
  -- negócio. Só mapeamentos aprovados; ciclo cortado pelo `path`.
  SELECT
    CASE WHEN cm.source_uri = e.uri THEN cm.target_uri ELSE cm.source_uri END,
    'skos:exactMatch',
    e.narrow_depth,
    e.close_hops,
    e.path || (CASE WHEN cm.source_uri = e.uri THEN cm.target_uri ELSE cm.source_uri END) || '|'
  FROM expansao e
  JOIN concept_mappings cm
    ON cm.status = 'approved'
   AND cm.predicate = 'skos:exactMatch'
   AND (cm.source_uri = e.uri OR cm.target_uri = e.uri)
  WHERE (length(e.path) / 45) < :max_technical_hops -- estimativa grosseira de saltos via tamanho do path; implementação real usa uma coluna de contagem dedicada
    AND e.path NOT LIKE '%|' || (CASE WHEN cm.source_uri = e.uri THEN cm.target_uri ELSE cm.source_uri END) || '|%'

  UNION ALL

  -- Perna 2 — skos:closeMatch: SIMÉTRICO, NÃO-TRANSITIVO (ADR-004 local).
  -- Só parte de um nó no fechamento transitivo de exactMatch da semente —
  -- ou seja, e.predicate_used IS NULL (a própria semente) OU 'skos:exactMatch'.
  -- close_hops < 1 garante EXATAMENTE 1 salto por caminho. O nó de chegada
  -- carrega predicate_used='skos:closeMatch', que a Perna 3 e esta mesma
  -- perna (via o filtro acima) recusam como ponto de partida — por isso
  -- nenhuma perna encadeia a partir de um closeMatch.
  SELECT
    CASE WHEN cm.source_uri = e.uri THEN cm.target_uri ELSE cm.source_uri END,
    'skos:closeMatch',
    e.narrow_depth,
    e.close_hops + 1,
    e.path || (CASE WHEN cm.source_uri = e.uri THEN cm.target_uri ELSE cm.source_uri END) || '|'
  FROM expansao e
  JOIN concept_mappings cm
    ON cm.status = 'approved'
   AND cm.predicate = 'skos:closeMatch'
   AND (cm.source_uri = e.uri OR cm.target_uri = e.uri)
  WHERE e.close_hops < 1
    AND (e.predicate_used IS NULL OR e.predicate_used = 'skos:exactMatch')
    AND e.path NOT LIKE '%|' || (CASE WHEN cm.source_uri = e.uri THEN cm.target_uri ELSE cm.source_uri END) || '|%'

  UNION ALL

  -- Perna 3 — skos:narrowMatch: DIRECIONAL, descendente (source = mais
  -- amplo, target = mais específico; segue só source_uri = e.uri, nunca o
  -- inverso). Bounded por SKOS_EXPANSION_MAX_DEPTH (narrow_depth). Nunca a
  -- partir de um nó alcançado por closeMatch (mesma regra da Perna 2).
  SELECT
    cm.target_uri,
    'skos:narrowMatch',
    e.narrow_depth + 1,
    e.close_hops,
    e.path || cm.target_uri || '|'
  FROM expansao e
  JOIN concept_mappings cm
    ON cm.status = 'approved'
   AND cm.predicate = 'skos:narrowMatch'
   AND cm.source_uri = e.uri
  WHERE e.narrow_depth < :max_narrow_depth
    AND (e.predicate_used IS NULL OR e.predicate_used != 'skos:closeMatch')
    AND e.path NOT LIKE '%|' || cm.target_uri || '|%'

  UNION ALL

  -- Perna 4 — skos:broadMatch: DIRECIONAL, ascendente. DESLIGADA por
  -- padrão — só entra na consulta (via WHERE :expand_broader = 1) quando o
  -- cliente da API pede expand_broader=true. Mesma disciplina de
  -- profundidade e de não-encadeamento após closeMatch que narrowMatch.
  SELECT
    cm.target_uri,
    'skos:broadMatch',
    e.narrow_depth + 1,
    e.close_hops,
    e.path || cm.target_uri || '|'
  FROM expansao e
  JOIN concept_mappings cm
    ON cm.status = 'approved'
   AND cm.predicate = 'skos:broadMatch'
   AND cm.source_uri = e.uri
  WHERE :expand_broader = 1
    AND e.narrow_depth < :max_narrow_depth
    AND (e.predicate_used IS NULL OR e.predicate_used != 'skos:closeMatch')
    AND e.path NOT LIKE '%|' || cm.target_uri || '|%'
)
-- Um conceito pode ser alcançado por mais de um caminho (ex.: exactMatch
-- direto E via um narrowMatch mais longo); DISTINCT + a MAX() da Etapa 6
-- garante que o predicado de maior fator vence, nunca o de menor.
SELECT DISTINCT uri, predicate_used, narrow_depth FROM expansao;
```

A verificação de profundidade da Perna 1 (`length(e.path) / 45`) é uma aproximação ilustrativa neste documento de referência — a implementação real mantém uma coluna de contagem de saltos técnicos dedicada na CTE (mais barata e exata que medir o tamanho do `path`), mas o *comportamento* — um teto puramente defensivo, bem acima de qualquer cadeia real de `exactMatch` curada por um Comitê humano — é o que este ADR fixa, não a forma exata de calculá-lo.

---

## Exemplo ponta a ponta: buscar "mandioca"

Cenário (o mesmo do `README.md` do Pluriverso, agora executável): três membros da federação descreveram o mesmo táxon com três rótulos diferentes, e o Comitê Federado já aprovou os mapeamentos entre os conceitos correspondentes de cada BioCultTermos membro.

| Membro | `member_type` | O que registrou | Rótulo do conceito próprio |
|---|---|---|---|
| Iniciativa USEFLORA | `fontes_secundarias` | `vernacular_names: ["macaxeira"]` | `pref_label: "mandioca"`, `alt_label: "macaxeira"` |
| Instituto Brasileiro de Florística | `obras_naturalistas` | `scientific_name: "Manihot esculenta"` (nenhum nome vernacular) | `pref_label: "Manihot esculenta"` |
| Comunidade Baniwa do Rio Içana | `comunidade_tradicional` | `vernacular_names: ["Maniaka"]` | `pref_label: "Maniaka"` (língua Baniwa) |

Mapeamentos `approved` em `concept_mappings`:
- `concept/useflora/mandioca` **`skos:exactMatch`** `concept/florabrasil/manihot-esculenta` — mesmo táxon, nomenclatura científica vs. popular, equivalência plena.
- `concept/useflora/mandioca` **`skos:closeMatch`** `concept/baniwa/maniaka` — termo culturalmente próximo, mas o Comitê não considerou equivalência taxonômica estrita (pode abranger variedades específicas na tradição Baniwa), por isso `closeMatch`, não `exactMatch`.

Busca `GET /api/v1/search?q=mandioca`:

1. **Normalizar**: `q_norm = "mandioca"`.
2. **Sementes**: casamento exato de `pref_label` encontra `concept/useflora/mandioca` (`label_score = 1.0`). Nenhuma outra semente — o casamento exato já teve sucesso, o casamento por prefixo não roda.
3. **Expandir**: a partir da semente, `exactMatch` alcança `concept/florabrasil/manihot-esculenta`; `closeMatch` (1 salto a partir da própria semente) alcança `concept/baniwa/maniaka`. Conjunto expandido: `{useflora/mandioca (semente), florabrasil/manihot-esculenta (exactMatch), baniwa/maniaka (closeMatch)}`.
4. **Candidatos**: `record_terms.concept_uri` liga cada um dos três conceitos a exatamente um registro do respectivo membro. Nenhum dos três registros do Instituto Florística ou da Comunidade Baniwa contém a palavra "mandioca" no seu próprio texto — só o registro da USEFLORA seria achado por FTS puro; os outros dois só entram pelo Ramo A (concept) da Etapa 4.
5. **Filtros**: nenhum filtro estrutural na consulta — os três passam.
6. **Ordenar**: registro da USEFLORA (via seed, `predicate_used IS NULL` → fator `1.0`) e do Instituto Florística (via `exactMatch` → fator `1.0`) empatam na base do fator; o registro da Comunidade Baniwa (via `closeMatch` → fator `0.7`) fica atrás dos dois.

Resposta (envelope completo em [`api.md`](api.md); aqui só o array `data`):

```json
{
  "data": [
    {
      "federated_id": "iniciativa-useflora/record_004",
      "member_id": "iniciativa-useflora",
      "member_name": "Iniciativa USEFLORA",
      "member_type": "fontes_secundarias",
      "member_url_base": "https://useflora.example.org",
      "license": "CC BY-NC-SA 4.0",
      "attribution": "Iniciativa USEFLORA, 2025",
      "member_updated_at": "2026-05-10T00:00:00Z",
      "harvested_at": "2026-05-11T03:00:07Z",
      "profile": { "vernacular_names": ["macaxeira"] },
      "data": { "species": { "commonNames": [{ "name": "macaxeira" }] } }
    },
    {
      "federated_id": "instituto-floristica/record_211",
      "member_id": "instituto-floristica",
      "member_name": "Instituto Brasileiro de Florística",
      "member_type": "obras_naturalistas",
      "member_url_base": "https://florabrasil.example.org",
      "license": "CC BY 4.0",
      "attribution": "Instituto Brasileiro de Florística, Herbário Digital, 2023",
      "member_updated_at": "2026-04-02T00:00:00Z",
      "harvested_at": "2026-04-03T03:00:11Z",
      "profile": { "scientific_name": "Manihot esculenta" },
      "data": { "species": { "scientificName": "Manihot esculenta" } },
      "matched_via": [
        { "concept_uri": "https://florabrasil.example.org/termos/concept/manihot-esculenta", "predicate": "skos:exactMatch", "member_id": "instituto-floristica" }
      ]
    },
    {
      "federated_id": "comunidade-baniwa/record_078",
      "member_id": "comunidade-baniwa",
      "member_name": "Comunidade Baniwa do Rio Içana",
      "member_type": "comunidade_tradicional",
      "member_url_base": "https://etnotermos-baniwa.example.org",
      "license": null,
      "license_notice": "não declarada pelo membro; consultar https://etnotermos-baniwa.example.org",
      "attribution": "Comunidade Baniwa do Rio Içana, 2024",
      "member_updated_at": "2026-06-01T00:00:00Z",
      "harvested_at": "2026-06-02T03:00:12Z",
      "profile": { "vernacular_names": ["Maniaka"] },
      "data": { "species": { "commonNames": [{ "name": "Maniaka", "language": "bwi" }] } },
      "matched_via": [
        { "concept_uri": "https://etnotermos-baniwa.example.org/termos/concept/maniaka", "predicate": "skos:closeMatch", "member_id": "comunidade-baniwa" }
      ]
    }
  ]
}
```

Cada item exibe o **rótulo do próprio membro** ("macaxeira", "Manihot esculenta", "Maniaka") — o Pluriverso nunca substitui o vocabulário de um membro pelo termo que o usuário digitou; ele é "tradutor, não ditador taxonômico" (conforme a intenção de [ADR-004/D2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)). O primeiro item não carrega `matched_via` porque foi alcançado diretamente pelo conceito-semente (nenhuma aresta de `concept_mappings` foi percorrida para chegar até ele); os outros dois carregam `matched_via` com o predicado exato que a Etapa 3 usou — é esse campo que torna a expansão semântica auditável do lado do cliente da API, não só do lado do Comitê.

---

## Fluxo de curadoria de mapeamento

`concept_mappings` tem três estados possíveis para `status`: `proposed`, `approved`, `rejected`. A regra que este documento e o [ADR-004 local](decisions/ADR-004-expansao-semantica-skos.md) protegem é absoluta: **a CTE recursiva da Etapa 3 só percorre arestas `status = 'approved'`** — um mapeamento `proposed` existe na tabela, é visível na fila de curadoria do Comitê, mas é invisível para todo usuário fazendo busca.

- **Sugestão automática.** Quando um conceito novo entra no índice (`concepts`) — por harvest ([`contrato-harvest.md` §5](contrato-harvest.md)) ou por cadastro manual — o `MappingService` compara seus rótulos (`pref_labels`/`alt_labels`) com os de conceitos já indexados de **outros** membros por similaridade textual, e cada par acima de um limiar de confiança gera uma linha em `concept_mappings` com `status = 'proposed'`, `proposed_by = 'system'`, `proposed_at = <timestamp>`. Esse é o mecanismo que [ADR-004 (Arquitetura-BioCultural), seção "Mitigações"](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) descreve como "iniciar com mapeamentos automáticos por similaridade, validar manualmente". O algoritmo de similaridade e o limiar exato são parâmetro de ajuste do `MappingService`, não uma decisão que este documento fixa — assim como o fator de pontuação por predicado (Etapa 6), podem ser recalibrados sem mudar a semântica de travessia.
- **Nunca `approved` automaticamente.** Uma sugestão automática **nunca** nasce com `status = 'approved'` — essa transição é exclusivamente uma ação humana do Comitê Federado, feita pelo painel de curadoria (`PATCH /api/v1/mappings/{id}`, ver [`api.md`](api.md) §5.2 e o workflow completo em [`governanca-e-seguranca.md`](governanca-e-seguranca.md)).
- **Submissão manual do Comitê.** Um curador também pode propor um mapeamento diretamente (`POST /api/v1/mappings`), sem passar pela sugestão automática — nasce igualmente em `status = 'proposed'`, sujeito à mesma decisão de aprovação por outro membro do Comitê (`decided_by` é sempre distinto de `proposed_by` no fluxo de dupla checagem descrito em `governanca-e-seguranca.md`).
- **Efeito imediato da aprovação.** Assim que um mapeamento muda para `status = 'approved'`, a próxima busca que passe por aquele conceito já o percorre — não há cache de expansão a invalidar, porque a CTE roda contra `concept_mappings` a cada consulta.
- **Rejeição e remoção por purge.** Um mapeamento `rejected` permanece na tabela (auditável, nunca apagado por decisão de curadoria — só um `purge_by_member` remove fisicamente mapeamentos, e só os que envolvem o membro removido, [`modelo-de-dados.md`](modelo-de-dados.md) §4.2), mas é tão invisível para a busca quanto um `proposed`: a CTE só olha `status = 'approved'`.

---

## Referências
- [`docs/decisions/ADR-004-expansao-semantica-skos.md`](decisions/ADR-004-expansao-semantica-skos.md) (Pluriverso) — decisão e justificativa das regras de travessia
- [ADR-004 — Arquitetura Federada, D2 (Camada de Mapeamento Semântico) e "Mitigações"](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)
- [ADR-008 — Motor de Banco de Dados do Pluriverso, DB5 (CTE Recursiva)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)
- [SKOS Simple Knowledge Organization System — Reference / SKOS-XL Namespace Document (W3C)](https://www.w3.org/TR/skos-reference/skos-xl.html)
- [`docs/modelo-de-dados.md`](modelo-de-dados.md) (Pluriverso) — DDL de `concepts`, `concept_mappings`, `record_terms`, `records_fts`
- [`docs/api.md`](api.md) e [`docs/api/openapi.yaml`](api/openapi.yaml) (Pluriverso) — `GET /api/v1/search`, `expand`, `expand_broader`
- [`docs/governanca-e-seguranca.md`](governanca-e-seguranca.md) (Pluriverso) — workflow de aprovação de mapeamentos, escape de consulta FTS5
- [`docs/arquitetura.md`](arquitetura.md) (Pluriverso) — componentes `SearchService`, `SemanticExpander`, `MappingService`
