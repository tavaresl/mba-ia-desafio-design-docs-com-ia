# Feature Design Document (FDD): Sistema de Webhooks de Notificação de Pedidos

---
**Versão:** v1.0  
**Data:** 2026-10-29  
**Responsável:** Larissa (Tech Lead)  
**Status:** Em Revisão  
---

> **Rastreabilidade.** Os itens deste documento têm identificador (por exemplo, `FDD-FLX-01`). A origem de cada um (fala da reunião ou arquivo do código) está no [TRACKER](TRACKER.md). Detalhes necessários para implementar que a reunião **não** definiu estão marcados como **(derivação técnica)**, e as lacunas que precisam de confirmação estão em "Premissas a confirmar" (seção 12).

## 1. Contexto e motivação técnica

**[FDD-CTX-01]** Hoje a aplicação não tem nenhum mecanismo de notificação externa, evento, fila ou webhook. Os clientes B2B descobrem mudanças de status consultando `GET /orders` em polling, o que torna a integração lenta e cara para eles.

**[FDD-CTX-02]** A mudança de status ocorre em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126-179), uma única transação Prisma que valida a transição, debita ou repõe estoque, atualiza `orders` e insere em `order_status_history`. Chamar HTTP externo dentro dela travaria outros pedidos e criaria o dilema de dar rollback quando o cliente está fora do ar.

**[FDD-CTX-03]** A solução é o padrão outbox no MySQL existente ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)): o evento é gravado na mesma transação da mudança de status ([ADR-007](adrs/ADR-007-publicacao-na-transacao-do-changestatus.md)), e um worker em processo separado ([ADR-002](adrs/ADR-002-worker-separado-polling.md)) faz o envio assinado ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)), com retry e DLQ ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)). A visão de alto nível está no [RFC](RFC.md).

---

## 2. Objetivos técnicos

* **[FDD-OBJ-01]** Garantir que **nenhuma mudança de status ocorra sem o evento correspondente**, e vice-versa, usando a mesma transação.
* **[FDD-OBJ-02]** Entregar a notificação em menos de 10 segundos, com latência mínima de 2s do polling.
* **[FDD-OBJ-03]** Permitir que o cliente autentique a origem e a integridade de cada chamada (HMAC-SHA256, secret por endpoint).
* **[FDD-OBJ-04]** Entrega at-least-once, com identificador estável para deduplicação.
* **[FDD-OBJ-05]** Reaproveitar integralmente os padrões do projeto, sem dependências novas.

---

## 3. Escopo e exclusões

### Incluído
* Tabelas `webhook_outbox`, `webhook_dead_letter` e de configuração dos webhooks (UUID como chave).
* `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamado de `OrderService.changeStatus`.
* Worker (`src/worker.ts` + `npm run worker`) com polling de 2s, envio HTTP com timeout de 10s, retry com backoff e DLQ.
* Módulo `src/modules/webhooks`: CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ.
* Erros `WEBHOOK_*`, logs estruturados e o filtro de eventos por status.

### Excluído
* **[FDD-ESC-01]** E-mail de alerta ao cliente.
* **[FDD-ESC-02]** Rate limiting de saída; "observar e decidir depois".
* **[FDD-ESC-03]** Dashboard visual; só endpoints.
* **[FDD-ESC-04]** Múltiplos workers e ordering global.
* **[FDD-ESC-05]** Arquivamento de linhas entregues da outbox.
* **[FDD-ESC-06]** Webhooks de entrada (o cliente só recebe).
* **[FDD-ESC-07]** Endurecer as permissões do CRUD.

---

## 4. Fluxos detalhados e diagramas

### 4.1 [FDD-FLX-01] Fluxo A — Criação do evento na outbox (dentro de `changeStatus`)

```
PATCH /api/v1/orders/:id/status
  └─ OrderService.changeStatus(id, input, userId)         order.service.ts:126
       prisma.$transaction(async tx => {                  order.service.ts:131
         1. order = tx.order.findUnique(items)            :132
         2. valida from→to (canTransition)                :140-149
         3. debita/repõe estoque                          :151-156
         4. tx.order.update(status = to)                  :158
         5. tx.orderStatusHistory.create(...)             :159-167
         6. publishWebhookEvent(tx, order, from, to)      ◄── NOVO, depois do passo 5
         7. relê o pedido e retorna                       :169-177
       })                                                 ← qualquer falha = rollback de TUDO
