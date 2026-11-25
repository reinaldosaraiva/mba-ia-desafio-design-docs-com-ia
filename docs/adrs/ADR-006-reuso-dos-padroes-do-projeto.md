# ADR-006: Reuso dos padrões existentes do projeto

| Campo | Valor |
|---|---|
| ID | ADR-006 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Larissa (Tech Lead, fechou a decisão), com Bruno (Engenheiro Pleno, time de Pedidos) e Diego (Engenheiro Sênior, time de Plataforma) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-002](./ADR-002-worker-separado-com-polling.md) · [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md) |

## Status

Aceito. Decisão fechada na reunião em `[09:30] Larissa`. O prefixo `WEBHOOK_` foi fechado em `[09:29] Larissa`. A decisão foi reconfirmada no resumo de `[09:48] Larissa` e aceita em `[09:49] Diego` e `[09:49] Bruno`.

## Contexto

1. **Padrão de módulos.** "Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas", e o webhook "vai seguir igual" (`[09:27] Bruno`). No código, `src/modules/customers/` é composto por `src/modules/customers/customer.controller.ts`, `src/modules/customers/customer.service.ts`, `src/modules/customers/customer.repository.ts`, `src/modules/customers/customer.routes.ts` e `src/modules/customers/customer.schemas.ts`. A função `buildControllers`, em `src/app.ts`, instancia o repository, o service e o controller dos módulos de domínio (o módulo `auth` reusa o `UserRepository`), e a função `buildApiRouter`, em `src/routes/index.ts`, monta o router de cada um.
2. **Entry-point e lógica de processamento.** O worker ganha uma entry separada, `src/worker.ts` (novo), e a lógica de processamento fica em um arquivo dentro do módulo (`[09:28] Bruno`). Diego concordou com "Beleza" em `[09:28] Diego`. O processo separado e o polling são tratados no [ADR-002](./ADR-002-worker-separado-com-polling.md).
3. **Padrão de erros.** Já existem a classe `AppError` e classes específicas, como `InsufficientStockError` e `InvalidStatusTransitionError`, com os códigos `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` (`[09:28] Bruno`). No código:
   - `AppError` (`src/shared/errors/app-error.ts`) carrega `statusCode`, `errorCode` e `details`.
   - Em `src/shared/errors/http-errors.ts`, `InvalidStatusTransitionError` estende `ConflictError` (409) e `InsufficientStockError` estende `UnprocessableEntityError` (422). As duas passam o código ao construtor da classe-mãe.
   - `NotFoundError` (404), no mesmo arquivo, não aceita código.
4. **Logger e middleware de erro.** O logger Pino "já tá no projeto inteiro", e o middleware de erro centralizado "já trata AppError, Zod e Prisma" (`[09:29] Bruno`). No código, o logger é criado por `createLogger`, em `src/shared/logger/index.ts`. O middleware é o `errorMiddleware`, em `src/middlewares/error.middleware.ts`, registrado por `buildApp` (`src/app.ts`) depois das rotas de `/api/v1`.
5. **Validação e autenticação.** As rotas autenticadas de domínio (customers, products, orders) aplicam `authenticate` (`src/middlewares/auth.middleware.ts`) e `validate` (`src/middlewares/validate.middleware.ts`) com schemas Zod, como em `buildCustomerRouter` (`src/modules/customers/customer.routes.ts`). O `requireRole` já existe no mesmo arquivo de autenticação.
6. **Pool de conexões.** "O pool de conexão do Prisma já tá lá", e ficou a pergunta se o worker abre o mesmo `PrismaClient` ou um separado (`[09:29] Diego`). No código, `createPrismaClient` (`src/config/database.ts`) é a fábrica do cliente. O `OrderService` (`src/modules/orders/order.service.ts`) recebe o `PrismaClient` no construtor e abre a transação de `changeStatus` com `$transaction`.

## Decisão

Nós reusamos ao máximo o que já existe no projeto: `AppError` e o padrão de classes de erro específicas, o logger Pino, o middleware de erro centralizado, o padrão de módulos de `src/modules`, o padrão de schemas Zod e o padrão de códigos de erro. O webhook fica como módulo igual aos outros (`[09:30] Larissa`). Fazem parte desta decisão:

