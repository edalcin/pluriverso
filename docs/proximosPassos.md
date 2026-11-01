# Próximos Passos — Pluriverso

**Estado em:** 2026-11-01

## Status

**Proposta, sem implementação; aberta a quem quiser assumir.** O repositório contém apenas documentação. Conforme o [ADR-021](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/architecture-decisions/ADR-021-escopo-de-implementacao.md), o único caso implementado da federação é o BioCultDB integrado ao BioCultTermos; o Pluriverso é visão de como a arquitetura poderia ser realizada por quem assumir a responsabilidade. A especificação anterior (API, modelo de dados, roadmap de fases, operação) foi removida, por ser escolha de tecnologia. Nenhuma tecnologia está definida.

Pendências da federação ficam em [`Arquitetura-BioCultural/docs/tecnico/proximosPassos.md`](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/proximosPassos.md). Aqui, só o que é desta unidade, em nível de arquitetura.

## Pendências

| # | Pendência | Origem |
|---|---|---|
| 1 | Harvest dos registros `public` conforme o contrato de harvest, incluindo remoção do índice do que deixou de ser publicado | ADR-004 D1, ADR-016 |
| 2 | Envio de apps em Custódia Local: recibo de uma via, nunca dado | ADR-018 |
| 3 | Respeito ao Regime Enunciativo e aos Labels/Notices na indexação e na exibição, com supressão declarada nunca omitida | ADR-015, ADR-016 |
| 4 | Admissão de membros pelo Comitê Federado, sem aprovação automática; verificação técnica só como sinal | ADR-006 |
| 5 | Mapeamentos entre vocabulários (`skos:exactMatch`, `closeMatch`, `broadMatch`) aprovados pelo Comitê Federado | ADR-004 D2, [rótulos SKOS-XL](https://github.com/edalcin/Arquitetura-BioCultural/blob/main/docs/tecnico/rotulos-skos-xl.md) |
| 6 | Saída de membro: remoção imediata e auditável de seus registros e mapeamentos | ADR-004 D4 |
| 7 | Harvest autenticado para registros não `public`: extensão futura, sem decisão | ADR-009 |
| 8 | Comitê Federado ainda não existe; a Reunião de Governança da Arquitetura decide só o desenho | CONTEXT.md da arquitetura |