```

`publishWebhookEvent` ([ADR-007](adrs/ADR-007-publicacao-na-transacao-do-changestatus.md)):
1. Busca os webhooks **ativos** do `order.customerId` cujo filtro de eventos inclui `toStatus`. Se não houver nenhum, retorna sem inserir.
2. Para cada webhook, renderiza o **payload completo (snapshot)** no momento da inserção e valida o limite de 64KB (ver 7.2).
3. Insere uma linha em `webhook_outbox` com status `PENDING`, usando UUID como chave, que é também o `event_id`.
4. Se qualquer inserção falhar, a exceção propaga e a transação inteira sofre rollback.

**(derivação técnica)** Uma linha por par evento × webhook: como `X-Webhook-Id` identifica qual cadastro originou cada envio, cada linha da outbox pertence a um webhook. O `event_id` é, portanto, único por linha.

### 4.2 [FDD-FLX-02] Fluxo B — Processamento pelo worker

```
src/worker.ts  (processo próprio, PrismaClient próprio)           [ADR-002]
 loop a cada 2s:
   1. SELECT eventos PENDING com next_attempt_at <= agora
      ORDER BY created_at ASC  LIMIT <batch pequeno>
   2. para cada evento, em sequência:
        a. marca PROCESSING
        b. carrega webhook (url, secret); se inativo/removido → ver 4.4
        c. serializa o payload salvo; assina: HMAC-SHA256(secret, corpo)
        d. POST <url> com os headers do item 5.3, timeout 10s
        e. 2xx  → marca DELIVERED + registra a entrega
           senão (não-2xx, erro de rede, timeout) → Fluxo C
```

* **Ordenação:** um único worker processa por `created_at`; a garantia vale **por `order_id`** e enquanto houver um só worker. Ver a ressalva sobre retry na seção 14.
* **Entry-point:** `src/worker.ts`, análoga a `src/server.ts`, com script `npm run worker`. Reutiliza `logger` e `env` do projeto.
* **(derivação técnica)** O critério de sucesso é qualquer resposta HTTP 2xx. A reunião só define que timeout de 10s é falha.

### 4.3 [FDD-FLX-03] Fluxo C — Retry com backoff

Em caso de falha, o evento volta para `PENDING` com `next_attempt_at` calculado pela tabela abaixo:

| Falha nº | Espera até a próxima tentativa |
|---|---|
| 1ª | 1 minuto |
| 2ª | 5 minutos |
| 3ª | 30 minutos |
| 4ª | 2 horas |
| 5ª | 12 horas |

Total de quase 15 horas entre a primeira falha e a última tentativa. Esgotadas as tentativas, a falha é permanente e o evento segue para a DLQ (Fluxo D). Ver a premissa sobre a contagem de tentativas na seção 12.

### 4.4 [FDD-FLX-04] Fluxo D — DLQ e replay

```
5ª falha ──► INSERT webhook_dead_letter (payload, motivo, timestamp)
          └► evento sai da fila (status FAILED na outbox)

POST /api/v1/admin/webhooks/dead-letter/:id/replay   (ADMIN)
   1. requireRole('ADMIN')                              ← 403 se não for ADMIN
   2. localiza o registro da DLQ                        ← 404 WEBHOOK_DEAD_LETTER_NOT_FOUND
   3. recoloca o evento na outbox como PENDING
   4. loga quem executou (req.user.id, req.user.email)
