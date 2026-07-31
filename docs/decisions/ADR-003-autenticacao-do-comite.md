# ADR-003: Autenticação do Comitê

## Status
Aceito

## Contexto
[ADR-006/E4](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md) delega explicitamente ao Pluriverso a escolha do mecanismo de autenticação de quem decide um pedido de adesão (`PATCH /api/federation/membership-requests/{id}`), registrando-a como **"decisão de implementação... bloqueador de produção"**: a fila de inscrição não pode ser aberta em produção com esse endpoint público e sem autenticação. A mesma lacuna se estende a todos os demais endpoints de governança do Comitê Federado — re-probe de pedido, `purge_by_member`, ativação/edição de membro, disparo manual de harvest, decisão de mapeamento semântico, cadastro manual de conceito e leitura do `audit_log` (todos listados em `docs/api.md` §5.2) — nenhum deles pode ficar público sem identificar quem age.

O requisito não é só "bloquear acesso não autorizado": o **Modelo de Dados Mínimo** de [ADR-006](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md) exige o campo `decided_by` ("identificador do membro do Comitê que decidiu") em `membership_requests`, e `docs/modelo-de-dados.md` exige o campo equivalente `actor` em `audit_log`. Qualquer mecanismo que não identifique uma pessoa individual do Comitê torna esses dois campos uma mentira gravada no banco. A governança do Comitê Federado em si — quem tem assento, como delibera — é [ADR-004/D3](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md); esta ADR decide apenas **como o software reconhece tecnicamente** uma pessoa já legitimada por aquele processo.

## Requisitos
### Funcionais
- Autenticar toda requisição aos endpoints do Comitê Federado (`docs/api.md` §5.2), rejeitando credencial ausente ou inválida com `401` e `error.code: UNAUTHORIZED` ([ADR-002 local](ADR-002-api-publica-rest.md)).
- Identificar de forma única e estável qual pessoa do Comitê autenticou cada requisição, para popular `decided_by` (em `membership_requests` e `concept_mappings`) e `actor` (em `audit_log`) na mesma transação da mudança.
- Permitir revogar ou rotacionar a credencial de uma pessoa do Comitê sem afetar as demais contas.
- Funcionar sem nenhum serviço de autenticação externo — auto-hospedagem trivial, inclusive para associações de recursos limitados que rodem sua própria instância ([ADR-009/MI2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md)).

### Não-Funcionais
- Nenhuma dependência nova de runtime: sem servidor OIDC, sem tabela de sessão, sem emissor de token.
- Verificação de credencial resistente a força bruta offline (hash lento, não reversível).
- Dimensionado para o volume esperado do Comitê Federado — poucas pessoas por instância (dezenas no limite superior, tipicamente ~5–20 no lançamento), não milhares de usuários.
- Sem estado de sessão no servidor: cada requisição se autentica sozinha, coerente com a API REST-only e sem sessão de [ADR-002 local](ADR-002-api-publica-rest.md).

## Opções Consideradas

### Opção 1: HTTP Basic Auth + bcrypt, contas nomeadas (escolhida)
**Prós:**
- `decided_by`/`actor` sempre correspondem a uma pessoa real — é requisito funcional de ADR-006, não estética de login.
- Zero infraestrutura nova: nenhum IdP, nenhuma tabela de sessão, nenhum emissor/verificador de token.
- Mesmo padrão já em produção no BioCultTermos (variável `ADMIN_USERS`) — reuso de convenção validada na federação, mesmo racional de [ADR-001 local](ADR-001-stack-e-framework.md).
- Stateless: a credencial viaja no header `Authorization` a cada requisição, sem sessão a gerenciar — combina com a natureza REST-only da API.
- Revogar ou trocar uma conta é editar a variável de ambiente `COMMITTEE_USERS` e reiniciar o container — sem migração de banco, sem tabela de usuários.

**Contras:**
- Sem MFA.
- Sem UI de autogestão de senha — trocar a senha exige gerar um novo hash bcrypt offline e reimplantar a variável.
- Escala mal além de ~20 contas: todas vivem numa única variável de ambiente JSON, sem ferramenta de gestão.

### Opção 2: Token único compartilhado entre o Comitê (rejeitada)
**Prós:**
- Trivial de implementar — uma comparação de string.

