# ADR-007 — Publicação do evento dentro da transação do `changeStatus`

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md) · [ADR-006](ADR-006-reuso-padroes-do-projeto.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

O [ADR-001](ADR-001-outbox-no-mysql.md) exige que o evento seja gravado na mesma transação da mudança de status. Falta decidir **como o código existente chama essa gravação**. Hoje não há nada parecido: a aplicação não tem eventos, filas ou webhooks.

**[ADR-007-C1]** O ponto de integração é `OrderService.changeStatus` em `src/modules/orders/order.service.ts`:
- Linha 131: `this.prisma.$transaction(async (tx) => { ... })` abre a transação.
- **[ADR-007-C2]** Linhas 138-149: valida a transição com `canTransition` (de `src/modules/orders/order.status.ts`) e lança `InvalidStatusTransitionError` ou `ConflictError`.
- Linhas 151-156: debita ou repõe estoque.
- Linhas 158-167: `tx.order.update` e `tx.orderStatusHistory.create`.
- Linhas 169-177: relê o pedido com relações e devolve.

**[ADR-007-C3]** Se o evento ficasse fora da transação, "perde a garantia toda".

## Decisão

1. **[ADR-007-D1]** Criar uma função **`publishWebhookEvent(tx, order, fromStatus, toStatus)`**, que aceita o client transacional (`tx`) da transação em curso. Ela vive no módulo `src/modules/webhooks` (a criar).
2. **[ADR-007-D2]** `OrderService.changeStatus` chama essa função **dentro** da transação, depois de gravar o histórico de status (`tx.orderStatusHistory.create`, linha 159).
3. **[ADR-007-D3]** A função é "pura recebendo o `tx`". Não se injeta o repository de webhook inteiro no `OrderService`.
4. **[ADR-007-D4]** Se a inserção na outbox falhar, a transação inteira sofre rollback. "Não pode ter caso de status mudar e evento não sair".
5. **[ADR-007-D5]** A função consulta os webhooks ativos do customer do pedido e insere uma linha por webhook cujo filtro inclui o `toStatus`. Se nenhum quiser aquele status, não insere nada.
6. **[ADR-007-D6]** O payload é renderizado nesse momento (snapshot), a partir do `order` e do `toStatus`.

## Alternativas consideradas

### A1. [ADR-007-A1] Injetar o `WebhookRepository` inteiro no `OrderService`
- **Prós:** reuso do padrão de injeção por construtor usado em `src/app.ts`.
- **Contras:** acopla o `OrderService` ao módulo de webhooks e aumenta o número de dependências de um serviço que já mistura `OrderRepository` e `PrismaClient`.
- **Motivo do descarte:** "Boa, função pura recebendo o tx. Não precisa injetar repository inteiro".

### A2. [ADR-007-A2] Publicar o evento depois do commit, fora da transação
- **Prós:** não aumenta o tempo da transação de status.
- **Contras:** se o processo cair entre o commit e a publicação, o status muda e o evento nunca sai.
- **Motivo do descarte:** viola a garantia central do outbox.

### A3. [ADR-007-A3] Disparar por trigger no banco, sem alterar o `order.service.ts`
- **Prós:** nenhum código novo no módulo de pedidos.
- **Contras:** a trigger só executa SQL e não tem acesso à lógica de filtro por webhook nem à renderização do payload.
- **Motivo do descarte:** alternativa plausível, não discutida para este ponto. Não resolve o filtro e o snapshot ([ADR-001](ADR-001-outbox-no-mysql.md)).

## Consequências

### Positivas
- Atomicidade garantida entre mudança de status e registro do evento.
- Mudança pequena e localizada em `changeStatus`, com uma única chamada nova.
- O módulo de pedidos não conhece o repository nem o worker de webhooks.

### Negativas
- O `order.service.ts` passa a depender de `src/modules/webhooks` e do shape do `order`.
- A transação de `changeStatus`, já pesada, fica mais longa, com leituras de webhooks e uma ou mais inserções.
- Falhas na publicação passam a **bloquear** a mudança de status do pedido, o que é intencional, mas amplia o impacto de bugs do módulo de webhooks.
- A transação lê e depois escreve sem trava de linha. Esse risco já existe hoje no débito de estoque e não é tratado por esta decisão.

### Trade-off explícito
Aceitamos acoplar o fluxo de pedidos ao módulo de webhooks e engordar a transação de `changeStatus` em troca da garantia de que nenhuma mudança de status ocorre sem o evento correspondente.
