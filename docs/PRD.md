# PRD: Sistema de Webhooks de Notificação de Pedidos

---
**Versão:** v1  
**Data:** 2026-10-29  
**Responsável:** Marcos (Product Manager)  
---

> Os itens deste documento têm identificador (por exemplo, `PRD-FR-01`). A origem de cada um (fala da reunião ou arquivo do código) está no [TRACKER](TRACKER.md). O que **não** foi dito na reunião está marcado como *(derivação)*. Este PRD fica no nível de produto; a arquitetura está no [RFC](RFC.md), as decisões nos [ADRs](adrs/README.md) e a implementação no [FDD](FDD.md).

## Resumo

Esta feature implementa **webhooks de saída** para que clientes B2B sejam notificados automaticamente quando o status de um pedido muda no OMS, em menos de 10 segundos. Hoje eles consultam `GET /orders` de tempos em tempos, o que deixa a integração lenta e cara. O cliente cadastra um endpoint `https`, escolhe os status de interesse e recebe um evento assinado a cada mudança. A solução usa o padrão outbox no MySQL existente, com um worker separado, retry com backoff e fila de falhas (DLQ), sem infraestrutura nova. O prazo estimado é de 3 sprints, para a entrega pedida pela Atlas até o fim de novembro.

---

## Contexto e problema

### Público-alvo
* **[PRD-CTX-01]** **Clientes B2B** que integram com a plataforma por API: Atlas Comercial, MaxDistribuição e Nova Cargo, que fizeram o pedido formal na semana anterior à reunião.
* **[PRD-CTX-02]** **Usuários do sistema que representam o cliente**, que cadastram os webhooks diretamente pela nossa API, autenticados com JWT.
* **[PRD-CTX-03]** **Administradores da plataforma**, responsáveis por reprocessar eventos que falharam.
* **[PRD-CTX-04]** **Time de produto**, que documenta a integração no portal de desenvolvedores.

### Cenários de uso chave
* **[PRD-CTX-05]** O cliente cadastra a URL do seu sistema e recebe a secret; a partir daí, é avisado quando o pedido muda de status, sem precisar consultar a API.
* **[PRD-CTX-06]** O cliente escolhe ouvir apenas alguns status, por exemplo `SHIPPED` e `DELIVERED`.
* **[PRD-CTX-07]** O cliente consulta o histórico das últimas entregas para investigar uma falha: sucesso ou falha, payload, resposta e tempo de resposta.
* **[PRD-CTX-08]** O cliente rotaciona a secret depois de um vazamento, sem derrubar a integração, porque a antiga continua válida por 24h.
* **[PRD-CTX-09]** Um administrador reprocessa um evento que esgotou as tentativas.

### Onde essa feature será implantada
* **[PRD-CTX-10]** No OMS atual (Node.js, TypeScript, MySQL via Prisma), como um novo módulo `webhooks` ao lado de `auth`, `users`, `customers`, `products` e `orders`.
* **[PRD-CTX-11]** Um **novo processo worker** roda junto da API, com o mesmo banco.
* Consumida por sistemas dos clientes e documentada no portal de desenvolvedores.

### Problemas priorizados
* **[PRD-PRB-01]** Os clientes fazem polling em `GET /orders`, o que torna a integração lenta e cara para eles.
* **[PRD-PRB-02]** Risco comercial: a Atlas indicou que pode migrar para o concorrente se a entrega não ocorrer até o fim do trimestre.
* **[PRD-PRB-03]** A aplicação não tem nenhum mecanismo de notificação externa. Chamar HTTP dentro da mudança de status travaria outros pedidos e não permite rollback se o cliente estiver fora do ar.

---

## Objetivos e métricas

| Objetivo | Métrica | Meta |
| :--- | :--- | :--- |
| **[PRD-OBJ-01]** Notificar o cliente "em tempo real" | Tempo entre a mudança de status e o envio do evento ao endpoint do cliente | **Menos de 10 segundos**; latência mínima de 2s pelo polling |
| **[PRD-OBJ-02]** Eliminar o polling dos clientes que pediram a feature | Clientes com webhook ativo entre Atlas, MaxDistribuição e Nova Cargo *(derivação)* | **3 de 3** até o fim de novembro |
| **[PRD-OBJ-03]** Nunca alterar um status sem gerar o evento | Mudanças de status commitadas sem evento correspondente na outbox *(derivação)* | **0**: "não pode ter caso de status mudar e evento não sair" |
| **[PRD-OBJ-04]** Cobrir indisponibilidades temporárias do cliente | Janela de retry antes de falha permanente | **~15 horas**, em 5 tentativas |
| **[PRD-OBJ-05]** Entregar no prazo | Duração do projeto | **3 sprints**, incluindo revisão de segurança |