**Contras:**
- Destrói `decided_by`: toda decisão fica atribuída ao mesmo token, sem identificar quem agiu — e ADR-006 exige explicitamente o identificador da pessoa do Comitê que decidiu.
- Revogar o acesso de uma pessoa revoga o acesso de todo o Comitê simultaneamente, obrigando redistribuir um novo token a todos.
- Nenhuma auditoria individual possível em `audit_log.actor` — o campo existiria só para conter sempre o mesmo valor.

### Opção 3: JWT + login com refresh token (rejeitada)
**Prós:**
- Modelo padrão para aplicações com muitos usuários finais, com expiração e renovação bem definidas.

**Contras:**
- Exige emissor de token, endpoint de login, armazenamento de refresh token e rotina de revogação — complexidade desproporcional para ~5 pessoas por instância.
- Já rejeitado para a API pública em [ADR-002 local](ADR-002-api-publica-rest.md) (Opção 3) pelo mesmo motivo: nenhum ator do Pluriverso precisa de sessão renovável; introduzir o mecanismo só para o Comitê duplicaria essa decisão sem um requisito novo que a justifique.
- Adiciona superfície de ataque (segredo de assinatura, algoritmo, expiração, revogação de refresh token) sem ganho de segurança sobre Basic Auth + HTTPS neste volume de usuários.

### Opção 4: OIDC/SSO (rejeitada)
**Prós:**
- Delegação de identidade a um provedor especializado, MFA nativo, gestão de conta centralizada.

**Contras:**
- Dependência de um provedor de identidade externo (ou a operação de um IdP próprio) — contradiz a auto-hospedagem trivial exigida por [ADR-009/MI2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md) justamente para associações de recursos limitados, o público-alvo típico de uma instância adicional do Pluriverso.
- Complexidade de configuração (client id/secret, redirect URIs, descoberta de metadados) desproporcional a um Comitê de poucas pessoas.
- Ponto de falha externo: a indisponibilidade do provedor de identidade bloquearia toda decisão de governança do Pluriverso, inclusive em instâncias pequenas sem equipe de TI dedicada.

## Decisão
Adotar a **Opção 1**: HTTP Basic Auth sobre HTTPS, com **uma conta nomeada por pessoa do Comitê Federado**, senha armazenada como hash bcrypt (custo 12), num único middleware aplicado a todos os endpoints de `docs/api.md` §5.2.

**Formato da credencial.** Variável de ambiente `COMMITTEE_USERS` (documentada em `docs/operacao.md`), um array JSON de objetos `{ "username": string, "passwordHash": string }` — mesmo padrão já em produção no BioCultTermos (`ADMIN_USERS`):

```
COMMITTEE_USERS=[{"username":"comite-useflora","passwordHash":"$2b$12$SUBSTITUIR_HASH"}]
```

**Verificação.** Middleware único (`requireCommittee`), executado antes de qualquer handler das rotas do Comitê: lê o header `Authorization: Basic <base64(username:password)>`; ausente ou malformado → `401` + `error.code: UNAUTHORIZED`; decodifica e procura `username` em `COMMITTEE_USERS`; ausente **ou** `bcrypt.compare(password, passwordHash)` falso → `401` + `UNAUTHORIZED` — a mesma resposta nos dois casos, para não revelar se um `username` existe. Sucesso → segue a requisição com `req.committeeUser = username`, valor usado para popular `decided_by` (em `membership_requests` e `concept_mappings`) e `actor` (em `audit_log`) na mesma transação da mudança.

**HTTPS é pré-requisito não negociável.** Basic Auth transmite a credencial em Base64 — reversível, não criptografado — e só é seguro sob TLS. Toda instância expõe `/api/federation/*` (rotas de governança) e `/api/v1/mappings`/`/api/v1/concepts` (rotas autenticadas) exclusivamente atrás de HTTPS, com terminação TLS no proxy reverso da implantação (fora do container — `docs/operacao.md`); o header HSTS aplicado via `helmet` (`docs/governanca-e-seguranca.md`) reforça isso no navegador do Comitê.

**Geração de hash.** Hashes bcrypt custo 12 são gerados **offline**, fora de qualquer rota da aplicação em runtime — nenhum endpoint cria ou expõe hash. Um hash real **nunca** é commitado no repositório (`docs/principios.md` §Versionamento); o exemplo acima usa deliberadamente o placeholder `$2b$12$SUBSTITUIR_HASH`.

**Sem sessão.** Cada requisição se autentica de forma independente — nenhum cookie, nenhum estado de servidor associado à credencial, coerente com a natureza stateless da API REST-only ([ADR-002 local](ADR-002-api-publica-rest.md)).

