# Contrato de Harvest

Este documento é lido por dois públicos: pelos **membros da federação**, que precisam implementar e manter o endpoint de harvest para serem coletáveis pelo Pluriverso; e pelo **implementador do Pluriverso**, que precisa escrever o coletor (`HarvestClient` + `RecordIndexer` + `ConceptHarvester`, ver [`arquitetura.md`](arquitetura.md)) contra este contrato. A Arquitetura BioCultural fixa o formato do endpoint ([ADR-004/D6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)); este documento fixa tudo o que o ADR não cobre — comportamento do coletor, modos de coleta, resiliência, o perfil de campos extraído e o contrato (ainda não normativo) de harvest de conceitos.

---

## §1 — Contrato de registros (cliente)

Reproduzido verbatim de [ADR-004/D6 — Protocolo de Publicação: Harvest REST Paginado](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md), decisão que vincula todo membro da federação ([ADR-004/D1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) — harvest periódico via REST paginado, sem pull em tempo real, sem push do membro):

```
GET /api/federation/records
  ?page=<int>          # paginação, obrigatório
  &size=<int>          # registros por página (máx. 500)
  &updated_since=<ISO> # coleta incremental, opcional

Resposta:
{
  "member_id": "iniciativa-useflora",
  "total": 1523,
  "page": 1,
  "records": [
    {
      "id": "<member_id>/<record_id>",
      "visibility": "public",
      "updated_at": "2026-06-01T00:00:00Z",
      "data": { ... }  // campos definidos pelo Comitê Federado
    }
  ]
}
```

Cada membro **deve** implementar exatamente este contrato: `page` e `size` como parâmetros de query inteiros, `size` limitado a 500 (o membro pode aceitar valores maiores, mas o coletor nunca solicita mais que isso), `updated_since` opcional em ISO-8601, e o envelope `{member_id, total, page, records[]}`. `id` é o identificador federado estável do registro, no formato `<member_id>/<record_id>` — é essa string, não `record_id` isolado, que o Pluriverso usa como chave primária no índice.

Isso é o que o ADR especifica. O que segue é o comportamento **do lado do coletor** — decisões de implementação do Pluriverso, não obrigações adicionais para o membro.

### Comportamento do coletor

- **Ordem de páginas.** Sequencial: `page=1, 2, 3, …` até `page * size >= total`. `size` é sempre `HARVEST_PAGE_SIZE` (default `100` — ver [`operacao.md`](operacao.md)), nunca o máximo de 500; um valor menor mantém cada página rápida de processar e reduz o impacto de uma falha no meio da varredura.
- **`total` é dica, não invariante.** Um membro com bug pode reportar um `total` que não bate com a contagem real. O coletor **para** assim que uma página vier com `records: []`, mesmo que `page * size` ainda não tenha alcançado `total`. Confiar cegamente em `total` para decidir quando parar criaria risco de loop indefinido contra um membro que superestima o total.
- **Guarda-corpo de páginas.** No máximo `HARVEST_MAX_PAGES` (default `1000`) páginas por execução — ainda que `total` sugira mais. Exceder o limite não é erro fatal: a run é marcada `partial` e o motivo (`max_pages_exceeded`) é gravado em `harvest_runs.error_message`. Em modo completo, uma run `partial` **nunca** aciona a remoção de registros ausentes (ver §2).
- **Idempotência.** Upsert em `records` por `federated_id = {member_id}/{record_id}` (a própria string `id` do envelope). Reharvestar o mesmo registro sobrescreve `data` e `member_updated_at` — não há acumulação de histórico de versões no índice; a versão anterior simplesmente deixa de existir.
- **Filtragem de visibilidade.** Todo registro cujo `visibility` no envelope seja diferente de `"public"` é descartado antes de chegar ao índice, e a contagem é incrementada em `harvest_runs.rejected`. O contrato já promete que o endpoint só retorna `visibility: public` (ADR-004/D6), mas o coletor não confia nisso — filtrar de novo do lado do Pluriverso é barato e remove uma classe inteira de bug de implementação de membro.
- **Validação estrutural mínima.** Um registro sem `id`, sem `updated_at`, ou cujo `id` não corresponda ao padrão `^<member_id>/.+$` (isto é, que não comece pelo `member_id` da própria resposta seguido de `/`) é descartado e contado em `harvest_runs.rejected`. Um registro malformado **nunca** aborta a run inteira — o coletor segue para o próximo item da página e para a próxima página.

---

## §2 — Modos de coleta e detecção de remoção