---

## Escopo

### Incluso
* Cadastro, edição, remoção e listagem de webhooks por customer, com filtro de eventos por status.
* Notificação de mudança de status de pedido, com payload enxuto e assinado.
* Rotação de secret com período de transição de 24h.
* Retry com backoff exponencial e fila de falhas (DLQ) com replay manual por administradores.
* Histórico de entregas por webhook.
* Documentação da integração no portal de desenvolvedores, a cargo do time de produto.

### Fora de escopo
* **[PRD-ESC-01]** **E-mail de alerta ao cliente** quando o webhook falha repetidamente: "fora de escopo dessa fase", talvez na próxima.
* **[PRD-ESC-02]** **Rate limiting de saída** por cliente: "observar e decidir depois".
* **[PRD-ESC-03]** **Dashboard visual** para o cliente: "só endpoints"; o painel é projeto separado do time de frontend.
* **[PRD-ESC-04]** **Múltiplos workers e ordering global**: a garantia vale só por pedido e com um único worker.
* **[PRD-ESC-05]** **Arquivamento** das linhas entregues da outbox.
* **[PRD-ESC-06]** **Endurecer as permissões** do CRUD de configuração, que hoje aceita qualquer role autenticada.
* **[PRD-ESC-07]** **Webhooks de entrada**: o cliente só recebe.
* **[PRD-ESC-08]** **Entrega exactly-once**: a garantia é at-least-once.

---

## Requisitos funcionais

### [PRD-FR-01] Notificação de mudança de status de pedido
Quando o status de um pedido muda, o sistema notifica os webhooks do customer interessados naquele status.

* **Fluxo principal:**
  1. Um operador altera o status do pedido (`PATCH /orders/:id/status`).
  2. O sistema registra o evento na mesma transação da mudança de status.
  3. Um processo em segundo plano envia o evento ao endpoint do cliente.
* **Fluxos alternativos e exceções:**
  * Se nenhum webhook do customer escuta aquele status, nenhum evento é gerado.
  * Se o registro do evento falhar, a mudança de status também falha.
* **Erros previstos:** endpoint do cliente indisponível ou lento (ver PRD-FR-09).
* **Prioridade:** Alta

### [PRD-FR-02] Cadastro de webhook
O cliente cadastra um webhook informando a `url`, os status de interesse e o `customer_id`. A `secret` é gerada pela plataforma e devolvida na criação.

* **Fluxo principal:**
  1. O usuário autenticado envia `POST` com `customerId`, `url` e a lista de status.
  2. O sistema valida os dados, gera a secret e grava o webhook.
  3. A resposta devolve o webhook e a secret.
* **Fluxos alternativos e exceções:** o `customer_id` vem no body ou no path, e não do JWT.
* **Erros previstos:** URL que não é `https` é recusada com erro de validação; dados inválidos.
* **Prioridade:** Alta

### [PRD-FR-03] Edição, remoção e listagem de webhooks
O cliente edita (`PATCH`), remove (`DELETE`) e lista (`GET`) os webhooks de um customer.

* **Fluxo principal:** o usuário autenticado chama o endpoint correspondente e recebe a configuração atualizada, a confirmação da remoção ou a lista.
* **Fluxos alternativos e exceções:** qualquer role autenticada pode usar o CRUD, por enquanto.
* **Erros previstos:** webhook inexistente; dados inválidos.
* **Prioridade:** Alta

### [PRD-FR-04] Filtro de eventos por status
Cada webhook define a lista de status que quer ouvir; por exemplo, só `SHIPPED` e `DELIVERED`.

* **Fluxo principal:** a mudança de status gera evento apenas para os webhooks cuja lista inclui o novo status; o filtro é aplicado na inserção na outbox, e não no envio.
* **Fluxos alternativos e exceções:** se nenhum webhook quer o status, nada é registrado, o que economiza linhas na tabela.
* **Erros previstos:** lista com status inválido *(derivação)*.
* **Prioridade:** Alta

