# ADR-005 — Garantia de entrega at-least-once com `X-Event-Id`

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança), Bruno (Eng. Pleno, Pedidos), Marcos (Product Manager)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md) · [ADR-003](ADR-003-retry-backoff-e-dlq.md) · [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

**[ADR-005-C1]** Com outbox e retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md)), pode acontecer de o cliente receber o mesmo evento mais de uma vez. Por exemplo, o cliente processa a chamada, mas o worker não recebe a resposta dentro do timeout de 10 segundos e tenta de novo. É preciso fixar a semântica de entrega e dar ao cliente um meio de lidar com duplicatas.

## Decisão

1. **[ADR-005-D1]** A plataforma garante entrega **at-least-once**. O cliente deve estar preparado para receber o mesmo evento duas vezes.
2. **[ADR-005-D2]** Cada evento leva um `event_id` (UUID), gerado quando o evento entra na outbox, enviado no header **`X-Event-Id`**. O cliente deduplica por esse valor do lado dele.
3. **[ADR-005-D3]** O mesmo UUID aparece no campo `event_id` do payload.
4. **[ADR-005-D4]** A responsabilidade de deduplicar fica **com o cliente**. Sofia notou que isso joga responsabilidade para ele, e Diego confirmou que é o padrão de mercado, como Stripe e GitHub.
5. **[ADR-005-D5]** Marcos vai documentar isso em destaque no portal de desenvolvedores.
6. **[ADR-005-D6]** Fora do escopo: exactly-once.

## Alternativas consideradas

### A1. [ADR-005-A1] Entrega exactly-once
- **Prós:** o cliente não precisa deduplicar.
- **Contras:** exige coordenação dos dois lados e fica muito mais complexo.
- **Motivo do descarte:** "At-least-once com event_id resolve 99% dos casos".

### A2. [ADR-005-A2] At-most-once (sem retry)
- **Prós:** nunca duplica evento.
- **Contras:** perde eventos sempre que o cliente está indisponível. Contradiz a política de retry de [ADR-003](ADR-003-retry-backoff-e-dlq.md).
- **Motivo do descarte:** alternativa plausível, não discutida na reunião. É incompatível com a decisão de retry em 5 tentativas.

## Consequências

### Positivas
- Nenhum evento é perdido enquanto o retry ou a DLQ o estiver segurando.
- Mecanismo simples e familiar para integradores, com um identificador único e estável por evento.
- Não exige coordenação transacional com o cliente.

### Negativas
- O cliente precisa implementar deduplicação por `X-Event-Id`. Quem não o fizer pode processar o mesmo pedido duas vezes.
- O replay manual da DLQ ([ADR-003](ADR-003-retry-backoff-e-dlq.md)) também pode gerar duplicatas. Não ficou definido se o replay mantém o `event_id` original (lacuna a registrar no FDD).
- Há dependência de documentação clara no portal.

### Trade-off explícito
Empurramos para o cliente a deduplicação em troca de evitar a complexidade e o custo de coordenação do exactly-once. Aceitamos duplicatas ocasionais para nunca perder um evento.
