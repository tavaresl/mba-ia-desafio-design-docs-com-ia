# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-10-29 |
| **Revisores** | Marcos (Product Manager), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) |

> Esta RFC consolida a proposta técnica discutida na reunião de quinta-feira (09:00 a 09:53) entre os cinco participantes acima, e a submete a eles para revisão antes do início da implementação. O "como construir" em detalhe está no [FDD](FDD.md), e cada decisão fechada tem seu [ADR](adrs/README.md).

---

## 1. TL;DR

**[RFC-PROP-01]** Vamos notificar clientes B2B por webhook quando o status de um pedido muda, usando o **padrão outbox no MySQL existente**. O evento é gravado na mesma transação de `OrderService.changeStatus`. Um **worker em processo separado** lê a outbox por polling a cada 2 segundos e envia o JSON assinado com **HMAC-SHA256** (uma secret por endpoint, com rotação e janela de 24h). Falhas passam por retry com backoff exponencial (5 tentativas) e depois vão para uma DLQ em tabela separada, com replay manual restrito a `ADMIN`. A entrega é **at-least-once**, e o cliente deduplica por `X-Event-Id`. Reaproveitamos os padrões do projeto (módulo em `src/modules`, `AppError`, Pino, error middleware). Estimativa: 3 sprints, incluindo a revisão de segurança, para atender o prazo de fim de novembro.

---

## 2. Contexto e problema

**[RFC-CTX-01]** Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para ser notificados em tempo real quando o status dos pedidos muda. Hoje consultam `GET /orders` de tempos em tempos, o que deixa a integração lenta e cara para eles. A Atlas indicou que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre.

**[RFC-CTX-02]** Para os clientes, "tempo real" significa menos de 10 segundos. Os webhooks são somente de saída.

**[RFC-CTX-03]** A aplicação hoje não tem eventos, filas ou notificação externa. A mudança de status é uma transação já pesada em `src/modules/orders/order.service.ts` (atualiza `orders`, insere em `order_status_history` e mexe no estoque), e não pode depender de HTTP externo.

---

## 3. Proposta técnica

### 3.1 Visão geral

```
 PATCH status ─► OrderService.changeStatus ──┐ (uma transação MySQL)
                  ├─ update orders           │
                  ├─ insert order_status_history
                  └─ publishWebhookEvent(tx) ─► insert webhook_outbox (payload snapshot)
                                              │
 worker (processo separado, poll 2s) ◄────────┘
   ├─ lê pendentes mais antigos, em batch
   ├─ POST https://cliente (HMAC-SHA256, timeout 10s, X-Event-Id)
   ├─ sucesso ─► marca entregue
   └─ falha ─► retry 1m/5m/30m/2h/12h ─► 5ª falha ─► webhook_dead_letter
                                                       └─ replay manual (ADMIN)
```

### 3.2 Componentes

| Componente | Responsabilidade | Decisão / Referência |
|---|---|---|
| **[RFC-COMP-01]** Outbox (`webhook_outbox`) | Registro atômico do evento, com payload já renderizado | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **[RFC-COMP-02]** `publishWebhookEvent(tx, …)` | Ponto único de publicação dentro do `changeStatus`, com filtro por status | [ADR-007](adrs/ADR-007-publicacao-na-transacao-do-changestatus.md) |
| **[RFC-COMP-03]** Worker (`src/worker.ts`) | Polling de 2s, envio HTTP, ordering por pedido, single-worker | [ADR-002](adrs/ADR-002-worker-separado-polling.md) |
| **[RFC-COMP-04]** Retry e DLQ (`webhook_dead_letter`) | Backoff exponencial, falha permanente, replay | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| **[RFC-COMP-05]** Assinatura | HMAC-SHA256, secret por endpoint, rotação com grace de 24h | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| **[RFC-COMP-06]** Semântica de entrega | At-least-once, dedup por `X-Event-Id` | [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md) |
| **[RFC-COMP-07]** Módulo `src/modules/webhooks` | CRUD, erros `WEBHOOK_*`, logs e schemas, no padrão do projeto | [ADR-006](adrs/ADR-006-reuso-padroes-do-projeto.md) |
| **[RFC-COMP-08]** Autorização | CRUD para qualquer role autenticada; replay só `ADMIN`, com auditoria em log | [ADR-008](adrs/ADR-008-autorizacao-crud-e-replay-admin.md) |

