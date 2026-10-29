# ADR-006 — Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Data da decisão:** 2026-10-29
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-polling.md) · [ADR-007](ADR-007-publicacao-na-transacao-do-changestatus.md) · [ADR-008](ADR-008-autorizacao-crud-e-replay-admin.md) · [RFC](../RFC.md) · [FDD](../FDD.md)

## Contexto

**[ADR-006-C1]** O módulo de webhooks entra em uma base de código com convenções claras. A decisão é construir a feature dentro dessas convenções, sem introduzir tecnologias ou estilos paralelos.

O código existente que a feature vai reutilizar:
- **[ADR-006-C2]** **Módulos por domínio** em `src/modules/*` (`auth`, `users`, `customers`, `products`, `orders`), cada um com controller, service, repository, routes e schemas. A composição é manual em `src/app.ts` (`buildControllers`), e `src/routes/index.ts` monta os routers sob `/api/v1`.
- **[ADR-006-C3]** **Erros**: `AppError` em `src/shared/errors/app-error.ts` (com `statusCode`, `errorCode` e `details`) e subclasses em `src/shared/errors/http-errors.ts` (`NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, `InsufficientStockError`, entre outras). Os códigos são strings em caixa alta, como `INSUFFICIENT_STOCK`.
- **[ADR-006-C4]** **`src/middlewares/error.middleware.ts`**, que centraliza o tratamento de `AppError`, Zod e Prisma.
- **[ADR-006-C5]** **Logger Pino** em `src/shared/logger/index.ts`.
- **[ADR-006-C6]** **Autorização** com `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`.
- **[ADR-006-C7]** **Validação** Zod via `src/middlewares/validate.middleware.ts` e schemas por módulo.

## Decisão

Reuso máximo do que já existe: `AppError`, Pino, error middleware, padrão de módulos, padrão de schemas Zod e padrão de códigos de erro.

1. **[ADR-006-D1]** Criar o módulo `src/modules/webhooks` com controller, service, repository, routes e schemas, igual aos demais. O diretório ainda não existe.
2. **[ADR-006-D2]** Os erros do módulo usam classes específicas derivadas de `AppError`, com códigos prefixados por **`WEBHOOK_`**. Exemplos citados: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`.
3. **[ADR-006-D3]** O logger é o Pino que já existe. "Não vamos botar nada novo".
4. **[ADR-006-D4]** O error middleware centralizado já trata `AppError`, Zod e Prisma, então os novos erros passam por ele **sem mudança**.
5. **[ADR-006-D5]** A validação (incluindo URL `https`) usa os schemas Zod do projeto.
6. **[ADR-006-D6]** A restrição por role reaproveita o `requireRole` existente. Ver [ADR-008](ADR-008-autorizacao-crud-e-replay-admin.md).
7. **[ADR-006-D7]** O worker tem entry-point própria (`src/worker.ts`) e `PrismaClient` próprio, mas na mesma stack ([ADR-002](ADR-002-worker-separado-polling.md)).

## Alternativas consideradas

### A1. [ADR-006-A1] Biblioteca ou serviço dedicado de eventos e webhooks
- **Prós:** funcionalidades prontas, como retry e assinatura.
- **Contras:** nova dependência e nova infraestrutura para um time pequeno.
- **Motivo do descarte:** o princípio de reuso máximo. Alternativa plausível, não discutida de forma específica.

### A2. [ADR-006-A2] Hierarquia de erros e logging próprios do módulo
- **Prós:** independência do restante do código.
- **Contras:** duplica o tratamento que o error middleware já faz, e quebra o padrão de códigos de erro.
- **Motivo do descarte:** o error middleware já cobre os casos sem alteração.

### A3. [ADR-006-A3] Colocar a lógica de webhooks dentro do módulo `orders`
- **Prós:** menos arquivos novos.
- **Contras:** mistura responsabilidades. O módulo `orders` já concentra 255 linhas de regra de negócio e acessa tabelas de outros módulos direto.
- **Motivo do descarte:** o padrão do projeto é um módulo por domínio.

## Consequências

### Positivas
- Menor curva de aprendizado e revisão mais simples, com código que parece o resto da base.
- Zero dependências ou infraestrutura novas, e tratamento de erros e logs já uniformes.
- A resposta de erro da API de webhooks segue o mesmo formato do resto da API.

### Negativas
- A feature herda as limitações atuais. O error middleware, por exemplo, só mapeia os códigos Prisma P2002 e P2025, então outros erros caem em 500.
- O uso de `requireRole` herda a granularidade atual de roles (`ADMIN` e `OPERATOR`).
- O worker, por ser outro processo, precisa de wiring próprio (logger, env e Prisma), que não é coberto pelo padrão da API.

### Trade-off explícito
Abrimos mão de liberdade de design e de ferramentas mais especializadas em troca de consistência, velocidade de entrega (3 sprints) e ausência de novas dependências.
