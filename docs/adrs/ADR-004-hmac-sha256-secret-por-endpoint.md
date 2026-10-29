# ADR-004 — Autenticação do webhook com HMAC-SHA256 e secret por endpoint

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Sofia (Eng. de Segurança), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Larissa (Tech Lead)
- **Relacionados:** [ADR-005](ADR-005-at-least-once-x-event-id.md) · [ADR-006](ADR-006-reuso-padroes-do-projeto.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

**[ADR-004-C1]** Os eventos carregam dados de pedidos e saem da nossa infraestrutura para endpoints dos clientes. O cliente precisa conseguir validar que a requisição veio de nós e que o payload não foi adulterado no caminho.

Forças em jogo:
- **[ADR-004-C2]** Um cliente já vazou uma secret em log de aplicação.
- **[ADR-004-C3]** O cliente precisa de tempo para migrar seus sistemas quando a secret muda.
- **[ADR-004-C4]** A aplicação já usa JWT com `jsonwebtoken` e `bcrypt` para autenticação interna (`src/middlewares/auth.middleware.ts`, `src/modules/auth/auth.service.ts`). Esse mecanismo não serve para assinar chamadas de saída.

## Decisão

1. **[ADR-004-D1]** Assinar o **corpo do request com HMAC-SHA256** usando uma secret compartilhada com o cliente. A assinatura vai no header `X-Signature`.
2. **[ADR-004-D2]** Cada endpoint de webhook tem uma **secret única**, e não existe secret global. A tabela de configuração guarda `url`, `secret`, `customer_id` e estado ativo.
3. **[ADR-004-D3]** A secret é gerada pela plataforma e devolvida na criação do webhook.
4. **[ADR-004-D4]** A secret é **rotacionável** por endpoint na API. Ao rotacionar, a secret antiga continua válida por **24 horas**, e depois deixa de valer.
5. **[ADR-004-D5]** Complemento: a URL do webhook deve ser `https`. Cadastro com `http` é recusado com erro de validação no schema Zod.
6. **[ADR-004-D6]** O header `X-Timestamp` leva o momento do envio, para o cliente detectar replay se quiser.

Pontos que a reunião **não** definiu e que ficam como premissas abertas (ver [RFC](../RFC.md)):
- com qual secret o payload é assinado durante o grace period de 24h;
- como a secret é armazenada, já que o HMAC exige a secret em forma utilizável para assinar. Sofia pediu revisão cuidadosa da geração de secret e do HMAC antes do deploy.

## Alternativas consideradas

### A1. [ADR-004-A1] Secret global da plataforma
- **Prós:** gestão simples, uma única chave.
- **Contras:** se vazar, vaza para todos os clientes.
- **Motivo do descarte:** o risco de vazamento já se concretizou em um cliente.

### A2. [ADR-004-A2] Sem assinatura, confiando apenas em HTTPS
- **Prós:** nenhuma gestão de secret.
- **Contras:** o cliente não consegue provar a origem nem a integridade do payload.
- **Motivo do descarte:** não atende ao requisito de segurança levantado. Esta é uma alternativa plausível, não uma opção discutida em detalhe na reunião.

### A3. [ADR-004-A3] Secret rotacionada sem grace period
- **Prós:** a secret antiga morre na hora, o que reduz a janela de exposição.
- **Contras:** o cliente precisa trocar a secret em todos os seus sistemas de forma instantânea e coordenada.
- **Motivo do descarte:** definiu-se uma janela de 24 horas.

## Consequências

### Positivas
- Autenticidade e integridade verificáveis pelo cliente, com HMAC-SHA256 ("padrão de mercado",).
- Um vazamento fica contido em um único endpoint.
- A rotação não derruba a integração do cliente.

### Negativas
- A plataforma precisa guardar a secret em forma recuperável para assinar, o que cria um ponto sensível no banco.
- Durante 24 horas existem duas secrets válidas por endpoint, o que exige lógica extra e mais superfície para revisão.
- O cliente é obrigado a implementar a verificação do HMAC e a rotação do seu lado.

### Trade-off explícito
Aceitamos a complexidade de gerir uma secret por endpoint, com rotação e janela de transição, em troca de isolamento de vazamentos e de uma migração sem queda para o cliente.