**Sem papéis dentro do Comitê.** Qualquer conta em `COMMITTEE_USERS` tem acesso igual a todos os endpoints de `docs/api.md` §5.2 — reflete [ADR-004/D3](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md) ("decisões... tomadas por consenso ou maioria qualificada"), que não distingue papéis técnicos dentro do Comitê Federado.

## Consequências
### Positivas
- `decided_by` e `actor` são sempre atribuíveis a uma pessoa real e identificável — a auditoria de governança prometida pelo `README.md` ("o processo é auditável") fica tecnicamente sustentada, não apenas declarada.
- Nenhuma infraestrutura nova: nenhum IdP, nenhum serviço de sessão, nenhuma tabela de usuários no índice — apenas uma variável de ambiente e uma dependência já presente na stack (`bcrypt`, [ADR-001 local](ADR-001-stack-e-framework.md)).
- Revogação e rotação de credencial são operações de implantação (editar `COMMITTEE_USERS`, reiniciar o container), sem deploy de código nem migração de banco.
- Compatível com a auto-hospedagem trivial exigida para qualquer instância do Pluriverso, inclusive as de associações com recursos limitados ([ADR-009/MI2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md)).

### Negativas
- Sem MFA: uma senha vazada é suficiente para agir como aquela pessoa do Comitê.
- Sem autogestão: trocar a própria senha não é uma ação que a pessoa do Comitê faz sozinha pela UI — exige coordenação com quem opera a implantação para gerar o novo hash e reimplantar a variável.
- `COMMITTEE_USERS` como única fonte de verdade das contas escala mal além de ~20 pessoas — não há paginação, busca ou UI de gestão sobre esse conjunto.

### Mitigações
- **Caminho de upgrade explícito:** se o Comitê Federado de uma instância ultrapassar ~20 pessoas, ou exigir MFA por política própria, migrar para OIDC. A fronteira de autenticação foi isolada num único middleware (`requireCommittee`) exatamente para tornar essa troca local — nenhum outro documento (`docs/api.md`, `docs/governanca-e-seguranca.md`, `docs/modelo-de-dados.md`) depende do mecanismo concreto, apenas do resultado (`req.committeeUser` identificando uma pessoa).
- A ausência de MFA é parcialmente compensada por HTTPS obrigatório, rate limiting nas rotas públicas de submissão (`docs/governanca-e-seguranca.md`) e pelo fato de bcrypt custo 12 tornar inviável força bruta offline sobre um hash vazado.
- A falta de autogestão é aceitável no volume esperado (poucas pessoas, baixa frequência de troca de senha); se isso se tornar fricção operacional relatada pelo Comitê, é sinal concreto de que a instância já cruzou o limiar de migração para OIDC acima.

## Referências
- [ADR-006: Protocolo de Inscrição na Federação, E4 (Autenticação da Decisão do Comitê)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)
- [ADR-006: Modelo de Dados Mínimo (`decided_by`)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-006-federation-membership-protocol.md)
- [ADR-004: Arquitetura Federada, D3 (Governança do Pluriverso: Comitê Federado)](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-004-federated-architecture.md)
- [ADR-009: Topologia Multi-Instância do Pluriverso, MI2](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/architecture-decisions/ADR-009-pluriverso-multi-instance-topology.md)
- `docs/decisions/ADR-001-stack-e-framework.md` (Pluriverso) — dependência `bcrypt` já fixada na stack
- `docs/decisions/ADR-002-api-publica-rest.md` (Pluriverso) — natureza REST-only e stateless da API, código `UNAUTHORIZED`
- `docs/api.md` §5.2 (Pluriverso) — lista completa dos endpoints protegidos por este mecanismo
- `docs/governanca-e-seguranca.md` (Pluriverso) — HTTPS/HSTS, rate limiting e demais controles complementares
- `docs/operacao.md` (Pluriverso) — variável de ambiente `COMMITTEE_USERS`
- BioCultTermos (repositório da federação) — origem do padrão `ADMIN_USERS` replicado aqui como `COMMITTEE_USERS`

## Data de Revisão
Revisitar quando o Comitê Federado de alguma instância ultrapassar ~20 pessoas ou exigir MFA por política própria — condição de gatilho para a migração a OIDC registrada em Mitigações. Até lá, sem prazo fixo de revisão.
