# ADR-003 — Política de retry com backoff exponencial e DLQ

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (Product Manager)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md) · [ADR-002](ADR-002-worker-separado-polling.md) · [ADR-008](ADR-008-autorizacao-crud-e-replay-admin.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

**[ADR-003-C1]** O endpoint do cliente pode estar fora do ar ou lento. O worker precisa de uma política para tentar de novo sem bloquear a fila e sem deixar eventos pendurados para sempre.

Forças em jogo:
- **[ADR-003-C2]** Houve cliente com indisponibilidade de 2 horas em manutenção planejada.
- **[ADR-003-C3]** O worker trata como falha qualquer chamada que não responda em 10 segundos.
- **[ADR-003-C4]** Eventos que esgotam as tentativas precisam ficar guardados para depuração e reprocessamento.

## Decisão

1. **[ADR-003-D1]** **Backoff exponencial com 5 tentativas**, nos intervalos **1 min, 5 min, 30 min, 2 h e 12 h**. Isso dá quase 15 horas entre a primeira falha e a última tentativa. Marcos considerou a janela aceitável.
2. **[ADR-003-D2]** Após o teto de tentativas, o evento é considerado falha permanente e movido para uma **DLQ em tabela separada**, `webhook_dead_letter`, com payload, motivo da falha e timestamp.
3. **[ADR-003-D3]** O reprocessamento é **manual**, por endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. A autorização está em [ADR-008](ADR-008-autorizacao-crud-e-replay-admin.md).
4. **[ADR-003-D4]** Timeout de 10 segundos por chamada HTTP; estourar o timeout conta como falha e marca para retry.
5. **[ADR-003-D5]** Fora do escopo: e-mail de alerta ao cliente quando o webhook falha.

Ponto não resolvido na reunião: não ficou claro se a primeira tentativa de envio conta entre as 5 ou se são 5 retries depois da falha inicial. Os intervalos sugerem 5 retries, e o resumo diz "total 5 tentativas". O ponto vai para as questões em aberto do [RFC](../RFC.md).

## Alternativas consideradas

### A1. [ADR-003-A1] Retry indefinido com backoff
- **Prós:** nenhum evento é descartado enquanto o cliente existir.
- **Contras:** se o cliente sumiu, o evento fica pendurado para sempre.
- **Motivo do descarte:** não há ponto de parada nem visibilidade do problema.

### A2. [ADR-003-A2] Apenas 3 tentativas
- **Prós:** falha confirmada mais rápido; "mais agressivo".
- **Contras:** as 3 tentativas caberiam em cerca de 30 minutos, e uma indisponibilidade de 2 horas descartaria o evento.
- **Motivo do descarte:** "3 é pouco".

### A3. [ADR-003-A3] Marcar como `failed` na própria outbox, sem DLQ separada
- **Prós:** uma tabela a menos.
- **Contras:** polui a leitura da outbox principal, e não serve tão bem como evidência de depuração e reprocessamento.
- **Motivo do descarte:** preferiu-se a tabela separada por limpeza e rastreabilidade.

## Consequências

### Positivas
- Cobre indisponibilidades planejadas de até 2 horas e janelas maiores sem intervenção.
- Todo evento terminal fica persistido com o motivo, o que permite depurar e reprocessar.
- A outbox principal fica enxuta.

### Negativas
- Quem tem um endpoint fora do ar por mais de ~15 horas perde a entrega automática e precisa do replay manual.
- O replay é uma operação manual de ADMIN, sem notificação automática ao cliente nesta fase.
- Mais uma tabela para manter. O estado `falhou` da outbox e a DLQ se sobrepõem conceitualmente (lacuna em `docs/SUMMARY.md`, observação 3).
- O retry combinado com at-least-once ([ADR-005](ADR-005-at-least-once-x-event-id.md)) aumenta a chance de eventos duplicados no cliente.

### Trade-off explícito
Escolhemos um teto finito de tentativas, com intervenção manual depois dele, em vez de insistir indefinidamente. Aceitamos mais trabalho operacional em troca de nenhum evento preso e de visibilidade total das falhas.