```

**(derivação técnica)** Se o webhook foi removido ou desativado antes do envio, o evento é movido para a DLQ com o motivo `WEBHOOK_INACTIVE`. A reunião não trata esse caso.

### 4.5 [FDD-FLX-05] Fluxo E — Cadastro e rotação de secret

* **Criação:** a plataforma gera a secret e a devolve **uma vez**, na resposta de criação.
* **Rotação:** o cliente pede uma nova secret pela API. A nova passa a ser a vigente, e a anterior continua válida por 24h em paralelo, depois "morre".
* Como a assinatura é gerada pela plataforma, a validade da secret antiga só tem efeito se o envio puder usar as duas. Esse ponto está em aberto (seção 12, FDD-PRM-02).

---

## 5. Contratos públicos (assinaturas, headers, exemplos)

Todas as rotas estão sob o prefixo `/api/v1` (`src/app.ts`, linha 67) e usam `authenticate`. O formato de erro é o do projeto: `{ "error": { "code", "message", "details?" } }` (`src/middlewares/error.middleware.ts`).

### 5.1 Endpoints HTTP

O módulo de webhooks segue o padrão de `buildOrderRouter` (`src/modules/orders/order.routes.ts`): `router.use(authenticate)` e `validate({...})` por rota.

#### [FDD-CONTRATO-01] `POST /api/v1/webhooks` — cadastrar webhook
O `customerId` vem no body. Qualquer role autenticada ([ADR-008](adrs/ADR-008-autorizacao-crud-e-replay-admin.md)).

Request:
```json
{
  "customerId": "c1f0f2a4-0b8e-4f55-9a53-3b2f8c9b1d01",
  "url": "https://api.atlas.example.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```
Response `201 Created`:
```json
{
  "id": "5d1e9c2e-6a3b-4a1f-8d77-0f9b3c1a7e22",
  "customerId": "c1f0f2a4-0b8e-4f55-9a53-3b2f8c9b1d01",
  "url": "https://api.atlas.example.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "<secret gerada pela plataforma, exibida somente nesta resposta>",
  "createdAt": "2026-10-29T14:00:00.000Z"
}
```
Status: `201` criado · `400` `VALIDATION_ERROR` ou `WEBHOOK_INVALID_URL` (URL não `https`) · `401` · `404` `WEBHOOK_CUSTOMER_NOT_FOUND`.

**(derivação técnica)** O nome do campo `events` e seu conteúdo (valores do enum `OrderStatus` em `prisma/schema.prisma`) vêm de "lista dos status que ele quer receber".

#### [FDD-CONTRATO-02] `GET /api/v1/webhooks?customerId=…` — listar webhooks do customer
Lista os webhooks de um customer.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "5d1e9c2e-6a3b-4a1f-8d77-0f9b3c1a7e22",
      "customerId": "c1f0f2a4-0b8e-4f55-9a53-3b2f8c9b1d01",
      "url": "https://api.atlas.example.com/hooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-29T14:00:00.000Z"
    }
  ]
}
```
A secret **não** é devolvida em listagens. Status: `200` · `400` `VALIDATION_ERROR` (sem `customerId`) · `401`.

#### [FDD-CONTRATO-03] `PATCH /api/v1/webhooks/:id` — editar webhook
Edita a configuração de um webhook.

Request:
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`: o webhook atualizado, no mesmo formato da listagem. Status: `200` · `400` `VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` · `401` · `404` `WEBHOOK_NOT_FOUND`.

#### [FDD-CONTRATO-04] `DELETE /api/v1/webhooks/:id` — remover webhook
Response `204 No Content`. Status: `204` · `401` · `404` `WEBHOOK_NOT_FOUND`.

#### [FDD-CONTRATO-05] `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret
Permite ao cliente pedir uma nova secret pela API. **(derivação técnica)** O caminho e o formato da resposta não foram definidos na reunião.

Response `200 OK`:
```json
{
  "id": "5d1e9c2e-6a3b-4a1f-8d77-0f9b3c1a7e22",
  "secret": "<nova secret, exibida somente nesta resposta>",
  "previousSecretValidUntil": "2026-10-30T14:00:00.000Z"
}
```
Status: `200` · `401` · `404` `WEBHOOK_NOT_FOUND`.

#### [FDD-CONTRATO-06] `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas
Histórico dos últimos 100 envios do webhook: sucesso ou falha, payload, resposta e tempo de resposta.

Response `200 OK`:
```json
{
  "data": [
    {
      "eventId": "9a6c0e1b-2f31-4d0e-b1c7-6d2d6a4f8e10",
      "attempt": 1,
      "success": true,
      "statusCode": 200,
      "durationMs": 184,
      "payload": { "event_id": "9a6c0e1b-2f31-4d0e-b1c7-6d2d6a4f8e10", "event_type": "order.status_changed" },
      "responseBody": "ok",
      "createdAt": "2026-10-29T14:05:02.000Z"
    }
  ]
}
```
Status: `200` · `401` · `404` `WEBHOOK_NOT_FOUND`. A tabela de origem é uma premissa (seção 12, FDD-PRM-03).

