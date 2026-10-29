# ADR-008 — Autorização: CRUD para qualquer role autenticada e replay restrito a ADMIN

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Sofia (Eng. de Segurança), Larissa (Tech Lead), Marcos (Product Manager)
- **Relacionados:** [ADR-003](ADR-003-retry-backoff-e-dlq.md) · [ADR-006](ADR-006-reuso-padroes-do-projeto.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

O módulo de webhooks expõe dois grupos de endpoints com riscos diferentes:
- O CRUD de configuração e a consulta de entregas, usados pelo cliente por meio da nossa API.
- O replay de eventos da DLQ, que mexe na fila de entrega de notificações.

**[ADR-008-C1]** O cliente cadastra webhooks direto pela nossa API, autenticado com um JWT do sistema. Existem usuários que representam o cliente. Portanto, o `customer_id` vem no body ou no path e **não** do JWT.

**[ADR-008-C2]** No código existente, `authenticate` define `req.user` com `id`, `email` e `role` (`ADMIN` ou `OPERATOR`), e `requireRole(...roles)` retorna 403 `Insufficient permissions` quando a role não bate (`src/middlewares/auth.middleware.ts`, linhas 27-61). Hoje `requireRole` só é usado em `GET /users/:id`.

## Decisão

1. **[ADR-008-D1]** O **replay de DLQ exige a role `ADMIN`**, reaproveitando o `requireRole` existente. "Mexer em fila de entrega de notificação não é coisa de operador."
2. **[ADR-008-D2]** O endpoint de replay **registra em log quem executou**, para auditoria. O log usa o Pino já existente ([ADR-006](ADR-006-reuso-padroes-do-projeto.md)).
3. **[ADR-008-D3]** O **restante do CRUD de configuração aceita qualquer role autenticada**, "por enquanto".
4. **[ADR-008-D4]** O `customer_id` é recebido no body ou no path.
5. **[ADR-008-D5]** Fora do escopo: endurecer as permissões do CRUD.

## Alternativas consideradas

### A1. [ADR-008-A1] Qualquer usuário autenticado pode fazer replay
- **Prós:** nenhuma regra extra.
- **Contras:** um operador poderia reinjetar eventos na fila de entrega sem restrição.
- **Motivo do descarte:** a Sofia exigiu `ADMIN`.

### A2. [ADR-008-A2] Restringir também todo o CRUD a `ADMIN`
- **Prós:** superfície mais protegida desde o início.
- **Contras:** os usuários que representam o cliente precisariam ser `ADMIN`, o que é desproporcional.
- **Motivo do descarte:** decidiu-se aceitar qualquer role autenticada por enquanto, com endurecimento futuro.

### A3. [ADR-008-A3] Derivar o `customer_id` do JWT
- **Prós:** impede um usuário de operar em outro customer.
- **Contras:** o JWT atual é de um usuário do sistema e não carrega o customer.
- **Motivo do descarte:** o `customer_id` vem do body ou do path.

## Consequências

### Positivas
- O replay, a operação mais sensível, fica restrito e auditável.
- Reuso total de `authenticate` e `requireRole`, sem código de autorização novo.
- O CRUD fica simples para os clientes integrarem.

### Negativas
- Como o `customer_id` não vem do JWT e o CRUD aceita qualquer role, um usuário autenticado pode, em tese, listar ou alterar webhooks de outro customer. A reunião não tratou autorização por customer (lacuna em `docs/SUMMARY.md`, observação 7).
- Ao herdar a convenção atual do projeto, `requireRole` fica concentrado em poucos endpoints.
- A auditoria do replay depende de log. Não há tabela de auditoria nesta fase.

### Trade-off explícito
Aceitamos um CRUD com controle de acesso frouxo, que pode ser endurecido depois, em troca de entregar rápido e de proteger de imediato o único endpoint com poder de reinjetar eventos.