### 3.3 Superfície pública (resumo)

- **[RFC-API-01]** **CRUD de configuração:** cadastrar, editar, remover e listar webhooks de um customer, com URL `https`, lista de status de interesse e secret gerada pela plataforma.
- **[RFC-API-02]** **Rotação de secret** pela API, com a secret antiga válida por 24h.
- **[RFC-API-03]** **Histórico de entregas:** `GET /webhooks/:id/deliveries`.
- **[RFC-API-04]** **Replay administrativo:** `POST /admin/webhooks/dead-letter/:id/replay`.
- **[RFC-API-05]** **Evento enviado:** JSON enxuto de `order.status_changed`, sem os `items`, com os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.

Contratos, schemas e fluxos detalhados estão no [FDD](FDD.md).

---

## 4. Alternativas consideradas

| # | Alternativa | Por que foi descartada (trade-off) |
|---|---|---|
| RFC-ALT-01 | Disparo HTTP síncrono dentro do `changeStatus` | Um cliente lento trava mudanças de status de outros pedidos, e um cliente fora do ar forçaria rollback. Ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md). |
| RFC-ALT-02 | Fila externa (Redis Streams ou Cluster) | Exige infraestrutura nova, "overengineering" para um time pequeno. Ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md). |
| RFC-ALT-03 | Trigger ou listener de banco para acordar o worker | O MySQL não tem `NOTIFY/LISTEN`, e a trigger só executa SQL. Ver [ADR-002](adrs/ADR-002-worker-separado-polling.md). |
| RFC-ALT-04 | Retry indefinido com backoff | Evento fica pendurado para sempre se o cliente sumiu. Ver [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md). |
| RFC-ALT-05 | Entrega exactly-once | Exige coordenação dos dois lados e é muito mais complexa. Ver [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md). |
| RFC-ALT-06 | Secret global da plataforma | Se vazar, vaza tudo. Ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md). |

---

## 5. Impacto

| Área | Impacto |
|---|---|
| **[RFC-IMP-01]** **Código existente** | Uma chamada nova em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). Novo módulo `src/modules/webhooks`, nova entry-point `src/worker.ts` e novos routers montados em `src/routes/index.ts`. Reuso de `AppError`, `requireRole`, error middleware e Pino, sem alterá-los. |
| **[RFC-IMP-02]** **Banco de dados** | Novas tabelas `webhook_outbox`, `webhook_dead_letter` e a de configuração de webhooks, com UUID como chave. Mais uma escrita na transação de `changeStatus`. |
| **[RFC-IMP-03]** **Operação** | Novo processo a operar (`npm run worker`), junto da API, com mesma `DATABASE_URL`. Passa a existir um single worker. |
| **[RFC-IMP-04]** **Clientes** | Devem implementar a verificação de HMAC e a deduplicação por `X-Event-Id`. Marcos documenta no portal de desenvolvedores. |
| **[RFC-IMP-05]** **Prazos** | 3 sprints, com a revisão de segurança incluída no fim; a Sofia pediu pelo menos 2 dias úteis para revisar HMAC e a geração de secret. Meta de entrega: fim de novembro. |

---

