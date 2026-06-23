# Pluriverso

Middleware de federação para o ecossistema de Conhecimento Tradicional Associado à Biodiversidade (CTA).

[![GitHub](https://img.shields.io/badge/GitHub-pluriverso-181717?logo=github)](https://github.com/edalcin/pluriverso)

---

## O que é o Pluriverso?

O **Pluriverso** é o middleware de federação da [EtnoArquitetura](https://github.com/edalcin/etnoArquitetura). Ele permite que iniciativas e comunidades tradicionais completamente independentes — cada uma com sua própria infraestrutura soberana de dados — sejam acessíveis de forma integrada por pesquisadores e aplicações.

O nome reflete o conceito filosófico e político do "pluriverso": não um universo único e centralizado, mas a coexistência de múltiplos mundos autônomos que se relacionam sem se subordinar.

---

## Posição na Arquitetura Federada

```mermaid
graph TD
    subgraph I1["Iniciativa de Fontes Secundárias"]
        I1A(etnoDB) --> I1M[(MongoDB)]
        I1B(etnopapers) --> I1A
        I1C(etnoTermos) <--> I1M
    end

    subgraph C2["Comunidade Tradicional #2"]
        C2A(etnoRelatos) --> C2M[(MongoDB)]
        C2B(etnoTermos) <--> C2M
    end

    subgraph C3["Comunidade Tradicional #N"]
        C3A(etnoRelatos) --> C3M[(MongoDB)]
        C3B(etnoTermos) <--> C3M
    end

    PL{{"Pluriverso\nMiddleware de Federação"}}
    U((Usuário /\nAplicação))

    I1 -->|harvest REST| PL
    C2 -->|harvest REST| PL
    C3 -->|harvest REST| PL
    U <-->|API| PL
```

Cada membro da federação (iniciativa ou comunidade) opera de forma completamente independente e soberana. O Pluriverso **não** gerencia os dados dos membros — ele indexa apenas o que cada membro decide tornar público.

---

## Responsabilidades

### 1. Harvest Periódico

Coleta registros públicos de cada membro via endpoint REST paginado. Cada membro expõe:

```
GET /api/federation/records?page=1&size=100&updated_since=<ISO>
```

O Pluriverso agenda coletas periódicas, mantém um índice central dos registros `visibility: public`, e detecta remoções (registro sumiu do endpoint → remove do índice).

### 2. Índice Central

Armazena e indexa os registros coletados para busca eficiente. O índice é uma **cópia derivada** dos dados públicos dos membros — a fonte de verdade permanece sempre no membro.

### 3. Camada de Mapeamento Semântico

Mantém mapeamentos SKOS entre os vocabulários (etnoTermos) dos diferentes membros:

- `skos:exactMatch` — conceitos idênticos em membros diferentes
- `skos:closeMatch` — conceitos muito similares
- `skos:broadMatch` / `skos:narrowMatch` — conceitos em relação hierárquica

Esses mapeamentos permitem que uma busca por "mandioca" retorne resultados de membros que usam "cassava", "Manihot esculenta", "macaxeira" ou termos em línguas indígenas — desde que o curador da federação tenha mapeado os conceitos.

### 4. API Pública Unificada

Expõe uma API única para usuários e aplicações acessarem o conjunto federado de CTAs, com:

- Busca textual e semântica (via mapeamentos SKOS)
- Filtros por membro, tipo de fonte, comunidade, espécie, região
- Atribuição clara da origem de cada registro (member_id)
- Respeito às licenças e restrições definidas por cada membro

### 5. Interface de Governança

Suporta o **Comitê Federado** — composto por representantes de cada membro — nas decisões sobre:

- Admissão e remoção de membros
- Contrato de publicação (campos obrigatórios do endpoint)
- Aprovação de mapeamentos semânticos
- Resolução de conflitos

---

## Princípios de Design

### Soberania dos Membros

O Pluriverso **nunca** acessa dados de um membro além do que o membro publica explicitamente. Não há backdoor, não há acesso direto ao MongoDB de ninguém.

### Remoção Imediata

Quando um membro sai da federação, todos os seus dados são removidos do índice central imediatamente (`purge_by_member`). Mapeamentos SKOS envolvendo seus conceitos também são removidos. O processo é auditável.

### Transparência de Origem

Cada registro no índice carrega `member_id` permanente. O Pluriverso nunca "apaga" a procedência de um dado.

### CARE na Prática

| Princípio | Implementação no Pluriverso |
|-----------|---------------------------|
| **Collective Benefit** | Acesso integrado beneficia pesquisadores e comunidades de todos os membros |
| **Authority to Control** | Membro decide o que publica; pode sair e remover tudo a qualquer momento |
| **Responsibility** | Auditoria de harvest; logs de remoção; mapeamentos semânticos revisados pelo comitê |
| **Ethics** | Atribuição de origem obrigatória; respeito a licenças por membro |

---

## Necessidades de Implementação (v3.0)

O Pluriverso é um **novo componente**, ainda sem implementação. As principais funcionalidades a desenvolver:

- [ ] Harvest scheduler: coleta periódica configurável por membro
- [ ] Parser do endpoint de harvest: consumir e normalizar respostas dos membros
- [ ] Índice central: armazenamento e busca dos registros coletados
- [ ] Camada de mapeamento SKOS: CRUD de mapeamentos entre ConceptSchemes
- [ ] Motor de busca semântica: busca expandida por mapeamentos SKOS
- [ ] API pública REST: endpoint de consulta federada
- [ ] `purge_by_member`: remoção completa de um membro do índice
- [ ] Interface de governança: painel para o Comitê Federado

---

## Relação com os Demais Componentes

| Componente | Relação com o Pluriverso |
|------------|--------------------------|
| **[etnoDB](https://github.com/edalcin/etnoDB)** | Membro da federação; expõe endpoint de harvest com registros secundários aprovados |
| **[etnopapers](https://github.com/edalcin/etnopapers)** | Alimenta o etnoDB; sem relação direta com o Pluriverso |
| **[etnoRelatos](https://github.com/edalcin/etnoRelatos)** | Membro da federação (por comunidade); expõe endpoint de harvest com registros primários consentidos |
| **[etnoTermos](https://github.com/edalcin/etnotermos)** | Cada instância é soberana; Pluriverso mantém mapeamentos entre instâncias de diferentes membros |
| **[etnoArquitetura](https://github.com/edalcin/etnoArquitetura)** | Repositório de arquitetura; documenta o Pluriverso e a federação como um todo |

---

## Documentação da Arquitetura

A arquitetura completa, incluindo diagramas C4, ADRs e decisões de design, está documentada em:

**[etnoArquitetura](https://github.com/edalcin/etnoArquitetura)** — especialmente:
- [ADR-004: Arquitetura Federada v3.0](https://github.com/edalcin/etnoArquitetura/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md)

---

## Licença

A definir — considerando licenças que respeitem os princípios C.A.R.E. e protejam adequadamente o conhecimento tradicional.

## Contato

[GitHub Issues](https://github.com/edalcin/pluriverso/issues)
