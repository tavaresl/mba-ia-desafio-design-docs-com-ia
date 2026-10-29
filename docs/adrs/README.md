# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature de **Sistema de Webhooks de Notificação de Pedidos**. Cada decisão arquitetural fechada na reunião técnica está em um arquivo individual, no formato `ADR-NNN-titulo-em-kebab-case.md`, seguindo o template [`docs/templates/ADR.md`](../templates/ADR.md).

Documentos relacionados: [PRD](../PRD.md) · [RFC](../RFC.md) · [FDD](../FDD.md) · [Tracker](../TRACKER.md)

| ADR | Decisão | Status |
| :--- | :--- | :--- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-separado-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ | Aceito |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace de 24h | Aceito |
| [ADR-005](ADR-005-at-least-once-x-event-id.md) | Entrega at-least-once com `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-padroes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-publicacao-na-transacao-do-changestatus.md) | Publicação do evento dentro da transação do `changeStatus` | Aceito |
| [ADR-008](ADR-008-autorizacao-crud-e-replay-admin.md) | CRUD para qualquer role autenticada e replay restrito a `ADMIN` | Aceito |