O `README.md` do Pluriverso promete: *"detecta remoções (registro sumiu do endpoint → remove do índice)"*. Harvest incremental (`updated_since`) não detecta isso por si só — um registro removido do membro simplesmente para de aparecer nas respostas incrementais, sem sinal explícito de exclusão. Por isso o coletor opera em **dois modos**, com semânticas distintas de remoção.

### Incremental (default, diário)

Envia `updated_since=<member_updated_at máximo já indexado para aquele membro>`. Rápido — normalmente varre poucas páginas, só os registros tocados desde a última coleta. **Não detecta remoção.** É o modo padrão, agendado por `HARVEST_CRON_INCREMENTAL` (default `0 3 * * *`).

### Completo (semanal, e sempre na primeira coleta de um membro)

Omite `updated_since` — varre **todas** as páginas do membro, do zero. Acumula em memória o conjunto de `federated_id` vistos ao longo da varredura inteira. Ao final, **na mesma transação** que grava o último lote de upserts, o coletor remove do índice todo registro daquele `member_id` cujo `federated_id` não esteja no conjunto visto. Só o modo completo remove registros — o modo incremental nunca executa um `DELETE` em `records`. Agendado por `HARVEST_CRON_FULL` (default `0 4 * * 0`), e forçado na primeira coleta de um membro recém-aprovado (não há `member_updated_at` anterior para basear um incremental).

### Regra de segurança da remoção

Uma run completa que falhe no meio — erro HTTP não recuperado após retry, timeout persistente, guarda-corpo de páginas excedido — vira `partial` (ou `failed`, ver §3) e **nunca** executa a etapa de remoção. Uma varredura incompleta não é prova de que os registros ausentes foram removidos do membro; podem simplesmente não ter sido alcançados ainda. Remover com base num conjunto parcial apagaria dados válidos do índice. A remoção só roda depois que a run completa esgota todas as páginas do membro com sucesso.

### Configuração por membro

Defaults globais: `HARVEST_CRON_INCREMENTAL='0 3 * * *'`, `HARVEST_CRON_FULL='0 4 * * 0'` (ver [`operacao.md`](operacao.md)). Sobrescrevíveis por membro em `members.doc.harvest_cron_incremental` / `members.doc.harvest_cron_full` — um membro com volume muito grande ou taxa de atualização atipicamente alta pode receber um cron próprio sem alterar o comportamento dos demais.

---

## §3 — Resiliência

**Timeouts.** Timeout de conexão: 5 s. Timeout total por página (conexão + resposta completa): 10 s, controlado por `HARVEST_TIMEOUT_MS` (default `10000`).

**Retry por página.** Até 3 tentativas por página, com backoff exponencial 1 s / 4 s / 16 s mais jitter (evita que múltiplas páginas retentando simultaneamente sincronizem e sobrecarreguem o membro no mesmo instante). Falha nas 3 tentativas → a run inteira é marcada `failed`. O índice permanece **inalterado**: nada do que já foi upsertado nas páginas anteriores é revertido (não há necessidade — upsert é idempotente e válido), mas nenhuma remoção acontece, mesmo em modo completo. O próximo agendamento tenta de novo do zero.

**Desativação por falhas consecutivas.** Um membro que acumula `HARVEST_MAX_FAILURES` (default `5`) runs `failed` consecutivas tem `members.doc.harvest_enabled` automaticamente definido como `false`, e uma entrada é gravada em `audit_log` (`action: "harvest_disabled"`). O agendador para de tentar coletar aquele membro. Reativação (`harvest_enabled=true`) é uma ação humana do Comitê Federado, via `PATCH /api/federation/members/{member_id}` (ver [`api.md`](api.md)) — não há retomada automática, para evitar que o coletor martele indefinidamente um membro fora do ar.

**Concorrência.** No máximo 1 run por membro simultaneamente — lock por `member_id` mantido em memória pelo `HarvestScheduler`; um disparo manual (`POST /api/federation/members/{member_id}/harvest`) durante uma run agendada em curso para o mesmo membro é rejeitado, não enfileirado. No máximo `HARVEST_MAX_CONCURRENT` (default `3`) runs simultâneas no total, entre membros diferentes. Isso limita a carga de rede e, principalmente, respeita a arquitetura de escritor único do SQLite ([ADR-008](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)): cada página processada gera uma transação curta de escrita ao final do seu processamento, e transações curtas e frequentes de múltiplas runs concorrentes mantêm o `busy_timeout=5000` (ADR-008/DB2) como margem suficiente em vez de gargalo.

---

## §4 — Perfil Mínimo de Publicação