1. **Módulo.** Nós criamos o módulo em `src/modules/webhooks` (novo), com controller, service, repository, routes e schemas (`[09:27] Bruno`, `[09:28] Diego`). Ele é registrado nos mesmos pontos de composição dos demais módulos, `buildControllers` e `buildApiRouter`, conforme a convenção de `src/app.ts` e `src/routes/index.ts`.
2. **Códigos de erro.** Nós usamos o prefixo `WEBHOOK_` em todos os códigos de erro do módulo (`[09:29] Larissa`). Os erros são modelados como classes específicas derivadas de `AppError` e tratados pelo `errorMiddleware` sem alteração (`[09:28] Bruno`, `[09:29] Bruno`). `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` são exemplos do padrão, e não uma lista fechada: a fala termina em "etc." (`[09:28] Bruno`).
3. **Logger.** Nós usamos o logger Pino existente (`createLogger`) em todo o módulo, sem biblioteca nova (`[09:29] Bruno`).
4. **Validação e autenticação.** Nós validamos a entrada com schemas Zod aplicados pelo `validate` existente (`[09:30] Larissa`). As rotas do módulo usam `authenticate`, por serem "endpoint autenticado normal" (`[09:32] Larissa`), seguindo a convenção de `buildCustomerRouter`. Nós reaproveitamos o `requireRole` existente para exigir a role ADMIN no replay da DLQ (`[09:36] Larissa`).
5. **Publicação do evento.** Nós publicamos o evento pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o cliente de transação corrente e é chamada por `OrderService.changeStatus` (`[09:41] Bruno`, `[09:41] Diego`). A atomicidade dessa escrita com a mudança de status é tratada no [ADR-001](./ADR-001-outbox-no-mysql.md).
6. **Worker.** O entry-point do worker fica em `src/worker.ts` (novo), e a lógica de processamento fica dentro de `src/modules/webhooks` (novo) (`[09:28] Bruno`). O worker usa uma instância própria de `PrismaClient`, no mesmo banco e com a mesma `DATABASE_URL` (`[09:30] Bruno`). Essa instância é criada pela fábrica existente `createPrismaClient` (consequência técnica inferida a partir do código: é em `src/config/database.ts` que o projeto constrói o `PrismaClient`).

**Forma final.** Em `[09:11] Bruno` falou-se em "usar o mesmo Prisma client". Vale a forma refinada em `[09:30] Bruno`: mesmo banco e mesma stack (`[09:11] Diego`), com instância separada.

**Interpretação a confirmar (alcance do prefixo).** A frase "Prefixo WEBHOOK_ pra tudo do módulo" (`[09:29] Larissa`) admite duas leituras:

- (a) todo código de erro que o módulo define;
- (b) toda resposta de erro das rotas do módulo.

Adotamos a leitura (a). Os componentes reusados emitem códigos próprios:

| Componente | Código emitido |
|---|---|
| `validate` | `VALIDATION_ERROR` |
| `authenticate` | `UNAUTHORIZED` |
| `requireRole` | `UNAUTHORIZED` (sem usuário autenticado) e `FORBIDDEN` |
| Mapeamento Prisma do `errorMiddleware` | `CONFLICT` e `NOT_FOUND` |

A leitura (b) exigiria mudar esses componentes, o que contraria "sem precisar mudar nada" (`[09:29] Bruno`). Pelo mesmo motivo, a recusa de URL http é "só uma validação no schema Zod" (`[09:23] Sofia`) e responde `VALIDATION_ERROR`. A outra leitura possível é que `WEBHOOK_INVALID_URL` (`[09:28] Bruno`) seja emitido por uma classe do módulo.

**Interpretação a confirmar (arquivo de processamento).** Em `[09:28] Bruno` aparecem, como exemplo ("tipo"), `src/modules/webhooks/webhook.worker.ts` (novo) "ou" `src/modules/webhooks/webhook.processor.ts` (novo). O corpo deste ADR registra apenas "arquivo dentro do módulo"; o nome do arquivo não foi fixado.

## Alternativas Consideradas

### ADR-006-ALT-1: Injetar o repository do webhook no `OrderService`

Levantada por `[09:41] Bruno`. Para `changeStatus` publicar o evento, seria preciso "passar um repository do webhook pro OrderService ou uma função de 'enqueue event'". Bruno propôs a função. A injeção foi descartada em `[09:41] Diego`: "função pura recebendo o tx. Não precisa injetar repository inteiro." O trade-off é que a injeção acoplaria um repository inteiro ao `OrderService` sem necessidade, enquanto a função precisa apenas do cliente da transação em curso.

(consequência técnica inferida a partir do código) A injeção mudaria a assinatura do construtor de `OrderService`, hoje `(orders, prisma)`, e a sua montagem em `buildControllers`. Além disso, os repositories existentes guardam o `PrismaClient` recebido no construtor, como faz `CustomerRepository`, e não o `tx` aberto por `$transaction` em `changeStatus`. Arquivos: `src/modules/orders/order.service.ts`, `src/app.ts`, `src/modules/customers/customer.repository.ts`.

### ADR-006-ALT-2: Worker usar a mesma instância de `PrismaClient` da API

Levantada como pergunta em `[09:29] Diego`: "O worker abre o mesmo PrismaClient ou um separado?". Descartada em `[09:30] Bruno`: "PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node." O trade-off é que compartilhar a instância, e com ela o pool já existente (`[09:29] Diego`), exigiria rodar o worker no mesmo processo da API. Isso "não pode ser" (`[09:11] Diego`), conforme registrado no [ADR-002](./ADR-002-worker-separado-com-polling.md).

## Consequências

### Positivas