### [PRD-FR-05] Histórico de entregas
O cliente consulta os últimos 100 envios de um webhook, com resultado (sucesso ou falha), payload, resposta e tempo de resposta, em `GET /webhooks/:id/deliveries`.

* **Fluxo principal:** o usuário autenticado consulta o histórico do seu webhook e investiga falhas.
* **Fluxos alternativos e exceções:** sem entregas, a lista vem vazia *(derivação)*.
* **Erros previstos:** webhook inexistente.
* **Prioridade:** Média

### [PRD-FR-06] Rotação de secret
O cliente pede uma nova secret pela API. A antiga continua válida por 24 horas e depois deixa de valer.

* **Fluxo principal:**
  1. O usuário solicita a rotação do webhook.
  2. O sistema gera a nova secret e a devolve.
  3. Durante 24h, a secret anterior continua aceita para dar tempo de migrar os sistemas do cliente.
* **Fluxos alternativos e exceções:** passadas as 24h, só a nova secret vale.
* **Erros previstos:** webhook inexistente.
* **Prioridade:** Alta

### [PRD-FR-07] Assinatura e headers de cada envio
Toda requisição enviada ao cliente é assinada com **HMAC-SHA256** sobre o corpo, com a secret do endpoint.

* **Fluxo principal:** o envio traz `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. O cliente verifica a assinatura do lado dele.
* **Fluxos alternativos e exceções:** o `X-Timestamp` permite ao cliente detectar replay, se quiser.
* **Erros previstos:** assinatura inválida é rejeitada pelo cliente, do lado dele.
* **Prioridade:** Alta

### [PRD-FR-08] Conteúdo do evento
O evento é um JSON com `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. Não inclui os `items`.

* **Fluxo principal:** o cliente recebe o evento e, se precisar de detalhes, consulta `GET /orders/:id`.
* **Fluxos alternativos e exceções:** o conteúdo reflete o estado do pedido no momento da mudança, mesmo que o pedido mude depois.
* **Erros previstos:** payload acima de 64KB não é enviado e gera erro (PRD-FR-12).
* **Prioridade:** Alta

### [PRD-FR-09] Retry e fila de falhas (DLQ)
Se o endpoint do cliente falha ou não responde em 10 segundos, o sistema tenta de novo com backoff exponencial: **5 tentativas**, com intervalos de 1 min, 5 min, 30 min, 2 h e 12 h.

* **Fluxo principal:** a cada falha o evento é reagendado; esgotadas as tentativas, é movido para uma tabela de falhas separada, com payload, motivo e horário.
* **Fluxos alternativos e exceções:** o cliente que fica fora do ar por mais de ~15h depende do replay manual (PRD-FR-10).
* **Erros previstos:** timeout, erro de rede e resposta de erro do cliente.
* **Prioridade:** Alta

### [PRD-FR-10] Replay manual de eventos da DLQ
Um administrador reprocessa um evento da DLQ por `POST /admin/webhooks/dead-letter/:id/replay`, que o recoloca como pendente.

* **Fluxo principal:** o administrador chama o endpoint, o evento volta para a fila, e quem fez o replay fica registrado em log para auditoria.
* **Fluxos alternativos e exceções:** só a role `ADMIN` pode usar o endpoint.
* **Erros previstos:** acesso negado para quem não é `ADMIN`; registro inexistente na DLQ.
* **Prioridade:** Média

### [PRD-FR-11] URL do webhook obrigatoriamente `https`
Cadastros com URL `http` são recusados com erro de validação.

* **Fluxo principal:** a validação ocorre no cadastro e na edição.
* **Fluxos alternativos e exceções:** nenhum; é regra de segurança.
* **Erros previstos:** erro de validação com código de webhook inválido.
* **Prioridade:** Alta

### [PRD-FR-12] Limite de tamanho do payload
O evento tem limite de **64KB**. Acima disso, o sistema não envia e retorna erro, sem truncar.

* **Fluxo principal:** o tamanho é verificado ao montar o evento.
* **Fluxos alternativos e exceções:** o payload é enxuto, então não se espera chegar perto do limite.
* **Erros previstos:** erro de payload grande.
* **Prioridade:** Média

### [PRD-FR-13] Entrega at-least-once com identificador para deduplicação
O sistema garante que o evento chega pelo menos uma vez, e o cliente pode receber o mesmo evento mais de uma vez. Cada evento leva um `X-Event-Id` único para que o cliente deduplique.