#### [FDD-CONTRATO-07] `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay manual da DLQ
Exige `ADMIN` (`requireRole('ADMIN')`) e registra quem executou. **(derivação técnica)** O prefixo `/api/v1` vem de como o roteador é montado em `src/routes/index.ts`; o resto do caminho é o da reunião.

Response `202 Accepted`:
```json
{ "id": "e3b5b2a0-7c1d-4e6f-9a10-2f4d8c6b1a33", "status": "PENDING" }
```
Status: `202` reenfileirado · `401` · `403` `FORBIDDEN` (não é `ADMIN`) · `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 5.2 Interface interna

```ts
// src/modules/webhooks/webhook.publisher.ts  (a criar)
import type { Order, OrderItem, OrderStatus, Prisma } from '@prisma/client';

export type OrderWithItems = Order & { items: OrderItem[] };

export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: OrderWithItems,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void>;
```

### 5.3 Requisição enviada ao cliente (worker → endpoint do cliente)

Headers:

* **[FDD-CONTRATO-08]** `Content-Type: application/json`
* **[FDD-CONTRATO-09]** `X-Event-Id`: UUID do evento (gerado ao entrar na outbox), usado pelo cliente para deduplicar ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)).
* **[FDD-CONTRATO-10]** `X-Signature`: HMAC-SHA256 do **corpo** do request com a secret do endpoint. **(derivação técnica)** Valor em hexadecimal; o formato exato do valor não foi definido.
* **[FDD-CONTRATO-11]** `X-Timestamp`: momento do envio, para o cliente detectar replay attack se quiser.
* **[FDD-CONTRATO-12]** `X-Webhook-Id`: id do cadastro de webhook que originou o envio.

**[FDD-CONTRATO-13]** Payload, sem os `items`; quem precisar de detalhes consulta `GET /orders/:id`:
```json
{
  "event_id": "9a6c0e1b-2f31-4d0e-b1c7-6d2d6a4f8e10",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-29T14:05:00.000Z",
  "order_id": "7f4b2f1e-9a1c-4c1d-8e0e-1c3a5b7d9f00",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "c1f0f2a4-0b8e-4f55-9a53-3b2f8c9b1d01",
  "total_cents": 129900
}
```
Os campos `order_number`, `customer_id`, `total_cents` e os status existem em `prisma/schema.prisma` (modelo `Order`).

### 5.4 Verificação pelo cliente (exemplo)

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

function isValidSignature(rawBody: string, secret: string, signatureHex: string): boolean {
  const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
  const a = Buffer.from(expected, 'hex');
  const b = Buffer.from(signatureHex, 'hex');
  return a.length === b.length && timingSafeEqual(a, b);
}
```

---

## 6. Configuração e opções

* **Requisitos de runtime:** Node.js ≥ 20, ESM (`"type": "module"`), como o resto do projeto.
* **Armazenamento:** o MySQL 8.0 existente (`docker-compose.yml`), via `PrismaClient` próprio do worker, mesma `DATABASE_URL`.
* **Scripts:** `npm run worker`.

### 6.1 Modelo de dados

Convenção do projeto: UUID como chave, `@db.Char(36)`, `@@map` em snake_case (`prisma/schema.prisma`).

| Tabela | Campos (principais) | Observações |
| --- | --- | --- |
| **[FDD-DAT-01]** `webhook_endpoints` | `id`, `customer_id` (FK `customers`), `url`, `secret`, `events` (lista de `OrderStatus`), `active`, `created_at`, `updated_at` | **(derivação técnica)** o nome da tabela e as colunas de rotação (`previous_secret`, `previous_secret_valid_until`) |
| **[FDD-DAT-02]** `webhook_outbox` | `id` (= `event_id`), `webhook_id`, `order_id`, `event_type`, `payload` (JSON, snapshot), `status` (`PENDING` / `PROCESSING` / `FAILED` / `DELIVERED`), `attempts`, `next_attempt_at`, `last_error`, `created_at`, `updated_at`. Índices em `status` e `created_at` | **(derivação técnica)** `attempts`, `next_attempt_at` e `last_error`, necessários para o backoff |
| **[FDD-DAT-03]** `webhook_dead_letter` | `id`, `outbox_event_id`, `webhook_id`, `payload`, `failure_reason`, `failed_at` | — |
| **[FDD-DAT-04]** `webhook_deliveries` | `id`, `webhook_id`, `event_id`, `attempt`, `success`, `status_code`, `response_body`, `duration_ms`, `created_at` | **(derivação técnica)** suporta `GET /webhooks/:id/deliveries`; modelagem em aberto (FDD-PRM-03) |