O Pluriverso extrai de `data` um conjunto fixo de campos — o **perfil** — para alimentar filtros estruturados, facetas e âncoras de mapeamento semântico. Os caminhos de origem em `data` seguem [ADR-003 — Modelo de Dados](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-003-data-model.md), cujo status é `Proposto` na Arquitetura BioCultural — por isso a tabela abaixo é citada como **perfil de referência**, não como obrigação contratual que o membro precise satisfazer para ser coletado.

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

### Política de campo ausente

A decisão está registrada formalmente em [`docs/decisions/ADR-005-perfil-minimo-de-publicacao.md`](decisions/ADR-005-perfil-minimo-de-publicacao.md) (a análise de opções não é repetida aqui — só a regra operacional):

Extração **best-effort**. Campo ausente em `data` ⇒ o registro **é indexado do mesmo jeito**; apenas o filtro/faceta correspondente àquele campo específico não responde por ele. O coletor **nunca** rejeita um registro por campo do perfil faltando — rejeitar equivaleria a o Pluriverso impor um esquema a um membro soberano, contrariando [ADR-004/D2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) e o princípio C.A.R.E. *Authority to Control*.

O `data` original é armazenado **íntegro**, sem perda, em paralelo ao `profile` extraído — os dois convivem no mesmo registro em `records`. O `profile` é derivado e descartável: pode ser reconstruído por reindexação a partir de `data`, sem precisar de um novo harvest contra o membro, caso o perfil de campos extraídos seja ampliado no futuro.

Um registro sem **nenhum** dos campos do perfil ainda entra no índice normalmente e continua alcançável por busca textual (FTS) sobre o `data` serializado — ver [`busca-semantica.md`](busca-semantica.md), §pipeline de busca, etapa 4.

---

## §5 — Contrato de conceitos

A tabela "Necessidades de Implementação por Componente" de [ADR-004](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) exige que o BioCultTermos de cada membro "publique `ConceptScheme` via endpoint para harvest pelo Pluriverso" — mas nenhum ADR aceito especifica o formato desse endpoint. A especificação abaixo é a materialização mínima dessa exigência, no mesmo formato do contrato de registros (§1 / ADR-004/D6):

```
GET /api/federation/concepts?page=<int>&size=<int>&updated_since=<ISO>

Resposta:
{
  "member_id": "<estável, igual ao de /records>",
  "total": 412,
  "page": 1,
  "size": 100,
  "concepts": [
    {
      "uri": "https://<url-base>/termos/concept/0192f...",
      "scheme_uri": "https://<url-base>/termos/scheme/etnobotanica",
      "pref_labels": [{ "value": "mandioca", "language": "pt-BR" }],
      "alt_labels":  [{ "value": "macaxeira", "language": "pt-BR" }],
      "broader":  ["https://<url-base>/termos/concept/0192a..."],
      "narrower": [],
      "updated_at": "2026-07-01T00:00:00Z"
    }
  ]
}
```

`uri` e `scheme_uri` são as URIs absolutas do conceito e do `ConceptScheme` a que ele pertence, publicadas pela instância soberana de BioCultTermos do membro. `pref_labels`/`alt_labels` seguem o modelo de rótulo multilíngue do SKOS-XL (`{value, language}`). `broader`/`narrower` são arrays de URIs de conceitos relacionados hierarquicamente, já publicados pelo mesmo membro. `page`/`size`/`updated_since` seguem exatamente a mesma semântica de paginação do endpoint de registros (§1).

> **Nota — endpoint não normativo.** Este endpoint **ainda não é normativo** na federação. Ele é a materialização mínima do que a tabela "Necessidades de Implementação" de ADR-004 pede, e deve ser levado ao Comitê Federado para ratificação formal antes de se tornar obrigatório para membros. Até lá, a ausência deste endpoint num membro não é uma falha de conformidade — significa apenas que os conceitos daquele membro entram no índice de conceitos por **cadastro manual do curador** (`POST /api/v1/concepts`, ver [`governanca-e-seguranca.md`](governanca-e-seguranca.md) e [`api.md`](api.md) §5.2), em vez de harvest automático.

O harvest de conceitos, executado pelo `ConceptHarvester` (ver [`arquitetura.md`](arquitetura.md)), é **opcional e independente** do harvest de registros: um membro sem este endpoint, ou uma falha ao coletá-lo, nunca marca a run de registros correspondente como `failed` ou `partial` — são duas runs distintas, com seu próprio ciclo de sucesso/falha. A ausência ou instabilidade do harvest de conceitos de um membro não impede o índice central de continuar recebendo e servindo seus registros normalmente.
