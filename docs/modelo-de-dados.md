# Modelo de Dados

Este documento especifica o esquema completo do índice SQLite do Pluriverso: as nove tabelas, suas colunas geradas, índices, invariantes, a estratégia de migração e o procedimento transacional de remoção de membro (`purge_by_member`). É referência normativa para a implementação de `HarvestClient`, `RecordIndexer`, `ConceptHarvester`, `SearchService`, `MembershipService`, `MappingService`, `PurgeService` e `AuditService` (ver [`arquitetura.md`](arquitetura.md)).

Convenções globais: ids internos são **UUIDv7** (ordenáveis por tempo de criação, sem coordenação central); `created_at`/`updated_at` são strings **ISO-8601 UTC** (`2026-07-31T00:00:00Z`). Todas as tabelas de domínio seguem o padrão de tabela único-documento-JSON de [ADR-005/DA2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-005-sqlite-json-persistence.md), com exceção declarada e justificada de `record_terms` (§4).

## Pragmas de conexão

Executados na abertura de toda conexão com o arquivo SQLite, conforme [ADR-005/DA1](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-005-sqlite-json-persistence.md):

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
```

`journal_mode = WAL` permite leitores concorrentes durante escrita; `foreign_keys = ON` é obrigatório para a cascata de `record_terms` (§4) e para os `REFERENCES` funcionarem; `busy_timeout = 5000` absorve contenção de escrita entre o `HarvestScheduler` e requisições da API sem erro imediato de `SQLITE_BUSY`.

---

## 1. `members`

**Propósito:** membros ativos da federação — a fonte de verdade sobre quem é coletado.

`id` é o `member_id`: UUIDv7 gerado no momento da aprovação do pedido de adesão, e **nunca reciclado**, mesmo após `purge_by_member` ([ADR-006/E5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-006-federation-membership-protocol.md)) — ver §"Invariantes" abaixo e a seção `purge_by_member` mais adiante.

```sql
CREATE TABLE members (
  id         TEXT PRIMARY KEY,
  doc        TEXT NOT NULL CHECK (json_valid(doc)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE INDEX idx_members_member_type     ON members (member_type);
CREATE INDEX idx_members_harvest_enabled ON members (harvest_enabled);
```

Colunas geradas:

```sql
ALTER TABLE members ADD COLUMN member_type     TEXT    GENERATED ALWAYS AS (json_extract(doc, '$.member_type')) VIRTUAL;
ALTER TABLE members ADD COLUMN harvest_enabled INTEGER GENERATED ALWAYS AS (json_extract(doc, '$.harvest_enabled')) VIRTUAL;
```

Chaves de `doc`:

| Chave | Tipo | Descrição |
|---|---|---|
| `member_name` | string | nome de exibição do membro |
| `member_type` | string (enum) | `fontes_secundarias` \| `comunidade_tradicional` \| `acervos_historicos` \| `obras_naturalistas` |
| `url_base` | string (URL) | raiz do endpoint federado do membro |
| `contact_email` | string | único dado pessoal armazenado (ver [`governanca-e-seguranca.md`](governanca-e-seguranca.md), LGPD) |
| `care_declaration` | string | declaração C.A.R.E. submetida na adesão |
| `harvest_enabled` | boolean | `false` após 5 falhas consecutivas ([`contrato-harvest.md`](contrato-harvest.md) §3) |
| `harvest_cron_incremental` | string (cron) | sobrescreve `HARVEST_CRON_INCREMENTAL` por membro |
| `harvest_cron_full` | string (cron) | sobrescreve `HARVEST_CRON_FULL` por membro |
| `joined_at` | string (ISO-8601) | data de aprovação |
| `consecutive_failures` | integer | contador de `harvest_runs.status='failed'` seguidas; zera em qualquer sucesso |

**Invariantes:** `id` é imutável e gerado uma única vez, na transição `membership_requests.status → 'active'` (§2); nunca reatribuído a outro pedido, mesmo que o membro original seja removido por `purge_by_member`. `member_type` restrito aos quatro literais acima — validado na aplicação antes do `INSERT`/`UPDATE`, não por `CHECK` em coluna gerada (SQLite não permite `CHECK` referenciando coluna `VIRTUAL` de outra linha, mas o valor pode ser validado com `CHECK (json_extract(doc,'$.member_type') IN (...))` diretamente na definição de `doc` — recomendado).

---

## 2. `membership_requests`

**Propósito:** fila de pedidos de adesão à federação, do cadastro self-service até a decisão do Comitê ([ADR-006/E1–E3](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-006-federation-membership-protocol.md)).

```sql
CREATE TABLE membership_requests (
  id         TEXT PRIMARY KEY,
  doc        TEXT NOT NULL CHECK (json_valid(doc)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE INDEX idx_membership_requests_status      ON membership_requests (status);
CREATE INDEX idx_membership_requests_member_type ON membership_requests (member_type);
CREATE INDEX idx_membership_requests_member_id   ON membership_requests (member_id);
```

Colunas geradas:

```sql
ALTER TABLE membership_requests ADD COLUMN status      TEXT GENERATED ALWAYS AS (json_extract(doc, '$.status'))      VIRTUAL;
ALTER TABLE membership_requests ADD COLUMN member_type TEXT GENERATED ALWAYS AS (json_extract(doc, '$.member_type')) VIRTUAL;
ALTER TABLE membership_requests ADD COLUMN member_id   TEXT GENERATED ALWAYS AS (json_extract(doc, '$.member_id'))   VIRTUAL;
```

Chaves de `doc` — nomes **exatamente** os de ADR-006:

| Chave | Tipo | Descrição |
|---|---|---|
| `member_name` | string | nome proposto |
| `member_type` | string (enum) | os quatro literais de §1 |
| `url_base` | string (URL) | endpoint a ser coletado |
| `contact_email` | string | contato do solicitante |
| `care_declaration` | string | declaração C.A.R.E. |
| `status` | string (enum) | `pending` \| `active` \| `rejected` |
| `technical_check` | objeto (JSON aninhado) | resultado do probe anti-SSRF: `{ ok, checked_at, checks, failure_reason, http_status, elapsed_ms }` — ver [`governanca-e-seguranca.md`](governanca-e-seguranca.md) |
| `member_id` | string (UUIDv7) \| null | preenchido só na aprovação |
| `decided_at` | string (ISO-8601) \| null | timestamp da decisão do Comitê |
| `decided_by` | string \| null | usuário do Comitê que decidiu (Basic Auth, `COMMITTEE_USERS`) |
| `rejection_reason` | string \| null | motivo textual obrigatório em rejeição |

**Invariantes:**
- `rejection_reason` **não-nulo** quando `status = 'rejected'` — validado na aplicação antes do `UPDATE` que decide o pedido (`PATCH /api/federation/membership-requests/{id}`).
- `member_id` **só é não-nulo** quando `status = 'active'` — é o `MembershipService` que gera o UUIDv7 e grava `member_id` atomicamente com a transição de `status`, na mesma transação em que insere a linha correspondente em `members`.
- Reenvio de um pedido `rejected` volta o `status` para `pending` ([ADR-006/E2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-006-federation-membership-protocol.md)) — só decisão humana do Comitê move o estado, nunca o próprio solicitante.

---

## 3. `records`

**Propósito:** índice central de registros federados — a tabela consultada por toda busca pública.

```sql
CREATE TABLE records (
  id         TEXT PRIMARY KEY,
  doc        TEXT NOT NULL CHECK (json_valid(doc)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE UNIQUE INDEX idx_records_federated_id     ON records (federated_id);
CREATE INDEX        idx_records_member_id        ON records (member_id);
CREATE INDEX        idx_records_member_updated_at ON records (member_updated_at);
CREATE INDEX        idx_records_source_type      ON records (source_type);
CREATE INDEX        idx_records_country          ON records (country);
CREATE INDEX        idx_records_state            ON records (state);
CREATE INDEX        idx_records_region           ON records (region);
CREATE INDEX        idx_records_community_name   ON records (community_name);
CREATE INDEX        idx_records_scientific_name  ON records (scientific_name);
CREATE INDEX        idx_records_family           ON records (family);
CREATE INDEX        idx_records_member_incremental ON records (member_id, member_updated_at);
```

Colunas geradas:

```sql
ALTER TABLE records ADD COLUMN federated_id      TEXT GENERATED ALWAYS AS (json_extract(doc, '$.federated_id'))        VIRTUAL;
ALTER TABLE records ADD COLUMN member_id         TEXT GENERATED ALWAYS AS (json_extract(doc, '$.member_id'))           VIRTUAL;
ALTER TABLE records ADD COLUMN member_updated_at TEXT GENERATED ALWAYS AS (json_extract(doc, '$.member_updated_at'))  VIRTUAL;
ALTER TABLE records ADD COLUMN source_type       TEXT GENERATED ALWAYS AS (json_extract(doc, '$.profile.source_type'))      VIRTUAL;
ALTER TABLE records ADD COLUMN country           TEXT GENERATED ALWAYS AS (json_extract(doc, '$.profile.country'))          VIRTUAL;
ALTER TABLE records ADD COLUMN state             TEXT GENERATED ALWAYS AS (json_extract(doc, '$.profile.state'))            VIRTUAL;
ALTER TABLE records ADD COLUMN region            TEXT GENERATED ALWAYS AS (json_extract(doc, '$.profile.region'))           VIRTUAL;
ALTER TABLE records ADD COLUMN community_name    TEXT GENERATED ALWAYS AS (json_extract(doc, '$.profile.community_name'))   VIRTUAL;
ALTER TABLE records ADD COLUMN scientific_name   TEXT GENERATED ALWAYS AS (json_extract(doc, '$.profile.scientific_name'))  VIRTUAL;
ALTER TABLE records ADD COLUMN family            TEXT GENERATED ALWAYS AS (json_extract(doc, '$.profile.family'))          VIRTUAL;
```

Chaves de `doc`:

| Chave | Tipo | Descrição |
|---|---|---|
| `federated_id` | string | `{member_id}/{record_id}`, chave federada estável ([ADR-004/D6](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md)) |
| `member_id` | string (UUIDv7) | referência lógica a `members.id` (sem `FOREIGN KEY` declarada — `doc` é JSON; a integridade é garantida por `PurgeService`, não pelo motor) |
| `record_id` | string | id local do registro no membro |
| `visibility` | string | sempre `"public"` neste índice — filtrado no harvest ([`contrato-harvest.md`](contrato-harvest.md) §1) |
| `member_updated_at` | string (ISO-8601) | `updated_at` reportado pelo membro; base do próximo `updated_since` incremental |
| `harvested_at` | string (ISO-8601) | timestamp local da última coleta |
| `harvest_run_id` | string (UUIDv7) | referência lógica à `harvest_runs.id` que produziu esta versão |
| `data` | objeto | payload íntegro do membro, armazenado sem perda |
| `profile` | objeto | Perfil Mínimo de Publicação, 13 campos, extração best-effort — ver [`contrato-harvest.md`](contrato-harvest.md) §4 |

**Invariantes:** `federated_id` é `UNIQUE` — todo upsert de harvest é `INSERT ... ON CONFLICT (federated_id) DO UPDATE`. O índice composto `(member_id, member_updated_at)` existe especificamente para a query que o `HarvestClient` roda antes de cada run incremental: `SELECT MAX(member_updated_at) FROM records WHERE member_id = ?`, que vira o `updated_since` da próxima página 1.

---

## 4. `record_terms`

**Propósito:** uma linha por valor multivalorado de um registro (nome vernacular, categoria de uso, URI de conceito) — o padrão de coluna gerada de `records` não indexa arrays, então valores repetíveis precisam de tabela própria para serem filtráveis e junction-friendly.

**Exceção declarada ao padrão DA2:** esta é uma tabela de junção puramente relacional — não armazena um "documento" com identidade própria, só a relação `(registro, tipo, valor)` — por isso usa colunas reais em vez de `doc` JSON; forçar JSON aqui só adicionaria uma camada de `json_extract` sobre dados que já nascem tabulares.

```sql
CREATE TABLE record_terms (
  id          TEXT PRIMARY KEY,
  record_id   TEXT NOT NULL REFERENCES records (id) ON DELETE CASCADE,
  term_type   TEXT NOT NULL CHECK (term_type IN ('vernacular_name', 'scientific_name', 'use_category', 'concept_uri')),
  term_value  TEXT NOT NULL,
  concept_uri TEXT
);

CREATE INDEX idx_record_terms_type_value  ON record_terms (term_type, term_value);
CREATE INDEX idx_record_terms_concept_uri ON record_terms (concept_uri);
CREATE INDEX idx_record_terms_record_id   ON record_terms (record_id);
```

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | TEXT (UUIDv7) | chave primária própria |
| `record_id` | TEXT | FK para `records.id`, `ON DELETE CASCADE` — depende de `PRAGMA foreign_keys = ON` |
| `term_type` | TEXT (enum) | `vernacular_name` \| `scientific_name` \| `use_category` \| `concept_uri` |
| `term_value` | TEXT | o valor em si (nome, categoria, ou a própria URI quando `term_type='concept_uri'`) |
| `concept_uri` | TEXT \| null | URI SKOS associada ao termo, quando aplicável (usada pela expansão semântica em `SemanticExpander`, ver [`busca-semantica.md`](busca-semantica.md)) |

**Invariante:** o `ON DELETE CASCADE` é o mecanismo pelo qual `DELETE FROM records WHERE member_id = ?` (passo 2 de `purge_by_member`, ver adiante) limpa `record_terms` sem instrução adicional.

---

## 5. `records_fts`

**Propósito:** busca textual full-text sobre o conteúdo de cada registro, com título (nomes) pesando mais que corpo.

```sql
CREATE VIRTUAL TABLE records_fts USING fts5(
  federated_id UNINDEXED,
  titulo,
  corpo,
  tokenize = 'unicode61 remove_diacritics 2'
);
```

**Autônoma, não external-content:** `records_fts` não usa `content='records'` porque `titulo`/`corpo` são texto **derivado** de JSON aninhado achatado — não correspondem a nenhuma coluna direta de `records` que o mecanismo external-content pudesse espelhar automaticamente; a sincronização é feita explicitamente pelo `RecordIndexer`, na mesma transação SQLite do upsert em `records` ([ADR-008/DB4](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md)).

**Composição de `titulo`:** `profile.scientific_name` concatenado com todos os `record_terms.term_value` onde `term_type = 'vernacular_name'` para aquele `record_id`, separados por espaço único.

**Composição de `corpo` — algoritmo de achatamento de `data`:**

1. Percorrer `data` recursivamente (é um valor JSON arbitrário — objeto, array, ou escalar).
2. Se o valor corrente é um **objeto**: recursar em cada valor de propriedade, na ordem em que aparecem no JSON; **ignorar as chaves** (nomes de propriedade não entram no texto).
3. Se o valor corrente é um **array**: recursar em cada elemento, na ordem.
4. Se o valor corrente é uma **string**: emitir a string como folha.
5. Se o valor corrente é **número, booleano ou `null`**: ignorar — não entra no corpo indexado.
6. Concatenar todas as folhas emitidas, na ordem de visita, separadas por um único espaço.

Pseudocódigo de referência:

```
function achatar(valor):
  if tipo(valor) == objeto:
    for cada v em valores(valor):     # ordem de inserção do JSON, chaves descartadas
      achatar(v)
  elif tipo(valor) == array:
    for cada item em valor:
      achatar(item)
  elif tipo(valor) == string:
    emitir(valor)
  # número, booleano, null: nenhuma ação

corpo = join(" ", folhas_emitidas)
```

**Ranking:** `bm25(records_fts, 10.0, 1.0)` — pesos posicionais na ordem das colunas indexadas (`titulo`, `corpo`); título pesa 10× o corpo. Usado por `SearchService` (ver [`busca-semantica.md`](busca-semantica.md), pipeline de busca).

**Sincronização:** toda escrita em `records` (upsert de harvest) e toda remoção (modo completo, ou `purge_by_member`) executa a operação correspondente em `records_fts` (`INSERT`/`DELETE` por `federated_id`) dentro da **mesma transação** — nunca em transação separada, para não haver janela onde o índice FTS e `records` divergem.

---

## 6. `concepts`

**Propósito:** conceitos SKOS publicados por membros, via harvest opcional (`docs/contrato-harvest.md` §5) ou cadastro manual do curador.

```sql
CREATE TABLE concepts (
  id         TEXT PRIMARY KEY,
  doc        TEXT NOT NULL CHECK (json_valid(doc)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE UNIQUE INDEX idx_concepts_uri       ON concepts (uri);
CREATE INDEX        idx_concepts_member_id ON concepts (member_id);
```

Colunas geradas:

```sql
ALTER TABLE concepts ADD COLUMN uri       TEXT GENERATED ALWAYS AS (json_extract(doc, '$.uri'))       VIRTUAL;
ALTER TABLE concepts ADD COLUMN member_id TEXT GENERATED ALWAYS AS (json_extract(doc, '$.member_id')) VIRTUAL;
```

Chaves de `doc`:

| Chave | Tipo | Descrição |
|---|---|---|
| `uri` | string (URL) | URI absoluta do conceito, publicada pelo BioCultTermos do membro |
| `member_id` | string (UUIDv7) | membro proprietário do conceito |
| `scheme_uri` | string (URL) | URI do `ConceptScheme` a que pertence |
| `pref_labels[]` | array de `{value, language}` | rótulos preferenciais (SKOS-XL) |
| `alt_labels[]` | array de `{value, language}` | rótulos alternativos |
| `broader[]` | array de string (URI) | conceitos mais amplos |
| `narrower[]` | array de string (URI) | conceitos mais específicos |
| `origin` | string (enum) | `harvest` \| `manual` |
| `updated_at` | string (ISO-8601) | timestamp de atualização no membro (ou de cadastro manual) |

**Invariante:** `uri` é `UNIQUE` — um conceito é identificado globalmente pela própria URI, que já é absoluta e inclui `url_base` do membro publicador.

---

## 7. `concept_mappings`

**Propósito:** relações SKOS aprovadas (ou propostas) entre conceitos de membros distintos — o insumo de `SemanticExpander`.

```sql
CREATE TABLE concept_mappings (
  id         TEXT PRIMARY KEY,
  doc        TEXT NOT NULL CHECK (json_valid(doc)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE UNIQUE INDEX idx_concept_mappings_triple    ON concept_mappings (source_uri, target_uri, predicate);
CREATE INDEX        idx_concept_mappings_source_uri ON concept_mappings (source_uri);
CREATE INDEX        idx_concept_mappings_target_uri ON concept_mappings (target_uri);
CREATE INDEX        idx_concept_mappings_status      ON concept_mappings (status);
CREATE INDEX        idx_concept_mappings_source_member ON concept_mappings (source_member_id);
CREATE INDEX        idx_concept_mappings_target_member ON concept_mappings (target_member_id);
CREATE INDEX        idx_concept_mappings_predicate     ON concept_mappings (predicate);
```

Colunas geradas:

```sql
ALTER TABLE concept_mappings ADD COLUMN source_uri       TEXT GENERATED ALWAYS AS (json_extract(doc, '$.source_uri'))        VIRTUAL;
ALTER TABLE concept_mappings ADD COLUMN target_uri       TEXT GENERATED ALWAYS AS (json_extract(doc, '$.target_uri'))        VIRTUAL;
ALTER TABLE concept_mappings ADD COLUMN status           TEXT GENERATED ALWAYS AS (json_extract(doc, '$.status'))            VIRTUAL;
ALTER TABLE concept_mappings ADD COLUMN source_member_id TEXT GENERATED ALWAYS AS (json_extract(doc, '$.source_member_id')) VIRTUAL;
ALTER TABLE concept_mappings ADD COLUMN target_member_id TEXT GENERATED ALWAYS AS (json_extract(doc, '$.target_member_id')) VIRTUAL;
ALTER TABLE concept_mappings ADD COLUMN predicate        TEXT GENERATED ALWAYS AS (json_extract(doc, '$.predicate'))         VIRTUAL;
```

Chaves de `doc`:

| Chave | Tipo | Descrição |
|---|---|---|
| `source_uri` | string (URI) | conceito de origem |
| `source_member_id` | string (UUIDv7) | membro do conceito de origem |
| `target_uri` | string (URI) | conceito de destino |
| `target_member_id` | string (UUIDv7) | membro do conceito de destino |
| `predicate` | string (enum) | `skos:exactMatch` \| `skos:closeMatch` \| `skos:broadMatch` \| `skos:narrowMatch` — verbatim, [ADR-008/DB5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-008-pluriverso-database-engine.md) |
| `status` | string (enum) | `proposed` \| `approved` \| `rejected` |
| `proposed_by` | string | quem propôs (curador ou sugestão automática — ver [`busca-semantica.md`](busca-semantica.md)) |
| `proposed_at` | string (ISO-8601) | |
| `decided_by` | string \| null | usuário do Comitê que decidiu |
| `decided_at` | string (ISO-8601) \| null | |
| `note` | string \| null | justificativa da decisão |

**Invariantes:**
- `source_member_id != target_member_id` — mapeamento é sempre **entre** membros distintos (`README.md` §3); validado na aplicação antes do `INSERT`.
- Par `(source_uri, target_uri, predicate)` é único — `UNIQUE INDEX idx_concept_mappings_triple`.
- Só `status = 'approved'` participa da expansão de busca ([ADR-004/D2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md), "aprovados pelo Comitê e nunca impostos") — `SemanticExpander` sempre filtra `WHERE status = 'approved'` na CTE recursiva.

---

## 8. `harvest_runs`

**Propósito:** auditoria de cada execução de coleta, incremental ou completa, por membro.

```sql
CREATE TABLE harvest_runs (
  id         TEXT PRIMARY KEY,
  doc        TEXT NOT NULL CHECK (json_valid(doc)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE INDEX idx_harvest_runs_member_id  ON harvest_runs (member_id);
CREATE INDEX idx_harvest_runs_status     ON harvest_runs (status);
CREATE INDEX idx_harvest_runs_started_at ON harvest_runs (started_at);
```

Colunas geradas:

```sql
ALTER TABLE harvest_runs ADD COLUMN member_id  TEXT GENERATED ALWAYS AS (json_extract(doc, '$.member_id'))  VIRTUAL;
ALTER TABLE harvest_runs ADD COLUMN status     TEXT GENERATED ALWAYS AS (json_extract(doc, '$.status'))     VIRTUAL;
ALTER TABLE harvest_runs ADD COLUMN started_at TEXT GENERATED ALWAYS AS (json_extract(doc, '$.started_at')) VIRTUAL;
```

Chaves de `doc`:

| Chave | Tipo | Descrição |
|---|---|---|
| `member_id` | string (UUIDv7) | membro coletado |
| `mode` | string (enum) | `incremental` \| `full` |
| `status` | string (enum) | `running` \| `ok` \| `partial` \| `failed` |
| `started_at` | string (ISO-8601) | |
| `finished_at` | string (ISO-8601) \| null | `null` enquanto `status='running'` |
| `pages_fetched` | integer | |
| `upserted` | integer | registros inseridos/atualizados |
| `removed` | integer | registros removidos (só possível em `mode='full'`) |
| `rejected` | integer | registros descartados por filtragem de visibilidade ou validação estrutural ([`contrato-harvest.md`](contrato-harvest.md) §1) |
| `http_status_last` | integer \| null | último código HTTP recebido do membro |
| `error_message` | string \| null | motivo de `partial`/`failed` (ex.: `max_pages_exceeded`) |

**Invariante:** `removed > 0` só é válido quando `mode = 'full'` **e** `status = 'ok'` — consequência direta da regra de segurança de remoção ([`contrato-harvest.md`](contrato-harvest.md) §2: uma run `partial` ou `failed` nunca remove).

---

## 9. `audit_log`

**Propósito:** trilha de auditoria de toda ação de governança — a única tabela cujo conteúdo sobrevive à saída de um membro da federação.

```sql
CREATE TABLE audit_log (
  id         TEXT PRIMARY KEY,
  doc        TEXT NOT NULL CHECK (json_valid(doc)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE INDEX idx_audit_log_action    ON audit_log (action);
CREATE INDEX idx_audit_log_target_id ON audit_log (target_id);
CREATE INDEX idx_audit_log_at        ON audit_log (at);
```

Colunas geradas:

```sql
ALTER TABLE audit_log ADD COLUMN action    TEXT GENERATED ALWAYS AS (json_extract(doc, '$.action'))    VIRTUAL;
ALTER TABLE audit_log ADD COLUMN target_id TEXT GENERATED ALWAYS AS (json_extract(doc, '$.target_id')) VIRTUAL;
ALTER TABLE audit_log ADD COLUMN at        TEXT GENERATED ALWAYS AS (json_extract(doc, '$.at'))        VIRTUAL;
```

Chaves de `doc`:

| Chave | Tipo | Descrição |
|---|---|---|
| `action` | string (enum) | `membership_approved` \| `membership_rejected` \| `member_purged` \| `harvest_disabled` \| `harvest_reenabled` \| `mapping_approved` \| `mapping_rejected` \| `mapping_removed_by_purge` \| `concept_registered` |
| `actor` | string | usuário do Comitê, ou `"system"` para ações automáticas (ex.: `harvest_disabled`) |
| `target_type` | string | tipo da entidade afetada (`member`, `membership_request`, `concept_mapping`, ...) |
| `target_id` | string | id da entidade afetada |
| `reason` | string \| null | motivo textual, quando aplicável |
| `before` | objeto \| null | estado anterior relevante (ex.: contagens em `member_purged`) |
| `after` | objeto \| null | estado posterior relevante |
| `at` | string (ISO-8601) | timestamp da ação |

**Invariante — append-only:** `audit_log` **nunca** recebe `UPDATE` nem `DELETE`, sob nenhuma circunstância, inclusive durante `purge_by_member` (§ seguinte) — é o único registro que precisa sobreviver à remoção completa dos dados de um membro, por isso a aplicação não deve expor nenhum caminho de escrita além de `INSERT`.

---

## Migrações

Controle de versão de esquema via tabela dedicada:

```sql
CREATE TABLE schema_migrations (
  version    INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL
);
```

Arquivos de migração residem em `migrations/NNN-descricao.sql` (numeração sequencial de três dígitos, ex.: `001-schema-inicial.sql`, `002-add-harvest-runs.sql`), cada um contendo DDL puro. Na inicialização do processo, a aplicação lê `schema_migrations` para descobrir a versão corrente, e aplica em ordem crescente todo arquivo com `version` maior, cada um dentro de **uma transação própria** que termina com `INSERT INTO schema_migrations (version, applied_at) VALUES (?, ?)`. Sem ORM e sem ferramenta de migração externa (Knex, Flyway, etc.) — o volume de tabelas (nove) e a ausência de múltiplos ambientes divergentes não justificam a dependência.

Esta seção **especifica** o mecanismo; não cria os arquivos `.sql` de migração — esses são artefato de implementação, fora do escopo desta documentação.

---

## `purge_by_member(member_id)`

Implementa [ADR-004/D4](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-004-federated-architecture.md) — remoção imediata e completa de um membro, executada pelo `PurgeService`. Sequência **exata**, executada como **uma única transação SQLite**:

1. `DELETE FROM records_fts WHERE federated_id IN (SELECT federated_id FROM records WHERE member_id = ?)`
2. `DELETE FROM records WHERE member_id = ?` — a cascata (`ON DELETE CASCADE`) limpa `record_terms` automaticamente.
3. `DELETE FROM concept_mappings WHERE source_member_id = ? OR target_member_id = ?`
4. `DELETE FROM concepts WHERE member_id = ?`
5. `DELETE FROM harvest_runs WHERE member_id = ?`
6. `DELETE FROM members WHERE id = ?`
7. `UPDATE membership_requests SET doc = json_set(doc, '$.status', 'rejected', '$.rejection_reason', 'Membro removido da federação (purge)') WHERE member_id = ?` — o pedido original volta a não-`active`; **`member_id` permanece gravado** no `doc` do pedido, para nunca ser reciclado ([ADR-006/E5](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-006-federation-membership-protocol.md)).
8. `INSERT INTO audit_log` com `action = 'member_purged'`, `target_type = 'member'`, `target_id = member_id`, e `before` contendo as contagens de cada `DELETE` (registros, mapeamentos, conceitos, runs) coletadas antes da execução dos passos 1–5.
9. Um `INSERT INTO audit_log` por mapeamento removido no passo 3, com `action = 'mapping_removed_by_purge'`, `target_type = 'concept_mapping'`, `target_id` = id do mapeamento.

Após a transação (fora dela — `VACUUM` não pode rodar dentro de uma transação): `VACUUM;` — devolve ao sistema de arquivos o espaço físico liberado pelos `DELETE`s. A remoção precisa ser física, não apenas lógica: sem `VACUUM`, o arquivo `.sqlite` mantém as páginas livres internamente sem reduzir de tamanho, e páginas livres do WAL podem preservar bytes do conteúdo apagado até serem sobrescritas.

`audit_log` é o **único** vestígio que sobrevive ao purge — e mesmo esse vestígio guarda apenas contagens agregadas e o `member_id`, nunca conteúdo de registro, rótulo de conceito, ou qualquer outro dado do membro removido. Essa é a forma de atender a auditabilidade exigida pela governança federada sem reter dado do membro que saiu da federação.
