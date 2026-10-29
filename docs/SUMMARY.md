# Sumário da Reunião Técnica — Sistema de Webhooks de Notificação de Pedidos

Este documento consolida os pontos identificados em [TRANSCRICAO.md](../TRANSCRICAO.md), organizados por tipo. Cada ponto possui um identificador (`PREFIXO-NUMERO`, incremental por prefixo), o responsável pela fala com o timestamp, e um link para a linha de origem.

**Participantes:** Larissa (Tech Lead), Marcos (Product Manager), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança).

## Índice

1. [Requisitos Funcionais (REQ)](#1-requisitos-funcionais-req)
2. [Decisões (DEC)](#2-decisões-dec)
3. [Definições Técnicas Principais (PTD)](#3-definições-técnicas-principais-ptd)
4. [Restrições (RES)](#4-restrições-res)
5. [Ganchos para o Código Pré-existente (SCL)](#5-ganchos-para-o-código-pré-existente-scl)
6. [Detalhes Técnicos Secundários (STD)](#6-detalhes-técnicos-secundários-std)
7. [Pontos Adiados para Fases Futuras (POS)](#7-pontos-adiados-para-fases-futuras-pos)
8. [Pontos Descartados (DIS)](#8-pontos-descartados-dis)
9. [Observações](#9-observações)

---

## 1. Requisitos Funcionais (REQ)

Comportamentos e capacidades que o sistema **DEVE** oferecer, levantados principalmente por Marcos (produto) e detalhados pelo time.

| ID      | Descrição                                                                                                                                                                                                              | Responsável   | Fonte                            |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | -------------------------------- |
| REQ-001 | O sistema **DEVE** notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) via webhook quando o status de um pedido mudar, em substituição ao polling no `GET /orders`                                    | Marcos (09:00) | [18](../TRANSCRICAO.md#L18)      |
| REQ-002 | Os webhooks são **somente outbound** (da plataforma para o cliente); o cliente não envia webhooks para a plataforma                                                                                                    | Marcos (09:02) | [26](../TRANSCRICAO.md#L26)      |
| REQ-003 | O sistema **DEVE** expor `POST` para cadastrar webhook com `url` e lista de status desejados; a `secret` é **gerada pela plataforma** e devolvida na criação                                                            | Marcos (09:31) | [184](../TRANSCRICAO.md#L184)    |
| REQ-004 | O `customer_id` **DEVE** ser informado no body ou no path (**NÃO** vem do JWT, pois o JWT é de um usuário do sistema que representa o cliente)                                                                          | Larissa (09:32) | [190](../TRANSCRICAO.md#L190)    |
| REQ-005 | O sistema **DEVE** expor `PATCH` (editar), `DELETE` (remover) e `GET` (listar webhooks de um customer)                                                                                                                 | Bruno (09:33) | [192](../TRANSCRICAO.md#L192)    |
| REQ-006 | Cada webhook **DEVE** ter filtro de eventos (lista de status que deseja ouvir, ex.: só `SHIPPED` e `DELIVERED`); o filtro é aplicado na inserção na outbox, e nada é inserido se nenhum webhook do customer quer o status | Marcos (09:33), Bruno (09:34) | [194](../TRANSCRICAO.md#L194), [198](../TRANSCRICAO.md#L198) |
| REQ-007 | O sistema **DEVE** expor `GET /webhooks/:id/deliveries` com o histórico de entregas (últimos 100: sucesso/falha, payload, response, tempo de resposta)                                                                  | Marcos (09:34) | [202](../TRANSCRICAO.md#L202)    |
| REQ-008 | O sistema **DEVE** expor endpoint para o cliente solicitar nova secret (rotação); a secret antiga permanece válida por 24h em paralelo                                                                                 | Sofia (09:21) | [130](../TRANSCRICAO.md#L130)    |
| REQ-009 | O sistema **DEVE** expor `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento da DLQ na outbox como pendente                                                                                           | Diego (09:18) | [114](../TRANSCRICAO.md#L114)    |
| REQ-010 | O replay **DEVE** exigir role `ADMIN` e **DEVE** registrar em log quem o executou (auditoria)                                                                                                                          | Sofia (09:36) | [210](../TRANSCRICAO.md#L210)    |
| REQ-011 | O payload **DEVE** ser JSON com `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order (ex.: `total_cents`); **NÃO** inclui `items` | Diego (09:43) | [256](../TRANSCRICAO.md#L256)    |
| REQ-012 | Cada requisição **DEVE** conter os headers `X-Event-Id` (UUID), `X-Signature` (HMAC), `X-Timestamp` (momento do envio), `Content-Type: application/json` e `X-Webhook-Id` (id do endpoint cadastrado)                    | Diego (09:44), Sofia (09:44) | [262](../TRANSCRICAO.md#L262), [264](../TRANSCRICAO.md#L264) |
| REQ-013 | A URL do webhook **DEVE** ser `https`; cadastro com `http` **DEVE** ser recusado com erro de validação                                                                                                                 | Sofia (09:23) | [138](../TRANSCRICAO.md#L138)    |
| REQ-014 | Eventos com payload acima de 64KB **NÃO** devem ser enviados e **DEVEM** resultar em erro (sem truncar)                                                                                                                | Larissa (09:24) | [144](../TRANSCRICAO.md#L144)    |
| REQ-015 | O sistema **DEVE** garantir entrega at-least-once, permitindo ao cliente deduplicar por `X-Event-Id`                                                                                                                   | Larissa (09:26) | [158](../TRANSCRICAO.md#L158)    |

## 2. Decisões (DEC)

Escolhas fechadas pelo grupo entre alternativas discutidas (arquitetura, processo, escopo, prazo).

| ID      | Descrição                                                                                                                                         | Responsável   | Fonte                         |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------- |
| DEC-001 | **NÃO** disparar o webhook de forma síncrona dentro do service de orders                                                                          | Diego (09:06) | [44](../TRANSCRICAO.md#L44)   |
| DEC-002 | Adotar o padrão **outbox** no MySQL existente, em vez de fila externa                                                                             | Larissa (09:08) | [58](../TRANSCRICAO.md#L58)   |
| DEC-003 | O worker lê a outbox por **polling a cada 2 segundos**, aceitando 2s como latência mínima no pior caso                                            | Larissa (09:10) | [68](../TRANSCRICAO.md#L68)   |
| DEC-004 | O worker roda como **processo separado** da API                                                                                                    | Diego (09:11) | [70](../TRANSCRICAO.md#L70)   |
| DEC-005 | Ordenação garantida apenas **por `order_id`** e enquanto houver **um único worker**; sem garantia de ordering global (limitação conhecida)         | Larissa (09:13) | [86](../TRANSCRICAO.md#L86)   |
| DEC-006 | Retry com backoff exponencial: **5 tentativas** com intervalos 1m / 5m / 30m / 2h / 12h; depois, DLQ                                              | Larissa (09:17) | [108](../TRANSCRICAO.md#L108) |
| DEC-007 | DLQ em **tabela separada** (`webhook_dead_letter`) com payload, motivo da falha e timestamp                                                       | Diego (09:18) | [110](../TRANSCRICAO.md#L110) |
| DEC-008 | Reprocessamento da DLQ é **manual**, via endpoint admin                                                                                           | Diego (09:18) | [114](../TRANSCRICAO.md#L114) |
| DEC-009 | Assinatura **HMAC-SHA256** sobre o corpo do request, **secret única por endpoint**, rotação com grace period de 24h                               | Sofia (09:22) | [134](../TRANSCRICAO.md#L134) |
| DEC-010 | Garantia **at-least-once** com `X-Event-Id` para deduplicação do lado do cliente (padrão de mercado, como Stripe e GitHub)                        | Larissa (09:26) | [158](../TRANSCRICAO.md#L158) |
| DEC-011 | Reuso máximo do que já existe: `AppError`, Pino, error middleware, padrão de módulos, schemas Zod e códigos de erro; webhook é um módulo como os demais | Larissa (09:30) | [180](../TRANSCRICAO.md#L180) |
| DEC-012 | O replay de DLQ exige role `ADMIN`, reaproveitando o `requireRole` existente                                                                      | Larissa (09:36) | [212](../TRANSCRICAO.md#L212) |
| DEC-013 | O restante do CRUD de configuração aceita **qualquer role autenticada** por enquanto                                                              | Sofia (09:37) | [216](../TRANSCRICAO.md#L216) |
| DEC-014 | Filtrar eventos **na inserção da outbox**, e não no envio                                                                                         | Bruno (09:34) | [198](../TRANSCRICAO.md#L198) |
| DEC-015 | Chaves primárias das novas tabelas em **UUID**, seguindo o padrão do projeto                                                                      | Larissa (09:51) | [304](../TRANSCRICAO.md#L304) |
| DEC-016 | A outbox guarda o **payload já renderizado (snapshot)** no momento da inserção                                                                    | Bruno (09:52) | [314](../TRANSCRICAO.md#L314) |
| DEC-017 | Prazo estimado de **3 sprints**, incluindo a revisão de segurança da Sofia no fim                                                                 | Larissa (09:47) | [276](../TRANSCRICAO.md#L276) |
| DEC-018 | Larissa abrirá o doc de design e marcará sessão de revisão com Bruno e Diego antes de codar                                                       | Larissa (09:50) | [292](../TRANSCRICAO.md#L292) |

## 3. Definições Técnicas Principais (PTD)

Definições de arquitetura, dados e contratos que guiam a implementação.

| ID      | Descrição                                                                                                                                                                  | Responsável   | Fonte                         |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------- |
| PTD-001 | Na mesma transação SQL que atualiza `orders` e `order_status_history`, inserir linha em `webhook_outbox`; falha na outbox causa rollback, e rollback descarta o evento      | Diego (09:06) | [48](../TRANSCRICAO.md#L48)   |
| PTD-002 | `webhook_outbox` com campo de status (`pendente`, `processando`, `falhou`, `entregue`) e índices em status e `created_at`; worker lê só pendentes, em batch pequeno          | Diego (09:08) | [56](../TRANSCRICAO.md#L56)   |
| PTD-003 | Worker em loop: a cada 2s busca os eventos pendentes mais antigos (por `created_at`), processa e marca                                                                     | Diego (09:09) | [60](../TRANSCRICAO.md#L60)   |
| PTD-004 | Nova entry-point `src/worker.ts` (análoga a `src/server.ts`) e script `npm run worker`                                                                                      | Larissa (09:11) | [72](../TRANSCRICAO.md#L72)   |
| PTD-005 | Lógica de processamento em arquivo do módulo (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`)                                                          | Bruno (09:28) | [166](../TRANSCRICAO.md#L166) |
| PTD-006 | Módulo `src/modules/webhooks` com controller, service, repository, routes e schemas                                                                                        | Bruno (09:27) | [162](../TRANSCRICAO.md#L162) |
| PTD-007 | Tabela de configuração de webhook com `url`, `secret`, `customer_id` e estado ativo                                                                                         | Bruno (09:21), Sofia (09:21) | [128](../TRANSCRICAO.md#L128), [130](../TRANSCRICAO.md#L130) |
| PTD-008 | Assinatura HMAC-SHA256 do payload enviada no header `X-Signature`; verificação feita pelo cliente                                                                          | Sofia (09:20) | [120](../TRANSCRICAO.md#L120) |
| PTD-009 | Rotação de secret: a nova passa a valer e a antiga continua válida por 24h antes de expirar                                                                                 | Sofia (09:21) | [130](../TRANSCRICAO.md#L130) |
| PTD-010 | `event_id` é um UUID gerado quando o evento entra na outbox e enviado em `X-Event-Id`                                                                                       | Diego (09:25) | [150](../TRANSCRICAO.md#L150) |
| PTD-011 | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` da transação atual, chamada pelo `order.service`; sem injetar repository inteiro             | Bruno (09:41), Diego (09:41) | [242](../TRANSCRICAO.md#L242), [244](../TRANSCRICAO.md#L244) |
| PTD-012 | Timeout de 10s na chamada HTTP do worker; estouro é tratado como falha e vai para retry                                                                                     | Diego (09:42) | [250](../TRANSCRICAO.md#L250) |
| PTD-013 | O worker usa **instância própria** de `PrismaClient` (um por processo), com o mesmo banco e a mesma `DATABASE_URL`                                                          | Bruno (09:30) | [178](../TRANSCRICAO.md#L178) |
| PTD-014 | Códigos de erro do módulo com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), com classes específicas derivadas de `AppError` | Bruno (09:28), Larissa (09:29) | [170](../TRANSCRICAO.md#L170), [172](../TRANSCRICAO.md#L172) |
| PTD-015 | Falha permanente (após a 5ª tentativa) move o evento para `webhook_dead_letter`; replay devolve à outbox como pendente                                                       | Diego (09:15), Diego (09:18) | [92](../TRANSCRICAO.md#L92), [114](../TRANSCRICAO.md#L114) |

## 4. Restrições (RES)

Limites, requisitos não funcionais e condições que a solução **DEVE** respeitar.

| ID      | Descrição                                                                                                                                         | Responsável   | Fonte                         |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------- |
| RES-001 | Latência de notificação **abaixo de 10 segundos** (é o que o cliente considera "tempo real")                                                      | Marcos (09:02) | [22](../TRANSCRICAO.md#L22)   |
| RES-002 | A mudança de status **NÃO** pode depender de HTTP externo; cliente lento ou fora do ar não pode travar nem reverter a transação                   | Bruno (09:04) | [32](../TRANSCRICAO.md#L32), [36](../TRANSCRICAO.md#L36) |
| RES-003 | **NÃO** pode existir caso em que o status muda e o evento não é registrado: a inserção na outbox é obrigatoriamente na mesma transação            | Bruno (09:40), Diego (09:41) | [238](../TRANSCRICAO.md#L238), [240](../TRANSCRICAO.md#L240) |
| RES-004 | Time pequeno: sem subir infraestrutura nova (Redis etc.); usar o MySQL existente                                                                   | Diego (09:07) | [52](../TRANSCRICAO.md#L52)   |
| RES-005 | O worker **NÃO** pode rodar na mesma instância/processo da API                                                                                     | Diego (09:11) | [76](../TRANSCRICAO.md#L76)   |
| RES-006 | Ordenação só por `order_id` e só com single-worker; escalar para múltiplos workers em paralelo perde a garantia                                    | Diego (09:12) | [80](../TRANSCRICAO.md#L80)   |
| RES-007 | A URL do webhook **DEVE** ser HTTPS (TLS obrigatório), validada no schema Zod                                                                      | Sofia (09:23) | [138](../TRANSCRICAO.md#L138) |
| RES-008 | Tamanho máximo de payload: 64KB (requisito não funcional, não decisão arquitetural)                                                                | Larissa (09:24) | [144](../TRANSCRICAO.md#L144) |
| RES-009 | A secret **DEVE** ser única por endpoint; **NÃO** pode haver secret global da plataforma                                                           | Sofia (09:21) | [126](../TRANSCRICAO.md#L126) |
| RES-010 | Entrega at-least-once: o cliente **DEVE** estar preparado para receber o mesmo evento mais de uma vez                                              | Diego (09:24) | [146](../TRANSCRICAO.md#L146) |
| RES-011 | O replay de DLQ **DEVE** ser restrito à role `ADMIN`                                                                                               | Sofia (09:36) | [210](../TRANSCRICAO.md#L210) |
| RES-012 | **NÃO** introduzir novas bibliotecas de log nem mudar o error middleware; usar o que o projeto já tem                                              | Bruno (09:29) | [174](../TRANSCRICAO.md#L174) |
| RES-013 | Prazo: a Atlas quer a entrega até o fim de novembro (risco de migrar para o concorrente se não entregue até o fim do trimestre)                    | Marcos (09:45), Marcos (09:00) | [270](../TRANSCRICAO.md#L270), [18](../TRANSCRICAO.md#L18) |
| RES-014 | Reservar no mínimo 2 dias úteis para a revisão de segurança (HMAC e geração de secret) antes do deploy                                             | Sofia (09:46) | [274](../TRANSCRICAO.md#L274) |

## 5. Ganchos para o Código Pré-existente (SCL)

Pontos de contato com código, padrões e infraestrutura que já existem no projeto.

| ID      | Descrição                                                                                                                                                         | Responsável   | Fonte                         |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------- |
| SCL-001 | `order.service`, método `changeStatus`: a transação atual (update em `orders`, insert em `order_status_history`, decremento de `stock_quantity`) receberá também o insert na outbox | Bruno (09:40) | [238](../TRANSCRICAO.md#L238) |
| SCL-002 | `src/server.ts` é o modelo de entry-point para o novo `src/worker.ts`                                                                                              | Larissa (09:11) | [72](../TRANSCRICAO.md#L72)   |
| SCL-003 | Padrão de módulos em `src/modules` (controller, service, repository, routes, schemas)                                                                              | Bruno (09:27) | [162](../TRANSCRICAO.md#L162) |
| SCL-004 | Classe `AppError` e derivadas (`InsufficientStockError`, `InvalidStatusTransitionError`) com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`          | Bruno (09:28) | [170](../TRANSCRICAO.md#L170) |
| SCL-005 | Logger Pino já usado em todo o projeto                                                                                                                             | Bruno (09:29) | [174](../TRANSCRICAO.md#L174) |
| SCL-006 | Middleware de erro centralizado já trata `AppError`, Zod e Prisma; captura os novos erros sem mudanças                                                             | Bruno (09:29) | [174](../TRANSCRICAO.md#L174) |
| SCL-007 | `PrismaClient`, banco MySQL e `DATABASE_URL` existentes (o worker cria instância própria)                                                                          | Bruno (09:30) | [178](../TRANSCRICAO.md#L178) |
| SCL-008 | `requireRole` existente, reaproveitado para exigir `ADMIN` no replay                                                                                                | Larissa (09:36) | [212](../TRANSCRICAO.md#L212) |
| SCL-009 | Padrão de schemas Zod do projeto (validação de HTTPS na URL)                                                                                                        | Sofia (09:23), Larissa (09:30) | [138](../TRANSCRICAO.md#L138), [180](../TRANSCRICAO.md#L180) |
| SCL-010 | Autenticação por JWT existente; usuários do sistema representam os clientes e o JWT atual é de usuário operador                                                    | Bruno (09:32), Marcos (09:32) | [186](../TRANSCRICAO.md#L186), [188](../TRANSCRICAO.md#L188) |
| SCL-011 | `GET /orders` (hoje usado em polling pelos clientes) e `GET /orders/:id` (para detalhes que o payload não traz)                                                    | Marcos (09:00), Diego (09:43) | [18](../TRANSCRICAO.md#L18), [256](../TRANSCRICAO.md#L256) |
| SCL-012 | Convenção de UUID como identificador em todo o projeto                                                                                                              | Larissa (09:51) | [304](../TRANSCRICAO.md#L304) |

## 6. Detalhes Técnicos Secundários (STD)

Informações de apoio, justificativas e detalhes de menor impacto arquitetural.

| ID      | Descrição                                                                                                                                         | Responsável   | Fonte                         |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------- |
| STD-001 | O MySQL não tem listener nativo (como `NOTIFY/LISTEN` do Postgres); trigger executa só SQL e não notifica processo externo                         | Diego (09:09) | [64](../TRANSCRICAO.md#L64)   |
| STD-002 | A janela total do backoff é de quase 15h entre a primeira falha e a última tentativa (1m+5m+30m+2h+12h)                                            | Diego (09:17) | [104](../TRANSCRICAO.md#L104) |
| STD-003 | Por que 5 tentativas: 3 cobriria apenas ~30 min e já houve cliente com 2h de indisponibilidade planejada                                           | Diego (09:16) | [100](../TRANSCRICAO.md#L100) |
| STD-004 | Se escalar para múltiplos workers, é possível particionar por `order_id` ou usar lock pessimista                                                   | Diego (09:13) | [84](../TRANSCRICAO.md#L84)   |
| STD-005 | `X-Timestamp` permite ao cliente detectar replay attack, se quiser                                                                                  | Diego (09:44) | [262](../TRANSCRICAO.md#L262) |
| STD-006 | `X-Webhook-Id` ajuda clientes com vários cadastros a saber qual deles originou o envio                                                              | Sofia (09:44) | [264](../TRANSCRICAO.md#L264) |
| STD-007 | Motivo de errar em vez de truncar payload grande: se chegou a esse tamanho, algo está errado                                                       | Sofia (09:23) | [140](../TRANSCRICAO.md#L140) |
| STD-008 | Motivo do snapshot: o evento reflete o estado do pedido no momento da mudança, mesmo que o pedido mude depois                                      | Larissa (09:52) | [310](../TRANSCRICAO.md#L310) |
| STD-009 | Já houve cliente que vazou secret em log de aplicação, justificando a rotação                                                                      | Diego (09:22) | [132](../TRANSCRICAO.md#L132) |
| STD-010 | Marcos documentará no portal de desenvolvedores a semântica at-least-once (em destaque) e a integração via API                                     | Marcos (09:26), Marcos (09:40) | [156](../TRANSCRICAO.md#L156), [236](../TRANSCRICAO.md#L236) |
| STD-011 | Decomposição da estimativa: outbox/DLQ (1 sprint), worker/retry (1), CRUD/deliveries (0,5), integração no `order.service`/E2E (0,5), HMAC/schemas/validações (resto) | Larissa (09:46) | [272](../TRANSCRICAO.md#L272) |
| STD-012 | Marcos confirmará o prazo com a Atlas e atualizará os clientes hoje à tarde                                                                          | Marcos (09:47), Marcos (09:49) | [278](../TRANSCRICAO.md#L278), [290](../TRANSCRICAO.md#L290) |
| STD-013 | Sofia pediu que a revisão de segurança seja agendada antes do deploy                                                                                | Sofia (09:49) | [288](../TRANSCRICAO.md#L288) |
| STD-014 | Os clientes nunca pediram ordering global; só querem saber se cada pedido mudou                                                                     | Marcos (09:14) | [88](../TRANSCRICAO.md#L88)   |

## 7. Pontos Adiados para Fases Futuras (POS)

Itens reconhecidos como relevantes, mas deliberadamente fora do escopo desta fase.

| ID      | Descrição                                                                                                          | Responsável   | Fonte                         |
| ------- | ------------------------------------------------------------------------------------------------------------------ | ------------- | ----------------------------- |
| POS-001 | E-mail de alerta ao cliente quando o webhook falhar (ex.: 3 falhas seguidas); talvez na próxima fase, após medir o impacto | Larissa (09:37) | [220](../TRANSCRICAO.md#L220) |
| POS-002 | Rate limiting de envio por cliente: observar e implementar apenas se virar problema (ponto em aberto)              | Larissa (09:39) | [230](../TRANSCRICAO.md#L230) |
| POS-003 | Escalar para múltiplos workers (particionamento por `order_id` ou lock pessimista) e ordering além do single-worker | Diego (09:13) | [84](../TRANSCRICAO.md#L84)   |
| POS-004 | Endurecer as permissões do CRUD de configuração (hoje qualquer role autenticada)                                   | Sofia (09:37) | [216](../TRANSCRICAO.md#L216) |
| POS-005 | Arquivamento das linhas entregues da outbox (cerca de 30 dias), fora do escopo desta feature                       | Diego (09:08) | [56](../TRANSCRICAO.md#L56)   |

## 8. Pontos Descartados (DIS)

Alternativas consideradas e rejeitadas, com o motivo quando mencionado.

| ID      | Descrição                                                                                                                       | Responsável   | Fonte                         |
| ------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------- |
| DIS-001 | Disparo síncrono no service de orders: um cliente lento travaria mudanças de status e não há rollback viável se o cliente cai   | Bruno (09:04) | [32](../TRANSCRICAO.md#L32), [36](../TRANSCRICAO.md#L36) |
| DIS-002 | Redis Streams / Redis Cluster: exigiria mais infra, overengineering para um time pequeno                                        | Diego (09:07) | [52](../TRANSCRICAO.md#L52)   |
| DIS-003 | Trigger de banco para acionar o worker: não notifica processo externo e exigiria gambiarras                                     | Diego (09:09) | [64](../TRANSCRICAO.md#L64)   |
| DIS-004 | Retry indefinido com backoff: deixaria evento pendurado para sempre se o cliente sumisse                                        | Diego (09:15) | [96](../TRANSCRICAO.md#L96)   |
| DIS-005 | Apenas 3 tentativas: janela curta demais para indisponibilidades longas                                                          | Diego (09:16) | [100](../TRANSCRICAO.md#L100) |
| DIS-006 | Marcar como `failed` na própria outbox em vez de tabela DLQ separada (preterido por clareza de leitura e uso em debug/reprocesso) | Diego (09:18) | [110](../TRANSCRICAO.md#L110) |
| DIS-007 | Secret global da plataforma: se vazasse, vazaria tudo                                                                           | Sofia (09:21) | [126](../TRANSCRICAO.md#L126) |
| DIS-008 | Truncar payload acima do limite (opta-se por erro)                                                                               | Sofia (09:23) | [140](../TRANSCRICAO.md#L140) |
| DIS-009 | Entrega exactly-once: exigiria coordenação dos dois lados e muita complexidade                                                  | Diego (09:25) | [154](../TRANSCRICAO.md#L154) |
| DIS-010 | Reaproveitar a mesma instância de `PrismaClient` no worker (é outro processo Node)                                               | Bruno (09:30) | [178](../TRANSCRICAO.md#L178) |
| DIS-011 | Obter o `customer_id` do JWT (o JWT é de usuário operador/representante, não do cliente)                                         | Larissa (09:32) | [190](../TRANSCRICAO.md#L190) |
| DIS-012 | Dashboard visual para o cliente: só endpoints; painel é projeto separado do time de frontend                                    | Larissa (09:40) | [234](../TRANSCRICAO.md#L234) |
| DIS-013 | Injetar o repository de webhook inteiro no `OrderService` (preferida a função `publishWebhookEvent` recebendo `tx`)             | Diego (09:41) | [244](../TRANSCRICAO.md#L244) |
| DIS-014 | Incluir `items` do pedido no payload (cliente consulta `GET /orders/:id` se precisar)                                            | Diego (09:43) | [256](../TRANSCRICAO.md#L256) |
| DIS-015 | ID auto incremental na outbox (usa-se UUID)                                                                                      | Larissa (09:51) | [304](../TRANSCRICAO.md#L304) |
| DIS-016 | Guardar só `order_id` na outbox e renderizar o payload no envio (usa-se snapshot)                                                | Larissa (09:52) | [310](../TRANSCRICAO.md#L310) |

## 9. Observações

Pontos que não se encaixam nas categorias acima, ambiguidades e lacunas identificadas na transcrição.

1. **Tabela de entregas não discutida.** O `GET /webhooks/:id/deliveries` exige guardar payload, response, tempo de resposta e resultado de cada tentativa ([202](../TRANSCRICAO.md#L202)), mas nenhuma tabela para isso foi definida. Só foram citadas `webhook_outbox` ([48](../TRANSCRICAO.md#L48)) e `webhook_dead_letter` ([110](../TRANSCRICAO.md#L110)).
2. **Ambiguidade sobre o endpoint admin.** Após o requisito de deliveries, Larissa diz "Esse endpoint admin pra reprocessar DLQ que o Diego falou" ([204](../TRANSCRICAO.md#L204)), o que parece ser um comentário solto. Diego esclarece em seguida que se trata do `POST /admin/webhooks/dead-letter/:id/replay` ([206](../TRANSCRICAO.md#L206)); `GET /webhooks/:id/deliveries` é um endpoint distinto, voltado ao cliente.
3. **Sobreposição entre status `falhou` e a DLQ.** A outbox prevê o status `falhou` ([56](../TRANSCRICAO.md#L56)) e a DLQ fica em tabela separada ([110](../TRANSCRICAO.md#L110)). Não ficou definido se `falhou` indica falha transitória (aguardando retry) ou permanente (antes de mover para a DLQ).
4. **Contagem das 5 tentativas.** Não foi esclarecido se a primeira tentativa de envio entra nas 5, ou se são 5 retries após a falha inicial ([104](../TRANSCRICAO.md#L104), [108](../TRANSCRICAO.md#L108)). Os intervalos 1m/5m/30m/2h/12h sugerem 5 retries; "total 5 tentativas" no resumo ([282](../TRANSCRICAO.md#L282)) sugere o contrário.
5. **Janela de retry divergente.** Diego afirma que 5 tentativas cobrem "12 ou 24 horas" ([96](../TRANSCRICAO.md#L96)) e depois calcula "quase 15 horas" ([104](../TRANSCRICAO.md#L104)). A soma dos intervalos é ~14,6h, então o valor de referência é o segundo.
6. **Estado `processando` sem tratamento de falha do worker.** Se o worker cair com eventos em `processando` ([56](../TRANSCRICAO.md#L56)), não foi discutido como recuperá-los (timeout/lease).
7. **Autorização por customer não discutida.** O CRUD aceita qualquer role autenticada ([216](../TRANSCRICAO.md#L216)) e o `customer_id` vem do body/path ([190](../TRANSCRICAO.md#L190)). Não foi tratado se um usuário só pode acessar webhooks do próprio cliente, o que combina com a nota de Sofia sobre endurecer isso depois.
8. **Armazenamento da secret.** Foi decidido que a secret é gerada pela plataforma e devolvida na criação ([184](../TRANSCRICAO.md#L184)). Não foi definido se ela é armazenada em texto claro ou criptografada, nem se pode ser consultada depois. Como o HMAC exige a secret para assinar ([120](../TRANSCRICAO.md#L120)), o armazenamento reversível é implícito. É o ponto que a Sofia quer revisar com calma ([274](../TRANSCRICAO.md#L274)).
9. **Rotação e assinatura.** Durante o grace period de 24h ([130](../TRANSCRICAO.md#L130)), não ficou definido com qual secret o payload é assinado nem se envia as duas assinaturas.
10. **Lista de status do filtro.** Os status citados na conversa são `PAID`, `PROCESSING`, `SHIPPED` e `DELIVERED` ([78](../TRANSCRICAO.md#L78), [194](../TRANSCRICAO.md#L194)). Não há lista completa do enum de status de pedido.
11. **Inconsistência de tempo na reunião.** Larissa diz "estamos a uma hora de reunião" às 09:27 ([160](../TRANSCRICAO.md#L160)), mas só haviam passado 27 minutos pelos timestamps. A reunião termina às 09:53 ([324](../TRANSCRICAO.md#L324)); o cabeçalho informa ~55 minutos ([4](../TRANSCRICAO.md#L4)).
12. **Decisões tomadas após o resumo final.** UUID como identificador ([304](../TRANSCRICAO.md#L304)) e snapshot do payload na outbox ([314](../TRANSCRICAO.md#L314)) foram decididos depois do resumo e da saída de Marcos e Sofia ([294](../TRANSCRICAO.md#L294), [296](../TRANSCRICAO.md#L296)), sem a confirmação deles. Convém validá-las no doc de design.
13. **Contexto de negócio.** Os três clientes que originaram a demanda são Atlas Comercial, MaxDistribuição e Nova Cargo ([18](../TRANSCRICAO.md#L18)). A Atlas condicionou a permanência à entrega até o fim do trimestre.
14. **Classificação de itens fronteiriços.** HTTPS ([138](../TRANSCRICAO.md#L138)) e limite de 64KB ([144](../TRANSCRICAO.md#L144)) aparecem em REQ e RES porque são validações funcionais e também requisitos não funcionais. Larissa os classifica como requisito não funcional, e não como decisão arquitetural ([144](../TRANSCRICAO.md#L144)).