## 6. Riscos

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| **[RFC-RSK-01]** Transação de `changeStatus` mais pesada (escrita extra na outbox) | Média | Médio | Inserção mínima e indexada; testes de integração do fluxo de status. |
| **[RFC-RSK-02]** Cliente lento ou fora do ar | Alta | Médio | Timeout de 10s, retry com backoff e DLQ; o worker é isolado da API. |
| **[RFC-RSK-03]** Eventos duplicados no cliente | Média | Médio | `X-Event-Id` e documentação em destaque no portal. |
| **[RFC-RSK-04]** Single worker é gargalo e ponto único de falha; ordering global não garantido | Média | Médio | Limitação documentada; escala futura por partição ou lock. |
| **[RFC-RSK-05]** Vazamento ou exposição indevida da secret | Baixa | Alto | Secret por endpoint, rotação com grace de 24h e revisão de segurança antes do deploy. |
| **[RFC-RSK-06]** Prazo apertado para fim de novembro | Média | Alto | Escopo enxuto (sem e-mail e sem dashboard) e estimativa de 3 sprints já com a revisão da Sofia. |

---

## 7. Questões em aberto

| # | Questão | Situação na discussão | Proposta para resolução |
|---|---|---|---|
| RFC-Q-01 | Rate limiting de saída por cliente (ex.: 50 pedidos mudando em 1 minuto) | Registrado como ponto em aberto: "observar e decidir depois" | Monitorar o volume real e reabrir se virar problema. |
| RFC-Q-02 | Alerta por e-mail ao cliente quando o webhook falha repetidamente | Fora do escopo desta fase; "talvez próxima fase" | Reavaliar depois de medir o impacto. |
| RFC-Q-03 | Escalar para múltiplos workers e ordering além de `order_id` | Limitação conhecida; particionar ou usar lock é "problema do futuro" | Reabrir se a vazão exigir mais de um worker. |
| RFC-Q-04 | Endurecer as permissões do CRUD de configuração | Por enquanto qualquer role autenticada | Revisar na próxima fase. |
| RFC-Q-05 | Arquivamento das linhas entregues da outbox (~30 dias) | Fora do escopo desta feature | Tratar em feature separada. |
| RFC-Q-06 | *Lacuna identificada na análise, não discutida:* a primeira tentativa conta entre as 5? | A reunião fala em "5 tentativas" com 5 intervalos | Os revisores definem; o FDD registra a premissa. |
| RFC-Q-07 | *Lacuna identificada na análise, não discutida:* com qual secret assinar durante os 24h de rotação, e como a secret é armazenada | Só se decidiu a existência do grace period | Sofia define na revisão de segurança. |
| RFC-Q-08 | *Lacuna identificada na análise, não discutida:* onde persistir o histórico de entregas de `GET /webhooks/:id/deliveries` | O endpoint foi pedido, mas nenhuma tabela foi definida | Diego e Bruno definem a modelagem no FDD. |
| RFC-Q-09 | *Lacuna identificada na análise, não discutida:* o retry pode inverter a ordem de eventos do mesmo pedido (o evento N em backoff é ultrapassado pelo N+1) | A reunião só garante ordering por `order_id` com single-worker, sem considerar o backoff | Diego e Larissa decidem se bloqueiam eventos seguintes do mesmo pedido ou se aceitam a limitação. Ver [FDD](FDD.md), seção 14. |

**[RFC-ESC-01]** **Explicitamente fora de escopo:** dashboard visual para o cliente; e-mail de alerta; rate limiting de saída; múltiplos workers; arquivamento da outbox.

---

## 8. Decisões relacionadas

* [ADR-001 — Padrão Outbox no MySQL existente](adrs/ADR-001-outbox-no-mysql.md)
* [ADR-002 — Worker em processo separado com polling de 2 segundos](adrs/ADR-002-worker-separado-polling.md)
* [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-backoff-e-dlq.md)
* [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
* [ADR-005 — Entrega at-least-once com `X-Event-Id`](adrs/ADR-005-at-least-once-x-event-id.md)
* [ADR-006 — Reuso máximo dos padrões existentes do projeto](adrs/ADR-006-reuso-padroes-do-projeto.md)
* [ADR-007 — Publicação do evento dentro da transação do `changeStatus`](adrs/ADR-007-publicacao-na-transacao-do-changestatus.md)
* [ADR-008 — CRUD para qualquer role autenticada e replay restrito a ADMIN](adrs/ADR-008-autorizacao-crud-e-replay-admin.md)