### 6.2 Parâmetros

| Parâmetro | Valor |
| --- | --- |
| **[FDD-CFG-01]** Intervalo de polling | 2 s |
| **[FDD-CFG-02]** Timeout da chamada HTTP | 10 s |
| **[FDD-CFG-03]** Tentativas e backoff | 5; 1m / 5m / 30m / 2h / 12h |
| **[FDD-CFG-04]** Limite de payload | 64 KB |
| **[FDD-CFG-05]** Grace da secret rotacionada | 24 h |
| **[FDD-CFG-06]** Tamanho do batch | "pequeno" (valor a definir) |

**(derivação técnica)** Esses valores devem ficar centralizados em constantes do módulo. Se forem configuráveis por ambiente, entram no schema Zod de `src/config/env.ts`.

---

## 7. Erros, exceções e fallback

### 7.1 Concorrência e atomicidade

* **Por que é crítico:** se a inserção na outbox ficasse fora da transação de `changeStatus`, poderia existir status alterado sem evento, "perde a garantia toda".
* **Garantias:** a gravação ocorre com o `tx` da transação corrente. Se a transação commita, o evento existe; se faz rollback, ele some junto. O envio ao cliente é at-least-once, então duplicatas são possíveis ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)).

### 7.2 Matriz de erros previstos

Os erros do módulo são classes derivadas de `AppError` (`src/shared/errors/app-error.ts`), com `errorCode` prefixado por `WEBHOOK_`. O error middleware os formata sem mudança.

| Código | HTTP | Quando ocorre |
| --- | --- | --- |
| **[FDD-ERR-01]** `WEBHOOK_NOT_FOUND` | 404 | Webhook inexistente em `PATCH`, `DELETE`, rotação ou deliveries |
| **[FDD-ERR-02]** `WEBHOOK_INVALID_URL` | 400 | URL que não é `https` ou é inválida |
| **[FDD-ERR-03]** `WEBHOOK_SECRET_REQUIRED` | 422 | Operação que exige secret e ela não está disponível. **(derivação técnica)** gatilho a confirmar |
| **[FDD-ERR-04]** `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload renderizado acima de 64KB |
| **[FDD-ERR-05]** `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` do cadastro não existe. **(derivação técnica)** |
| **[FDD-ERR-06]** `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de registro inexistente na DLQ. **(derivação técnica)** |
| **[FDD-ERR-07]** `FORBIDDEN` (existente) | 403 | Replay por usuário que não é `ADMIN`, via `requireRole` |
| **[FDD-ERR-08]** `VALIDATION_ERROR` (existente) | 400 | Body, query ou params inválidos, via `validate` + Zod |

Códigos **internos do worker**, que não saem por HTTP e são gravados em `last_error`, `failure_reason` e nas entregas **(derivação técnica)**:

| Código | Situação |
|---|---|
| **[FDD-ERR-09]** `WEBHOOK_DELIVERY_TIMEOUT` | O cliente não respondeu em 10s |
| **[FDD-ERR-10]** `WEBHOOK_DELIVERY_FAILED` | Resposta não-2xx ou erro de rede |
| **[FDD-ERR-11]** `WEBHOOK_INACTIVE` | O webhook foi removido ou desativado antes do envio |

**Atenção:** como `WEBHOOK_PAYLOAD_TOO_LARGE` é avaliado no momento do snapshot, dentro da transação de `changeStatus`, ele faria a mudança de status falhar. Como o payload não inclui `items`, é improvável alcançar 64KB. O comportamento exato é a premissa FDD-PRM-04.

### 7.3 Fallback

* **Cliente indisponível:** retry com backoff e, ao final, DLQ ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)). Não há outro canal de fallback: o e-mail de alerta está fora do escopo.
* **Worker parado:** os eventos permanecem na outbox como `PENDING` e são processados quando o worker voltar, já que a API e o worker são processos independentes.

