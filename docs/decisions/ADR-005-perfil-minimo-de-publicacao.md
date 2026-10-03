# ADR-005: Perfil Mínimo de Publicação

## Status
Aceito

## Contexto

O Pluriverso extrai de `data` — o payload íntegro que cada membro publica em `GET /api/federation/records` ([ADR-004/D6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)) — um conjunto fixo de 13 campos, o **Perfil Mínimo de Publicação**, para alimentar filtros estruturados, facetas e âncoras de mapeamento semântico na busca federada. A tabela completa de campos, seus caminhos de origem em `data` (segundo [ADR-003 — Modelo de Dados](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-003-data-model.md), status `Proposto`) e seu uso no Pluriverso já está especificada em [`docs/contrato-harvest.md` §4](../contrato-harvest.md) — este ADR não a repete, só decide **formalmente** a política que rege campo ausente, porque essa é a decisão do conjunto mais contestável: ela define o que o Pluriverso espera de cada membro atual e futuro, e um museu, uma comunidade tradicional e uma iniciativa de fontes secundárias não têm o mesmo formato de dado.

Nenhum ADR aceito da Arquitetura-BioCultural resolve essa questão diretamente — [ADR-004/D2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) fixa que a camada de mapeamento semântico vive no Pluriverso e que os `ConceptScheme` mapeados pertencem a membros soberanos, mas não trata de campos estruturados de `data`. É, portanto, uma decisão de implementação do Pluriverso, registrada aqui para não ficar implícita no código do `RecordIndexer`.

## Requisitos

### Funcionais
- Alimentar os filtros estruturados prometidos pelo `README.md` §4 ("Filtros por membro, tipo de fonte, comunidade, espécie, região") a partir de `data`.
- Aceitar registros de membros com formatos de `data` heterogêneos — um acervo museológico, uma comunidade tradicional e uma iniciativa de fontes secundárias não compartilham o mesmo esquema de payload.
- Preservar o payload original de cada registro sem perda, independentemente de quanto do perfil foi extraído com sucesso.

### Não-Funcionais
- Não impor esquema a nenhum membro soberano — coerente com [ADR-004/D2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) e com o princípio C.A.R.E. *Authority to Control* (o membro decide o que e como publica).
- Permitir ampliar o perfil no futuro sem exigir um novo harvest contra os membros já coletados.

## Opções Consideradas

### Opção 1: Extração best-effort, registro sempre indexado (escolhida)

O `RecordIndexer` tenta extrair cada um dos 13 campos do perfil pelo caminho JSON declarado em `docs/contrato-harvest.md` §4. Um campo ausente em `data` não impede a indexação do registro — apenas o filtro ou a faceta correspondente àquele campo específico não responde por ele. Nenhum registro é rejeitado por campo do perfil faltando.

**Prós:**
- Nenhum membro soberano é obrigado a reestruturar seu `data` para ser coletável — um acervo museológico legitimamente não tem `species.taxonomy`, uma comunidade tradicional pode não reportar `source.type`.
- Coerente com [ADR-004/D2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) e o princípio C.A.R.E. *Authority to Control*: o Pluriverso adapta-se ao dado que o membro publica, nunca o contrário.
- Todo registro, mesmo sem nenhum campo do perfil, continua alcançável por busca textual (FTS5) sobre o `data` serializado.

**Contras:**
- Filtros e facetas ficam inconsistentes entre membros — um membro que não popula `location.region` nunca aparece no filtro `region`, mesmo tendo registros geograficamente localizáveis em outro formato.
- Exige que a UI trate ausência de faceta como estado normal, não como erro de dado.

### Opção 2: Exigir os campos do perfil e rejeitar registro incompleto (rejeitada)

O `RecordIndexer` validaria a presença dos 13 campos (ou de um subconjunto obrigatório) antes de indexar; registros sem eles seriam descartados, contados em `harvest_runs.rejected`.

**Prós:**
- Todo registro indexado responde a todos os filtros — nenhuma inconsistência de faceta entre membros.
- Simplifica a lógica de exibição na UI: nunca há filtro "vazio" para um registro presente.

**Contras:**
- O Pluriverso passaria a **impor um esquema** a um membro soberano — direta e explicitamente contra [ADR-004/D2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) e o princípio C.A.R.E. *Authority to Control*: a federação existe para agregar dados heterogêneos sem subordinar quem os produz a um formato central.
- Um acervo histórico/museológico legitimamente não tem `species.taxonomy` nem `community.name` — aplicar essa opção excluiria dessa federação exatamente o tipo de membro (`acervos_historicos`) que ela foi desenhada para incluir.
- Rejeitar registro é uma penalidade desproporcional a um campo de faceta ausente — o registro em si pode ser perfeitamente válido e público.

### Opção 3: Nenhum perfil, só FTS sobre o `data` serializado (rejeitada)

Abandonar a extração de perfil inteiramente; toda busca seria full-text sobre o `data` bruto achatado, sem colunas estruturadas.

**Prós:**
- Elimina toda a complexidade de mapeamento de campo por `member_type`/formato — um único caminho de indexação (FTS) para todos os membros.
- Zero acoplamento entre o esquema do Pluriverso e o formato de `data` de qualquer membro.