* **Fluxo principal:** o cliente guarda os `event_id` processados e ignora repetidos.
* **Fluxos alternativos e exceções:** o time de produto documenta isso de forma destacada no portal.
* **Erros previstos:** cliente que não deduplica pode processar o mesmo evento duas vezes.
* **Prioridade:** Alta

---

## Requisitos não funcionais

* **[PRD-NFR-01]** **Performance:** notificação em menos de 10 segundos; latência mínima de 2s no pior caso; timeout de 10s por chamada ao cliente.
* **[PRD-NFR-02]** **Disponibilidade e isolamento:** a mudança de status não pode depender de HTTP externo; o worker é um processo separado da API.
* **[PRD-NFR-03]** **Segurança e autorização:** HMAC-SHA256 por endpoint, secret única e rotacionável; HTTPS obrigatório; replay restrito a `ADMIN`, com auditoria; revisão de segurança antes do deploy, com no mínimo 2 dias úteis.
* **[PRD-NFR-04]** **Observabilidade:** logs estruturados com o logger Pino existente; métricas e tracing são proposta no [FDD](FDD.md) *(derivação)*.
* **[PRD-NFR-05]** **Confiabilidade e integridade de dados:** registro do evento atômico com a mudança de status, via outbox na mesma transação; entrega at-least-once; ordenação garantida apenas por pedido e com um único worker.
* **[PRD-NFR-06]** **Limites de dados:** payload de até 64KB.
* **[PRD-NFR-07]** **Compatibilidade e consistência:** reuso de `AppError`, Pino, error middleware, módulos em `src/modules` e prefixo `WEBHOOK_` nos códigos de erro, sem dependência nova.

---

## Arquitetura e abordagem

* **Abordagem:** a mudança de status grava o evento numa tabela outbox, na mesma transação; um worker separado lê a tabela por polling, assina e envia o evento, com retry, backoff e DLQ. Detalhes em [RFC](RFC.md) e [FDD](FDD.md).
* **Componentes:**
  * Módulo `webhooks` no OMS (CRUD, histórico, replay)
  * Tabelas `webhook_outbox` e `webhook_dead_letter`, mais a configuração dos webhooks, no MySQL existente
  * Worker em processo separado
* **Integrações:** o fluxo de pedidos (`OrderService.changeStatus`) e os endpoints dos clientes.
* **Decisões e trade-offs:**
  * **Decisão:** outbox no MySQL, e não fila externa ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
    * *Justificativa:* time pequeno, sem infraestrutura nova, atomicidade com a mudança de status.
    * *Trade-off:* mais uma escrita na transação de status.
  * **Decisão:** worker separado com polling de 2s ([ADR-002](adrs/ADR-002-worker-separado-polling.md)).
    * *Justificativa:* o MySQL não tem listener nativo, e 2s atende o requisito de 10s.
    * *Trade-off:* latência mínima de 2s e um único worker.
  * **Decisão:** 5 tentativas com backoff e DLQ ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)).
    * *Justificativa:* cobre indisponibilidades de horas sem deixar eventos pendurados.
    * *Trade-off:* acima de ~15h de queda, replay manual.
  * **Decisão:** HMAC-SHA256 com secret por endpoint ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).
    * *Justificativa:* o cliente valida origem e integridade, e um vazamento afeta um endpoint.
    * *Trade-off:* gestão e rotação de secrets.
  * **Decisão:** at-least-once com `X-Event-Id` ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)).
    * *Justificativa:* padrão de mercado, muito mais simples que exactly-once.
    * *Trade-off:* o cliente deduplica.
  * **Decisão:** reuso dos padrões do projeto ([ADR-006](adrs/ADR-006-reuso-padroes-do-projeto.md)).
    * *Justificativa:* velocidade e consistência.
    * *Trade-off:* herdar limitações atuais.

---

## Dependências

* **[PRD-DEP-01]** **Dependência externa:** os clientes precisam implementar a verificação do HMAC e a deduplicação por `X-Event-Id` do lado deles.
* **[PRD-DEP-02]** **Dependência organizacional:** revisão de segurança da Sofia antes do deploy, com 2 dias úteis reservados; confirmação do prazo com a Atlas, a cargo do Marcos; documentação no portal de desenvolvedores; sessão de revisão do doc de design com Bruno e Diego.
* **[PRD-DEP-03]** **Dependência técnica:** MySQL existente; Prisma e a máquina de estados de pedidos em `src/modules/orders`; processo adicional para o worker.