---

## 8. Estratégias de resiliência

* **[FDD-RES-01]** **Timeout:** 10s por chamada; estourar conta como falha e marca para retry.
* **[FDD-RES-02]** **Retry:** 5 tentativas com backoff exponencial 1m/5m/30m/2h/12h.
* **[FDD-RES-03]** **DLQ:** falha permanente vai para `webhook_dead_letter`, com replay manual.
* **[FDD-RES-04]** **Isolamento:** a API e o worker são processos distintos; o worker falhar ou ficar lento não afeta `changeStatus`.
* **[FDD-RES-05]** **Idempotência no cliente:** `X-Event-Id`.
* **[FDD-RES-06]** **Encerramento:** o worker deve tratar `SIGINT`/`SIGTERM` como `src/server.ts` faz. **(derivação técnica)** Diferente do `server.ts` atual, convém concluir o evento em curso antes de `process.exit`, para não deixar um evento em `PROCESSING`.

---

## 9. Observabilidade

> A reunião só confirma o **logger Pino** do projeto. Métricas e tracing abaixo são **(derivação técnica)**: o projeto hoje não tem biblioteca de métricas nem de tracing, e a decisão de não adicionar dependências novas vale para o logger.

* **[FDD-OBS-01]** **Logs estruturados (Pino, `src/shared/logger/index.ts`):** eventos em `snake_case`, como `server_started` e `http_request` já usados no projeto. Propostos:
  * **[FDD-OBS-02]** `webhook_event_enqueued` (`eventId`, `webhookId`, `orderId`, `toStatus`)
  * **[FDD-OBS-03]** `webhook_delivery_succeeded` (`eventId`, `statusCode`, `durationMs`, `attempt`)
  * `webhook_delivery_failed` (`eventId`, `attempt`, `reason`, `nextAttemptAt`)
  * `webhook_moved_to_dead_letter` (`eventId`, `reason`)
  * **[FDD-OBS-04]** `webhook_dead_letter_replayed` (`deadLetterId`, `userId`, `email`): obrigatório para a auditoria do replay
  * **[FDD-OBS-05]** Nunca registrar a secret nem a assinatura em log.
* **Métricas (propostas):**
  * `webhook_outbox_pending` (gauge): eventos pendentes
  * `webhook_outbox_oldest_pending_age_seconds` (gauge): acompanha o requisito de menos de 10s
  * `webhook_delivery_attempts_total{outcome}` (contador)
  * `webhook_delivery_duration_ms` (histograma)
  * `webhook_dead_letter_total` (contador)
  * Enquanto não houver infraestrutura de métricas, esses números podem ser emitidos como logs periódicos do worker.
* **[FDD-OBS-06]** **Tracing (proposta):** sem OpenTelemetry. Para correlação, gravar o `requestId` (gerado em `src/middlewares/request-logger.middleware.ts`) na linha da outbox e incluí-lo nos logs do worker.

---

## 10. Integração com o sistema existente

Os caminhos abaixo existem no repositório, exceto os marcados **(a criar)**.