- Nenhuma biblioteca nova de logging ou de tratamento de erro entra no projeto: "Não vamos botar nada novo" (`[09:29] Bruno`).
- As respostas de erro do módulo saem no mesmo formato das demais rotas (`[09:29] Bruno`). (consequência técnica inferida a partir do código) O ramo `instanceof AppError` do `errorMiddleware` usa `statusCode`, `errorCode` e `details` da própria instância. Arquivo: `src/middlewares/error.middleware.ts`.
- O webhook é lido e mantido como os demais módulos: "Webhook fica como módulo igual aos outros" (`[09:30] Larissa`).
- (consequência técnica inferida a partir do código) Ao receber o `tx`, `publishWebhookEvent` repete o formato dos helpers privados de `OrderService` que já recebem o cliente de transação (`debitStock`, `replenishStock`, `reserveOrderNumber`). Arquivo: `src/modules/orders/order.service.ts`.

### Negativas e trade-off explícito

- (consequência técnica inferida a partir do código) `NotFoundError` grava o código fixo `NOT_FOUND`, e o seu construtor só recebe `resource`. Para responder `WEBHOOK_NOT_FOUND`, é preciso uma classe que estenda `AppError` diretamente. Ainda assim, o `errorMiddleware` trata essa classe sem alteração. Arquivos: `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`.
- (consequência técnica inferida a partir do código) Um campo rejeitado por schema Zod produz `VALIDATION_ERROR` (400), e não um código `WEBHOOK_*`. Isso ocorre porque `validate` converte o `ZodError` em `ValidationError`, e o `errorMiddleware` responde `400 VALIDATION_ERROR` diretamente para um `ZodError` não capturado. Do mesmo modo, `authenticate` e `requireRole` respondem `UNAUTHORIZED` e `FORBIDDEN`. Arquivos: `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`.
- O worker não compartilha a instância nem o pool de conexões da API: abre os seus próprios, contra o mesmo banco (`[09:29] Diego`, `[09:30] Bruno`).
- (consequência técnica inferida a partir do código) Reusar `createLogger` sem alteração tem dois efeitos:
  - mantém a lista `redactPaths` atual, que não cobre um campo chamado `secret` (nome usado em `[09:21] Bruno`; a secret por endpoint é tratada no [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md));
  - mantém `service: 'order-management-api'` na base dos logs, inclusive nos logs do worker.

  Arquivo: `src/shared/logger/index.ts`.
- (consequência técnica inferida a partir do código) O `errorMiddleware` só atua sobre as requisições da app montada por `buildApp`. As falhas no processamento do worker não passam por ele. Arquivo: `src/app.ts`.
- (consequência técnica inferida a partir do código) Registrar o módulo exige editar arquivos compartilhados: o tipo `Controllers` e a função `buildApiRouter`, em `src/routes/index.ts`, e a função `buildControllers`, em `src/app.ts`.

**Trade-off aceito:** aceitamos ficar restritos ao que já existe, ou seja, `AppError`, Pino, error middleware, padrão de módulos, schemas Zod e códigos de erro (`[09:30] Larissa`), e abrir no worker uma instância nova de `PrismaClient` (`[09:30] Bruno`), em troca de não botar nada novo no projeto e de o middleware pegar os erros do módulo sem precisar mudar nada (`[09:29] Bruno`).

## Referências

- **Transcrição:** `[09:11] Bruno`, `[09:11] Diego`, `[09:21] Bruno`, `[09:23] Sofia`, `[09:27] Bruno`, `[09:28] Bruno`, `[09:28] Diego`, `[09:29] Larissa`, `[09:29] Bruno`, `[09:29] Diego`, `[09:30] Bruno`, `[09:30] Larissa`, `[09:32] Larissa`, `[09:36] Larissa`, `[09:41] Bruno`, `[09:41] Diego`, `[09:48] Larissa`, `[09:49] Diego`, `[09:49] Bruno`.
- **Código existente:**

| Arquivo | Símbolos |
|---|---|
| `src/shared/errors/app-error.ts` | `AppError` |
| `src/shared/errors/http-errors.ts` | `InsufficientStockError`, `InvalidStatusTransitionError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `ValidationError` |
| `src/middlewares/error.middleware.ts` | `errorMiddleware` |
| `src/middlewares/validate.middleware.ts` | `validate` |
| `src/middlewares/auth.middleware.ts` | `authenticate`, `requireRole` |
| `src/shared/logger/index.ts` | `createLogger`, `redactPaths` |
| `src/app.ts` | `buildControllers`, `buildApp` |
| `src/routes/index.ts` | `buildApiRouter`, `Controllers` |
| `src/modules/customers/` | composição de um módulo |
| `src/modules/customers/customer.routes.ts` | `buildCustomerRouter` |
| `src/modules/customers/customer.repository.ts` | `CustomerRepository` |
| `src/modules/orders/order.service.ts` | `OrderService`, `changeStatus`, `debitStock`, `replenishStock`, `reserveOrderNumber` |
| `src/config/database.ts` | `createPrismaClient` |

- **Documentos:** [RFC](../RFC.md) (seção de decisões relacionadas) · [FDD](../FDD.md) (detalhamento de implementação do módulo de webhooks) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-002](./ADR-002-worker-separado-com-polling.md) · [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md).
