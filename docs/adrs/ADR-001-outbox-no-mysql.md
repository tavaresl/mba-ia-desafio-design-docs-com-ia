# ADR-001 — Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-polling.md) · [ADR-003](ADR-003-retry-backoff-e-dlq.md) · [ADR-007](ADR-007-publicacao-na-transacao-do-changestatus.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

**[ADR-001-C1]** Três clientes B2B querem ser notificados quando o status dos pedidos muda, em menos de 10 segundos. Hoje o único mecanismo é o polling do cliente em `GET /orders`.

**[ADR-001-C2]** A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126-179). É uma única transação Prisma (`this.prisma.$transaction`) que valida a transição, debita ou repõe estoque, atualiza `orders` e insere em `order_status_history`.

Forças em jogo:
- **[ADR-001-C3]** Um cliente lento não pode travar a mudança de status de outros pedidos. Um cliente fora do ar também não pode causar rollback da mudança.
- **[ADR-001-C4]** Se a transação principal commitou, o evento tem que existir. Se deu rollback, o evento tem que sumir junto.
- **[ADR-001-C5]** O time é pequeno e não quer subir infraestrutura nova. Hoje a única infraestrutura é o MySQL do `docker-compose.yml`.

## Decisão

Adotar o padrão **outbox no MySQL existente**.

1. **[ADR-001-D1]** Dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, o sistema insere uma linha em `webhook_outbox`. Se a inserção falhar, a transação inteira sofre rollback.
2. **[ADR-001-D2]** A tabela tem um campo de status (`pendente`, `processando`, `falhou`, `entregue`) e índices em status e `created_at`.
3. **[ADR-001-D3]** A linha guarda o **payload já renderizado (snapshot)** no momento da inserção, e não só o `order_id`. O evento reflete o estado do pedido quando o status mudou.
4. **[ADR-001-D4]** Chaves primárias em **UUID**, seguindo o padrão do projeto.
5. **[ADR-001-D5]** O filtro de eventos por webhook é aplicado **na inserção**. Se nenhum webhook do customer quer aquele status, nada é inserido.
6. **[ADR-001-D6]** Fora do escopo desta decisão: arquivar linhas entregues (cerca de 30 dias).

Observação de processo: o UUID e o snapshot foram decididos depois do resumo final, já sem Marcos e Sofia na call. Convém confirmá-los na revisão do RFC.

## Alternativas consideradas

### A1. [ADR-001-A1] Disparo HTTP síncrono dentro do `changeStatus`
- **Prós:** nenhum componente novo; menor latência no caso feliz.
- **Contras:** um cliente lento trava mudanças de status de outros pedidos. Um cliente fora do ar obriga a escolher entre falhar a mudança de status ou perder o evento.
- **Motivo do descarte:** "Síncrono está fora de questão".

### A2. [ADR-001-A2] Fila externa (Redis Streams ou Redis Cluster)
- **Prós:** desacopla o envio sem tocar o banco transacional; reativo.
- **Contras:** exige infraestrutura nova, e o evento deixa de estar na mesma transação da mudança de status.
- **Motivo do descarte:** "overengineering" para um time pequeno; "outbox no MySQL existente resolve".

### A3. [ADR-001-A3] Guardar só `order_id` na outbox e renderizar o payload no envio
- **Prós:** linha menor; payload sempre com dados atuais.
- **Contras:** se o pedido mudar depois, o evento deixa de refletir o estado de quando o status mudou, o que gera "caso esquisito".
- **Motivo do descarte:** preferência por snapshot na inserção.

## Consequências

### Positivas
- Consistência forte: evento e mudança de status são atômicos, e não há status alterado sem evento.
- Zero infraestrutura nova.
- A latência de clientes externos sai do caminho da transação de pedidos.
- O snapshot torna o payload determinístico e reprocessável (replay de DLQ, [ADR-003](ADR-003-retry-backoff-e-dlq.md)).

### Negativas
- Mais uma escrita dentro da transação de `changeStatus`, que já é pesada. Aumenta a carga e o tempo de lock no MySQL.
- A tabela cresce, e o arquivamento ficou fora do escopo.
- O estado `processando` não tem regra de recuperação definida na reunião (lacuna registrada em `docs/SUMMARY.md`, observação 6).
- Há latência mínima de entrega (ver [ADR-002](ADR-002-worker-separado-polling.md)).

### Trade-off explícito
Aceitamos carregar a transação de `changeStatus` com uma escrita extra e acoplar o mecanismo ao MySQL. Em troca ganhamos atomicidade garantida entre mudança de status e evento, e não precisamos de infraestrutura nova.