---

## Riscos e mitigação

* **[PRD-RSK-01]** **Cliente lento ou fora do ar**
  * **Probabilidade:** Alta | **Impacto:** Médio
  * **Mitigação:** timeout de 10s, retry com backoff e DLQ; o worker é isolado da API.
  * **Plano de contingência:** replay manual pelo administrador.

* **[PRD-RSK-02]** **Eventos duplicados processados em dobro pelo cliente**
  * **Probabilidade:** Média | **Impacto:** Médio
  * **Mitigação:** `X-Event-Id` em todo envio e documentação em destaque.
  * **Plano de contingência:** suporte orienta o cliente a deduplicar; o histórico de entregas ajuda a investigar.

* **[PRD-RSK-03]** **Vazamento da secret de um cliente**
  * **Probabilidade:** Média | **Impacto:** Alto
  * **Mitigação:** secret única por endpoint e rotação com 24h de transição; já houve vazamento em log de aplicação de um cliente.
  * **Plano de contingência:** o cliente rotaciona a secret pela API.

* **[PRD-RSK-04]** **Atraso na entrega frente ao prazo de fim de novembro**
  * **Probabilidade:** Média | **Impacto:** Alto
  * **Mitigação:** escopo enxuto (sem e-mail e sem dashboard), 3 sprints com a revisão da Sofia já incluída.
  * **Plano de contingência:** alinhar o prazo com a Atlas antes do estouro.

* **[PRD-RSK-05]** **Limitação de um único worker (gargalo e ordenação só por pedido)**
  * **Probabilidade:** Média | **Impacto:** Médio
  * **Mitigação:** limitação conhecida e documentada; os clientes não pediram ordering global.
  * **Plano de contingência:** particionar por pedido ou usar lock, numa fase futura.

---

## Critérios de aceitação

* [ ] Um usuário autenticado cadastra um webhook com `url` `https`, status de interesse e `customer_id`, e recebe a secret na resposta.
* [ ] Cadastro com URL `http` é recusado com erro de validação.
* [ ] É possível editar, remover e listar os webhooks de um customer.
* [ ] Ao mudar o status de um pedido, o endpoint do cliente recebe o evento em menos de 10 segundos, apenas se ele escuta aquele status.
* [ ] Se o registro do evento falhar, a mudança de status do pedido não é efetivada.
* [ ] O envio traz os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type`, e o cliente consegue validar o HMAC-SHA256.
* [ ] Falhas são reenviadas em 5 tentativas (1m, 5m, 30m, 2h, 12h) e, depois, o evento vai para a DLQ.
* [ ] Um `ADMIN` reprocessa um evento da DLQ, e um usuário sem a role recebe acesso negado; o replay fica registrado em log.
* [ ] O cliente consulta os últimos 100 envios com resultado, payload, resposta e tempo de resposta.
* [ ] Após rotacionar a secret, a antiga vale por 24h e depois deixa de valer.
* [ ] O evento não inclui os `items` e respeita o limite de 64KB.
* [ ] A revisão de segurança foi concluída antes do deploy.

---

## Testes e validação

* **Tipos de teste obrigatórios:**
  * **[PRD-TST-01]** Testes de integração do fluxo de mudança de status com a inserção na outbox, incluindo o rollback (: "integração no `order.service` e testes ponta a ponta").
  * **[PRD-TST-02]** Testes ponta a ponta do ciclo completo: evento, envio assinado, retry e DLQ.
  * **[PRD-TST-03]** Testes dos endpoints de CRUD, deliveries e replay, no estilo de `tests/orders.test.ts` (Vitest + Supertest, contra um MySQL real) *(derivação)*.
  * **[PRD-TST-04]** Revisão de segurança do código de HMAC e de geração de secret.
* **Estratégia de validação:**
  * **[PRD-TST-05]** Validar a latência abaixo de 10s e o comportamento de retry em ambiente de teste *(derivação)*.
  * **[PRD-TST-06]** Validar com os três clientes que pediram a feature (Atlas, MaxDistribuição e Nova Cargo) antes de desligar o polling deles *(derivação)*.
  * Os critérios técnicos de aceite detalhados estão na seção 13 do [FDD](FDD.md).