**Contras:**
- Destrói os filtros por membro, tipo de fonte, comunidade, espécie e região prometidos pelo `README.md` §4 — a busca federada regride a um grep sobre texto serializado, sem faceta estruturada alguma.
- Nenhuma âncora estruturada para a junção com mapeamentos SKOS via `record_terms.concept_uri` — a busca semântica (`docs/busca-semantica.md`) perde seu principal ponto de entrada estruturado.

## Decisão

Adotar a **Opção 1**: extração **best-effort** dos 13 campos do Perfil Mínimo de Publicação, listados em [`docs/contrato-harvest.md` §4](../contrato-harvest.md) (não repetidos aqui — a tabela de campo/caminho/uso é normativa naquele documento). Um registro é **sempre** indexado, independentemente de quantos campos do perfil `RecordIndexer` conseguiu extrair de `data`; campo ausente apenas desativa o filtro/faceta correspondente para aquele registro específico. O `data` original é armazenado íntegro, em paralelo ao `profile` extraído.

Tabela de referência (reproduzida de `docs/contrato-harvest.md` §4, para consulta rápida — a fonte normativa permanece aquele documento):

| Campo do perfil | Caminho em `data` (ADR-003) | Uso no Pluriverso |
|---|---|---|
| `scientific_name` | `species.scientificName` | filtro `species`, faceta, FTS |
| `vernacular_names[]` | `species.commonNames[].name` | FTS, ancoragem SKOS |
| `family` | `species.taxonomy.family` | faceta |
| `use_categories[]` | `uses[].category` | filtro `use_category` |
| `community_name` | `community.name` | filtro `community` |
| `ethnicity` | `community.ethnicity` | faceta |
| `region` | `location.region` | filtro `region` |
| `state` | `location.state` | faceta |
| `country` | `location.country` | faceta |
| `source_type` | `source.type` (`primary`\|`secondary`) | filtro `source_type` |
| `license` | `permissions.license` | atribuição obrigatória na resposta |
| `attribution` | `permissions.attribution` | atribuição obrigatória na resposta |
| `concept_uris[]` | `extensions.bioculttermos.conceptUris[]` | junção com mapeamentos SKOS |

## Consequências

### Positivas
- Todo membro soberano é coletável independentemente do formato interno de `data` — nenhum `member_type` (`fontes_secundarias`, `comunidade_tradicional`, `acervos_historicos`, `obras_naturalistas`) é estruturalmente excluído por não popular um campo do perfil.
- O `profile` é **derivado e descartável**: reconstruível por reindexação a partir do `data` íntegro já armazenado, sem exigir um novo harvest contra o membro. Ampliar o perfil no futuro (adicionar um 14º campo, por exemplo) é uma migração local sobre dado já coletado, não uma nova rodada de coleta.
- Um registro sem nenhum campo do perfil continua alcançável por busca textual sobre `data` serializado ([`docs/modelo-de-dados.md`](../modelo-de-dados.md), `records_fts`) — nunca fica invisível à busca.

### Negativas
- Filtros e facetas são inconsistentes entre membros: um usuário pode não conseguir filtrar por `region` registros de um membro que nunca populou `location.region`, mesmo que esses registros sejam geograficamente localizáveis em outro campo do `data` bruto.
- A UI precisa tratar ausência de faceta como estado normal — um filtro "vazio" para um dado membro não é um bug a ser corrigido, é o reflexo de uma decisão soberana daquele membro.

### Mitigações
- A documentação pública do Pluriverso (`docs/api.md`) deixa explícito, no schema de resposta, que `profile` é best-effort e pode ter campos ausentes — nenhuma integração externa deve assumir que todo registro popula todos os 13 campos.
- Se a federação identificar um subconjunto de campos que, na prática, todo `member_type` consegue popular sem esforço, esse subconjunto pode ser promovido a uma recomendação (não obrigação) no fluxo de onboarding de novo membro (`docs/governanca-e-seguranca.md`), sem reabrir esta decisão de política de campo ausente.

## Referências
- [`docs/contrato-harvest.md` §4 — Perfil Mínimo de Publicação](../contrato-harvest.md) — tabela normativa de campo/caminho/uso e a política operacional de campo ausente.
- [ADR-004 — Arquitetura Federada, D2 (Camada de Mapeamento Semântico)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)
- [ADR-003 — Modelo de Dados (Arquitetura-BioCultural, status `Proposto`)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-003-data-model.md) — origem dos caminhos JSON usados como perfil de referência.
- `README.md` §4 — promessa de filtros por membro, tipo de fonte, comunidade, espécie, região, que motiva a existência do perfil.
- Princípios C.A.R.E. — *Authority to Control*.

## Data de Revisão
Revisitar se [ADR-003 — Modelo de Dados](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-003-data-model.md) da Arquitetura-BioCultural mudar de status (`Proposto` → `Aceito` ou reescrita) com caminhos de campo distintos dos usados aqui, ou se o Comitê Federado propuser ampliar o conjunto de 13 campos do perfil. Até lá, sem prazo fixo de revisão.