| Arquivo | Como o módulo de webhooks se integra |
|---|---|
| **[FDD-INT-01]** `src/modules/orders/order.service.ts` | `changeStatus` (linha 126) passa a chamar `publishWebhookEvent(tx, order, from, to)` dentro de `this.prisma.$transaction` (linha 131), logo após `tx.orderStatusHistory.create` (linhas 159-167) e antes da releitura do pedido (linha 169). É a única alteração em código de domínio existente. A variável `order` já traz `items`; `from` e `to` já existem (linhas 138-139). |
| **[FDD-INT-02]** `src/modules/orders/order.status.ts` | `OrderStatus` e `canTransition` definem os status válidos usados no filtro de eventos. O módulo de webhooks valida `events` com o enum `OrderStatus` do Prisma e não altera a máquina de estados. |
| **[FDD-INT-03]** `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Classes como `WebhookNotFoundError` estendem `AppError` (campos `statusCode`, `errorCode`, `details`) com códigos `WEBHOOK_*`, no mesmo estilo de `InsufficientStockError` e `InvalidStatusTransitionError`. `ForbiddenError` e `ValidationError` são reutilizados como estão. |
| **[FDD-INT-04]** `src/middlewares/error.middleware.ts` | Não muda: já formata `AppError`, `ZodError` e erros Prisma, e vai capturar os erros `WEBHOOK_*`. Limitação herdada: só mapeia os códigos Prisma P2002 e P2025. |
| **[FDD-INT-05]** `src/middlewares/auth.middleware.ts` | `authenticate` protege todo o módulo. `requireRole('ADMIN')` (linha 49) protege o replay. `req.user` fornece o usuário para o log de auditoria. |
| **[FDD-INT-06]** `src/middlewares/validate.middleware.ts` | Validação de body, query e params com Zod, como em `buildOrderRouter`; inclui a regra de URL `https`. |
| **[FDD-INT-07]** `src/shared/logger/index.ts` | O `logger` Pino é reutilizado pela API e pelo worker, sem logger novo. |
| **[FDD-INT-08]** `src/routes/index.ts` | `buildApiRouter` ganha `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e o router administrativo. O tipo `Controllers` ganha o campo `webhooks`. |
| **[FDD-INT-09]** `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController` e os devolve em `Controllers`, como já faz para os outros módulos. |
| **[FDD-INT-10]** `src/server.ts` e `package.json` | `src/server.ts` é o modelo para a entry-point `src/worker.ts` **(a criar)**; `package.json` ganha o script `worker`, ao lado de `dev` e `start`. |
| **[FDD-INT-11]** `src/config/env.ts` e `src/config/database.ts` | O worker reutiliza o schema de ambiente. O `PrismaClient` do worker é instância própria, porque é outro processo. |
| **[FDD-INT-12]** `prisma/schema.prisma` | Novos modelos (`webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter` e `webhook_deliveries`) e uma nova migration, com UUID como chave. |
| **[FDD-INT-13]** `src/modules/webhooks/` **(a criar)** | `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, `webhook.publisher.ts` e `webhook.worker.ts` (ou `webhook.processor.ts`), no padrão de `src/modules/*`. |

---

## 11. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| :--- | :--- | :--- |
| **[FDD-DEP-01]** Node.js | 20 | Já exigido pelo projeto; `fetch` global disponível, o que dispensa biblioteca HTTP nova **(derivação técnica)** |
| **[FDD-DEP-02]** MySQL | 8.0 | Mesma instância do `docker-compose.yml` |
| **[FDD-DEP-03]** Prisma / `@prisma/client` | 5.22.0 | Versão atual; migration nova, sem mudança de versão |
| **[FDD-DEP-04]** Zod | 3.23.8 | Validação de schemas do módulo |
| **[FDD-DEP-05]** Express | 4.21.1 | Novos routers montados como os demais |
| **[FDD-DEP-06]** `node:crypto` | embutido | `createHmac` para HMAC-SHA256, sem dependência nova **(derivação técnica)** |

Nenhuma biblioteca nova é necessária. A API continua compatível com os clientes atuais: a única mudança em `changeStatus` é interna e não altera a resposta de `PATCH /orders/:id/status`.

---

## 12. Premissas a confirmar

Lacunas identificadas na análise (não foram discutidas na reunião). Estão também nas questões em aberto do [RFC](RFC.md).

| # | Premissa adotada neste FDD | O que precisa ser confirmado |
|---|---|---|
| FDD-PRM-01 | "5 tentativas" = 5 envios no total; após a 5ª falha, o evento vai para a DLQ, e os 5 intervalos de espera entre elas | Se a 1ª tentativa conta nas 5 ou se são 5 retries |
| FDD-PRM-02 | Durante o grace de 24h, o envio é assinado com a secret nova | Com qual secret assinar e se as duas assinaturas são enviadas |
| FDD-PRM-03 | Tabela `webhook_deliveries` registra cada tentativa | A modelagem do histórico de entregas |
| FDD-PRM-04 | O limite de 64KB é checado no snapshot e, se excedido, lança `WEBHOOK_PAYLOAD_TOO_LARGE` | O que fazer com a mudança de status nesse caso; a reunião diz "não envia... erra" |
| FDD-PRM-05 | A secret é guardada em forma recuperável, porque o HMAC precisa dela para assinar | Armazenamento, criptografia em repouso e a revisão da Sofia |
| FDD-PRM-06 | Ao iniciar, o worker devolve eventos presos em `PROCESSING` para `PENDING` | Regra de recuperação de eventos presos |
| FDD-PRM-07 | O replay mantém o `event_id` original | Se o replay gera um novo `event_id`, o que afeta a deduplicação do cliente |

---

## 13. Critérios de aceite técnicos

* [ ] **[FDD-CRT-01]** Mudar o status de um pedido insere uma linha em `webhook_outbox` **na mesma transação**, e um rollback em `changeStatus` não deixa linha na outbox.
* [ ] **[FDD-CRT-02]** Se nenhum webhook do customer escuta o status, nenhuma linha é inserida.
* [ ] **[FDD-CRT-03]** O worker roda como processo separado (`npm run worker`) e a API reiniciar não o interrompe.
* [ ] **[FDD-CRT-04]** O evento chega ao endpoint do cliente em menos de 10s em condições normais (polling de 2s).
* [ ] **[FDD-CRT-05]** A requisição traz `X-Event-Id`, `X-Signature` (HMAC-SHA256 do corpo), `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`, e o cliente consegue validar a assinatura.
* [ ] **[FDD-CRT-06]** Falhas seguem o backoff 1m/5m/30m/2h/12h e, esgotadas as tentativas, o evento aparece em `webhook_dead_letter`.
* [ ] **[FDD-CRT-07]** O replay recoloca o evento na outbox, responde `403` para quem não é `ADMIN` e registra em log quem o executou.
* [ ] **[FDD-CRT-08]** Cadastro com URL `http` é recusado com `WEBHOOK_INVALID_URL`.
* [ ] **[FDD-CRT-09]** Todos os erros do módulo usam códigos `WEBHOOK_*` e passam pelo error middleware sem alteração dele.
* [ ] **[FDD-CRT-10]** A secret é devolvida só na criação e na rotação, e a antiga continua válida por 24h após a rotação.
* [ ] **[FDD-CRT-11]** Testes automatizados (Vitest + Supertest, como `tests/orders.test.ts`) cobrem os fluxos A a D. O teste ponta a ponta de integração no `order.service` está no plano de entrega.
* [ ] **[FDD-CRT-12]** A revisão de segurança da Sofia foi feita antes do deploy.

---

## 14. Riscos e mitigação

| Risco | Impacto | Mitigação |
| :--- | :--- | :--- |
| **[FDD-RSK-01]** Transação de `changeStatus` mais pesada | Médio | Inserção mínima e indexada; testes dos fluxos de status |
| **[FDD-RSK-02]** **Retry pode inverter a ordem de eventos do mesmo pedido** (análise nossa, não discutida): se o evento N falha e vai para retry em 1 minuto, o evento N+1 do mesmo pedido pode ser entregue antes | Médio | Registrar como limitação conhecida junto à de ordering; o payload traz `from_status`, `to_status` e `timestamp` para o cliente reordenar; decisão em aberto com os revisores |
| **[FDD-RSK-03]** Estoque debitado sem trava de linha em `changeStatus` (risco já existente em `debitStock`, `order.service.ts` linha 204): a nova inserção estende a mesma transação | Médio | Não mudar nesta feature; acompanhar em item separado |
| **[FDD-RSK-04]** Evento preso em `PROCESSING` se o worker cair | Médio | Premissa FDD-PRM-06; encerramento gracioso do worker |
| **[FDD-RSK-05]** Erros Prisma além de P2002 e P2025 viram 500 (`error.middleware.ts`) | Baixo | Validar existência de `customerId` e `webhook` antes de gravar, para evitar violações de FK |
| **[FDD-RSK-06]** Vazamento ou má guarda da secret | Alto | Secret por endpoint, não registrada em log, rotação com grace de 24h, revisão da Sofia |
| **[FDD-RSK-07]** Autorização por customer não definida: o CRUD aceita qualquer role e o `customerId` vem do body | Médio | Aceito "por enquanto"; endurecer na próxima fase |
| **[FDD-RSK-08]** Single worker é gargalo e ponto único de falha | Médio | Limitação documentada; `webhook_outbox_oldest_pending_age_seconds` para alertar |
| **[FDD-RSK-09]** Duplicatas no cliente | Médio | `X-Event-Id` e documentação em destaque no portal |
