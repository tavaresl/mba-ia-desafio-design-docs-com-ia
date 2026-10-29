# ADR-002 — Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (Product Manager)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md) · [ADR-003](ADR-003-retry-backoff-e-dlq.md) · [ADR-006](ADR-006-reuso-padroes-do-projeto.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), é preciso decidir quem lê a tabela e dispara as chamadas HTTP, e como esse processo descobre que há eventos novos.

Forças em jogo:
- **[ADR-002-C1]** O requisito de negócio é entrega abaixo de 10 segundos.
- **[ADR-002-C2]** O MySQL não tem listener nativo como o `NOTIFY/LISTEN` do Postgres. Triggers só executam SQL e não notificam processos externos.
- **[ADR-002-C3]** A API sobe por `src/server.ts` (`npm run dev` e `npm run start` em `package.json`). Se o worker rodasse dentro dela, um restart da API perderia o worker.
- **[ADR-002-C4]** A ordem de entrega por pedido importa: PAID, depois PROCESSING, depois SHIPPED. Clientes não pediram ordering global.

## Decisão

1. **[ADR-002-D1]** O worker lê a outbox por **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos em batch pequeno. A latência mínima de 2 segundos no pior caso foi aceita.
2. **[ADR-002-D2]** O worker roda como **processo separado da API**, com uma entry-point nova `src/worker.ts` e um script `npm run worker`. O arquivo `src/worker.ts` ainda não existe no repositório.
3. **[ADR-002-D3]** A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`.
4. **[ADR-002-D4]** O worker usa o mesmo banco e a mesma `DATABASE_URL`, mas uma **instância própria de `PrismaClient`**, porque é outro processo Node.
5. **[ADR-002-D5]** O worker é **single-worker**. A ordenação é por `created_at` da outbox e vale apenas por `order_id`. Não há garantia de ordering global, e isso é uma limitação conhecida e documentada.
6. **[ADR-002-D6]** Fora do escopo: escalar para múltiplos workers. Particionar por `order_id` ou usar lock pessimista fica para o futuro.

## Alternativas consideradas

### A1. [ADR-002-A1] Reação por trigger ou listener do banco
- **Prós:** reativo, sem esperar o intervalo de polling.
- **Contras:** o MySQL não tem listener nativo. A trigger só executa SQL. Avisar o worker exigiria improvisar, como escrever em arquivo ou chamar um endpoint.
- **Motivo do descarte:** a solução fica "esquisita", e o polling de 2s já atende o requisito de 10s.

### A2. [ADR-002-A2] Worker dentro da instância da API
- **Prós:** um único processo para operar e fazer deploy.
- **Contras:** se a API reinicia, o worker é perdido.
- **Motivo do descarte:** "Só não pode ser o mesmo processo".

### A3. [ADR-002-A3] Múltiplos workers em paralelo desde o início
- **Prós:** maior vazão de entrega.
- **Contras:** perde a garantia de ordenação por pedido, a menos que haja particionamento ou lock pessimista.
- **Motivo do descarte:** complexidade sem necessidade atual. "Isso é problema do futuro, não agora".

## Consequências

### Positivas
- A API e o worker têm ciclos de vida independentes. O restart da API não interrompe o envio.
- O ordering por pedido sai de graça com um único worker.
- Nenhuma tecnologia nova: mesma stack e mesmo banco.

### Negativas
- Há um novo processo para fazer deploy e monitorar.
- A latência mínima é 2s. O polling gera consultas periódicas ao MySQL mesmo sem eventos.
- Um único worker é gargalo de vazão e ponto único de falha. Ordering global não é garantido.
- Não foi decidido como recuperar eventos presos em `processando` se o worker cair (lacuna em `docs/SUMMARY.md`, observação 6).

### Trade-off explícito
Trocamos reatividade imediata e escalabilidade horizontal por simplicidade operacional e ordering por pedido, aceitando 2 segundos de latência mínima e um único worker.
