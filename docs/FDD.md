# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Autor | Larissa (Tech Lead) |
| Data | 2026-11-25 |
| Status | Em revisão — sessão de revisão com Bruno e Diego agendada na reunião (`[09:50] Larissa`) |
| Decisões de origem | [RFC](RFC.md) (proposta) · [ADRs](adrs/README.md) (decisões fechadas) · [PRD](PRD.md) (requisitos) |
| Rastreabilidade | [Tracker](TRACKER.md) — cada ID deste documento tem uma linha com a origem |

## Como ler este documento

Este documento detalha como construir a feature. As decisões já estão fechadas e registradas em ADRs; aqui elas viram especificação de implementação. Cada requisito, contrato, erro, fluxo e ponto de integração tem um ID no padrão `FDD-TIPO-NN`, e o [Tracker](TRACKER.md) aponta a origem de cada um: uma fala da transcrição (`[hh:mm] Falante`) ou um arquivo do código.

Três marcações aparecem ao longo do texto e não devem ser lidas como decisão:

- `(novo)` depois de um caminho: arquivo que a implementação vai criar; os demais caminhos citados já existem no repositório.
- `(proposta, ver FDD-PROP-NN)`: detalhe de implementação que a reunião não decidiu. A proposta serve de base de trabalho e está listada, com a lacuna que atende, na seção 15.
- `(consequência técnica inferida)`: dedução a partir do código, não dita na reunião.

A seção 15 concentra tudo o que a reunião deixou em aberto e precisa ser confirmado na revisão deste documento.

## Sumário

- 1. Contexto e motivação técnica
- 2. Objetivos técnicos
- 3. Escopo e exclusões
- 4. Visão geral da solução
- 5. Modelo de dados
- 6. Fluxos detalhados
- 7. Contratos públicos
- 8. Matriz de erros previstos
- 9. Estratégias de resiliência
- 10. Observabilidade
- 11. Dependências e compatibilidade
- 12. Integração com o sistema existente
- 13. Critérios de aceite técnicos
- 14. Riscos e mitigação
- 15. Lacunas não decididas e propostas a confirmar


## 1. Contexto e motivação técnica
- **FDD-CTX-01.** Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para ser notificados em tempo real quando o status dos pedidos deles muda. Hoje eles consultam `GET /orders` de tempos em tempos, o que deixa a integração lenta e cara para eles (`[09:00] Marcos`). O router de pedidos é montado em `/api/v1/orders`.
- **FDD-CTX-02.** Para esses clientes, qualquer coisa abaixo de 10 segundos já é "tempo real". O importante é que a notificação não fique pendurada (`[09:02] Marcos`).
- **FDD-CTX-03.** O alcance é só notificação de saída (outbound webhook): os clientes querem receber, não mandar (`[09:02] Marcos`, `[09:03] Sofia`).
- **FDD-CTX-04.** O código não tem hoje notificação, evento, fila nem webhook. A função `buildApiRouter`, em `src/routes/index.ts`, monta apenas os módulos auth, users, customers, products e orders.
- **FDD-CTX-05.** A transação de mudança de status já é pesada (`[09:04] Bruno`). Em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), o callback de `this.prisma.$transaction` faz, nesta ordem:
  - busca o pedido com os itens;
  - valida a transição;
  - debita ou repõe estoque;
  - atualiza o pedido;
  - grava o histórico.

  Uma chamada HTTP nesse trecho faria qualquer cliente lento travar a mudança de status de outros pedidos (`[09:04] Bruno`).

## 2. Objetivos técnicos
| ID | Objetivo | Verificação |
|---|---|---|
| FDD-OBJ-01 | Registrar o evento na mesma transação SQL da mudança de status (`[09:06] Diego`, `[09:40] Bruno`). | O commit grava juntos a mudança de status e as linhas da `webhook_outbox`. Se a inserção na outbox falha, há rollback e o status não muda. Quando o filtro (ADR-008) não insere nenhuma linha, isso não conta como falha (leitura do ADR-001). |
| FDD-OBJ-02 | Entregar o evento em menos de 10 segundos (`[09:02] Marcos`). | Com o endpoint do cliente disponível, o envio inicial (FDD-CONTRATO-08) sai em menos de 10 segundos após o commit. O polling de 2 segundos atende essa meta (`[09:09] Diego`, ADR-002). Tamanho do lote e concorrência: proposta, ver FDD-PROP-04. |
| FDD-OBJ-03 | Suportar a indisponibilidade do cliente com novas tentativas (`[09:15] Diego`). | Um envio com falha é retentado nos intervalos 1m/5m/30m/2h/12h, até o teto de 5 tentativas. Esgotado o teto, o evento fica em `webhook_dead_letter` (`[09:17] Larissa`, `[09:18] Diego`). O timeout de 10 segundos conta como falha (`[09:42] Diego`). Critério de falha: proposta, ver FDD-PROP-05. Contagem das 5 tentativas: proposta, ver FDD-PROP-11. |
| FDD-OBJ-04 | Permitir ao cliente validar a origem e a integridade de cada entrega (`[09:19] Sofia`). | Toda entrega leva `X-Signature` com o HMAC-SHA256 do corpo, calculado com a secret do cadastro (`[09:22] Sofia`). Recalculado com a mesma secret, o valor confere; com o corpo alterado, não confere. Formato: proposta, ver FDD-PROP-06. |
| FDD-OBJ-05 | Reusar os padrões existentes do projeto (`[09:30] Larissa`). | O módulo tem controller, service, repository, routes e schemas, como os demais (`[09:27] Bruno`). Os códigos de erro usam o prefixo `WEBHOOK_` (`[09:29] Larissa`), em classes derivadas de `AppError`. O log usa o Pino de `src/shared/logger/index.ts`. O arquivo `src/middlewares/error.middleware.ts` não é alterado (`[09:29] Bruno`). |
| FDD-OBJ-06 | Isolar a mudança de status da latência e da disponibilidade do endpoint do cliente (`[09:04] Bruno`). | Nenhuma chamada HTTP ocorre dentro da transação de `OrderService.changeStatus` (`[09:06] Diego`). As entregas são feitas pelo worker, em processo separado da API (`[09:11] Diego`). |

**Interpretação a confirmar (FDD-OBJ-02).** A reunião não diz onde termina a contagem dos 10 segundos (`[09:02] Marcos`).
- Leitura adotada: a contagem vai do commit da mudança de status até o envio inicial do request.
- Outra leitura: a contagem vai até a resposta do cliente. Nesse caso, um cliente que responda perto do timeout de 10 segundos (`[09:42] Diego`) estouraria a meta sozinho.
- Nas duas leituras, a meta não cobre as novas tentativas, cujo primeiro intervalo é de 1 minuto (`[09:17] Diego`).

## 3. Escopo e exclusões
### 3.1 No escopo

| ID | Item |
|---|---|
| FDD-ESC-01 | Registro do evento na `webhook_outbox` dentro da transação de `OrderService.changeStatus`, pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:40] Bruno`, `[09:41] Bruno`). O registro inclui o filtro por status na inserção (ADR-008) e o payload em snapshot (ADR-007). A criação do pedido não emite evento (proposta, ver FDD-PROP-09). |
| FDD-ESC-02 | Worker em processo separado (`[09:10] Larissa`, `[09:11] Diego`, `[09:13] Larissa`): entry-point `src/worker.ts` (novo), script `npm run worker` (novo), polling de 2 segundos e worker único. |
| FDD-ESC-03 | Novas tentativas nos intervalos 1m/5m/30m/2h/12h, com teto de 5 tentativas (`[09:17] Larissa`), lido como 1 envio inicial mais 5 novas tentativas (proposta, ver FDD-PROP-11), e timeout de 10 segundos. Fila de mortos em `webhook_dead_letter` (`[09:17] Larissa`, `[09:18] Diego`, `[09:42] Diego`). |
| FDD-ESC-04 | Assinatura das entregas com HMAC-SHA256 sobre o corpo, usando secret única por endpoint de webhook, gerada pela plataforma e devolvida na criação (`[09:22] Sofia`, `[09:31] Marcos`). A entrega leva os headers FDD-HEADER-01 a FDD-HEADER-05 (`[09:44] Diego`, `[09:45] Diego`). O payload tem limite de 64KB, com erro se ultrapassar (`[09:24] Larissa`). |
| FDD-ESC-05 | API de gerenciamento dos cadastros (FDD-CONTRATO-01 a FDD-CONTRATO-04), autenticada e aberta a qualquer role (`[09:32] Larissa`, `[09:37] Sofia`). A URL tem de ser `https`, validada no schema Zod (`[09:23] Sofia`). Cada cadastro tem a sua lista de status a receber (`[09:33] Marcos`). A abertura a qualquer role parte da premissa de `[09:32] Marcos` (os usuários representam os clientes), que não vale no código: ver FDD-RISCO-05, FDD-GAP-46 e FDD-PROP-44. |
| FDD-ESC-06 | Histórico de entregas por cadastro (FDD-CONTRATO-05), com sucesso/falha, payload, response e tempo de resposta (`[09:34] Marcos`). A fala cita "os últimos 100 webhooks" dentro de um exemplo ("Tipo"); ver "Lacunas adicionais sugeridas nesta parte". |
| FDD-ESC-07 | Rotação de secret pela API (FDD-CONTRATO-06). A secret antiga continua válida por 24 horas em paralelo com a nova (`[09:21] Sofia`). Qual secret assina durante a carência: proposta, ver FDD-PROP-21. |
| FDD-ESC-08 | Reprocessamento manual da DLQ (FDD-CONTRATO-07), restrito à role ADMIN pelo `requireRole` existente, com registro de quem fez o replay (`[09:18] Diego`, `[09:36] Sofia`, `[09:36] Larissa`). |

### 3.2 Exclusões

| ID | Item excluído |
|---|---|
| FDD-OOS-01 | E-mail de alerta ao cliente quando o webhook dele está com problema. Fica para uma próxima fase (`[09:37] Larissa`, `[09:48] Larissa`). |
| FDD-OOS-02 | Painel visual para o cliente ver os webhooks dele. Nesta fase há só endpoints; o painel é projeto separado do time de frontend (`[09:40] Larissa`). |
| FDD-OOS-03 | Limite de envio (rate limiting) por cliente. Fica como "observar e decidir depois" (`[09:39] Diego`, `[09:39] Larissa`). |
| FDD-OOS-04 | Arquivamento das linhas entregues depois de cerca de 30 dias (valor aproximado na fala), fora do escopo desta feature (`[09:08] Diego`); ver FDD-GAP-22. |
| FDD-OOS-05 | Múltiplos workers em paralelo. O worker único fica como limitação conhecida, sem garantia de ordenação global (`[09:13] Diego`, `[09:13] Larissa`). |
| FDD-OOS-06 | Webhooks de entrada, isto é, chamadas dos clientes para a plataforma (`[09:02] Marcos`). |
| FDD-OOS-07 | Itens do pedido no payload. Para ver detalhes, o cliente consulta `GET /api/v1/orders/:id` (citado como "GET /orders/:id" em `[09:43] Diego`). |

## 4. Visão geral da solução
### 4.1 Componentes

A solução roda em dois processos Node: a API, iniciada por `src/server.ts`, e o worker, iniciado por `src/worker.ts` (novo). Os dois usam o mesmo MySQL e a mesma `DATABASE_URL`, cada um com a sua instância de `PrismaClient` (`[09:30] Bruno`).

| ID | Componente | Processo | Responsabilidade | Onde |
|---|---|---|---|---|
| FDD-COMP-01 | Módulo de webhooks | API | Expõe FDD-CONTRATO-01 a FDD-CONTRATO-07 com `authenticate` e `validate`, e com `requireRole` no replay. Gera a secret na criação e na rotação (gerá-la é decisão da reunião, `[09:31] Marcos`; o formato e a entropia são proposta, ver FDD-PROP-19). É registrado em `buildControllers` (`src/app.ts`) e em `buildApiRouter` (`src/routes/index.ts`). | `src/modules/webhooks/webhook.controller.ts` (novo), `src/modules/webhooks/webhook.service.ts` (novo), `src/modules/webhooks/webhook.repository.ts` (novo), `src/modules/webhooks/webhook.routes.ts` (novo), `src/modules/webhooks/webhook.schemas.ts` (novo) |
| FDD-COMP-02 | Ponto de publicação | API, dentro da transação | Chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` no mesmo callback de `this.prisma.$transaction` que executa `tx.order.update` e `tx.orderStatusHistory.create`. A função aplica o filtro por status (ADR-008) e insere o payload renderizado (ADR-007). | Chamada em `src/modules/orders/order.service.ts`; a função é do módulo `src/modules/webhooks/` (novo) |
| FDD-COMP-03 | Entry-point do worker | Worker | Implementa `bootstrap` e `shutdown` (`SIGINT`/`SIGTERM`, com `prisma.$disconnect()`) no modelo de `src/server.ts`. Usa instância própria de `PrismaClient`, vinda de `src/config/database.ts`. É iniciado por `npm run worker` (novo). | `src/worker.ts` (novo), `package.json` |
| FDD-COMP-04 | Processador de entregas | Worker | A cada 2 segundos, lê as linhas pendentes mais antigas, em lote pequeno (proposta, ver FDD-PROP-04). Assina e envia cada uma com timeout de 10 segundos, registra o resultado e decide entre entregue, nova tentativa ou DLQ. | `src/modules/webhooks/webhook.processor.ts` (novo); nome proposto, ver FDD-PROP-14 |
| FDD-COMP-05 | Tabela `webhook_outbox` | MySQL (API insere; worker lê e atualiza) | Guarda os eventos a entregar, com UUID do evento, payload em snapshot e status (pendente, processando, falhou, entregue), com índices em status e em `created_at` (`[09:08] Diego`). Granularidade e papel dos status: propostas, ver FDD-PROP-01 e FDD-PROP-02. | `prisma/schema.prisma` |
| FDD-COMP-06 | Tabela `webhooks` (nome proposto, ver FDD-PROP-14) | MySQL (API escreve; publicação e worker leem) | Guarda os cadastros: url, secret, customer_id e estado ativo (`[09:21] Bruno`), mais a lista de status assinados (`[09:33] Marcos`). Armazenamento da secret: não decidido na reunião (ver FDD-PROP-08) | `prisma/schema.prisma` |
| FDD-COMP-07 | Tabela `webhook_deliveries` (nome proposto, ver FDD-PROP-14) | MySQL (worker escreve; API lê) | Guarda o histórico de entregas por cadastro: sucesso/falha, payload, response e tempo de resposta (`[09:34] Marcos`). | `prisma/schema.prisma` |
| FDD-COMP-08 | Tabela `webhook_dead_letter` | MySQL (worker escreve; API lê no replay) | Guarda os eventos esgotados, com payload, motivo da falha e timestamp (`[09:18] Diego`). É a origem do replay de FDD-CONTRATO-07. | `prisma/schema.prisma` |

### 4.2 Caminho do evento

Visão geral, do commit da mudança de status até a entrega. Os fluxos detalhados estão na seção 6.

1. Uma requisição de mudança de status chega a `OrderService.changeStatus`. Dentro da transação, o método busca o pedido com os itens, valida a transição, debita ou repõe estoque e executa `tx.order.update` e `tx.orderStatusHistory.create`.
2. No mesmo callback, `publishWebhookEvent` busca, pelo `tx`, os cadastros do cliente do pedido que assinam o status de destino (leitura adotada no ADR-008).
   - Se nenhum cadastro assina, nada é inserido.
   - Se algum assina, a função insere na `webhook_outbox` o evento, com UUID (`[09:25] Diego`) e payload renderizado, uma linha por destino (proposta, ver FDD-PROP-01).
   - Se a inserção falha, a transação inteira faz rollback (ADR-001).
3. O commit grava juntos a mudança de status e o evento. Até aqui, nenhuma chamada HTTP aconteceu.
4. A cada 2 segundos, o worker lê as linhas pendentes mais antigas, pela ordem de `created_at`, em lote pequeno, e as marca como processando. Linhas presas nesse status: proposta, ver FDD-PROP-03.
5. Para cada linha, o worker usa o payload gravado como corpo.
   - Se o corpo passa de 64KB, a linha não é enviada (proposta, ver FDD-PROP-16).
   - Caso contrário, o worker calcula o HMAC-SHA256 com a secret do cadastro (propostas, ver FDD-PROP-06 e FDD-PROP-21) e monta os headers FDD-HEADER-01 a FDD-HEADER-05.
   - Em seguida, faz o request FDD-CONTRATO-08 com timeout de 10 segundos.
6. O worker registra o resultado do envio em `webhook_deliveries` e o classifica como sucesso ou falha (proposta, ver FDD-PROP-05).
7. O worker trata o resultado assim:
   - Com sucesso, a linha é marcada como entregue.
   - Com falha, a nova tentativa segue os intervalos 1m/5m/30m/2h/12h (estado da linha: proposta, ver FDD-PROP-02).
   - Esgotado o teto de 5 tentativas (`[09:17] Larissa`; contagem lida conforme FDD-PROP-11), o evento vai para `webhook_dead_letter` com o motivo da falha.
8. Um ADMIN pode fazer o replay por FDD-CONTRATO-07. O replay recoloca o evento na outbox como pendente (`[09:18] Diego`), com o mesmo id de evento e o mesmo payload (proposta, ver FDD-PROP-23). O caminho recomeça no passo 4.

## 5. Modelo de dados
### 5.1 Convenções herdadas

As tabelas novas seguem o que `prisma/schema.prisma` já define:

- **FDD-DADOS-01: Chave primária UUID.** Todo model novo declara `id String @id @default(uuid()) @db.Char(36)`, como `User`, `Order` e `OrderStatusHistory`. A coluna que guarda o id de outro registro usa `String @db.Char(36)`, como `Order.customerId`.
- **FDD-DADOS-02: Datas de controle.** Um model cujo registro é alterado depois de gravado tem `createdAt DateTime @default(now())` e `updatedAt DateTime @updatedAt`, como `Customer` e `Order`. Um model cujo registro não muda tem só `createdAt`, como `OrderItem`.
- **FDD-DADOS-03: Nomes.** O model fica em PascalCase, com `@@map` para um nome de tabela em snake_case (`OrderStatusHistory` vira `order_status_history`). Os campos ficam em camelCase, sem `@map` por coluna. Por isso, o `created_at` citado em `[09:08] Diego` e `[09:12] Diego` corresponde, no schema, ao campo `createdAt`.
- **FDD-DADOS-04: Índices.** O índice é simples, por coluna, e declarado com `@@index([campo])`, como `@@index([status])`, `@@index([createdAt])` e `@@index([customerId])` em `Order`.
- **FDD-DADOS-05: Enums.** Um conjunto fechado de valores vira `enum` do Prisma, com nome em PascalCase e valores em maiúsculas, como `UserRole` (`ADMIN`, `OPERATOR`) e `OrderStatus`.

### 5.2 Tabelas novas

Como ler a coluna Origem das tabelas abaixo:

- uma fala da reunião indica de onde o campo decorre;
- `proposta (FDD-PROP-NN)` indica que o campo não decorre da reunião;
- "convenção do código" remete à seção 5.1.

Os nomes de model, de coluna e de enum não foram ditados na reunião. Eles seguem a seção 5.1 e o camelCase de FDD-PROP-12. Onde a reunião não definiu tipo ou tamanho, o tipo aparece marcado como proposta e nenhum tamanho é fixado.

### 5.2.1 `webhooks`

- **Finalidade:** cada registro é um endpoint cadastrado por um cliente, e o mesmo cliente pode ter vários (`[09:44] Sofia`).
- **Composição:** o conjunto url, secret, cliente e estado ativo foi levantado em `[09:21] Bruno` e confirmado em `[09:21] Sofia`. A lista de status vem de `[09:33] Marcos`.
- **Nome:** `webhooks` (model `Webhook`) é proposta (FDD-PROP-14).

| ID | Campo | Tipo | Nulo | Finalidade | Origem |
|---|---|---|---|---|---|
| FDD-DADOS-06 | `id` | `String @db.Char(36)`, PK (FDD-DADOS-01) | não | Identifica o cadastro. É o `:id` das rotas e o valor de `X-Webhook-Id` (FDD-HEADER-05) | `[09:51] Larissa` |
| FDD-DADOS-07 | `customerId` | `String @db.Char(36)` | não | Cliente dono do cadastro. Chave estrangeira para `customers` (FDD-DADOS-54) | `[09:21] Sofia` |
| FDD-DADOS-08 | `url` | `String`, tamanho a definir (proposta) | não | Destino das entregas. Só `https`; o schema Zod recusa outro esquema (`[09:23] Sofia`) | `[09:31] Marcos` |
| FDD-DADOS-09 | `secret` | `String`, tamanho a definir (proposta, ver FDD-PROP-19) | não | Secret mais recente, gerada pela plataforma e devolvida ao cliente (`[09:31] Marcos`). Fica guardada de forma recuperável, ao contrário de `User.passwordHash`, porque o worker assina com ela (proposta, ver FDD-PROP-08) | `[09:21] Sofia` |
| FDD-DADOS-10 | `previousSecret` | `String`, mesmo tipo de `secret` (proposta) | sim | Secret anterior à última rotação, válida em paralelo por 24 horas. Fica `null` enquanto não houver rotação | `[09:21] Sofia` |
| FDD-DADOS-11 | `previousSecretExpiresAt` | `DateTime` | sim | Instante da rotação mais 24 horas. Depois dele, `previousSecret` deixa de valer | `[09:21] Sofia` |
| FDD-DADOS-12 | `statuses` | `Json` com uma lista de valores de `OrderStatus` (proposta; o schema já usa `Json` em `Customer.address`) | não | Status de destino que o cadastro assina, usados pelo filtro na inserção (ADR-008). Os valores são validados com `z.nativeEnum(OrderStatus)`. O valor `PENDING` é aceito, mas não gera entrega (proposta, ver FDD-PROP-09) | `[09:33] Marcos` |
| FDD-DADOS-13 | `active` | `Boolean @default(true)`, como `Product.active` | não | Estado ativo do cadastro. O efeito sobre entregas pendentes está em FDD-PROP-15 | `[09:21] Sofia` |
| FDD-DADOS-14 | `createdAt`, `updatedAt` | `DateTime` (FDD-DADOS-02) | não | Datas de controle | convenção do código (FDD-DADOS-02) |

- **FDD-DADOS-15: Índice (proposta).** `@@index([customerId])`, como em `Order`. Ele atende a dois usos:
  - a listagem por cliente (FDD-CONTRATO-02);
  - a leitura dos cadastros do cliente no filtro na inserção (ADR-008), em que a aplicação verifica a lista `statuses`.
- **FDD-DADOS-16: Uso das colunas de secret (proposta, ver FDD-PROP-21).**
  1. Na rotação (FDD-CONTRATO-06), o valor de `secret` é copiado para `previousSecret`, a secret nova é gravada em `secret` e `previousSecretExpiresAt` é preenchido.
  2. A partir da rotação, o worker assina com `secret`, a secret nova (proposta, ver FDD-PROP-21).
  3. Depois de `previousSecretExpiresAt`, `previousSecret` deixa de valer também para o cliente.

### 5.2.2 `webhook_outbox`

- **Finalidade:** guardar as entregas a fazer, que o worker lê (ADR-001, ADR-002).
- **Nome:** `webhook_outbox` foi citado em `[09:06] Diego`. O model `WebhookOutbox` é proposta (FDD-PROP-14).
- **Granularidade:** cada linha representa um evento para um cadastro de destino (proposta, ver FDD-PROP-01).

| ID | Campo | Tipo | Nulo | Finalidade | Origem |
|---|---|---|---|---|---|
| FDD-DADOS-17 | `id` | `String @db.Char(36)`, PK (FDD-DADOS-01) | não | Identifica a linha, ou seja, a entrega para um destino | `[09:51] Larissa` |
| FDD-DADOS-18 | `eventId` | `String @db.Char(36)`, sem `@default` | não | UUID do evento, gerado em `publishWebhookEvent` no momento da inserção. É enviado em `X-Event-Id` e no campo `event_id` do payload, e é igual em todas as linhas do mesmo evento | `[09:25] Diego` |
| FDD-DADOS-19 | `webhookId` | `String @db.Char(36)` | não | Cadastro de destino. A linha não copia a URL nem a secret: o envio lê do cadastro os valores vigentes | proposta (FDD-PROP-01) |
| FDD-DADOS-20 | `orderId` | `String @db.Char(36)`, sem chave estrangeira | não | Pedido do evento | proposta (FDD-PROP-22) |
| FDD-DADOS-21 | `customerId` | `String @db.Char(36)`, sem chave estrangeira | não | Cliente do pedido | proposta (FDD-PROP-22) |
| FDD-DADOS-22 | `status` | enum `WebhookOutboxStatus` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`), `@default(PENDING)` | não | Estados pendente, processando, falhou e entregue, em maiúsculas (FDD-DADOS-05). As transições estão em FDD-PROP-02 | `[09:08] Diego` |
| FDD-DADOS-23 | `payload` | `Json` (proposta, como `Customer.address`) | não | Evento já renderizado na inserção, com os campos de `[09:43] Diego`. O worker serializa o payload e assina os bytes enviados (proposta, ver FDD-PROP-06) | `[09:52] Larissa` |
| FDD-DADOS-24 | `attempts` | `Int @default(0)` | não | Quantidade de envios já feitos para a linha. O teto é de 5 tentativas (`[09:17] Larissa`); a forma de contar está em FDD-PROP-11 | proposta (FDD-PROP-02) |
| FDD-DADOS-25 | `nextAttemptAt` | `DateTime @default(now())` | não | Instante a partir do qual a linha `PENDING` pode ser enviada. Após uma falha, recebe o intervalo seguinte de 1m/5m/30m/2h/12h | proposta (FDD-PROP-02) |
| FDD-DADOS-26 | `createdAt` | `DateTime @default(now())` | não | Ordem de processamento do worker único (ADR-002) | `[09:12] Diego` |
| FDD-DADOS-27 | `updatedAt` | `DateTime @updatedAt` | não | Última mudança de estado. Serve de base para retomar linhas presas em `PROCESSING` (proposta, ver FDD-PROP-03) | convenção do código (FDD-DADOS-02) |

- **FDD-DADOS-28: Índices.** `@@index([status])` e `@@index([createdAt])`, decididos em `[09:08] Diego`.

> **Interpretação a confirmar:** em `[09:25] Diego`, o UUID é "único por evento".
> - Leitura adotada: com uma linha por destino, as linhas de um mesmo evento repetem o `eventId`. O cliente que tem vários cadastros distingue os envios por `X-Webhook-Id` (`[09:44] Sofia`).
> - Outra leitura: um UUID por linha, o que tornaria `eventId` igual a `id` (ver FDD-GAP-01).

### 5.2.3 `webhook_deliveries`

- **Finalidade:** guardar o histórico consultado em `GET /api/v1/webhooks/:id/deliveries` (FDD-CONTRATO-05), com resultado, payload, resposta e tempo de resposta (`[09:34] Marcos`).
- **Nome:** `webhook_deliveries` (model `WebhookDelivery`) é proposta (FDD-PROP-14).
- **Datas:** o registro não muda depois de gravado, por isso só existe `createdAt` (FDD-DADOS-02).

- **FDD-DADOS-29: Granularidade (proposta).** Cada request feito pelo worker gera uma linha, inclusive cada nova tentativa.

> **Interpretação a confirmar:** a fala "os últimos 100 webhooks que vocês mandaram" (`[09:34] Marcos`) admite duas leituras.
> - Leitura adotada: cada request enviado gera uma linha, para que cada falha apareça no histórico.
> - Outra leitura: uma linha por evento, com o resultado final.

| ID | Campo | Tipo | Nulo | Finalidade | Origem |
|---|---|---|---|---|---|
| FDD-DADOS-30 | `id` | `String @db.Char(36)`, PK (FDD-DADOS-01) | não | Identifica o registro do envio | `[09:51] Larissa` |
| FDD-DADOS-31 | `webhookId` | `String @db.Char(36)` | não | Cadastro para o qual o request foi feito. É o filtro da consulta | `[09:34] Marcos` |
| FDD-DADOS-32 | `eventId` | `String @db.Char(36)` | não | Evento enviado. Correlaciona histórico, outbox, DLQ e logs | proposta (FDD-PROP-20) |
| FDD-DADOS-33 | `success` | `Boolean` | não | Resultado do envio, pelo critério de FDD-PROP-05 | `[09:34] Marcos` |
| FDD-DADOS-34 | `payload` | `Json` (proposta) | não | Payload enviado, copiado da linha da outbox | `[09:34] Marcos` |
| FDD-DADOS-35 | `responseStatus` | `Int` | sim | Status HTTP da resposta. Fica `null` em erro de rede ou timeout | `[09:34] Marcos` |
| FDD-DADOS-36 | `responseBody` | `String @db.Text` (proposta, como `Order.notes`) | sim | Corpo da resposta do cliente | `[09:34] Marcos` |
| FDD-DADOS-37 | `errorMessage` | `String @db.Text` (proposta) | sim | Descrição da falha sem resposta HTTP: erro de rede ou timeout de 10 segundos (`[09:42] Diego`) | proposta (FDD-PROP-05) |
| FDD-DADOS-38 | `responseTimeMs` | `Int`, em milissegundos (proposta) | não | Tempo entre o início do request e a resposta, o erro ou o timeout | `[09:34] Marcos` |
| FDD-DADOS-39 | `createdAt` | `DateTime @default(now())` | não | Instante do envio. Ordena "os últimos" envios | `[09:34] Marcos` |

- **FDD-DADOS-40: Índices (proposta).** `@@index([webhookId])` e `@@index([createdAt])`, no mesmo modelo de `@@index([orderId])` e `@@index([changedAt])` de `OrderStatusHistory`.

### 5.2.4 `webhook_dead_letter`

- **Finalidade:** guardar as entregas que esgotaram o teto de tentativas ou que foram desviadas para a DLQ (FDD-PROP-15, FDD-PROP-16). Elas ficam como evidência para debug e reprocessamento (`[09:18] Diego`).
- **Nome:** `webhook_dead_letter` foi citado em `[09:18] Diego`. O model `WebhookDeadLetter` é proposta (FDD-PROP-14).
- **Criação:** a linha é criada quando a entrega sai da outbox (FDD-PROP-02). Ela guarda o necessário para que o replay recoloque a mesma linha na outbox (FDD-PROP-23).

| ID | Campo | Tipo | Nulo | Finalidade | Origem |
|---|---|---|---|---|---|
| FDD-DADOS-41 | `id` | `String @db.Char(36)`, PK (FDD-DADOS-01) | não | Identifica a entrada. É o `:id` do replay (FDD-CONTRATO-07) | `[09:51] Larissa` |
| FDD-DADOS-42 | `outboxId` | `String @db.Char(36)` | não | `id` original da linha da outbox, reusado no replay | proposta (FDD-PROP-23) |
| FDD-DADOS-43 | `eventId` | `String @db.Char(36)` | não | Mesmo `eventId` da outbox, preservado no replay | proposta (FDD-PROP-23) |
| FDD-DADOS-44 | `webhookId` | `String @db.Char(36)` | não | Cadastro de destino | proposta (FDD-PROP-01) |
| FDD-DADOS-45 | `orderId` | `String @db.Char(36)`, sem chave estrangeira | não | Pedido do evento | proposta (FDD-PROP-22) |
| FDD-DADOS-46 | `customerId` | `String @db.Char(36)`, sem chave estrangeira | não | Cliente do pedido | proposta (FDD-PROP-22) |
| FDD-DADOS-47 | `payload` | `Json` (proposta) | não | Payload da outbox, sem alteração | `[09:18] Diego` |
| FDD-DADOS-48 | `reason` | `String @db.Text` (proposta) | não | Motivo da falha. Pode ser: tentativas esgotadas, com o último erro; cadastro inativo ou removido (FDD-PROP-15); ou payload acima de 64KB (FDD-PROP-16) | `[09:18] Diego` |
| FDD-DADOS-49 | `attempts` | `Int` | não | Envios feitos até a ida para a DLQ. No replay, a contagem volta a zero (FDD-PROP-23) | proposta (FDD-PROP-02) |
| FDD-DADOS-50 | `createdAt` | `DateTime @default(now())` | não | Timestamp da ida para a DLQ | `[09:18] Diego` |

- **FDD-DADOS-51: Índices (proposta).** Nenhum índice além da chave primária, porque o único acesso decidido é por `id`, no replay.

### 5.3 Relações e exclusão

Fatos do código que servem de base para esta seção:

- `OrderService.delete` remove pedidos em `PENDING` ou `CANCELLED`, por meio de `this.orders.deleteById`.
- Na migration inicial, `order_items` e `order_status_history` apontam para `orders` com `ON DELETE CASCADE`.
- Também na migration inicial, `orders` aponta para `customers` com `ON DELETE RESTRICT ON UPDATE CASCADE`, que é a ação gerada para uma relação sem `onDelete`.

As regras abaixo aplicam a proposta FDD-PROP-22 (lacuna FDD-GAP-23) e, no último item, a FDD-PROP-15. Nenhuma delas foi decidida na reunião.

1. **FDD-DADOS-52: Pedido removido.** `orderId` em `webhook_outbox` e `webhook_dead_letter` não tem chave estrangeira para `orders` (proposta, ver FDD-PROP-22).
   - Efeito: remover um pedido `CANCELLED` não apaga nem bloqueia as linhas do evento `CANCELLED` desse pedido.
   - O que permite isso: o payload já renderizado (`[09:52] Larissa`) dispensa a leitura do pedido no envio.
   - Pedido em `PENDING`: não tem evento (FDD-PROP-09).
   - Com cascata, como em `order_status_history`, a entrega pendente seria apagada junto com o pedido.
   - Com `ON DELETE RESTRICT`, a remoção do pedido passaria a falhar.
2. **FDD-DADOS-53: Cliente removido, nas tabelas de evento.** `customerId` em `webhook_outbox` e `webhook_dead_letter` também não tem chave estrangeira (proposta, ver FDD-PROP-22). Assim, essas linhas não bloqueiam a remoção de um cliente e não são apagadas junto com ele.
3. **FDD-DADOS-54: Cadastro ligado ao cliente.** `Webhook.customer` é declarado com `@relation(fields: [customerId], references: [id])`, sem `onDelete`, como `Order.customer`. A migration gera `ON DELETE RESTRICT ON UPDATE CASCADE` (proposta, ver FDD-PROP-22).
   - Efeito: o banco recusa remover um cliente que tenha cadastro, como já recusa remover um cliente com pedido.
   - Resposta da API: o `errorMiddleware` só mapeia `P2002` e `P2025` entre os erros do Prisma, então essa recusa chegaria como `500 INTERNAL_SERVER_ERROR`.
4. **FDD-DADOS-55: Cadastro removido ou desativado.** As linhas `PENDING` da outbox desse cadastro vão para `webhook_dead_letter`, com o motivo registrado em `reason` (proposta, ver FDD-PROP-15). A mudança de URL não move nenhuma linha: o envio usa a URL vigente do cadastro (FDD-DADOS-19). O tratamento desse caso continua aberto em FDD-GAP-15.

Esta parte não define a chave estrangeira de `webhookId` em `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`. Com ela fica sem definição o destino das linhas `DELIVERED` e do histórico quando o cadastro é removido (ver "Lacunas adicionais sugeridas nesta parte"). A limpeza por idade continua fora do escopo (FDD-GAP-22).

### 5.4 Migration

- **FDD-DADOS-56: Geração.** O schema é alterado em `prisma/schema.prisma`, e a migration é gerada com `npm run db:migrate` (`prisma migrate dev`). O comando cria uma pasta nova em `prisma/migrations/`, ao lado de `20260519182739_init`. O nome da pasta é dado na geração e não é fixado aqui.
- **FDD-DADOS-57: Conteúdo.** A feature usa uma migration única (proposta, ver FDD-PROP-42). O SQL gerado deve seguir o formato da migration inicial e conter:
  - `CREATE TABLE` de `webhooks`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`;
  - o enum como `ENUM(...)` na própria coluna, como em `orders.status`;
  - datas em `DATETIME(3)`;
  - `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`;
  - os índices `webhook_outbox_status_idx` e `webhook_outbox_createdAt_idx`;
  - os índices propostos `webhooks_customerId_idx`, `webhook_deliveries_webhookId_idx` e `webhook_deliveries_createdAt_idx`, no padrão `<tabela>_<coluna>_idx`;
  - a chave estrangeira `webhooks_customerId_fkey` (FDD-DADOS-54);
  - nenhuma chave estrangeira das tabelas de evento para `orders` ou `customers`.
- **FDD-DADOS-58: Model `Customer`.** O model ganha o campo de relação inverso `webhooks Webhook[]`, como já tem `orders Order[]`. A tabela `customers` não ganha coluna: a migration inicial também não tem coluna para `orders`.
- **FDD-DADOS-59: Sem migração de dados.** As tabelas começam vazias. Mudanças de status anteriores à implantação não geram eventos, porque a emissão nasce em `changeStatus`.
- **FDD-DADOS-60: Testes.** A limpeza feita por `tests/setup.ts` em `beforeEach` passa a incluir as quatro tabelas novas e precisa respeitar a chave estrangeira de FDD-DADOS-54.

## 6. Fluxos detalhados
### 6.1 Criação do evento na outbox

- **Disparo:** chamada de `OrderService.changeStatus`, em `src/modules/orders/order.service.ts`, depois que `canTransition` aprova a transição. `OrderService.create` não chama a função (proposta, ver FDD-PROP-09).
- **Processo responsável:** API (processo de `src/server.ts`), dentro do callback de `this.prisma.$transaction` de `changeStatus`.

1. **FDD-FLUXO-01** — No mesmo callback que executa `tx.order.update` e `tx.orderStatusHistory.create`, `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`, função do módulo `src/modules/webhooks` (novo). A função recebe o `tx` (`TxClient`, alias de `Prisma.TransactionClient`), o pedido carregado no início da transação e os dois status. Nenhum repository é injetado no `OrderService` (`[09:41] Diego`).
2. **FDD-FLUXO-02** — Pelo `tx`, a função consulta em `webhooks` (proposta, ver FDD-PROP-14) os cadastros do `customerId` do pedido que estão ativos (o estado ativo vem de `[09:21] Sofia`; ignorar os inativos no filtro é proposta, ver FDD-PROP-27) e cuja lista `statuses` (proposta, ver FDD-PROP-12) contém o status de destino (`[09:33] Marcos`). A comparação com o destino segue a interpretação registrada no ADR-008.
3. **FDD-FLUXO-03** — Se nenhum cadastro é selecionado, a função retorna sem inserir nada e a transação segue normalmente (`[09:34] Bruno`). A ausência de inserção causada pelo filtro não provoca rollback (interpretação registrada no ADR-001).
4. **FDD-FLUXO-04** — Se há cadastros selecionados, a função gera um UUID para o evento (`[09:25] Diego`). Para isso usa o pacote `uuid`, que já é dependência em `package.json` e já é usado por `src/middlewares/request-logger.middleware.ts` (escolha proposta, ver FDD-PROP-17).
5. **FDD-FLUXO-05** — A função monta o payload uma única vez, com o estado do pedido naquele instante. Os campos são:
   - `event_id`: o UUID do passo anterior;
   - `event_type`: `order.status_changed`;
   - `timestamp`: em ISO 8601, fixado na renderização (interpretação registrada no ADR-007);
   - `order_id`, `order_number`, `customer_id` e `total_cents`: de `order.id`, `order.orderNumber`, `order.customerId` e `order.totalCents`;
   - `from_status` e `to_status`.

   O payload não inclui itens (`[09:43] Diego`).
6. **FDD-FLUXO-06** — Pelo `tx`, a função insere em `webhook_outbox` uma linha por cadastro selecionado (proposta, ver FDD-PROP-01), todas com o mesmo `event_id` e o mesmo payload (o `event_id` compartilhado entre destinos é interpretação a confirmar, registrada na seção 7.2). Cada linha entra como pendente (`[09:06] Diego`, `[09:08] Diego`), com a próxima tentativa devida de imediato (proposta, ver FDD-PROP-02).
7. **FDD-FLUXO-07** — Se a consulta ou a inserção lança erro, o erro se propaga e `$transaction` desfaz a atualização do pedido, o histórico e o movimento de estoque. O status não muda (`[09:40] Bruno`), e a resposta segue o mapeamento existente do `errorMiddleware`.

- **Estado resultante:** o pedido fica no novo status, o histórico fica gravado e há zero ou mais linhas pendentes em `webhook_outbox`, tudo no mesmo commit. Em caso de erro, nada é gravado.

### 6.2 Loop do worker

- **Disparo:** início do processo por `npm run worker` (novo), no entry-point `src/worker.ts` (novo); depois disso, um ciclo a cada 2 segundos (`[09:09] Diego`).
- **Processo responsável:** worker, em processo separado da API (`[09:11] Diego`). A lógica fica em `src/modules/webhooks/webhook.processor.ts` (novo; nome proposto, ver FDD-PROP-14 — a reunião deixou `webhook.worker.ts` ou `webhook.processor.ts`, `[09:28] Bruno`), e o processo usa a sua própria instância do singleton `prisma` de `src/config/database.ts` (`[09:30] Bruno`).

1. **FDD-FLUXO-08** — `bootstrap` segue o modelo de `src/server.ts` (os comportamentos listados a seguir são os desse arquivo). Falha na inicialização registra `bootstrap_failed` e encerra com `process.exit(1)`. `SIGINT` e `SIGTERM` acionam `shutdown`, que chama `prisma.$disconnect()`.
2. **FDD-FLUXO-09** — A cada ciclo, o worker lê em `webhook_outbox` as linhas pendentes cuja próxima tentativa já é devida (proposta, ver FDD-PROP-02). A leitura segue a ordem crescente de `created_at` (`[09:09] Diego`, `[09:12] Diego`), usa os índices de status e `created_at` e traz um lote pequeno (`[09:08] Diego`). O tamanho do lote está em FDD-PROP-04.
3. **FDD-FLUXO-10** — Antes de qualquer envio, o worker marca as linhas lidas como em processamento (`[09:08] Diego`).
4. **FDD-FLUXO-11** — O worker processa as linhas do lote pelos fluxos 6.3 e 6.4, em sequência, na ordem de `created_at`, para preservar a ordem por pedido (`[09:12] Diego`). O ciclo seguinte só começa quando o lote termina, para que haja um único processador ativo. Processar em paralelo apenas linhas de pedidos diferentes é proposta, a adotar só se a meta de latência não for atingida e sempre sujeita a FDD-PROP-24 (proposta, ver FDD-PROP-04).
5. **FDD-FLUXO-12** — Se o processo cai ou é encerrado com um lote em andamento, as linhas ficam em processamento. A reunião não tratou da recuperação; a retomada para pendente é a proposta FDD-PROP-03.

- **Estado resultante:** as linhas do lote ficam em processamento e seguem para 6.3. Com um único worker, a ordem por `created_at` dá a ordem por `order_id` (`[09:12] Diego`).

> Nota: a espera por uma nova tentativa pode fazer um evento posterior do mesmo pedido sair antes de um anterior. A reunião não tratou desse caso (lacuna FDD-GAP-25); a base de trabalho é processar em sequência e segurar o evento posterior (FDD-PROP-04 e FDD-PROP-24).

### 6.3 Assinatura e envio

- **Disparo:** cada linha em processamento do lote (FDD-FLUXO-11).
- **Processo responsável:** worker, em `src/modules/webhooks/webhook.processor.ts` (novo).

1. **FDD-FLUXO-13** — O worker carrega o cadastro de destino da linha: URL, secret e estado ativo. Se o cadastro está inativo ou foi removido, a linha vai direto para a fila de mortos pelo fluxo 6.5, com o motivo registrado (proposta, ver FDD-PROP-15).
2. **FDD-FLUXO-14** — O worker serializa em JSON o payload gravado na inserção, sem renderizar de novo a partir do pedido (`[09:52] Larissa`). Os bytes resultantes formam o corpo enviado.
3. **FDD-FLUXO-15** — Se o corpo passa de 64KB, o worker não envia (`[09:24] Larissa`). A linha vai direto para a fila de mortos, sem consumir tentativas (proposta, ver FDD-PROP-16). A verificação fica no worker porque, na inserção, um erro desfaria a mudança de status (FDD-FLUXO-07).
4. **FDD-FLUXO-16** — O worker calcula o HMAC-SHA256 do corpo com a secret do cadastro (`[09:22] Sofia`):
   - sobre os bytes exatos enviados, em hexadecimal e sem prefixo (proposta, ver FDD-PROP-06);
   - com `node:crypto` (proposta, ver FDD-PROP-17);
   - durante a carência de 24 horas, com a secret indicada em FDD-PROP-21 (proposta).

   A secret não entra em log; para isso, a redação de `src/shared/logger/index.ts` é estendida (proposta, ver FDD-PROP-08).
5. **FDD-FLUXO-17** — O worker monta os headers FDD-HEADER-01 a FDD-HEADER-05:
   - `X-Event-Id`: o `event_id` do payload (leitura do ADR-005);
   - `X-Timestamp`: o instante deste envio, recalculado a cada tentativa (`[09:44] Diego`);
   - `X-Webhook-Id`: o id do cadastro (`[09:44] Sofia`, `[09:45] Diego`).

   Os headers que ficam fora do HMAC seguem FDD-PROP-07.
6. **FDD-FLUXO-18** — O worker envia o request de FDD-CONTRATO-08 à URL do cadastro, com o `fetch` global do Node 20 (proposta, ver FDD-PROP-17) e timeout de 10 segundos (`[09:42] Diego`). Passado o prazo, a requisição é abortada e a tentativa conta como falha.
7. **FDD-FLUXO-19** — O worker mede o tempo de resposta e guarda, para o histórico, o código HTTP e o corpo da resposta, ou o erro de rede ou timeout (`[09:34] Marcos`).

- **Estado resultante:** a tentativa tem um resultado (resposta, erro de rede ou timeout) e segue para 6.4; ou a linha foi desviada para 6.5 sem envio.

### 6.4 Resultado da tentativa

- **Disparo:** fim de cada tentativa (FDD-FLUXO-19).
- **Processo responsável:** worker.

1. **FDD-FLUXO-20** — O worker classifica o resultado. Resposta 2xx é sucesso; qualquer outra resposta, erro de rede ou timeout é falha (proposta, ver FDD-PROP-05). O timeout ser falha é decisão da reunião (`[09:42] Diego`).
2. **FDD-FLUXO-21** — Em caso de sucesso, o worker marca a linha como entregue (`[09:08] Diego`). Também grava a entrega em `webhook_deliveries` (proposta, ver FDD-PROP-14), com sucesso, payload, resposta e tempo de resposta (`[09:34] Marcos`).
3. **FDD-FLUXO-22** — Em caso de falha, o worker grava a tentativa em `webhook_deliveries` com o motivo (`[09:34] Marcos`) e incrementa a contagem de tentativas da linha (proposta, ver FDD-PROP-01).
4. **FDD-FLUXO-23** — Se ainda restam novas tentativas, a linha volta a pendente (proposta, ver FDD-PROP-02). A próxima tentativa fica devida após o intervalo seguinte da tabela, contado a partir da falha (`[09:17] Larissa`):

   | Falha de | Intervalo até a próxima tentativa |
   |---|---|
   | envio inicial | 1 minuto |
   | 1ª nova tentativa | 5 minutos |
   | 2ª nova tentativa | 30 minutos |
   | 3ª nova tentativa | 2 horas |
   | 4ª nova tentativa | 12 horas |

5. **FDD-FLUXO-24** — Se a falha foi da 5ª nova tentativa, não há intervalo seguinte e a linha segue para 6.5 (proposta, ver FDD-PROP-11; interpretação registrada no ADR-003).
6. **FDD-FLUXO-25** — Cada resultado gera um log estruturado no logger Pino com `event_id`, id do cadastro, número da tentativa e resultado (proposta, ver FDD-PROP-20).

- **Estado resultante:** a linha está entregue; ou está pendente com a próxima tentativa agendada; ou está esgotada, a caminho da fila de mortos.

### 6.5 Esgotamento e fila de mortos

- **Disparo:** uma destas situações:
  - falha da 5ª nova tentativa (FDD-FLUXO-24);
  - desvio direto por cadastro inativo ou removido (FDD-FLUXO-13);
  - desvio direto por corpo acima de 64KB (FDD-FLUXO-15).
- **Processo responsável:** worker.

1. **FDD-FLUXO-26** — O worker marca a linha como falhou, estado do evento esgotado antes da ida à fila de mortos (proposta, ver FDD-PROP-02).
2. **FDD-FLUXO-27** — Em uma única transação, o worker:
   - insere em `webhook_dead_letter` o payload, o motivo da falha e o timestamp (`[09:18] Diego`);
   - inclui nesse registro o id do evento e o cadastro de destino, necessários ao replay (proposta, ver FDD-PROP-23);
   - remove a linha de `webhook_outbox` (proposta, ver FDD-PROP-02).
3. **FDD-FLUXO-28** — Se essa transação falha, a linha continua em falhou e o ciclo seguinte do worker repete o passo anterior (proposta, ver FDD-PROP-02).
4. **FDD-FLUXO-29** — O worker registra um log estruturado da ida à fila de mortos, com `event_id`, id do cadastro e motivo (proposta, ver FDD-PROP-20).

- **Estado resultante:** o evento sai da outbox e fica registrado em `webhook_dead_letter`. Nenhum novo envio acontece sem replay manual (`[09:18] Diego`).

### 6.6 Reprocessamento (replay)

- **Disparo:** um usuário com role ADMIN chama FDD-CONTRATO-07 (`/api/v1/admin/webhooks/dead-letter/:id/replay`; o prefixo `/api/v1` é o da API, montada em `src/app.ts`).
- **Processo responsável:** API, módulo `src/modules/webhooks` (novo).

1. **FDD-FLUXO-30** — `authenticate` e `requireRole` exigem a role ADMIN (`[09:36] Larissa`). Sem token ou com token inválido, a resposta é `401 UNAUTHORIZED`; com outro papel, `403 FORBIDDEN`.
2. **FDD-FLUXO-31** — `validate` confere o `:id` com `z.string().uuid()`, como nas rotas de módulo existentes. Um id inválido gera `400 VALIDATION_ERROR`.
3. **FDD-FLUXO-32** — O service busca o registro em `webhook_dead_letter`. Se ele não existe, a resposta é `404` com o código `WEBHOOK_DEAD_LETTER_NOT_FOUND` (seção 8).
4. **FDD-FLUXO-33** — O service recoloca o evento em `webhook_outbox` como pendente (`[09:18] Diego`). O evento volta com o mesmo id de evento, o mesmo payload, a contagem de tentativas zerada e a próxima tentativa devida de imediato (proposta, ver FDD-PROP-23). O destino do registro da fila de mortos após o replay não foi decidido (lacuna FDD-GAP-32).
5. **FDD-FLUXO-34** — O service registra em log, para auditoria, quem fez o replay (`sub` do JWT), o id do registro e o `event_id` (`[09:36] Sofia`).
6. **FDD-FLUXO-35** — Se o cadastro de destino está inativo ou foi removido, a linha recolocada volta à fila de mortos quando o worker a pega (proposta, ver FDD-PROP-15; FDD-FLUXO-13).

- **Estado resultante:** o evento fica pendente em `webhook_outbox` e é pego pelo loop de 6.2 no ciclo seguinte. A resposta segue FDD-CONTRATO-07. O cliente pode receber o evento de novo e deduplicar pelo `X-Event-Id` (ADR-005).

### 6.7 Rotação de secret

- **Disparo:** chamada a FDD-CONTRATO-06, na rota `POST /api/v1/webhooks/:id/rotate-secret` (proposta, ver FDD-PROP-13).
- **Processo responsável:** API, módulo `src/modules/webhooks` (novo).

1. **FDD-FLUXO-36** — O router aplica `authenticate`, e `validate` confere o `:id` com `z.string().uuid()`. Um cadastro inexistente gera a resposta `404` com o código `WEBHOOK_NOT_FOUND` (seção 8).
2. **FDD-FLUXO-37** — O service gera a nova secret com o gerador criptograficamente seguro do Node (proposta, ver FDD-PROP-19).
3. **FDD-FLUXO-38** — O service grava a nova secret e mantém a anterior com validade de 24 horas a partir da rotação (`[09:21] Sofia`). As duas ficam guardadas de forma recuperável e não aparecem em log (proposta, ver FDD-PROP-08).
4. **FDD-FLUXO-39** — A resposta de FDD-CONTRATO-06 devolve a nova secret (`[09:21] Sofia`).
5. **FDD-FLUXO-40** — A partir da rotação, o worker assina com a secret nova; a antiga continua aceita pelo cliente durante as 24 horas e deixa de valer ao fim do prazo (proposta, ver FDD-PROP-21).

- **Estado resultante:** o cadastro tem a secret nova e a antiga em carência. Passadas as 24 horas, só a nova vale.

### 6.8 Cadastro, edição, remoção e listagem

- **Disparo:** chamadas a FDD-CONTRATO-01 a FDD-CONTRATO-04.
- **Processo responsável:** API, em `src/modules/webhooks/webhook.routes.ts` (novo), `src/modules/webhooks/webhook.controller.ts` (novo), `src/modules/webhooks/webhook.service.ts` (novo) e `src/modules/webhooks/webhook.repository.ts` (novo).

1. **FDD-FLUXO-41** — O router aplica `authenticate`. Nesta fase, qualquer usuário autenticado pode usar estas operações (`[09:37] Sofia`). Essa abertura parte da premissa de `[09:32] Marcos`, que não vale no código; ver FDD-RISCO-05 e FDD-GAP-46. Cada rota aplica `validate` com os schemas de `src/modules/webhooks/webhook.schemas.ts` (novo). Na edição e na remoção, o `:id` é validado com `z.string().uuid()`, e um cadastro inexistente gera a resposta `404` com o código `WEBHOOK_NOT_FOUND` (seção 8).
2. **FDD-FLUXO-42** — O cliente vem da requisição, nunca do JWT (`[09:32] Larissa`). O `customerId` vai no body na criação e em query na listagem; a edição não troca de cliente (proposta, ver FDD-PROP-13 e FDD-PROP-31). A reunião disse "no body ou no path"; o uso de query na listagem é uma divergência registrada em FDD-PROP-13. Na criação e na listagem, o service confere se o cliente existe, como `OrderService.create` faz com `tx.customer.findUnique`; se ele não existe, a resposta é o erro de não encontrado.
3. **FDD-FLUXO-43** — O schema exige URL `https`. Uma URL `http` é recusada pelo schema Zod com `400 VALIDATION_ERROR` (`[09:23] Sofia`; código conforme FDD-PROP-10). Cada valor de `statuses` é validado com `z.nativeEnum(OrderStatus)` (ADR-008; nome do campo em FDD-PROP-12).
4. **FDD-FLUXO-44** — Na criação, a plataforma gera a secret (proposta, ver FDD-PROP-19), grava o cadastro e responde `201` com a secret (`[09:31] Marcos`). Entre as operações deste fluxo, a criação é a única que devolve a secret; fora delas, só a rotação devolve a nova (FDD-FLUXO-39).
5. **FDD-FLUXO-45** — Na edição, o service altera os campos enviados e responde `200`, sem a secret. Uma nova lista `statuses` só vale para mudanças de status futuras, porque o filtro é aplicado na inserção (ADR-008). Entregas pendentes de um cadastro desativado vão para a fila de mortos (proposta, ver FDD-PROP-15). Com a URL alterada, o carregamento do cadastro no envio (FDD-FLUXO-13) usa a URL vigente; a confirmação desse efeito fica em FDD-GAP-15.
6. **FDD-FLUXO-46** — Na remoção, a resposta é `204` sem corpo. As entregas pendentes do cadastro removido vão para a fila de mortos quando o worker as pega (proposta, ver FDD-PROP-15; FDD-FLUXO-13).
7. **FDD-FLUXO-47** — Na listagem, a resposta é `200` com o envelope de `paginated()` de `src/shared/http/response.ts`, sem a secret.

- **Estado resultante:** o cadastro é criado, alterado ou removido. As mudanças de status seguintes do cliente passam a ser filtradas pela configuração vigente no momento da inserção (FDD-FLUXO-02).

### 6.9 Consulta do histórico de entregas

- **Disparo:** chamada a FDD-CONTRATO-05 (`GET /api/v1/webhooks/:id/deliveries`).
- **Processo responsável:** API, módulo `src/modules/webhooks` (novo).

1. **FDD-FLUXO-48** — O router aplica `authenticate`, e `validate` confere o `:id` com `z.string().uuid()`. Um cadastro inexistente gera a resposta `404` com o código `WEBHOOK_NOT_FOUND` (seção 8).
2. **FDD-FLUXO-49** — O service lê em `webhook_deliveries` as últimas 100 entregas do cadastro, da mais recente para a mais antiga. Cada entrega traz sucesso ou falha, payload, resposta e tempo de resposta (`[09:34] Marcos`).
3. **FDD-FLUXO-50** — A resposta é `200`, como nas listagens existentes, no formato de FDD-CONTRATO-05.

- **Estado resultante:** leitura apenas, sem efeito em `webhook_outbox` nem em `webhook_deliveries`.

## 7. Contratos públicos
Esta seção define os contratos expostos pela feature: a API de gerenciamento (7.1) e a entrega do evento ao cliente (7.2).

### 7.1 API de gerenciamento (rotas que a plataforma expõe)

Os caminhos completos levam o prefixo `/api/v1`, com que a API é montada em `src/app.ts`. Os valores dos exemplos desta subseção são ilustrativos: ids, URLs, datas, números de pedido e valores. As regras abaixo valem para os sete endpoints.

1. Todas as rotas passam por `authenticate`.
   - Os endpoints de FDD-CONTRATO-01 a FDD-CONTRATO-06 aceitam qualquer papel autenticado (`[09:37] Sofia`). A resposta vale para "o resto do CRUD de configuração" (`[09:36] Marcos`); o histórico de entregas (FDD-CONTRATO-05) é tratado como parte desse gerenciamento (leitura adotada, a confirmar junto com a questão do endurecimento, RFC-QA-05).    A resposta vale para "o resto do CRUD de configuração" (`[09:36] Marcos`); o histórico de entregas (FDD-CONTRATO-05) é tratado como parte desse gerenciamento (leitura adotada, a confirmar junto com a questão do endurecimento, RFC-QA-05).
   - O JWT carrega `sub`, `email` e `role`, mas não carrega o cliente (`src/middlewares/auth.middleware.ts`). Por isso o cliente é indicado por `customerId` na requisição (`[09:32] Larissa`).
2. Os nomes de campos seguem camelCase e a lista de status fica no campo `statuses` (proposta, ver FDD-PROP-12).
   - `statuses` é validado com `z.array(z.nativeEnum(OrderStatus))`, no padrão de `src/modules/orders/order.schemas.ts`.
3. A `url` deve usar `https` (`[09:23] Sofia`). O schema Zod rejeita qualquer outro esquema, e a resposta é `400 VALIDATION_ERROR` (proposta, ver FDD-PROP-10).
4. O parâmetro `:id` é validado com `z.string().uuid()`. Entrada fora do schema vira `400 VALIDATION_ERROR`, com `details` no formato `{ path, message }` (`src/middlewares/validate.middleware.ts`).
5. Os erros seguem o envelope `{ "error": { "code", "message", "details"? } }` do `errorMiddleware` (`src/middlewares/error.middleware.ts`). Os códigos novos, com prefixo `WEBHOOK_`, são detalhados na seção 8.
6. A `secret` só aparece nas respostas de FDD-CONTRATO-01 e FDD-CONTRATO-06. A redação dela no logger segue a proposta FDD-PROP-08.

#### `POST /api/v1/webhooks` — FDD-CONTRATO-01

- **Autenticação:** `authenticate`.
- **Request:** o corpo tem três campos.
  - `url`: obrigatória, com `https`.
  - `customerId`: `z.string().uuid()`, como em `createOrderSchema`.
  - `statuses`: ao menos um valor de `OrderStatus`. Exigir ao menos um valor é proposta (ver FDD-PROP-28).

  O corpo não traz `secret`, porque ela é gerada pela plataforma (`[09:31] Marcos`).

```json
{
  "url": "https://cliente.example.com/webhooks/pedidos",
  "customerId": "c1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

- **Sucesso:** `201`, como em `CustomerController.create` (`src/modules/customers/customer.controller.ts`). A resposta traz o cadastro e a secret gerada.

```json
{
  "id": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e",
  "customerId": "c1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b",
  "url": "https://cliente.example.com/webhooks/pedidos",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "<secret gerada pela plataforma>",
  "createdAt": "2026-11-25T10:00:00.000Z",
  "updatedAt": "2026-11-25T10:00:00.000Z"
}
```

- **Erros:**
  - `400` — `VALIDATION_ERROR` — `url` ausente ou sem `https`; `customerId` fora do formato UUID; `statuses` vazia ou com valor fora de `OrderStatus`.
  - `401` — `UNAUTHORIZED` — sem token ou com token inválido.
  - `404` — `NOT_FOUND` — `customerId` não corresponde a um cliente existente (`NotFoundError('Customer')`, no padrão de `src/modules/orders/order.service.ts`).
- **Semântica:**
  - Cria o cadastro, gera a secret (proposta, ver FDD-PROP-19) e a devolve nesta resposta (`[09:31] Marcos`).
  - O cadastro nasce com `active: true` (proposta, ver FDD-PROP-29).
  - `PENDING` é aceito pelo schema, mas não gera entrega, porque só `changeStatus` emite evento (proposta, ver FDD-PROP-09).

#### `GET /api/v1/webhooks` — FDD-CONTRATO-02

- **Autenticação:** `authenticate`.
- **Request:** a query tem três parâmetros.
  - `customerId`: obrigatório (proposta, ver FDD-PROP-13; a reunião disse "no body ou no path" em `[09:32] Larissa` — o uso de query segue os filtros de listagem existentes e é uma divergência registrada).
  - `page`: padrão 1.
  - `pageSize`: padrão 20, máximo 100.

  `page` e `pageSize` copiam `listCustomersQuerySchema` (`src/modules/customers/customer.schemas.ts`). Query após a validação:

```json
{ "customerId": "c1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b", "page": 1, "pageSize": 20 }
```

- **Sucesso:** `200`, com o envelope de `paginated()` (`src/shared/http/response.ts`).

```json
{
  "data": [
    { "id": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e", "customerId": "c1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b",
      "url": "https://cliente.example.com/webhooks/pedidos", "statuses": ["SHIPPED", "DELIVERED"],
      "active": true, "createdAt": "2026-11-25T10:00:00.000Z", "updatedAt": "2026-11-25T10:00:00.000Z" }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

- **Erros:**
  - `400` — `VALIDATION_ERROR` — `customerId` ausente ou fora do formato UUID; `page` ou `pageSize` fora dos limites.
  - `401` — `UNAUTHORIZED` — sem token ou com token inválido.
  - `404` — `NOT_FOUND` — `customerId` não corresponde a um cliente existente.
- **Semântica:**
  - Lista os cadastros do cliente (`[09:33] Bruno`), do mais recente para o mais antigo (convenção do código: `orderBy: { createdAt: 'desc' }`, como em `CustomerRepository.list`; proposta).
  - Os inativos também aparecem, com `active: false` (proposta, ver FDD-PROP-30).
  - A `secret` não aparece (consequência técnica inferida). Ela é única por endpoint para limitar o alcance de um vazamento (`[09:21] Sofia`), e já houve cliente que vazou secret em log (`[09:22] Diego`). O cliente já a recebe na criação e na rotação.

#### `PATCH /api/v1/webhooks/:id` — FDD-CONTRATO-03

- **Autenticação:** `authenticate`.
- **Request:** o `id` do cadastro vai no path.
  - O corpo aceita `url`, `statuses` e `active`, todos opcionais, no padrão `.partial()` de `updateCustomerSchema`.
  - O schema de edição não contém `customerId`, logo o cadastro não troca de cliente. Esse ponto não foi discutido (proposta, ver FDD-PROP-31).

```json
{ "statuses": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

- **Sucesso:** `200`, como em `CustomerController.update`. A resposta traz o cadastro atualizado, sem `secret`.

```json
{
  "id": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e", "customerId": "c1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b",
  "url": "https://cliente.example.com/webhooks/pedidos", "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false, "createdAt": "2026-11-25T10:00:00.000Z", "updatedAt": "2026-11-25T11:30:00.000Z"
}
```

- **Erros:**
  - `400` — `VALIDATION_ERROR` — `id` fora do formato UUID; `url` sem `https`; `statuses` vazia ou com valor fora de `OrderStatus`; `active` não booleano.
  - `401` — `UNAUTHORIZED` — sem token ou com token inválido.
  - `404` — `WEBHOOK_NOT_FOUND` — não existe cadastro com o `id` informado (ver seção 8).
- **Semântica:**
  - Altera só os campos enviados, como `CustomerService.update`.
  - O filtro é aplicado na inserção (ADR-008). Por isso, um novo `statuses` vale apenas para as mudanças de status seguintes, e as linhas já inseridas na outbox não mudam.
  - Com `active: false`, as entregas pendentes vão para a DLQ, com o motivo registrado (proposta, ver FDD-PROP-15).
  - O destino das entregas pendentes após troca de `url` é a lacuna FDD-GAP-15.

#### `DELETE /api/v1/webhooks/:id` — FDD-CONTRATO-04

- **Autenticação:** `authenticate`.
- **Request:** o `id` do cadastro vai no path; não há corpo. Parâmetros após a validação:

```json
{ "id": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e" }
```

- **Sucesso:** `204`, sem corpo, como em `CustomerController.delete`. Por isso não há bloco de exemplo.
- **Erros:**
  - `400` — `VALIDATION_ERROR` — `id` fora do formato UUID.
  - `401` — `UNAUTHORIZED` — sem token ou com token inválido.
  - `404` — `WEBHOOK_NOT_FOUND` — não existe cadastro com o `id` informado (ver seção 8).
- **Semântica:**
  - Remove o cadastro (`[09:33] Bruno`), com busca prévia e erro de inexistência, como em `CustomerService.delete`.
  - As entregas pendentes do cadastro vão para a DLQ, com o motivo registrado (proposta, ver FDD-PROP-15).
  - O efeito da remoção sobre o histórico de entregas e sobre a DLQ desse cadastro é a lacuna FDD-GAP-23.

#### `GET /api/v1/webhooks/:id/deliveries` — FDD-CONTRATO-05

- **Autenticação:** `authenticate`.
- **Request:** o `id` do cadastro vai no path; não há query nem corpo. Parâmetros após a validação:

```json
{ "id": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e" }
```

- **Sucesso:** `200`, como em `CustomerController.list`.
  - A resposta traz até 100 itens em `data`, do mais recente para o mais antigo (a ordem é proposta, ver FDD-PROP-26).
  - O `payload` é o corpo do evento enviado, com campos em snake_case (seção 7.2).

```json
{
  "data": [
    {
      "id": "d4c3b2a1-0f9e-4d8c-b7a6-5e4d3c2b1a09",
      "eventId": "e9f8a7b6-c5d4-4e3f-a2b1-0f9e8d7c6b5a",
      "success": false,
      "payload": {
        "event_id": "e9f8a7b6-c5d4-4e3f-a2b1-0f9e8d7c6b5a", "event_type": "order.status_changed",
        "timestamp": "2026-11-25T10:05:00.000Z", "order_id": "0a1b2c3d-4e5f-4a6b-8c7d-9e0f1a2b3c4d",
        "order_number": "ORD-000123", "from_status": "PROCESSING", "to_status": "SHIPPED",
        "customer_id": "c1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b", "total_cents": 15990
      },
      "responseStatus": 503, "responseBody": "Service Unavailable", "responseTimeMs": 412,
      "createdAt": "2026-11-25T10:05:02.000Z"
    }
  ]
}
```

- **Erros:**
  - `400` — `VALIDATION_ERROR` — `id` fora do formato UUID.
  - `401` — `UNAUTHORIZED` — sem token ou com token inválido.
  - `404` — `WEBHOOK_NOT_FOUND` — não existe cadastro com o `id` informado (ver seção 8).
- **Semântica:**
  - Devolve as últimas 100 entregas do cadastro. Cada uma traz resultado (sucesso ou falha), payload, resposta e tempo de resposta (`[09:34] Marcos`).
  - `success` vale `true` quando a resposta é 2xx e `false` nos demais casos (proposta, ver FDD-PROP-05).
  - Quando não há resposta HTTP (erro de rede ou timeout de 10 segundos, `[09:42] Diego`), `responseStatus` e `responseBody` vêm `null`.
  - O formato é proposta (ver FDD-PROP-26): lista simples em `data`, sem `pagination`. O pedido é uma janela fixa ("os últimos 100"), e não navegação pelo histórico completo. A chave `data` mantém a forma que o cliente já lê nas demais listagens.
- **Interpretação a confirmar:**
  - Leitura adotada: cada item é um envio HTTP, porque resposta e tempo de resposta existem por envio.
  - Outra leitura possível de `[09:34] Marcos`: um item por evento, com o resultado final.

#### `POST /api/v1/webhooks/:id/rotate-secret` — FDD-CONTRATO-06

- **Autenticação:** `authenticate`.
- **Request:** a rota é proposta (ver FDD-PROP-13). O `id` do cadastro vai no path; não há corpo. Parâmetros após a validação:

```json
{ "id": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e" }
```

- **Sucesso:** `200` (proposta, ver FDD-PROP-32), como nas atualizações dos controllers existentes, porque a rotação altera um cadastro e não cria recurso. A resposta traz a nova secret e o instante em que a anterior deixa de valer (formato proposto, ver FDD-PROP-32).

```json
{
  "id": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e",
  "secret": "<secret gerada pela plataforma>",
  "previousSecretValidUntil": "2026-11-26T14:00:00.000Z"
}
```

- **Erros:**
  - `400` — `VALIDATION_ERROR` — `id` fora do formato UUID.
  - `401` — `UNAUTHORIZED` — sem token ou com token inválido.
  - `404` — `WEBHOOK_NOT_FOUND` — não existe cadastro com o `id` informado (ver seção 8).
- **Semântica:**
  - Gera uma nova secret (proposta, ver FDD-PROP-19) e a devolve.
  - A secret anterior continua válida por 24 horas, em paralelo com a nova, e depois deixa de valer (`[09:21] Sofia`). `previousSecretValidUntil` é o instante da rotação somado às 24 horas.
  - A secret que assina as entregas nesse intervalo segue a proposta FDD-PROP-21.
  - O efeito de uma nova rotação antes do fim das 24 horas não foi discutido (lacuna FDD-GAP-33).

#### `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — FDD-CONTRATO-07

- **Autenticação:** `authenticate` e `requireRole('ADMIN')` (`[09:36] Sofia`, `[09:36] Larissa`).
- **Request:** o `id` do item em `webhook_dead_letter` vai no path; não há corpo. Parâmetros após a validação:

```json
{ "id": "f1e2d3c4-b5a6-4978-8a6b-5c4d3e2f1a0b" }
```

- **Sucesso:** `200` (proposta, ver FDD-PROP-32), como nas atualizações dos controllers existentes, que respondem `200` quando a operação não cria recurso. A resposta traz os identificadores do evento recolocado na outbox.

```json
{
  "eventId": "e9f8a7b6-c5d4-4e3f-a2b1-0f9e8d7c6b5a",
  "webhookId": "7b2e4c1a-5d3f-4a8b-9c6e-1f0a2b3c4d5e"
}
```

- **Erros:**
  - `400` — `VALIDATION_ERROR` — `id` fora do formato UUID.
  - `401` — `UNAUTHORIZED` — sem token ou com token inválido.
  - `403` — `FORBIDDEN` — usuário autenticado sem role ADMIN (`requireRole`).
  - `404` — `WEBHOOK_DEAD_LETTER_NOT_FOUND` — não existe item na fila de mortos com o `id` informado (código novo, proposto; ver seção 8).
- **Semântica:**
  - Recoloca o evento na outbox como pendente (`[09:18] Diego`). O id de evento e o payload são os mesmos, e a contagem de tentativas é zerada (proposta, ver FDD-PROP-23).
  - Registra em log, para auditoria, quem fez o replay (`[09:36] Sofia`). O registro contém o `req.user.id` preenchido por `authenticate`, o `id` do item e o id do evento.
  - Dois pontos não foram discutidos (lacuna FDD-GAP-32): se o item continua na fila de mortos após o replay, e o replay de item cujo cadastro foi removido ou está inativo.

A consulta por id (`GET /api/v1/webhooks/:id`) não consta das rotas que Bruno enumerou na reunião (`[09:33] Bruno`): não foi pedida e não faz parte destes contratos (ver FDD-PROP-13).


### 7.2 Entrega do evento — FDD-CONTRATO-08

FDD-CONTRATO-08 é o request HTTP que o worker faz para a URL cadastrada pelo cliente, a partir de uma entrega pendente da outbox (ADR-001, ADR-002). Nenhuma chamada HTTP acontece dentro da transação de mudança de status (ADR-001).

**Método e URL**

- O worker faz `POST` para a URL do cadastro. Essa URL é sempre `https`, porque a URL `http` é recusada no cadastro com erro de validação (`[09:23] Sofia`; ver seção 8).
- Interpretação a confirmar: a reunião não ditou o método da entrega. O FDD usa `POST` porque a entrega carrega corpo JSON (`Content-Type: application/json`, `[09:44] Diego`), e não segue redirecionamento (proposta, ver FDD-PROP-33; lacuna FDD-GAP-30). O ponto fica para a revisão com Bruno e Diego (`[09:50] Larissa`).

**Headers**

| ID | Header | Valor ou exemplo | Finalidade | Origem |
|---|---|---|---|---|
| FDD-HEADER-01 | `Content-Type` | `application/json` | Indica que o corpo é JSON | `[09:44] Diego` |
| FDD-HEADER-02 | `X-Event-Id` | UUID do evento, gerado na entrada na outbox; ex.: `5f0c7a2e-3b1d-4c8e-9a6f-2d4b8e1c7a90` | Permite ao cliente deduplicar na garantia at-least-once (ADR-005) | `[09:25] Diego`, `[09:44] Diego` |
| FDD-HEADER-03 | `X-Signature` | HMAC-SHA256 do corpo com a secret do cadastro. Codificação hexadecimal sem prefixo (proposta, ver FDD-PROP-06). Secret usada na carência de 24 horas: proposta, ver FDD-PROP-21 | Permite ao cliente validar a origem e a integridade do corpo (ADR-004) | `[09:20] Sofia`, `[09:22] Sofia`, `[09:44] Diego` |
| FDD-HEADER-04 | `X-Timestamp` | Instante do envio, gerado a cada tentativa; em ISO 8601 (formato não ditado na reunião; proposta, ver FDD-PROP-34) | Permite ao cliente detectar replay attack, se quiser | `[09:44] Diego` |
| FDD-HEADER-05 | `X-Webhook-Id` | UUID do cadastro em `webhooks` (nome proposto, ver FDD-PROP-14) | Permite ao cliente com vários cadastros saber qual deles recebeu o envio | `[09:44] Sofia`, `[09:45] Diego` |

- FDD-HEADER-02 e o `event_id` do corpo carregam o mesmo valor (leitura adotada no ADR-005). Esse valor se mantém em todas as tentativas do evento, porque o UUID é único por evento (`[09:25] Diego`), e também no replay (proposta, ver FDD-PROP-23).
- Interpretação a confirmar: a proposta é ter uma linha por evento × destino (ver FDD-PROP-01), e o UUID é "único por evento" (`[09:25] Diego`). Nessa leitura, um evento destinado a dois cadastros do mesmo cliente sai com o mesmo `event_id` e com `X-Webhook-Id` diferentes. Na outra leitura, cada linha da outbox gera um UUID próprio, e cada destino recebe um `event_id` diferente.

**Corpo**

O corpo é o payload gravado na inserção (snapshot, ADR-007) e é enviado sem nova renderização. Os campos são exatamente os ditos em `[09:43] Diego`, em snake_case. Os valores abaixo são ilustrativos:

```json
{
  "event_id": "5f0c7a2e-3b1d-4c8e-9a6f-2d4b8e1c7a90",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-25T13:00:00.000Z",
  "order_id": "a3e1c9d2-7b4f-4e2a-8c1d-6f9b0e2a4c71",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "c7b2e4f1-9a3d-4b6c-8e0f-1d2a3b4c5d6e",
  "total_cents": 15990
}
```

| Campo | Tipo | Origem |
|---|---|---|
| `event_id` | texto (UUID), com o mesmo valor de FDD-HEADER-02 (leitura adotada no ADR-005) | `[09:25] Diego`, `[09:43] Diego` |
| `event_type` | texto, com o valor `"order.status_changed"` | `[09:43] Diego` |
| `timestamp` | texto ISO 8601, fixado na renderização feita na inserção (leitura adotada no ADR-007) | `[09:43] Diego` |
| `order_id` | texto (UUID), vindo de `Order.id` | `[09:43] Diego`; `prisma/schema.prisma` |
| `order_number` | texto, vindo de `Order.orderNumber`; formato `ORD-` seguido de ao menos 6 dígitos, com zeros à esquerda, gerado por `reserveOrderNumber` | `[09:43] Diego`; `src/modules/orders/order.service.ts` |
| `from_status` | texto, valor de `OrderStatus` | `[09:43] Diego`; `src/modules/orders/order.status.ts` |
| `to_status` | texto, valor de `OrderStatus` | `[09:43] Diego`; `src/modules/orders/order.status.ts` |
| `customer_id` | texto (UUID), vindo de `Order.customerId` | `[09:43] Diego`; `prisma/schema.prisma` |
| `total_cents` | inteiro, vindo de `Order.totalCents` | `[09:43] Diego`; `prisma/schema.prisma` |

- O corpo não leva `items`, para não inflar o payload. Quem quiser detalhes consulta `GET /api/v1/orders/:id` (`[09:43] Diego`).
- `from_status` é sempre preenchido. A emissão parte só de `OrderService.changeStatus`, e nenhuma transição leva a `PENDING` (`src/modules/orders/order.status.ts`; ver FDD-PROP-09).
- Interpretação a confirmar: em "os campos básicos da order tipo total_cents" (`[09:43] Diego`), `total_cents` aparece como exemplo. Este FDD inclui só os campos nomeados. Acrescentar outros campos do model `Order` fica para a revisão do FDD.

**Resposta esperada e critério de sucesso**

1. **Sucesso:** resposta com código 2xx (proposta, ver FDD-PROP-05). A entrega é marcada como entregue.
2. **Falha:** resposta fora de 2xx, erro de rede ou timeout (proposta, ver FDD-PROP-05). A entrega segue para nova tentativa conforme o ADR-003. Os motivos estão em FDD-ERRO-04 a FDD-ERRO-06.
3. **Timeout:** a chamada sem resposta em 10 segundos é tratada como falha e marcada para retry (`[09:42] Diego`).
4. **Limite de tamanho:** o worker não envia corpo acima de 64KB (`[09:23] Sofia`, `[09:24] Larissa`). A entrega vai direto para a DLQ com o código de FDD-ERRO-03 (proposta, ver FDD-PROP-16).
5. **Corpo da resposta:** a reunião não exigiu formato para a resposta do cliente, e o worker decide pelo código HTTP (FDD-PROP-05). O histórico guarda sucesso ou falha, payload, resposta e tempo de resposta (`[09:34] Marcos`).

**Verificação do lado do cliente**

1. Recomputar o HMAC-SHA256 do corpo recebido com a secret do cadastro e comparar o resultado com `X-Signature` (`[09:20] Sofia`). Durante as 24 horas de carência da rotação, aceitar as duas secrets (proposta, ver FDD-PROP-21).
2. Descartar repetições pelo identificador do evento, `X-Event-Id` ou `event_id` (`[09:25] Diego`).
3. Usar `X-Timestamp` contra replay, se quiser (`[09:44] Diego`). Os headers não entram no HMAC; o corpo inteiro é assinado, inclusive os campos `event_id` e `timestamp` (lacuna FDD-GAP-07; proposta, ver FDD-PROP-07).

## 8. Matriz de erros previstos
Os códigos novos usam o prefixo `WEBHOOK_` (`[09:29] Larissa`). Os que são respostas da API viram classes específicas derivadas de `AppError`, tratadas pelo `errorMiddleware` sem alteração (ADR-006). `NotFoundError` grava o código fixo `NOT_FOUND` (`src/shared/errors/http-errors.ts`). Por isso, os erros 404 do módulo estendem `AppError` diretamente, com `statusCode` 404, como faz `NotFoundError`.

| ID | Código | HTTP | Onde ocorre | Gatilho | Origem |
|---|---|---|---|---|---|
| FDD-ERRO-01 | `WEBHOOK_NOT_FOUND` | 404 | FDD-CONTRATO-03, FDD-CONTRATO-04, FDD-CONTRATO-05 e FDD-CONTRATO-06 | O `:id`, já validado como UUID, não corresponde a cadastro em `webhooks` (nome proposto, ver FDD-PROP-14) | `[09:28] Bruno` |
| FDD-ERRO-02 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | FDD-CONTRATO-07, depois de `requireRole` | O `:id`, já validado como UUID, não corresponde a item de `webhook_dead_letter` | proposta |
| FDD-ERRO-03 | `WEBHOOK_PAYLOAD_TOO_LARGE` | não se aplica | Worker, antes do envio de FDD-CONTRATO-08 | Corpo acima de 64KB (`[09:24] Larissa`); a entrega vai direto para a DLQ (proposta, ver FDD-PROP-16) | proposta |

- FDD-ERRO-03 não é resposta da API. É o motivo gravado na DLQ, e por isso não precisa de classe `AppError`.

**Códigos citados na reunião e não utilizados.** `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` foram apenas exemplos do padrão em `[09:28] Bruno` ("etc.") e não recebem ID. URL sem `https` responde `400 VALIDATION_ERROR`, porque a recusa é "só uma validação no schema Zod" (`[09:23] Sofia`; proposta, ver FDD-PROP-10). A secret é gerada pela plataforma e devolvida na criação, sem ser informada pelo cliente (`[09:31] Marcos`). Assim, não existe o caso de secret ausente na entrada.

**Códigos que as rotas do módulo respondem pelos componentes reusados.** Esses códigos não levam o prefixo `WEBHOOK_`, porque vêm de componentes usados sem alteração (leitura (a) do ADR-006).

| Código | HTTP | Quando ocorre nas rotas do módulo | Origem |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | `params`, `query` ou `body` fora do schema Zod: id que não é UUID (`z.string().uuid()`), URL sem `https` (proposta, ver FDD-PROP-10) ou status fora de `z.nativeEnum(OrderStatus)` | `src/middlewares/validate.middleware.ts` |
| `UNAUTHORIZED` | 401 | Ausência de header `Authorization` Bearer, ou token inválido ou expirado, em todas as rotas do módulo | `src/middlewares/auth.middleware.ts` |
| `FORBIDDEN` | 403 | Usuário sem role ADMIN em FDD-CONTRATO-07 (`[09:36] Sofia`) | `src/middlewares/auth.middleware.ts` |
| `NOT_FOUND` | 404 | `customerId` inexistente no body da criação ou na query da listagem (FDD-CONTRATO-01 e FDD-CONTRATO-02; proposta, ver FDD-PROP-13). A verificação é feita antes da escrita, como em `OrderService.create`, porque o `errorMiddleware` só mapeia `P2002` e `P2025`: uma falha da chave estrangeira proposta em FDD-PROP-22 cairia em 500. Também cobre o `P2025` do Prisma | `src/shared/errors/http-errors.ts` |
| `CONFLICT` | 409 | Violação de unicidade (`P2002`) mapeada pelo `errorMiddleware`; nenhuma restrição única das tabelas novas foi decidida | `src/middlewares/error.middleware.ts` |
| `INTERNAL_SERVER_ERROR` | 500 | Qualquer erro não mapeado, logado como `Unhandled error in request` com `requestId` | `src/middlewares/error.middleware.ts` |

**Motivos de falha de entrega registrados pelo worker.** Os motivos abaixo não são respostas HTTP da API. São gravados como texto no histórico de entregas e no motivo da DLQ (`[09:18] Diego`). Os nomes dos códigos são proposta.

| ID | Código (proposta) | Motivo | Efeito | Origem |
|---|---|---|---|---|
| FDD-ERRO-04 | `WEBHOOK_DELIVERY_TIMEOUT` | Sem resposta em 10 segundos | Consome tentativa | `[09:42] Diego`; nome: proposta |
| FDD-ERRO-05 | `WEBHOOK_DELIVERY_CONNECTION_ERROR` | Erro de rede ou de conexão antes de qualquer resposta | Consome tentativa | proposta (FDD-PROP-05) |
| FDD-ERRO-06 | `WEBHOOK_DELIVERY_UNEXPECTED_STATUS` | Resposta fora de 2xx; o código recebido vai para o histórico | Consome tentativa | proposta (FDD-PROP-05) |
| FDD-ERRO-07 | `WEBHOOK_INACTIVE` | Cadastro desativado com entrega pendente | Direto para a DLQ | proposta (FDD-PROP-15) |
| FDD-ERRO-08 | `WEBHOOK_REMOVED` | Cadastro removido com entrega pendente | Direto para a DLQ | proposta (FDD-PROP-15) |

- Nos motivos que consomem tentativa, a entrega volta a pendente com a próxima tentativa agendada pela tabela 1m/5m/30m/2h/12h (ADR-003; estados em FDD-PROP-02). Esgotado o teto, segue para `webhook_dead_letter`. O teto é de 1 envio inicial + 5 novas tentativas (proposta, ver FDD-PROP-11).
- Payload acima de 64KB também é registrado como motivo, com o código de FDD-ERRO-03, e vai direto para a DLQ sem envio.

**Envelope do corpo de erro.** As rotas do módulo respondem com o envelope do `errorMiddleware` (`src/middlewares/error.middleware.ts`). A mensagem abaixo é ilustrativa e segue o padrão de `NotFoundError`:

```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

- `details` só aparece quando o erro o informa. Em `VALIDATION_ERROR`, é uma lista de `{ path, message }` (`src/middlewares/validate.middleware.ts`).

## 9. Estratégias de resiliência
| ID | Mecanismo | Valor ou parâmetro | Quando entra | Origem |
|---|---|---|---|---|
| FDD-RES-01 | Atomicidade entre mudança de status e registro do evento | Inserção em `webhook_outbox` no mesmo `this.prisma.$transaction` de `OrderService.changeStatus`; erro na inserção desfaz a transação inteira | Toda chamada de `changeStatus` com cadastro selecionado (FDD-FLUXO-06, FDD-FLUXO-07) | `[09:40] Bruno`, `[09:41] Diego` |
| FDD-RES-02 | Leitura da outbox em polling | 2 segundos entre ciclos; lote pequeno (`[09:08] Diego`); tamanho do lote e concorrência: proposta, ver FDD-PROP-04 | Enquanto o worker está no ar (FDD-FLUXO-09) | `[09:08] Diego`, `[09:09] Diego`, `[09:10] Larissa` |
| FDD-RES-03 | Timeout do envio | 10 segundos; estourado o prazo, a tentativa é falha | Cada envio (FDD-FLUXO-18) | `[09:42] Diego` |
| FDD-RES-04 | Tabela de intervalos | 1m/5m/30m/2h/12h, contados a partir de cada falha | Cada falha com tentativas restantes (FDD-FLUXO-23) | `[09:17] Diego`, `[09:17] Larissa` |
| FDD-RES-05 | Teto de tentativas | 5 tentativas, lidas como 1 envio inicial + 5 novas tentativas (proposta, ver FDD-PROP-11) | Falha da 5ª nova tentativa (FDD-FLUXO-24) | `[09:15] Diego`, `[09:17] Larissa` |
| FDD-RES-06 | Fila de mortos | Tabela `webhook_dead_letter` com payload, motivo da falha e timestamp | Esgotamento (`[09:15] Diego`) ou desvio direto (proposta, ver FDD-PROP-15 e FDD-PROP-16; FDD-FLUXO-27) | `[09:18] Diego` |
| FDD-RES-07 | Reprocessamento manual | FDD-CONTRATO-07, com role ADMIN e registro de quem fez o replay | Ação de um ADMIN (FDD-FLUXO-33) | `[09:18] Diego`, `[09:36] Sofia`, `[09:36] Larissa` |
| FDD-RES-08 | Entrega ao menos uma vez com deduplicação | FDD-HEADER-02 com o UUID do evento, igual nas novas tentativas e, no replay, conforme FDD-PROP-23 (proposta); o cliente deduplica | Toda entrega | `[09:25] Diego`, `[09:26] Larissa` |
| FDD-RES-09 | Carência na rotação de secret | Secret antiga válida por 24 horas em paralelo com a nova | Cada rotação (FDD-FLUXO-38) | `[09:21] Sofia`, `[09:22] Sofia` |
| FDD-RES-10 | Limite de tamanho do corpo | 64KB; acima disso, não há envio | Antes de cada envio (FDD-FLUXO-15) | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| FDD-RES-11 | Ordem por pedido | Worker único lendo por `created_at` (ver a nota de 6.2 sobre nova tentativa e paralelismo) | Enquanto houver um único worker (FDD-FLUXO-09) | `[09:12] Diego`, `[09:13] Larissa` |

**Limites conhecidos.** O desenho deliberadamente não garante:

- ordenação global: a ordem vale apenas por `order_id` e enquanto houver um único worker (`[09:13] Larissa`);
- limite de envio por cliente: o tema fica para observar e decidir depois (`[09:39] Larissa`);
- entrega exatamente uma vez: o cliente pode receber o mesmo evento mais de uma vez (`[09:25] Diego`);
- retomada automática de eventos presos em processamento após queda do worker: para isso existe apenas a proposta FDD-PROP-03.

## 10. Observabilidade
### 10.1 Logs

- **FDD-OBS-01.** O módulo `src/modules/webhooks` (novo), inclusive `src/modules/webhooks/webhook.processor.ts` (novo; nome proposto, ver FDD-PROP-14), e o entry-point `src/worker.ts` (novo) registram logs somente pelo logger Pino existente (`logger` e `createLogger` de `src/shared/logger/index.ts`), sem biblioteca nova (`[09:29] Bruno`). O formato segue o do evento `http_request` de `src/middlewares/request-logger.middleware.ts`: o objeto de campos, em camelCase, vai no primeiro argumento, e o nome do evento, em snake_case, vai como mensagem. O worker não usa `console.log`, que a regra `no-console` de `.eslintrc.json` sinaliza.
- **FDD-OBS-02.** Regra proposta para o módulo (proposta, ver FDD-PROP-08): nenhum log, da API ou do worker, carrega a secret de um cadastro. O caso de secret vazada em log citado em `[09:22] Diego` ocorreu na aplicação de um cliente e serviu de argumento para a rotação; a regra o estende aos logs da plataforma. Ela se cumpre em três pontos:
  1. (proposta, ver FDD-PROP-08) A lista `redactPaths` de `src/shared/logger/index.ts` ganha `*.secret`, no padrão de `*.password` e `*.token`. O campo que guardar a secret anterior durante as 24 horas de carência entra na lista do mesmo modo (proposta, ver FDD-PROP-08).
  2. O `http_request` de `src/middlewares/request-logger.middleware.ts` não registra corpo de resposta. Por isso, a secret devolvida na criação (FDD-CONTRATO-01) e na rotação (FDD-CONTRATO-06) não chega ao log.
  3. Os eventos da tabela abaixo não registram o corpo do evento, os headers da entrega (entre eles `X-Signature`) nem o corpo da resposta do cliente. O corpo e a resposta ficam no histórico de entregas, FDD-CONTRATO-05, como pedido em `[09:34] Marcos`; mantê-los fora dos logs é proposta (ver FDD-PROP-20).
- **FDD-OBS-03.** Nas rotas sob `/api/v1/webhooks` e `/api/v1/admin/webhooks`, o `requestLogger` existente continua gerando ou repassando o `X-Request-Id` e emitindo `http_request` com `userId`. Os eventos da API disparados por requisições (FDD-OBS-05 e FDD-OBS-06) levam o mesmo `requestId`, lido de `req.id`.
- **FDD-OBS-04.** `createLogger` fixa `base.service` em `order-management-api` para qualquer processo. Para que os eventos do worker se distingam dos da API sem alterar `createLogger`, o worker registra por um logger filho, `logger.child({ process: 'webhook-worker' })` (proposta, ver FDD-PROP-20).

**Eventos de log.** A reunião exigiu apenas FDD-OBS-05. Os demais itens são proposta (ver FDD-PROP-20), com exceção do nome `bootstrap_failed`, que segue `src/server.ts`.

| ID | Evento | Nível | Campos | Processo | Origem |
|---|---|---|---|---|---|
| FDD-OBS-05 | `webhook_dead_letter_replayed` | `info` | `requestId`, `userId`, `deadLetterId`, `eventId`, `webhookId` | API, em FDD-CONTRATO-07 | `[09:36] Sofia` (registro de quem fez o replay); nome e demais campos: proposta, ver FDD-PROP-20 |
| FDD-OBS-06 | `webhook_config_changed` | `info` | `requestId`, `userId`, `webhookId`, `customerId`, `action` (`created`, `updated`, `deleted`, `secret_rotated`) | API, em FDD-CONTRATO-01, FDD-CONTRATO-03, FDD-CONTRATO-04 e FDD-CONTRATO-06 | proposta, ver FDD-PROP-20 |
| FDD-OBS-07 | `webhook_event_published` | `info` | `orderId`, `customerId`, `fromStatus`, `toStatus`, `outboxRows`, `eventId` (ausente quando `outboxRows` é 0) | API, em `publishWebhookEvent` | proposta, ver FDD-PROP-20 |
| FDD-OBS-08 | `webhook_worker_started` | `info` | `pollIntervalSeconds` (2 segundos, ADR-002), `batchSize` (ver FDD-PROP-04) | worker | proposta, ver FDD-PROP-20 |
| FDD-OBS-09 | `webhook_worker_stopped` | `info` | `signal` (`SIGINT` ou `SIGTERM`) | worker | proposta, ver FDD-PROP-20 |
| FDD-OBS-10 | `bootstrap_failed` | o mesmo usado em `src/server.ts` | erro capturado | worker | convenção de `src/server.ts` (ADR-002) |
| FDD-OBS-11 | `webhook_delivery_succeeded` | `info` | `eventId`, `webhookId`, `customerId`, `orderId`, `attempt`, `statusCode`, `durationMs`, `latencyMs` | worker | proposta, ver FDD-PROP-20; sucesso conforme FDD-PROP-05 |
| FDD-OBS-12 | `webhook_delivery_failed` | `warn` | `eventId`, `webhookId`, `customerId`, `orderId`, `attempt`, `failureKind`, `statusCode` (só com resposta), `durationMs`, `nextAttemptAt` (ausente no último envio) | worker | proposta, ver FDD-PROP-20; falha conforme FDD-PROP-05 |
| FDD-OBS-13 | `webhook_dead_lettered` | `error` | `eventId`, `webhookId`, `customerId`, `orderId`, `attempts`, `reason` | worker (ou a API, conforme FDD-PROP-15) | proposta, ver FDD-PROP-20 |
| FDD-OBS-14 | `webhook_stuck_recovered` | `warn` | `count`, `stuckThreshold` (limite de FDD-PROP-03) | worker | proposta, ver FDD-PROP-20 e FDD-PROP-03 |
| FDD-OBS-15 | `webhook_poll_failed` | `error` | erro capturado | worker | proposta, ver FDD-PROP-20 |

**Campos comuns aos eventos de FDD-OBS-05 a FDD-OBS-15** (nomes propostos, ver FDD-PROP-20):

| Campo | Conteúdo |
|---|---|
| `requestId` | valor de `req.id`, o mesmo do header `X-Request-Id` da resposta da API |
| `userId` | `req.user.id`, como em `http_request`; identifica quem executou a operação |
| `eventId` | id do evento; mesmo valor do header `X-Event-Id` (FDD-HEADER-02) e do campo `event_id`, na leitura adotada no ADR-005 |
| `webhookId` | id do cadastro; mesmo valor do header `X-Webhook-Id` (FDD-HEADER-05) |
| `deadLetterId` | id da linha de `webhook_dead_letter` recebido no path de FDD-CONTRATO-07 |
| `customerId`, `orderId` | ids do cliente e do pedido gravados com o evento |
| `fromStatus`, `toStatus`, `outboxRows` | status de origem e de destino da mudança; quantidade de linhas inseridas na `webhook_outbox` (0 quando o filtro do ADR-008 não casa nenhum cadastro) |
| `attempt` | número do envio: 1 no envio inicial; vai até 6 na leitura de 1 envio inicial + 5 novas tentativas (proposta, ver FDD-PROP-11) |
| `attempts` | total de envios feitos antes da ida à fila de mortos |
| `statusCode`, `durationMs` | status HTTP devolvido pelo cliente e duração da chamada, no padrão de `http_request` |
| `latencyMs` | tempo entre a criação da linha na `webhook_outbox` e o fim da entrega bem-sucedida |
| `failureKind` | `http_status` (resposta fora de 2xx), `timeout` (sem resposta em 10 segundos, `[09:42] Diego`) ou `network` (proposta, ver FDD-PROP-05) |
| `nextAttemptAt` | instante da próxima tentativa, calculado pela tabela 1m/5m/30m/2h/12h do ADR-003 |
| `reason` | o mesmo motivo gravado em `webhook_dead_letter`: tentativas esgotadas (ADR-003), payload acima de 64KB (proposta, ver FDD-PROP-16) ou cadastro inativo ou removido (proposta, ver FDD-PROP-15) |

- FDD-OBS-07 é emitido dentro da transação de `OrderService.changeStatus`, antes do commit. Se houver rollback, o log existe sem a linha correspondente na `webhook_outbox`.
- Entregas que vão direto para a fila de mortos (FDD-PROP-15 e FDD-PROP-16) geram apenas FDD-OBS-13. Se a movimentação de FDD-PROP-15 ocorrer na API, durante a edição ou a remoção do cadastro, é a API que emite FDD-OBS-13.
- Nenhum evento da tabela fica abaixo de `info`. Assim, todos aparecem com o valor padrão de `LOG_LEVEL` em `src/config/env.ts` (`info`).

**Interpretação a confirmar (FDD-OBS-05).**

- Leitura adotada: a exigência "tem que logar quem fez o replay" (`[09:36] Sofia`) é atendida pelo evento de log estruturado, que leva o `userId` do JWT de quem chamou FDD-CONTRATO-07.
- Outra leitura: o ADR-003 registra o replay "para auditoria". Nessa leitura, o autor também seria gravado no banco, junto do registro reprocessado, e o campo entraria no modelo de dados deste FDD. Encaminhar essa escolha à revisão de segurança é proposta deste documento; a existência da revisão vem de `[09:46] Sofia`.

### 10.2 Métricas

- **FDD-OBS-16.** A reunião não decidiu métricas, e o projeto não tem biblioteca de métricas: nenhuma dependência de `package.json` cumpre esse papel. O conjunto mínimo abaixo é proposta (ver FDD-PROP-20) e deriva dos objetivos da feature. Cada métrica se obtém por contagem sobre os eventos de 10.1 ou por consulta às tabelas, sem dependência nova e sem endpoint de métricas.

| ID | Métrica | Como obter | Referência |
|---|---|---|---|
| FDD-OBS-17 | Eventos inseridos na outbox | soma de `outboxRows` em FDD-OBS-07 | ADR-001, ADR-008 |
| FDD-OBS-18 | Entregas por resultado | contagem de FDD-OBS-11 e de FDD-OBS-12, separada por `failureKind` e `statusCode` | FDD-PROP-05 |
| FDD-OBS-19 | Tentativas por evento | `attempt` em FDD-OBS-11 e `attempts` em FDD-OBS-13 | ADR-003, FDD-PROP-11 |
| FDD-OBS-20 | Eventos na fila de mortos | entradas: contagem de FDD-OBS-13 por `reason`; total atual: contagem de linhas de `webhook_dead_letter` | ADR-003 |
| FDD-OBS-21 | Tempo entre a inserção e a entrega | `latencyMs` em FDD-OBS-11 com `attempt` igual a 1, comparado ao limite de 10 segundos (`[09:02] Marcos`) | ADR-002 |
| FDD-OBS-22 | Eventos pendentes | contagem de linhas da `webhook_outbox` nos estados pendente e processando, apoiada no índice de status (`[09:08] Diego`) | ADR-002, FDD-PROP-02, FDD-PROP-03 |

- Em FDD-OBS-21, entregas com `attempt` maior que 1 ficam fora da comparação, porque já esperaram ao menos o primeiro intervalo do ADR-003 (1 minuto). A relação entre o tamanho do lote, o timeout de 10 segundos e esse limite está em FDD-PROP-04.
- FDD-OBS-22 é uma consulta feita sob demanda, e não a cada ciclo do worker, para não somar carga ao polling de 2 segundos (proposta, ver FDD-PROP-20).

### 10.3 Tracing

- **FDD-OBS-23.** O projeto não tem tracing distribuído (nenhuma dependência para isso em `package.json`), e a reunião não tratou do tema. Esta entrega não o introduz (proposta, ver FDD-PROP-20).
- **FDD-OBS-24.** A correlação entre processos é feita pelo identificador do evento (proposta, ver FDD-PROP-20). O mesmo valor aparece nestes pontos:
  1. na linha da `webhook_outbox`;
  2. na linha de `webhook_dead_letter`, quando o evento chega lá;
  3. como `eventId` em FDD-OBS-05, FDD-OBS-07 e de FDD-OBS-11 a FDD-OBS-13;
  4. no histórico de entregas (FDD-CONTRATO-05);
  5. no header `X-Event-Id` (FDD-HEADER-02) e no campo `event_id` do corpo enviado (`[09:43] Diego`; o header foi dito em `[09:25] Diego`), que o cliente usa para deduplicar.

  A assinatura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (ADR-006) não recebe o `requestId`. Por isso, a ligação entre a requisição que mudou o status e o evento é feita por `orderId` e `toStatus` em FDD-OBS-07. No replay, FDD-OBS-05 liga o `requestId` da chamada administrativa ao `eventId` reenviado.

## 11. Dependências e compatibilidade
A implementação reusa as dependências já instaladas e os recursos do próprio Node. A tabela abaixo lista cada item e a situação dele nesta entrega.

| ID | Item | Situação | Origem |
|---|---|---|---|
| FDD-DEP-01 | Node.js `>=20` | Já exigido pelo projeto; é a base de FDD-DEP-07 e FDD-DEP-08 | `engines` em `package.json` |
| FDD-DEP-02 | `express` 4.21.1 | Instalada; reusada no router do módulo, com os `authenticate`, `requireRole` e `validate` existentes | `package.json`; `[09:30] Larissa` |
| FDD-DEP-03 | `@prisma/client` 5.22.0 e `prisma` 5.22.0 (`devDependencies`) | Instaladas; reusadas na API e no worker, cada processo com a sua instância (ADR-002); as tabelas novas entram por migration gerada com o script `db:migrate` existente | `package.json`; `[09:30] Bruno` |
| FDD-DEP-04 | `zod` 3.23.8 | Instalada; reusada em `src/modules/webhooks/webhook.schemas.ts` (novo), inclusive na recusa de URL sem `https` | `package.json`; `[09:23] Sofia` |
| FDD-DEP-05 | `pino` 9.5.0 | Instalada; reusada pelos dois processos via `src/shared/logger/index.ts` (seção 10) | `package.json`; `[09:29] Bruno` |
| FDD-DEP-06 | `uuid` 11.0.3 | Instalada; já gera o `X-Request-Id`; disponível para gerar o id do evento em código, caso ele precise existir antes do insert para compor o `event_id` do snapshot (ADR-007) | `package.json` |
| FDD-DEP-07 | `node:crypto` | Módulo nativo; calcula o HMAC-SHA256 da assinatura (ADR-004) e gera a secret (proposta, ver FDD-PROP-19) | proposta, ver FDD-PROP-17 |
| FDD-DEP-08 | `fetch` global | Nativo no Node 20; faz a entrega de FDD-CONTRATO-08, com cancelamento ao fim do timeout de 10 segundos (`[09:42] Diego`) | proposta, ver FDD-PROP-17 |
| FDD-DEP-09 | MySQL 8.0 | Banco existente; recebe `webhook_outbox`, `webhook_dead_letter`, `webhooks` e `webhook_deliveries` (os dois últimos nomes são proposta, ver FDD-PROP-14) | `docker-compose.yml`; ADR-001 |
| FDD-DEP-10 | Dependências novas | Nenhuma em `dependencies` nem em `devDependencies` | proposta, ver FDD-PROP-17 |
| FDD-DEP-11 | Variáveis de ambiente | Nenhuma nova (proposta, ver FDD-PROP-18). O worker importa `env` por meio de `src/config/database.ts` e de `src/shared/logger/index.ts` e, por isso, precisa das mesmas variáveis de ambiente válidas da API, inclusive `JWT_SECRET`; sem elas, `loadEnv` encerra o processo com `process.exit(1)` | `src/config/env.ts` |

**Compatibilidade.** As rotas existentes não mudam de contrato. O módulo novo entra com dois sub-routers em `buildApiRouter` (`src/routes/index.ts`), conforme a seção 12, e o `errorMiddleware` trata os erros novos sem alteração (`[09:29] Bruno`). Em pedidos, a mudança se limita à chamada a `publishWebhookEvent` dentro de `OrderService.changeStatus`, que não altera o retorno do método (FDD-DEP-12). Os testes de `tests/orders.test.ts` devem continuar passando sem ajuste, porque, sem cadastro de webhook para o cliente, nenhuma linha é inserida (ADR-008). As tabelas novas entram na limpeza de `tests/setup.ts`, com `webhooks` apagada antes de `customers` por causa da chave estrangeira do cadastro para o cliente (proposta, ver FDD-PROP-22) (FDD-DEP-13). O `include` de `tsconfig.build.json` (`src/**/*.ts`) já leva `src/worker.ts` (novo) e os arquivos de `src/modules/webhooks` (novo) para `dist`, sem mudança na configuração de build (FDD-DEP-14). O código novo segue `.eslintrc.json`, que exige importação de tipos com `import type` (`@typescript-eslint/consistent-type-imports`), como em `import type { Prisma } from '@prisma/client'` para `Prisma.TransactionClient` (FDD-DEP-15). Segue também `.prettierrc`, com aspas simples (`singleQuote: true`) e vírgula final (`trailingComma: "all"`) (FDD-DEP-16). O projeto é ESM (`"type": "module"` em `package.json`), e os imports relativos terminam em `.js`, como em `src/app.ts` e `src/shared/logger/index.ts`; o código novo segue a mesma forma (FDD-DEP-17).

## 12. Integração com o sistema existente
Cada subseção trata de um arquivo existente e descreve três pontos: o que ele faz hoje (fato do código), o que a feature muda ou acrescenta e o que permanece igual.

### 12.1 `src/modules/orders/order.service.ts` — chamada de `publishWebhookEvent` na transação de `changeStatus` (FDD-INT-01)

**Hoje.** `OrderService.changeStatus` roda inteiro dentro de `this.prisma.$transaction(async (tx) => ...)`, nesta ordem:

- busca o pedido com `include: { items: true }`;
- recusa `from === to` com `ConflictError` (`INVALID_STATUS_TRANSITION`) e valida a transição com `canTransition`;
- debita ou repõe estoque;
- executa `tx.order.update` e `tx.orderStatusHistory.create`;
- relê o pedido com `items`, `history` e `customer` para montar a resposta.

**O que muda.**

1. Entra a chamada `await publishWebhookEvent(tx, order, from, to)` depois de `tx.orderStatusHistory.create` e antes da releitura (`refreshed`), no mesmo callback da transação (`[09:40] Bruno`, `[09:41] Bruno`, `[09:41] Diego`). A reunião fixou apenas que a inserção acontece dentro da mesma transação; a posição dela dentro do callback é escolha deste documento, por ser indiferente à atomicidade.
2. A função recebe:
   - `tx`: o cliente da transação corrente;
   - `order`: o objeto carregado no início do método, já garantido não nulo pelo `NotFoundError`;
   - `from` e `to`: as variáveis locais já calculadas.

   Como o pedido foi lido antes do `update`, `order.status` ainda guarda o status de origem. Por isso os dois status vão como argumentos explícitos.
3. A função lê os cadastros ativos de `order.customerId` que assinam `to` (ADR-008), monta o snapshot (ADR-007) e insere as linhas em `webhook_outbox`, uma por destino (proposta, ver FDD-PROP-01). Se nenhum cadastro tiver interesse, ela não insere nada e retorna normalmente. A ausência de linha não é erro, conforme a leitura do ADR-001 sobre rollback e filtro.
4. O arquivo passa a importar `publishWebhookEvent`, exportada como função de módulo (não como método de classe) por `src/modules/webhooks/webhook.service.ts` (novo). A escolha desse arquivo é proposta deste documento, dentro do conjunto de nomes de FDD-PROP-14.
5. Qualquer exceção lançada pela função sai do callback, e o Prisma desfaz a transação inteira: status, histórico e movimento de estoque voltam ao estado anterior (`[09:40] Bruno`, `[09:41] Diego`). O erro segue pelo `next(err)` de `OrderController.changeStatus` e é respondido pelo `errorMiddleware` conforme o tipo (ver 12.4).

```ts
// OrderService.changeStatus — assinatura e tipo de retorno sem mudança
return this.prisma.$transaction(async (tx) => {
  const order = await tx.order.findUnique({ where: { id }, include: { items: true } });
  if (!order) throw new NotFoundError('Order');
  // ... from/to, ConflictError, canTransition e débito/reposição de estoque: sem mudança
  await tx.order.update({ where: { id }, data: { status: to } });
  await tx.orderStatusHistory.create({
    // ... data: sem mudança
  });

  // novo: usa o tx recebido, sem transação própria; exceção aqui desfaz a transação inteira
  await publishWebhookEvent(tx, order, from, to);

  const refreshed = await tx.order.findUnique({
    // ... where e include de items, history e customer: sem mudança
  });
  return refreshed!;
});
```

**O que não muda.**

- O construtor `(orders: OrderRepository, prisma: PrismaClient)` e a montagem `new OrderService(orderRepository, prisma)` em `buildControllers`. A injeção de um repository do webhook foi descartada (ADR-006-ALT-1).
- O método `create` (proposta, ver FDD-PROP-09). Ele grava `fromStatus: null` e `toStatus: PENDING` na própria transação e não publica evento. Um cadastro que assine `PENDING` não recebe nada, porque nenhuma transição de `src/modules/orders/order.status.ts` leva a `PENDING`.
- O método `delete` (proposta, ver FDD-PROP-09). Ele continua removendo pedidos em `PENDING` ou `CANCELLED`. As tabelas de evento não têm chave estrangeira para `orders` (proposta, ver FDD-PROP-22), então eventos já gravados não bloqueiam a remoção e o snapshot permanece.
- As validações existentes e o corpo da resposta, que continua vindo de `refreshed`.

**Pontos de atenção (fatos, não requisitos).**

1. A transação interativa é aberta sem objeto de opções. Valem, portanto, os padrões do Prisma descritos na documentação do Prisma 5 (`@prisma/client` 5.22.0 em `package.json`):
   - limite de 5 segundos para a transação (`timeout`);
   - espera de 2 segundos para iniciá-la (`maxWait`).

   A consulta aos cadastros e a inserção na outbox passam a contar dentro desse limite. A reunião não tratou disso; o ponto fica para a revisão deste documento (lacuna FDD-GAP-37).
2. `TxClient` é um alias local (`type TxClient = Prisma.TransactionClient`) e não é exportado. `publishWebhookEvent` tipa o parâmetro diretamente como `Prisma.TransactionClient`, importado de `@prisma/client`. Esse tipo não oferece `$transaction`, então a função não abre transação própria: todas as leituras e escritas usam o `tx` recebido.

### 12.2 `prisma/schema.prisma` — models novos e migration aditiva (FDD-INT-02)

**Hoje.** O schema tem:

- datasource MySQL com `shadowDatabaseUrl`;
- enums `UserRole` e `OrderStatus`;
- ids `String @id @default(uuid()) @db.Char(36)`, campos `createdAt`/`updatedAt` e `@@map` em snake_case.

Existe uma única migration, `prisma/migrations/20260519182739_init/migration.sql`.

**O que muda.**

1. Entram quatro models, mapeados para `webhook_outbox`, `webhook_dead_letter`, `webhooks` e `webhook_deliveries`. Os dois últimos nomes são proposta (ver FDD-PROP-14). Campos, índices e o enum de status da outbox estão na seção 5.
2. Esta seção chama os models de `Webhook`, `WebhookOutbox`, `WebhookDelivery` e `WebhookDeadLetter`. Se a seção 5 usar outros nomes, valem os dela.
3. A lista de status assinados usa o enum `OrderStatus` existente (ADR-008), sem valor novo.
4. `Customer` ganha apenas o campo de relação inversa que o Prisma exige para a chave estrangeira do cadastro (proposta, ver FDD-PROP-22). A tabela `customers` não ganha coluna.
5. Uma nova migration é gerada por `npm run db:migrate` (`prisma migrate dev`). Ela contém apenas a criação das tabelas novas e das suas chaves.

**O que não muda.** Nenhuma tabela existente ganha ou perde coluna, e a migration inicial fica intacta.

Fato relevante: com a ação de exclusão restritiva, a mesma de `orders_customerId_fkey`, remover um cliente que tenha cadastro falharia com um erro do Prisma que o `errorMiddleware` não mapeia, e a resposta seria `500`. É o que já acontece hoje com um cliente que tem pedidos. A regra final está em FDD-GAP-23.

### 12.3 `src/shared/errors/http-errors.ts`, `src/shared/errors/app-error.ts` e `src/shared/errors/index.ts` — classes de erro com prefixo `WEBHOOK_` (FDD-INT-03)

**Hoje.**

- `AppError` recebe `message`, `statusCode`, `errorCode` e `details`.
- `BadRequestError`, `ConflictError` e `UnprocessableEntityError` aceitam código próprio, e delas derivam as classes específicas `InvalidStatusTransitionError` e `InsufficientStockError`.
- `NotFoundError` aceita apenas o nome do recurso e grava o código fixo `NOT_FOUND`.
- `src/shared/errors/index.ts` reexporta todas.

**O que muda.**

1. As classes do módulo entram em `src/shared/errors/http-errors.ts` (local inferido do padrão do arquivo), ao lado das específicas existentes, e são reexportadas em `src/shared/errors/index.ts`; o padrão de classes e o prefixo vêm de `[09:28] Bruno` e `[09:29] Larissa`.
2. Erros 400, 409 ou 422 estendem `BadRequestError`, `ConflictError` ou `UnprocessableEntityError`, como faz `InvalidStatusTransitionError`.
3. `WEBHOOK_NOT_FOUND` não pode estender `NotFoundError`, que fixa `NOT_FOUND`. A classe estende `AppError` diretamente.
4. O service verifica se o cadastro existe e lança essa classe antes de `update` ou `delete`. Se a operação do Prisma falhasse com `P2025`, o `errorMiddleware` responderia o código genérico `NOT_FOUND`.
5. A lista completa de códigos está na seção de erros. `WEBHOOK_INVALID_URL` fica sem uso (proposta, ver FDD-PROP-10).

```ts
// src/shared/errors/http-errors.ts (acréscimo)
export class WebhookNotFoundError extends AppError {
  constructor(message = 'Webhook not found') {
    super(message, 404, 'WEBHOOK_NOT_FOUND');
  }
}

// src/shared/errors/index.ts (acréscimo à lista exportada de './http-errors.js')
//   WebhookNotFoundError,
```

**O que não muda.** `src/shared/errors/app-error.ts` e as classes existentes.

### 12.4 `src/middlewares/error.middleware.ts` — sem mudança (FDD-INT-04)

**Hoje.** O middleware trata os erros assim:

| Erro | Resposta |
|---|---|
| `AppError` | `statusCode` da classe com `{ "error": { "code", "message", "details"? } }` |
| `ZodError` | `400 VALIDATION_ERROR` |
| Prisma `P2002` | `409 CONFLICT` |
| Prisma `P2025` | `404 NOT_FOUND` |
| Qualquer outro | registra `Unhandled error in request` e responde `500 INTERNAL_SERVER_ERROR` |

**Na feature.** O arquivo não muda (`[09:29] Bruno`), porque as classes de 12.3 são `AppError`. Duas consequências:

- Um erro que não seja `AppError` lançado por `publishWebhookEvent`, como uma falha do banco na inserção, chega ao `PATCH /api/v1/orders/:id/status` como `500 INTERNAL_SERVER_ERROR`, já com o status desfeito. A exceção são os códigos `P2002` e `P2025`, mapeados conforme a tabela.
- O worker não passa por este middleware, porque não monta Express. Os erros de entrega são tratados e registrados pelo próprio processamento.

### 12.5 `src/middlewares/validate.middleware.ts` — sem mudança (FDD-INT-05)

**Hoje.** O middleware aplica os schemas a `body`, `query` e `params`. Ele troca `req.body` e `req.params` pelo resultado do parse e copia a query com `Object.assign`. Um `ZodError` vira `ValidationError` com `details` no formato `{ path, message }`.

**Na feature.** O arquivo não muda. As regras ficam em `src/modules/webhooks/webhook.schemas.ts` (novo):

- `url` com refinamento que exige o protocolo `https:`. Uma URL `http` recebe `400 VALIDATION_ERROR` (`[09:23] Sofia`; proposta, ver FDD-PROP-10).
- `statuses` como `z.array(z.nativeEnum(OrderStatus))` (ADR-008). O nome do campo é proposta (ver FDD-PROP-12).
- `customerId` como `z.string().uuid()`, no body da criação e na query da listagem; o schema de edição não contém `customerId` (proposta, ver FDD-PROP-13 e FDD-PROP-31). A reunião disse "no body ou no path" (`[09:32] Larissa`); a query na listagem segue os filtros de listagem existentes e é uma divergência registrada em FDD-PROP-13.
- Parâmetro `id` como `z.string().uuid()`.
- `page` e `pageSize` com `z.coerce.number()`, como em `listCustomersQuerySchema`.

### 12.6 `src/middlewares/auth.middleware.ts` — sem mudança (FDD-INT-06)

**Hoje.**

- `authenticate` valida o Bearer JWT com `env.JWT_SECRET` e preenche `req.user` com `id` (vindo de `sub`), `email` e `role`. Se falhar, responde `401 UNAUTHORIZED`.
- `requireRole` responde `401` quando não há usuário e `403 FORBIDDEN` quando o papel está fora da lista.
- O JWT não carrega cliente.

**Na feature.** O arquivo não muda.

1. O router de `/api/v1/webhooks` aplica `router.use(authenticate)`, como `buildCustomerRouter`. Qualquer papel autenticado acessa os contratos FDD-CONTRATO-01 a FDD-CONTRATO-06 (`[09:37] Sofia`).
2. O router de admin aplica `authenticate` e, na rota de FDD-CONTRATO-07, `requireRole('ADMIN')` antes do `validate` (`[09:36] Larissa`).
3. Como o JWT não identifica o cliente, o `customerId` vem da requisição, "no body ou no path" (`[09:32] Larissa`; posição por endpoint em FDD-PROP-13).
4. `req.user.id` é o identificador que o replay registra para auditoria (`[09:36] Sofia`).

### 12.7 `src/app.ts` — `buildControllers` instancia o módulo novo (FDD-INT-07)

**Hoje.**

- `buildControllers` cria repository, service e controller de cada módulo e devolve o objeto `Controllers`.
- `buildApp` desativa `x-powered-by`, aplica `express.json({ limit: '1mb' })` e `requestLogger`, expõe `/health`, monta `/api/v1`, responde 404 para rota desconhecida e termina com `errorMiddleware`.

**O que muda.**

1. Depois de pedidos, `buildControllers` passa a criar `new WebhookRepository(prisma)`, `new WebhookService(...)` e `new WebhookController(webhookService)`, e devolve `webhooks: webhookController`.
2. Se o service precisar abrir transação, ele recebe também o `PrismaClient`, como `OrderService`. É o caso do replay, que insere a linha de novo na outbox (o destino da linha na fila de mortos é proposta, ver FDD-PROP-23 e FDD-PROP-35).

**O que não muda.**

- `buildApp`. O limite de `1mb` vale para requisições recebidas e não se relaciona com o limite de 64KB do evento de saída (tratamento proposto em FDD-PROP-16).
- O worker não chama `buildApp` nem `buildControllers`.

### 12.8 `src/routes/index.ts` — `Controllers` e montagem das rotas (FDD-INT-08)

**Hoje.** O tipo `Controllers` tem `auth`, `users`, `customers`, `products` e `orders`. `buildApiRouter` monta um sub-router para cada um.

**O que muda.**

1. `Controllers` ganha `webhooks: WebhookController`, com import de tipo de `src/modules/webhooks/webhook.controller.ts` (novo).
2. `buildApiRouter` acrescenta duas montagens:
   - `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`;
   - `router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks))`.

   As duas funções vêm de `src/modules/webhooks/webhook.routes.ts` (novo) e seguem o padrão de nomes `build<Módulo>Router`.
3. Os caminhos finais ficam assim:
   - `/api/v1/webhooks` e subcaminhos para FDD-CONTRATO-01 a FDD-CONTRATO-06;
   - `/api/v1/admin/webhooks/dead-letter/:id/replay` para FDD-CONTRATO-07.

   O router separado restringe `requireRole('ADMIN')` ao replay.

**O que não muda.** As cinco montagens existentes.

### 12.9 `src/server.ts` — sem mudança; modelo de `src/worker.ts` (novo) (FDD-INT-09)

**Hoje.**

- `bootstrap` chama `buildApp({ prisma })`, escuta em `env.PORT` e registra `server_started`.
- `shutdown`, acionado por `SIGINT` e `SIGTERM`, registra `shutdown_initiated`, fecha o servidor, chama `prisma.$disconnect()` e executa `process.exit(0)`.
- Uma falha no bootstrap registra `bootstrap_failed` e encerra com `process.exit(1)`.

**Na feature.** `src/worker.ts` (novo) repete essa estrutura (`[09:11] Larissa`):

1. `bootstrap` inicia o loop de `src/modules/webhooks/webhook.processor.ts` (novo; nome proposto, ver FDD-PROP-14) com o intervalo de 2 segundos (ADR-002), no lugar de `app.listen`.
2. `shutdown`, acionado por `SIGINT` e `SIGTERM`, interrompe o loop e chama `prisma.$disconnect()` antes de `process.exit(0)`.
3. Uma falha no bootstrap registra `bootstrap_failed` e encerra com `process.exit(1)`.

O worker não importa `buildApp` nem abre porta HTTP, e `src/server.ts` não importa nada do worker. Linhas deixadas em processamento por uma parada no meio do lote seguem FDD-PROP-03; o que o `shutdown` faz com o lote em andamento é a lacuna FDD-GAP-42.

### 12.10 `src/config/database.ts` — instância própria de `PrismaClient` no worker (FDD-INT-10)

**Hoje.** `createPrismaClient()` constrói o `PrismaClient`, com `log` conforme `NODE_ENV`. O singleton `prisma` é criado por essa fábrica quando o módulo é carregado.

**Na feature.** O arquivo não muda.

- `src/worker.ts` (novo) importa `prisma` deste arquivo. No processo do worker, o carregamento executa `createPrismaClient()` e gera uma instância própria, distinta da instância da API, no mesmo banco e com a mesma `DATABASE_URL` (`[09:30] Bruno`).
- Como o import já cria uma instância, uma chamada explícita a `createPrismaClient()` no worker criaria uma segunda instância no mesmo processo.
- O processador recebe o `PrismaClient` por parâmetro, como os repositories existentes.

### 12.11 `src/config/env.ts` — sem mudança (FDD-INT-11)

**Hoje.** `envSchema` valida `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET` e `JWT_EXPIRES_IN`. Se a validação falhar, o processo encerra com `process.exit(1)`.

**Na feature.** Nenhuma variável nova. Os valores abaixo ficam como constantes do módulo (proposta, ver FDD-PROP-18; leitura das tentativas em FDD-PROP-11):

- polling de 2 segundos;
- timeout de 10 segundos;
- limite de 64KB;
- 5 tentativas;
- intervalos 1m/5m/30m/2h/12h.

Fato: o worker carrega `env` por meio de `src/config/database.ts` e `src/shared/logger/index.ts`. Por isso precisa de `JWT_SECRET` válido, com no mínimo 16 caracteres, mesmo sem usá-lo (lacuna FDD-GAP-41).

### 12.12 `src/shared/logger/index.ts` — sem mudança por decisão; redação em proposta (FDD-INT-12)

**Hoje.**

- `createLogger()` monta o Pino com `LOG_LEVEL`, `redact.paths` (`req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`) e censura `[REDACTED]`.
- A `base` fixa `service: 'order-management-api'`.
- O evento `http_request` não inclui corpo de requisição nem de resposta.

**Na feature.** A API e o worker usam o `logger` existente (`[09:29] Bruno`). A reunião não decidiu mudar o arquivo. Três fatos:

- nenhum caminho atual cobre o campo de secret;
- a secret devolvida por FDD-CONTRATO-01 e FDD-CONTRATO-06 não aparece em `http_request`;
- os logs do worker saem com o mesmo `service` da base.

Acrescentar o caminho do campo de secret a `redactPaths` é proposta (ver FDD-PROP-08).

### 12.13 `src/shared/http/response.ts` — `paginated()` na listagem (FDD-INT-13)

**Hoje.** `paginated(data, page, pageSize, total)` devolve `{ "data": [...], "pagination": { "page", "pageSize", "total", "totalPages" } }`. É usado por `OrderService.list` e `CustomerService.list`.

**Na feature.** O arquivo não muda. A listagem de FDD-CONTRATO-02 usa `paginated()`, com `page` e `pageSize` vindos da query. O formato de FDD-CONTRATO-05 ("últimos 100", `[09:34] Marcos`) está na seção 7.1.

### 12.14 `package.json` — scripts do worker, sem dependência nova (FDD-INT-14)

**Hoje.** O projeto declara `engines.node >= 20` e tem os scripts `dev` (`tsx watch --env-file=.env src/server.ts`), `build` (`tsc -p tsconfig.build.json`) e `start` (`node --env-file=.env dist/server.js`).

**O que muda.** Entram dois scripts, propostos por este documento no molde de `dev` e `start`. O nome `worker` vem de `[09:11] Larissa`.

| Script | Comando proposto | Molde |
|---|---|---|
| `worker` | `tsx watch --env-file=.env src/worker.ts` | `dev` |
| `start:worker` | `node --env-file=.env dist/worker.js` | `start` |

**O que não muda.**

- `build`: o `include` `src/**/*.ts` e o `rootDir` `./src` de `tsconfig.build.json` já geram `dist/worker.js`.
- As dependências (proposta, ver FDD-PROP-17): o envio usa o `fetch` global do Node 20, e o HMAC e a geração da secret usam `node:crypto` (FDD-PROP-19).

### 12.15 `tests/setup.ts`, `tests/helpers/factories.ts` e `tests/orders.test.ts` — limpeza, fábricas e testes atuais (FDD-INT-15)

**Hoje.**

- `tests/setup.ts` apaga, em `beforeEach`, as tabelas de histórico, itens, pedidos, sequência, produtos, clientes e usuários.
- `tests/helpers/factories.ts` oferece `getTestApp`, `createTestCustomer`, `createTestProduct` e `bootstrapAuthenticatedUser(role)`.
- `tests/orders.test.ts` cobre `changeStatus`.

**O que muda.**

1. **`tests/setup.ts`.** No início do `beforeEach`, antes de `prisma.order.deleteMany()`, entram os `deleteMany()` das tabelas novas: primeiro entregas, outbox e DLQ, depois os cadastros.
   - A ordem vem das chaves estrangeiras: a chave do cadastro para `customers` (proposta, ver FDD-PROP-22) e as eventuais chaves das tabelas de evento para o cadastro (seção 5).
   - As tabelas de evento não têm chave para `orders`.
2. **`tests/helpers/factories.ts`.** As fábricas existentes atendem os testes novos:
   - `getTestApp()` já monta as rotas novas por `buildControllers`;
   - `createTestCustomer` fornece o `customerId`;
   - `bootstrapAuthenticatedUser('ADMIN')` cobre o replay, e `bootstrapAuthenticatedUser()` cobre o `403 FORBIDDEN`.

**O que não muda.** `tests/orders.test.ts`. Sem cadastro, o filtro não insere linha e `changeStatus` responde como hoje. No caso `422`, o débito falha antes da publicação.

## 13. Critérios de aceite técnicos
Os critérios abaixo podem ser verificados com a stack de testes do projeto: `vitest`, `supertest` e os helpers de `tests/helpers/factories.ts`. Nos critérios que envolvem entrega, um servidor HTTP local faz o papel do cliente. Quando há espera longa, o teste controla o relógio.

- **FDD-CA-01** — Atomicidade entre a mudança de status e o registro do evento (`[09:40] Bruno`).
  - Dado um pedido `PENDING` cujo cliente tem um cadastro ativo que assina `PAID`,
  - quando `PATCH /api/v1/orders/:id/status` com `toStatus: PAID` é chamado e o teste força a falha da gravação na outbox (por exemplo, `publishWebhookEvent` lançando erro),
  - então a resposta é `500 INTERNAL_SERVER_ERROR`, o pedido continua `PENDING`, `order_status_history` mantém só a entrada de criação, o estoque não é debitado e `webhook_outbox` não tem linha. Sem a falha forçada, o status, o histórico e a linha da outbox aparecem juntos após o commit.
- **FDD-CA-02** — Sem cadastro que assine o status, nada é registrado (`[09:34] Bruno`).
  - Dado um cliente sem cadastro e outro cujo único cadastro ativo assina apenas `CANCELLED`,
  - quando um pedido de cada cliente passa de `PENDING` para `PAID`,
  - então as duas mudanças respondem `200` e nenhuma linha é inserida em `webhook_outbox`.
- **FDD-CA-03** — O filtro compara o status de destino (`[09:33] Marcos`).
  - Dado um cliente com um único cadastro ativo que assina apenas `PAID`,
  - quando um pedido passa de `PENDING` para `PAID` e, em seguida, de `PAID` para `CANCELLED`,
  - então `webhook_outbox` tem exatamente uma linha, com `to_status` igual a `PAID` no payload. A transição cuja origem é `PAID` não gera linha. Esta é a leitura adotada no ADR-008; a interpretação a confirmar está registrada nele.
- **FDD-CA-04** — Coleta dentro do ciclo de 2 segundos (`[09:09] Diego`).
  - Dado o worker em execução e nenhuma outra entrega pendente na outbox,
  - quando é commitada uma mudança de status que tem cadastro assinante,
  - então o worker coleta a entrega e inicia o envio em até 2 segundos após o commit. Os 2 segundos são lidos como limite superior da espera; a interpretação a confirmar está no ADR-002.
- **FDD-CA-05** — Headers e corpo da entrega (`[09:44] Diego`, `[09:44] Sofia`).
  - Dado um cadastro ativo que assina `PAID` e aponta para o servidor de teste,
  - quando o worker entrega o evento `PENDING` para `PAID` (FDD-CONTRATO-08),
  - então o request traz os headers FDD-HEADER-01 a FDD-HEADER-05: `Content-Type: application/json`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
  - `X-Webhook-Id` é igual ao id do cadastro, e `X-Event-Id` é igual ao `event_id` do corpo (leitura adotada no ADR-005).
  - O corpo contém `event_id`, `event_type` (com o valor `order.status_changed`, dado como exemplo em `[09:43] Diego`), `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, e não contém os itens do pedido.
- **FDD-CA-06** — A assinatura pode ser verificada pelo cliente (`[09:20] Sofia`).
  - Dado a secret devolvida na criação do cadastro (FDD-CONTRATO-01),
  - quando o servidor de teste calcula, com `node:crypto`, o HMAC-SHA256 dos bytes exatos do corpo recebido usando essa secret,
  - então o resultado, em hexadecimal e sem prefixo (proposta, ver FDD-PROP-06), é igual a `X-Signature`. Com outra secret, ou com um byte do corpo alterado, a comparação falha.
- **FDD-CA-07** — A ausência de resposta em 10 segundos é falha (`[09:42] Diego`).
  - Dado um servidor de teste que não responde em 10 segundos,
  - quando o worker faz o envio inicial,
  - então o worker interrompe a espera ao atingir 10 segundos e registra a tentativa como falha por timeout. A entrega volta a pendente, com nova tentativa marcada para 1 minuto após a falha (proposta, ver FDD-PROP-02).
- **FDD-CA-08** — Intervalos e teto de tentativas (`[09:17] Larissa`).
  - Dado um servidor de teste que sempre responde `500` (falha pela proposta FDD-PROP-05) e o relógio controlado no teste,
  - quando o worker processa a entrega até o fim,
  - então o servidor recebe exatamente 6 requests: 1 envio inicial e 5 novas tentativas (proposta, ver FDD-PROP-11).
  - Cada nova tentativa só sai depois do intervalo correspondente, contado da falha anterior: 1m/5m/30m/2h/12h, nessa ordem, com tolerância de um ciclo de 2 segundos.
  - Nenhum request sai após a sexta falha.
  - Interpretação a confirmar: se prevalecer a leitura de 5 envios no total (ADR-003, `[09:48] Larissa`), o critério passa a 5 requests e o intervalo de 12h deixa de ser usado.
- **FDD-CA-09** — Conteúdo da fila de mortos (`[09:18] Diego`).
  - Dado uma entrega que esgotou o teto do FDD-CA-08,
  - quando o worker registra a falha permanente,
  - então `webhook_dead_letter` tem uma linha com o payload igual ao gravado na outbox, o motivo da última falha e um timestamp. A entrega não está mais em `webhook_outbox` (proposta, ver FDD-PROP-02).
- **FDD-CA-10** — O reprocessamento é restrito a ADMIN e fica auditado (`[09:36] Sofia`).
  - Dado uma linha em `webhook_dead_letter`,
  - quando `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (FDD-CONTRATO-07) é chamado três vezes: sem token, com token de `OPERATOR` e com token de `ADMIN` (obtidos via `bootstrapAuthenticatedUser(role)`),
  - então a primeira chamada recebe `401 UNAUTHORIZED` e a segunda `403 FORBIDDEN`, sem alterar `webhook_dead_letter` nem `webhook_outbox`.
  - A terceira recoloca a entrega na outbox como pendente, com o mesmo `event_id`, o mesmo payload e a contagem de tentativas zerada (proposta, ver FDD-PROP-23).
  - O log estruturado do replay contém o id do usuário (`req.user.id`), exigido em `[09:36] Sofia`; os demais campos, como o id da linha da DLQ, são proposta (ver FDD-PROP-20).
- **FDD-CA-11** — Carência de 24 horas na rotação (`[09:21] Sofia`).
  - Dado um cadastro com a secret S1 e o relógio controlado no teste,
  - quando a secret é rotacionada por FDD-CONTRATO-06, que gera a nova secret S2,
  - então, a partir da rotação, toda entrega tem `X-Signature` verificável com S2, inclusive as feitas dentro das 24 horas, e S1 não volta a ser usada para assinar (proposta, ver FDD-PROP-21); cabe ao cliente aceitar S1 e S2 até o fim das 24 horas.
  - Passadas as 24 horas, S1 deixa de valer também para o cliente.
- **FDD-CA-12** — O histórico devolve as últimas 100 entregas (`[09:34] Marcos`).
  - Dado um cadastro A com mais de 100 entregas registradas e um cadastro B com outras entregas,
  - quando FDD-CONTRATO-05 é chamado para A,
  - então a resposta `200` traz as 100 entregas mais recentes de A e nenhuma de B (o limite fixo de 100 é proposta, ver FDD-PROP-26). Cada entrega informa resultado (sucesso/falha), payload, response e tempo de resposta, com os nomes de campo da seção de contratos (proposta, ver FDD-PROP-12).
- **FDD-CA-13** — URL sem `https` é recusada (`[09:23] Sofia`).
  - Dado um usuário autenticado,
  - quando ele envia FDD-CONTRATO-01 ou FDD-CONTRATO-03 com `url` iniciada por `http://`,
  - então a resposta é `400 VALIDATION_ERROR`, com `details` apontando o campo `url` (proposta, ver FDD-PROP-10). Nenhum cadastro é criado, e o cadastro editado mantém a URL anterior.
- **FDD-CA-14** — Payload acima de 64KB é rejeitado (`[09:24] Larissa`).
  - Dado uma entrega pendente cujo payload serializado ultrapassa 64KB, inserida diretamente pelo teste,
  - quando o worker a coleta,
  - então nenhum request chega ao servidor de teste. A entrega vai direto para `webhook_dead_letter`, com motivo de tamanho excedido e sem consumir tentativas (proposta, ver FDD-PROP-16).
- **FDD-CA-15** — Os testes existentes continuam passando (`tests/orders.test.ts`).
  - Dado a feature implementada,
  - quando `npm test` executa a suíte,
  - então todos os casos de `tests/orders.test.ts` passam sem alteração no arquivo, inclusive os de `changeStatus`: `PENDING` para `PAID` com débito de estoque, `409 INVALID_STATUS_TRANSITION`, `422 INSUFFICIENT_STOCK` sem débito e reposição em `PAID` para `CANCELLED`.
- **FDD-CA-16** — O snapshot sobrevive ao pedido (`[09:52] Larissa`).
  - Dado um pedido que passou de `PAID` para `CANCELLED` e cujo evento foi registrado para um cadastro que assina `CANCELLED`, ainda não entregue,
  - quando o pedido é removido por `DELETE /api/v1/orders/:id` antes do envio,
  - então o worker entrega um corpo igual ao payload gravado na inserção (ADR-007), sem erro de chave estrangeira (proposta, ver FDD-PROP-22).

## 14. Riscos e mitigação
Probabilidade e impacto são avaliação deste documento, pois a reunião não os classificou; essas classificações não recebem ID. FDD-RISCO-05 e FDD-RISCO-06 são bloqueantes de deploy: o desenho depende de uma premissa de autorização que o código atual não sustenta, e a feature não deve ir a produção antes da revisão de segurança que o próprio time agendou (`[09:46] Sofia`, `[09:49] Sofia`). Eles não bloqueiam a entrega documental, que apenas registra o risco. As mitigações que não remetem a ADR, proposta ou critério de aceite são sugestões para a revisão do FDD com Bruno e Diego (`[09:50] Larissa`) e, nos temas de segurança, com Sofia (`[09:46] Sofia`).

| ID | Risco | Probabilidade | Impacto | Mitigação | Origem |
|---|---|---|---|---|---|
| FDD-RISCO-01 | Um endpoint lento ou fora do ar segura o ciclo do worker único. Somados o timeout de 10 segundos (`[09:42] Diego`) e a espera de até 2 segundos do polling, eventos de outros clientes podem passar da meta de "abaixo de 10 segundos" (`[09:02] Marcos`). (amplificação não discutida na reunião) Como não há teto de cadastros por cliente e cada mudança de status gera uma linha por cadastro (proposta FDD-PROP-01), muitos cadastros lentos para um mesmo cliente multiplicam o tempo do worker por evento. Lacunas FDD-GAP-04 e FDD-GAP-47. | Média | Alto | Lote pequeno, com envio em sequência por `created_at` como base; paralelismo só entre linhas de pedidos diferentes (proposta, ver FDD-PROP-04). Fixar o tamanho do lote na revisão do FDD. Fixar um teto de cadastros por cliente (proposta, ver FDD-PROP-45). Acompanhar a espera entre o `timestamp` do evento e o envio pelos logs (proposta, ver FDD-PROP-20). | `[09:42] Diego` |
| FDD-RISCO-02 | A falha no registro do evento impede a mudança de status. Por decisão (ADR-001), qualquer erro em `publishWebhookEvent` desfaz a transação de `changeStatus` (`[09:40] Bruno`). Com isso, um defeito no módulo de webhooks bloqueia a operação de pedidos, e o operador recebe `500`. | Baixa | Alto | Restringir `publishWebhookEvent` à leitura de cadastros e à escrita no mesmo banco, sem chamada HTTP. Manter a checagem de 64KB no worker (proposta, ver FDD-PROP-16). Cobrir com FDD-CA-01 e FDD-CA-15. O erro chega ao `errorMiddleware` e fica no log. | `[09:40] Bruno` |
| FDD-RISCO-03 | A secret acaba registrada em log. Isso pode ocorrer do lado do cliente, como no caso citado em `[09:22] Diego`, ou nos logs da plataforma, cuja lista de redação não cobre a secret. Há também um caminho indireto: o middleware de erro registra o erro inteiro quando ele não é um `AppError` (`src/middlewares/error.middleware.ts`), e a mensagem de um erro de validação do Prisma pode ecoar os dados passados na chamada, inclusive a secret (consequência técnica inferida). Lacuna FDD-GAP-08. | Média | Alto | Para o caso do cliente, usar a rotação com carência de 24 horas (FDD-CONTRATO-06). Na plataforma, estender a lista de redação de `src/shared/logger/index.ts` com os caminhos dos campos que carregam a secret e tratar os erros do Prisma no service antes de repassá-los ao middleware (proposta, ver FDD-PROP-08); não logar o objeto do cadastro inteiro no worker. Incluir o tema na revisão de Sofia. | `[09:22] Diego` |
| FDD-RISCO-04 | O cliente processa o mesmo evento mais de uma vez. Essa é uma consequência aceita do at-least-once (`[09:24] Diego`); a objeção de que isso joga a responsabilidade para o cliente está em `[09:25] Sofia`. | Alta | Médio | Manter `X-Event-Id` e `event_id` estáveis em todos os reenvios, inclusive no replay (proposta, ver FDD-PROP-23). Deduplicar pelo `event_id` do corpo assinado (proposta, ver FDD-PROP-07). Documentar o comportamento com destaque no portal (`[09:26] Marcos`). | `[09:24] Diego` |
| FDD-RISCO-05 | (inferência deste documento a partir da decisão de acesso e do código) Qualquer pessoa pode passar a receber os eventos de pedidos de qualquer cliente. A reunião aceitou que qualquer role autenticada gerencia os cadastros "por enquanto" (`[09:36] Marcos`, `[09:37] Sofia`), sobre a premissa de que os usuários representam os clientes (`[09:32] Marcos`). No código, o registro de usuário é público e aceita `role` (`src/modules/auth/auth.routes.ts`, `src/modules/auth/auth.schemas.ts`), não existe vínculo entre `User` e `Customer` (`prisma/schema.prisma`) e a listagem de clientes e de pedidos exige apenas `authenticate` (`src/modules/customers/customer.routes.ts`, `src/modules/orders/order.routes.ts`). Pré-existente, não introduzido por esta feature: qualquer pessoa registrada já lê todos os pedidos pela API, com dados mais completos que o payload do webhook. Acrescentado pela feature: o envio automático e persistente desses dados a um endereço escolhido pelo usuário; a listagem, que expõe as URLs dos cadastros de outros clientes; o poder de trocar a URL ou remover um cadastro alheio (FDD-CONTRATO-03, FDD-CONTRATO-04) e de obter a secret nova pela rotação (FDD-CONTRATO-06); e a resposta do destino guardada no histórico (FDD-CONTRATO-05). As rotas por `:id` não checam a quem o cadastro pertence. | Alta | Alto | Bloqueante de deploy, a resolver na revisão de segurança que o time agendou (`[09:46] Sofia`): a decisão de `[09:37] Sofia` precisa ser reexaminada, porque a premissa dela não vale no código. A proposta FDD-PROP-44 restringe o gerenciamento a ADMIN até existir vínculo entre usuário e cliente e exige checagem de propriedade nas rotas por id; como reabre uma decisão da reunião, não é requisito. Restringir a ADMIN só tem efeito se o registro público deixar de aceitar `role` (FDD-RISCO-06). | `[09:37] Sofia` |
| FDD-RISCO-06 | Fraqueza anterior à feature e fora do escopo dela. A rota pública `POST /api/v1/auth/register` não usa `authenticate` (`src/modules/auth/auth.routes.ts`) e aceita `role` `ADMIN` no corpo (`src/modules/auth/auth.schemas.ts`). `AuthService.register` repassa o valor a `UserService.createUser`, que o grava (`src/modules/auth/auth.service.ts`, `src/modules/users/user.service.ts`). Assim, qualquer pessoa obtém ADMIN e, com ele, o replay de DLQ (`[09:36] Sofia`); combinada com FDD-RISCO-05, dispensa até o papel de ADMIN. | Alta | Alto | Bloqueante de deploy: corrigir o registro público antes do deploy e levar à revisão de segurança (`[09:49] Sofia`). A correção é anterior à feature e fica fora do escopo dela, mas a feature não deve ir a produção sem ela: com esse caminho, a exigência de ADMIN no replay (FDD-CONTRATO-07) não é barreira efetiva. | `src/modules/auth/auth.schemas.ts` |
| FDD-RISCO-07 | A transação de `changeStatus`, já descrita como pesada (`[09:04] Bruno`), ganha a consulta aos cadastros e a inserção na outbox. Ela é aberta sem opções em `src/modules/orders/order.service.ts`, fica sujeita aos limites padrão de transação interativa do `@prisma/client` e passa a manter locks por mais tempo. | Média | Médio | Fazer uma única consulta aos cadastros ativos do cliente e uma única escrita em lote das entregas (`createMany`). Montar o snapshot com o pedido já carregado na transação, sem nova consulta. Não alterar as opções de `$transaction` sem decisão na revisão do FDD. | `src/modules/orders/order.service.ts` |
| FDD-RISCO-08 | O worker único para ou cai no meio de um lote. Nenhuma entrega sai até o processo voltar (consequência técnica inferida), e linhas ficam presas em "processando" (estado citado em `[09:08] Diego`). Lacuna FDD-GAP-03. | Média | Alto | Retomar para "pendente" as linhas em processamento há mais que um limite a definir (proposta, ver FDD-PROP-03); o reenvio fica coberto pelo at-least-once. Implementar `shutdown` por `SIGINT`/`SIGTERM` em `src/worker.ts` (novo), no padrão de `src/server.ts`. Definir com o time a supervisão do processo de `npm run worker` (novo). | `[09:08] Diego`, `[09:11] Diego` |
| FDD-RISCO-09 | Um evento em nova tentativa segura os eventos posteriores do mesmo pedido por até quase 15 horas (`[09:17] Diego`), caso se adote a preservação de ordem. Lacuna FDD-GAP-25. | Média | Médio | Restringir o bloqueio ao pedido afetado (proposta, ver FDD-PROP-24). Logar o adiamento com o `event_id` (proposta, ver FDD-PROP-20). Confirmar com Diego o equilíbrio entre atraso e ordem. | `[09:17] Diego` |
| FDD-RISCO-10 | (inferência deste documento; não discutido) O worker faz `POST` para a URL cadastrada pelo usuário. Um usuário autenticado pode apontar o cadastro para um endereço da infraestrutura interna, e a plataforma passa a enviar requisições assinadas para lá. Como a resposta do destino é guardada (`responseStatus`, `responseBody`) e devolvida pelo histórico (FDD-CONTRATO-05), o envio vira também leitura de serviços internos. | Alta | Alto | Validar o destino no cadastro e a cada envio: recusar hosts que resolvem para faixas privadas, loopback ou link-local, conectar ao endereço validado e exigir certificado TLS válido, sem desligar a verificação (proposta, ver FDD-PROP-43). Não guardar nem devolver o corpo da resposta de entregas com destino recusado. Levar à revisão de segurança. | `[09:31] Marcos` (URL informada pelo cliente); `[09:06] Diego` (o worker faz as chamadas HTTP) |

## 15. Lacunas não decididas e propostas a confirmar
A reunião não decidiu nenhum dos pontos abaixo: alguns não foram discutidos e outros ficaram sem fechamento. Nenhuma proposta é requisito nem decisão; cada uma é hipótese de trabalho até a confirmação indicada em "A confirmar com".

### 15.1 Lacunas

| ID | Lacuna | Onde a reunião parou | Por que importa para implementar | Proposta | A confirmar com |
|---|---|---|---|---|---|
| FDD-GAP-01 | Granularidade da linha de `webhook_outbox`: uma linha por evento ou uma por evento × cadastro de destino; onde ficam a contagem de tentativas e o instante da próxima | `[09:06] Diego` fala em inserir "uma linha" "com o evento"; `[09:44] Sofia` mostra que um cliente pode ter vários cadastros; `[09:34] Marcos` pede histórico por webhook | Define o model de `webhook_outbox` em `prisma/schema.prisma`, o que `publishWebhookEvent` insere por mudança de status e a consulta de `src/modules/webhooks/webhook.processor.ts` (novo) | FDD-PROP-01 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-02 | Papel do status "falhou" e transições da outbox, já que a DLQ é tabela separada; como representar "aguardando nova tentativa" | `[09:08] Diego` lista pendente, processando, falhou e entregue; `[09:18] Diego` sugere a tabela `webhook_dead_letter` separada, com concordância de `[09:18] Bruno`; fechada no resumo de `[09:48] Larissa` | Define o enum de status da outbox, o uso do índice em status e `created_at` (`[09:08] Diego`) e as transições que o processor aplica após cada envio | FDD-PROP-02 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-03 | Linhas presas em "processando" após queda do worker | `[09:11] Diego` separa os processos, mas não trata recuperação | O worker lê só pendentes (`[09:08] Diego`): sem retomada, a linha nunca é enviada; o limite precisa superar a duração de um envio com timeout de 10 segundos (`[09:42] Diego`) | FDD-PROP-03 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-04 | Tamanho do lote e concorrência dentro do worker; efeito na meta de latência | `[09:08] Diego` diz "batch pequeno"; `[09:02] Marcos` fixa abaixo de 10 segundos; `[09:42] Diego` fixa timeout de 10 segundos | Com envio sequencial, um único cliente que esgota o timeout de 10 segundos atrasa as entregas seguintes do lote além da meta de 10 segundos | FDD-PROP-04 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-05 | O que conta como entrega bem-sucedida | `[09:34] Marcos` fala em sucesso/falha no histórico; `[09:42] Diego` trata timeout como falha | Define a classificação gravada em `webhook_deliveries` e a escolha do processor entre marcar entregue, agendar nova tentativa ou mover para a DLQ | FDD-PROP-05 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-06 | Formato da assinatura: codificação, prefixo e quais bytes entram no HMAC | `[09:22] Sofia` fixa HMAC-SHA256 sobre o corpo do request | O cliente só valida se o HMAC cobrir os mesmos bytes enviados; define o valor de FDD-HEADER-03 e o que Marcos documenta no portal (`[09:40] Marcos`) | FDD-PROP-06 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`); encaminhar à revisão de segurança de Sofia (`[09:46] Sofia`) é proposta deste documento |
| FDD-GAP-07 | `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id` ficam fora do HMAC do corpo | `[09:22] Sofia` (HMAC sobre o corpo) e `[09:44] Diego` (`X-Timestamp` "pra cliente conseguir detectar replay attack") | Esses headers podem ser alterados sem invalidar a assinatura; o `timestamp` do corpo é fixado na inserção (leitura adotada no ADR-007) e não mede a idade de uma nova tentativa | FDD-PROP-07 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`); encaminhar à revisão de segurança de Sofia (`[09:46] Sofia`) é proposta deste documento |
| FDD-GAP-08 | Armazenamento da secret em repouso e redação no logger | `[09:21] Bruno` e `[09:21] Sofia` (a tabela guarda url + secret); `[09:22] Diego` (cliente que vazou secret em log) | O worker precisa recuperar a secret para assinar, o que exclui guardar só um hash de mão única como o de senha (`*.passwordHash` no logger, `bcrypt` em `package.json`); `redact.paths` de `src/shared/logger/index.ts` não cobre secret | FDD-PROP-08 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`); encaminhar à revisão de segurança de Sofia (`[09:46] Sofia`) é proposta deste documento |
| FDD-GAP-09 | A criação do pedido não emite evento: a emissão foi ancorada em `changeStatus` | `[09:40] Bruno` cita só o método `changeStatus` | `OrderService.create` grava `fromStatus: null` e `toStatus: PENDING`, e nenhuma transição leva a `PENDING` (`src/modules/orders/order.status.ts`): um cadastro que assine `PENDING` nunca recebe evento | FDD-PROP-09 | Marcos, por ser ponto de produto |
| FDD-GAP-10 | Código de erro para URL sem `https` | `[09:23] Sofia` (validação no schema Zod) e `[09:28] Bruno` (exemplo `WEBHOOK_INVALID_URL`) | Define se a regra fica em `src/modules/webhooks/webhook.schemas.ts` (novo), respondida pelo `validate`, ou numa classe derivada de `AppError` | FDD-PROP-10 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-11 | Leitura de "5 tentativas" | `[09:15] Diego`, `[09:17] Diego`, `[09:17] Larissa`, `[09:48] Larissa` | Define o teto que o processor compara antes de mover para a DLQ e se o intervalo de 12 horas chega a ser usado | FDD-PROP-11 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-12 | Nomes dos campos JSON da API de gerenciamento e da lista de status assinados | `[09:31] Marcos` e `[09:33] Marcos` citam url, secret e lista de status, sem nomes de campo | Define os schemas de `src/modules/webhooks/webhook.schemas.ts` (novo) e os exemplos de FDD-CONTRATO-01 a FDD-CONTRATO-07 | FDD-PROP-12 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-13 | Rotas não ditadas: rotação de secret, consulta por id e posição do `customerId` | `[09:21] Sofia` pede endpoint de rotação sem rota; `[09:32] Larissa` diz "no body ou no path"; `[09:33] Bruno` lista PATCH, DELETE e GET de listagem | O JWT não carrega cliente (`src/middlewares/auth.middleware.ts`); define `src/modules/webhooks/webhook.routes.ts` (novo) e o que `validate` checa em `params`, `body` e `query` | FDD-PROP-13 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-14 | Nomes de arquivos e tabelas não ditados | `[09:28] Bruno` dá dois nomes possíveis ao arquivo de processamento; `[09:21] Bruno` descreve a tabela de configuração sem nomeá-la | Define os models e os `@@map` em `prisma/schema.prisma` e os caminhos de import do módulo | FDD-PROP-14 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-15 | Destino das entregas pendentes quando o cadastro é removido, desativado ou muda de URL | não discutido; vizinhos `[09:33] Bruno` (PATCH e DELETE) e `[09:21] Sofia` (confirma o estado ativo do cadastro) | Define o efeito de FDD-CONTRATO-03 e FDD-CONTRATO-04 sobre as linhas da outbox e o que o processor faz diante de cadastro inativo ou removido | FDD-PROP-15 | Marcos, por ser ponto de produto; Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-16 | Payload acima de 64KB: consome tentativas ou vai direto para a DLQ | `[09:23] Sofia` ("Eu sou a favor de erra") e `[09:24] Larissa` ("erro caso ultrapasse") | Se o erro for lançado na inserção, dentro da transação de `changeStatus`, a mudança de status sofre rollback; se for tratado no envio, cada nova tentativa repete a mesma falha | FDD-PROP-16 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-17 | Cliente HTTP e biblioteca de HMAC | não discutido | `package.json` não tem cliente HTTP nem biblioteca de HMAC; o processor precisa do timeout de 10 segundos (`[09:42] Diego`) e de HMAC-SHA256 | FDD-PROP-17 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`); Sofia, na revisão de segurança (`[09:46] Sofia`) |
| FDD-GAP-18 | Configuração do worker: constantes do módulo ou variáveis de ambiente | `[09:09] Diego` (2 segundos) e `[09:42] Diego` (10 segundos) dão os valores sem dizer onde ficam; `src/config/env.ts` valida só `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET` e `JWT_EXPIRES_IN` | Define se `envSchema` muda e como os testes exercitam a tabela 1m/5m/30m/2h/12h sem esperar os intervalos reais | FDD-PROP-18 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-19 | Geração da secret: formato e entropia | `[09:31] Marcos` (secret gerada pela plataforma e devolvida na criação); `[09:46] Sofia` quer revisar a geração de secret | Define a função de geração em `src/modules/webhooks/webhook.service.ts` (novo) e o tamanho da coluna em `prisma/schema.prisma` | FDD-PROP-19 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`); encaminhar à revisão de segurança de Sofia (`[09:46] Sofia`) é proposta deste documento |
| FDD-GAP-20 | Métricas e tracing | não discutido; vizinhos `[09:29] Bruno` (logger Pino) e `[09:36] Sofia` (logar quem fez o replay) | Define quais eventos de log o processor emite e como seguir uma entrega entre outbox, histórico e DLQ | FDD-PROP-20 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-21 | Qual secret assina durante as 24 horas de carência da rotação | `[09:21] Sofia` define a validade paralela de 24 horas, sem dizer qual secret a plataforma usa para assinar | Define quais secrets o cadastro guarda, com que validade, e qual delas o processor usa em FDD-HEADER-03 | FDD-PROP-21 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`); encaminhar à revisão de segurança de Sofia (`[09:46] Sofia`) é proposta deste documento |
| FDD-GAP-22 | Limpeza e arquivamento da outbox e do histórico de entregas | `[09:08] Diego` ("Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature") | Nesta entrega as tabelas crescem sem limpeza; a leitura depende dos índices em status e `created_at` (`[09:08] Diego`) | Sem proposta: fora do escopo desta entrega | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-23 | Regras de chave estrangeira e de exclusão das tabelas novas quando pedido, cliente ou cadastro é removido | não discutido; vizinhos `[09:52] Larissa` (snapshot) e `OrderService.delete` em `src/modules/orders/order.service.ts` | `OrderService.delete` remove pedidos em `PENDING` ou `CANCELLED`; com chave restritiva, o pedido cancelado com evento falha (erro Prisma fora de `P2002`/`P2025` vira `500 INTERNAL_SERVER_ERROR`); com cascata, perde-se a evidência da DLQ (`[09:18] Diego`) | FDD-PROP-22 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-24 | O replay preserva o id de evento e o payload? A contagem de tentativas reinicia? | `[09:18] Diego` ("Recoloca na outbox como pendente"); `[09:25] Diego` (UUID "único por evento") | Com id novo, o cliente não reconhece o evento reprocessado como duplicata (ADR-005); define o que FDD-CONTRATO-07 grava na outbox | FDD-PROP-23 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`) |
| FDD-GAP-25 | Ordem por pedido quando um evento anterior do mesmo pedido entra em nova tentativa | `[09:12] Diego` (ordem por `created_at`), `[09:13] Larissa` (ordenação só por `order_id`), `[09:17] Larissa` (intervalos de até 12 horas) | O evento posterior, já pronto, seria entregue antes do anterior em espera, o que quebra a ordem por `order_id` registrada no ADR-002 | FDD-PROP-24 | Larissa, na revisão do documento de design com Bruno e Diego (`[09:50] Larissa`), em especial Diego |
| FDD-GAP-26 | Ponto de medição da meta de 10 segundos: até o início do envio ou até a resposta do cliente | `[09:02] Marcos`; `[09:42] Diego` | Define o que os testes e a métrica de latência medem; pela segunda leitura, um cliente que responde perto do timeout estoura a meta sozinho | FDD-PROP-25 | Larissa |
| FDD-GAP-27 | "Últimos 100": limite fixo ou exemplo; formato da resposta do histórico | `[09:34] Marcos` | Sem isso o endpoint de histórico não tem contrato fechado | FDD-PROP-26 | Marcos |
| FDD-GAP-28 | Filtro de inserção ignora cadastros inativos? | `[09:21] Sofia`; `[09:34] Bruno` | Decide se um cadastro desativado ainda recebe eventos novos | FDD-PROP-27 | Marcos |
| FDD-GAP-29 | Regras do cadastro: mínimo e repetição de status, valor inicial do estado ativo, inativos na listagem, campos editáveis | `[09:21] Sofia`; `[09:33] Bruno`; `[09:33] Marcos` | São as validações do schema Zod e o corpo das rotas de criação e edição | FDD-PROP-28, FDD-PROP-29, FDD-PROP-30, FDD-PROP-31 | Marcos |
| FDD-GAP-30 | Método HTTP da entrega e tratamento de redirecionamento | `[09:06] Diego`; `[09:44] Diego` | A reunião ditou headers e corpo, não o método; um redirecionamento levaria o corpo assinado a destino não cadastrado | FDD-PROP-33 | Diego e Sofia |
| FDD-GAP-31 | Formato do `X-Timestamp` | `[09:44] Diego` | O cliente só consegue comparar o instante se souber o formato | FDD-PROP-34 | Diego |
| FDD-GAP-32 | Replay: resposta, permanência do item na fila de mortos, replay repetido e replay de cadastro removido ou inativo | `[09:18] Diego`; `[09:36] Sofia` | Sem isso o endpoint não tem contrato fechado e o replay repetido pode duplicar linhas | FDD-PROP-32, FDD-PROP-35 | Diego e Sofia |
| FDD-GAP-33 | Rotação repetida antes do fim das 24 horas de carência | `[09:21] Sofia` | O modelo guarda uma única secret anterior; uma segunda rotação precisa dizer qual prevalece | FDD-PROP-36 | Sofia |
| FDD-GAP-34 | Conteúdo e tamanho da resposta do cliente guardada no histórico | `[09:34] Marcos` | Uma resposta muito grande inflaria a tabela de histórico | FDD-PROP-37 | Diego |
| FDD-GAP-35 | Chave estrangeira de `webhookId` nas tabelas de evento e destino do histórico quando o cadastro é removido | `[09:33] Bruno`; `[09:34] Marcos` | Decide se remover um cadastro apaga ou bloqueia entregas e histórico | FDD-PROP-38 | Diego |
| FDD-GAP-36 | Tamanho das colunas de URL e de secret | `[09:21] Sofia` | A migration precisa de um tipo para essas colunas | — (depende de FDD-PROP-08 e FDD-PROP-19) | Sofia |
| FDD-GAP-37 | Opções de `timeout` e `maxWait` da transação interativa de `changeStatus` | não discutido; `src/modules/orders/order.service.ts` | A transação é aberta sem opções e passa a incluir a consulta aos cadastros e a inserção na outbox | FDD-PROP-39 | Diego |
| FDD-GAP-38 | Alerta interno ao time quando a fila de mortos cresce | `[09:37] Larissa` (o aviso por e-mail ao cliente saiu de escopo) | As métricas da seção 10 não têm limiar nem destinatário | — (observar junto com a questão do limite de envio, RFC-QA-01) | Larissa |
| FDD-GAP-39 | Log de publicação emitido dentro da transação, antes do commit | não discutido; decorre da proposta FDD-PROP-20 (log de publicação emitido por `publishWebhookEvent`) | Em caso de rollback fica um log de publicação sem a linha correspondente na outbox | — | Diego |
| FDD-GAP-40 | Identificação do processo nos logs: `createLogger` fixa `service` para os dois processos | `src/shared/logger/index.ts` | Sem distinção, os logs do worker e da API são indistinguíveis | FDD-PROP-20 (logger filho no worker) | Diego |
| FDD-GAP-41 | O worker exige o mesmo `.env` da API, inclusive `JWT_SECRET` | `src/config/env.ts` | Sem esse arquivo o worker não sobe, mesmo sem usar o JWT | — (fato de operação: documentar na implantação) | Diego |
| FDD-GAP-42 | Comportamento do encerramento do worker com lote em andamento | `[09:11] Larissa` (modelo de `src/server.ts`) | Interrar o lote no meio deixa linhas em processamento | FDD-PROP-40 | Diego |
| FDD-GAP-43 | Unidade de medida do limite de 64KB: bytes do corpo JSON serializado ou outra grandeza | `[09:24] Diego`; `[09:24] Larissa` | O critério FDD-CA-14 e a checagem feita pelo worker dependem dessa definição | FDD-PROP-41 | Diego |
| FDD-GAP-44 | Quantidade e granularidade das migrations da feature | não discutido; `package.json` define apenas o script `db:migrate` | Decide se as quatro tabelas entram de uma vez ou em passos | FDD-PROP-42 | Diego |
| FDD-GAP-45 | Validação do destino das entregas: a URL é informada pelo usuário e o worker faz a chamada (risco de requisições a endereços internos) | não discutido; `[09:31] Marcos` (URL informada pelo cliente) e `[09:06] Diego` (o worker faz as chamadas HTTP) | Sem validação, a plataforma vira cliente HTTP de qualquer host https escolhido por um usuário autenticado | FDD-PROP-43 | Diego e Sofia |
| FDD-GAP-46 | Autorização do cadastro diante do registro público: a premissa de que todo usuário autenticado representa um cliente não vale no código | `[09:32] Marcos` (os usuários representam o cliente) e `[09:37] Sofia` (qualquer role autenticada, por enquanto); o registro é público em `src/modules/auth/auth.routes.ts` | Sem vínculo entre usuário e cliente, qualquer pessoa registrada cadastra webhook para qualquer cliente e recebe os eventos dos pedidos dele | FDD-PROP-44 (reabre a decisão de `[09:37] Sofia`) | Larissa e Sofia, antes da produção |
| FDD-GAP-47 | Teto de cadastros por cliente (amplificação do trabalho do worker único) | não discutido; `[09:38] Diego` e `[09:39] Larissa` tratam do limite de chamadas ao cliente, não da quantidade de cadastros | Cada cadastro gera uma linha por mudança de status, e cada linha pode custar até 10 segundos por tentativa no worker sequencial | FDD-PROP-45 | Diego |

### 15.2 Propostas

| ID | Proposta | Atende à | O que quebra se for recusada |
|---|---|---|---|
| FDD-PROP-01 | Uma linha de `webhook_outbox` por evento × cadastro de destino (entrega), porque a contagem de tentativas e o histórico são por destino | FDD-GAP-01 | Com uma linha por evento, o estado de retry de cada cadastro precisa de outra tabela, e um evento pode ficar entregue para um cadastro e pendente para outro |
| FDD-PROP-02 | A linha que aguarda nova tentativa fica em "pendente", com o instante da próxima tentativa; "falhou" marca a linha que esgotou as tentativas, antes da ida à DLQ; ao ir para `webhook_dead_letter`, a linha sai da outbox | FDD-GAP-02 | O enum de status e a consulta do processor mudam; se a linha esgotada ficar na outbox, volta a mistura que o ADR-003 preteriu |
| FDD-PROP-03 | Retomar para "pendente" as linhas em "processando" há mais que um limite a definir | FDD-GAP-03 | Linhas presas após queda do worker não são enviadas nem vão para a DLQ, e a recuperação vira intervenção manual no banco |
| FDD-PROP-04 | Lote pequeno, com envio em sequência por `created_at` como base, para preservar a ordem por pedido (`[09:12] Diego`); tamanho do lote a fixar na revisão do FDD; paralelismo só entre linhas de pedidos diferentes, e só se a meta de latência não for atingida, sempre sujeito a FDD-PROP-24 | FDD-GAP-04 | Com envio sequencial, a meta abaixo de 10 segundos (`[09:02] Marcos`) passa a depender do cliente mais lento do lote |
| FDD-PROP-05 | Resposta 2xx é sucesso; qualquer outra resposta, erro de rede ou timeout é falha | FDD-GAP-05 | Tratar 3xx ou 4xx de outro modo exige uma regra própria no processor e muda o sentido de sucesso/falha no histórico de FDD-CONTRATO-05 |
| FDD-PROP-06 | HMAC-SHA256 sobre os bytes exatos do corpo enviado, em hexadecimal, sem prefixo, a confirmar na revisão do documento de design (encaminhar à revisão de Sofia é proposta deste documento) | FDD-GAP-06 | Outra codificação ou um prefixo mudam o valor de FDD-HEADER-03 e o portal; o HMAC sobre JSON reserializado faz a verificação do cliente falhar |
| FDD-PROP-07 | O cliente deduplica pelo `event_id` do corpo, que é assinado: essa é a defesa contra reenvio. `X-Timestamp` não é assinado, e o `timestamp` do corpo é fixado na inserção, de modo que nenhum dos dois prova frescor; a submeter à revisão do documento de design (encaminhar à revisão de Sofia é proposta deste documento) | FDD-GAP-07 | Exigir headers assinados muda a entrada do HMAC, decidida sobre o corpo no ADR-004, e obriga a refazer FDD-PROP-06 |
| FDD-PROP-08 | Guardar a secret de forma recuperável, porque o worker precisa dela para assinar; acrescentar a `redact.paths` de `src/shared/logger/index.ts` os campos que carregam secret (por exemplo `*.secret`); criptografia em repouso a decidir com Sofia; e tratar os erros do Prisma no service antes de repassá-los ao middleware, porque a mensagem de um erro de validação do Prisma pode ecoar os dados passados na chamada, inclusive a secret (inferência a partir de `src/middlewares/error.middleware.ts`) | FDD-GAP-08 | Um hash de mão única impede o worker de calcular o HMAC; sem a redação, a secret pode aparecer em log, como no caso relatado em `[09:22] Diego` |
| FDD-PROP-09 | Manter a emissão só em `OrderService.changeStatus`; um cadastro que assine `PENDING` não recebe nada; a confirmar com Marcos | FDD-GAP-09 | Emitir na criação exige chamar `publishWebhookEvent` em `OrderService.create` com `fromStatus` nulo, o que admite `from_status` nulo no evento |
| FDD-PROP-10 | O schema Zod de `src/modules/webhooks/webhook.schemas.ts` (novo) rejeita URL sem `https` e a resposta é `400 VALIDATION_ERROR`; `WEBHOOK_INVALID_URL` fica sem uso | FDD-GAP-10 | Emitir `WEBHOOK_INVALID_URL` exige uma classe derivada de `AppError` e tirar a regra do schema, contra "só uma validação no schema Zod" (`[09:23] Sofia`) |
| FDD-PROP-11 | 1 envio inicial + 5 novas tentativas, nos intervalos 1m/5m/30m/2h/12h, como registrado no ADR-003 | FDD-GAP-11 | Com 5 envios no total, o intervalo de 12 horas fica sem uso, o que contraria as "quase 15 horas" de `[09:17] Diego` |
| FDD-PROP-12 | camelCase nos campos da API de gerenciamento; lista de status no campo `statuses`; demais nomes conforme os exemplos da seção de contratos | FDD-GAP-12 | Outra convenção diverge de `customerId` e `orderNumber` do código e refaz os exemplos de FDD-CONTRATO-01 a FDD-CONTRATO-07 |
| FDD-PROP-13 | `customerId` no body na criação; a edição não troca de cliente (FDD-PROP-31); consulta por id não entra, por não ter sido pedida; rotação em `POST /api/v1/webhooks/:id/rotate-secret`. Na listagem, a reunião disse "no body ou no path" (`[09:32] Larissa`): este documento usa `customerId` em query, como os filtros de listagem existentes, e registra a divergência para a revisão | FDD-GAP-13 | `customerId` no path muda os caminhos de FDD-CONTRATO-01 e FDD-CONTRATO-02 e o que o `validate` checa; outra rota de rotação muda FDD-CONTRATO-06 |
| FDD-PROP-14 | Os nomes de arquivos e tabelas usados neste documento: `src/modules/webhooks/webhook.processor.ts` (novo), `webhooks` e `webhook_deliveries`, ao lado de `webhook_outbox` e `webhook_dead_letter` | FDD-GAP-14 | Renomear após a migração inicial exige nova migração em `prisma/schema.prisma` e ajuste de imports e testes |
| FDD-PROP-15 | Entregas pendentes de um cadastro inativo ou removido vão para `webhook_dead_letter`, com o motivo registrado | FDD-GAP-15 | Descartá-las apaga a evidência que a DLQ guarda (`[09:18] Diego`); entregá-las envia eventos a um cadastro que o cliente desligou. A mudança de URL segue sem proposta |
| FDD-PROP-16 | Payload acima de 64KB vai direto para `webhook_dead_letter`, sem consumir tentativas, por ser falha determinística | FDD-GAP-16 | Consumir tentativas repete o mesmo erro por quase 15 horas; lançar o erro na inserção desfaz a mudança de status por causa do webhook |
| FDD-PROP-17 | `fetch` global do Node 20 (`engines` em `package.json`), `node:crypto` para o HMAC e a geração da secret e o pacote `uuid` já instalado para o id do evento; nenhuma dependência nova | FDD-GAP-17 | Uma biblioteca nova entra em `package.json` e no escopo da revisão de Sofia (`[09:46] Sofia`), e precisa cobrir o timeout de 10 segundos |
| FDD-PROP-18 | Valores decididos (2 segundos, 10 segundos, 64KB, 5 tentativas, 1m/5m/30m/2h/12h, 24 horas) como constantes do módulo; do ambiente, só o que `src/config/env.ts` já valida | FDD-GAP-18 | Ler do ambiente exige estender `envSchema` e a configuração dos dois processos, e permite que API e worker divirjam, nos valores adotados |
| FDD-PROP-19 | Gerador criptograficamente seguro do Node (`node:crypto`); tamanho a definir com Sofia | FDD-GAP-19 | A função de geração em `src/modules/webhooks/webhook.service.ts` (novo) e a coluna em `prisma/schema.prisma` mudam conforme o formato que Sofia definir |
| FDD-PROP-20 | Métricas por contagem nos logs estruturados do Pino e correlação por `event_id`; tracing distribuído fica fora | FDD-GAP-20 | Métricas fora dos logs exigem dependência ou infraestrutura nova, fora do reuso do ADR-006 |
| FDD-PROP-21 | A partir da rotação a plataforma assina com a secret nova; a antiga continua aceita pelo cliente por 24 horas, ao fim das quais deixa de valer. Leitura alternativa: assinar com a antiga até o fim do prazo. A primeira deixa antes de usar uma secret que pode ter vazado (`[09:22] Diego`); a segunda protege quem só verifica com a antiga. Decisão pendente (RFC-QA-07), a confirmar na revisão do documento de design (encaminhar à revisão de Sofia é proposta deste documento) | FDD-GAP-21 | Assinar com a nova desde a rotação quebra, até o cliente atualizar a verificação, quem só aceita a antiga; assinar com a antiga mantém em uso, por 24 horas, uma secret que pode ter vazado |
| FDD-PROP-22 | As tabelas de evento guardam apenas os ids de pedido e de cliente, sem chave estrangeira para `orders` nem `customers`, para que o snapshot sobreviva ao pedido; o cadastro tem chave estrangeira para o cliente | FDD-GAP-23 | Chave restritiva faz `OrderService.delete` falhar com `500 INTERNAL_SERVER_ERROR` para pedido cancelado com evento; cascata apaga a DLQ e o histórico |
| FDD-PROP-23 | O replay recoloca a mesma linha: mesmo id de evento e mesmo payload, com a contagem de tentativas zerada | FDD-GAP-24 | Com id novo, o cliente não deduplica o evento reprocessado (ADR-005); sem zerar a contagem, a primeira falha devolve a linha à DLQ |
| FDD-PROP-24 | O worker não entrega evento posterior de um pedido enquanto houver um anterior dele pendente ou em nova tentativa; preserva a ordem ao custo de atrasar esse pedido; a confirmar com Diego | FDD-GAP-25 | Sem a regra, a ordem por `order_id` do ADR-002 deixa de valer sempre que um evento anterior aguarda nova tentativa |
| FDD-PROP-25 | A meta de 10 segundos é medida do commit da mudança de status até o início do envio inicial; as novas tentativas ficam fora da meta. A métrica de produto do PRD (PRD-OBJ-01) mede até a chegada ao cliente: as duas leituras devem ser reconciliadas na revisão | FDD-GAP-26 | Sem critério, a meta não é testável |
| FDD-PROP-26 | O histórico devolve as últimas 100 entregas em lista simples, na chave `data`, sem paginação | FDD-GAP-27 | O contrato de FDD-CONTRATO-05 fica sem formato |
| FDD-PROP-27 | O filtro de inserção só considera cadastros ativos | FDD-GAP-28 | Cadastro desativado continuaria recebendo eventos |
| FDD-PROP-28 | `statuses` aceita ao menos um valor, sem repetição | FDD-GAP-29 | Cadastro sem status não receberia nada e seria indistinguível de erro |
| FDD-PROP-29 | O cadastro nasce com `active: true` | FDD-GAP-29 | A criação precisaria de um campo a mais no corpo |
| FDD-PROP-30 | A listagem inclui os cadastros inativos, com `active: false` | FDD-GAP-29 | Cadastro desativado desapareceria da listagem |
| FDD-PROP-31 | A edição aceita `url`, `statuses` e `active`; não troca o cliente do cadastro | FDD-GAP-29 | A troca de cliente exigiria regras de autorização não discutidas |
| FDD-PROP-32 | Rotação e replay respondem `200`; a rotação devolve a nova secret e o instante em que a anterior deixa de valer; o replay devolve os identificadores do evento e do cadastro | FDD-GAP-32 | O contrato de FDD-CONTRATO-07 fica sem resposta definida |
| FDD-PROP-33 | A entrega usa `POST` e não segue redirecionamento; resposta 3xx conta como falha | FDD-GAP-30 | O corpo assinado poderia chegar a destino não cadastrado |
| FDD-PROP-34 | `X-Timestamp` vai em ISO 8601, em UTC | FDD-GAP-31 | O cliente não poderia comparar o instante do envio |
| FDD-PROP-35 | O item da fila de mortos permanece nela após o replay, com a data do último replay | FDD-GAP-32 | A evidência de debug citada em `[09:18] Diego` seria perdida |
| FDD-PROP-36 | Em rotação repetida, a secret que ainda estava em carência é descartada; só a última anterior é mantida | FDD-GAP-33 | O modelo precisaria de uma lista de secrets anteriores |
| FDD-PROP-37 | O histórico guarda o status HTTP e o corpo da resposta do cliente, com o corpo truncado num limite a definir | FDD-GAP-34 | Uma resposta grande inflaria a tabela de histórico |
| FDD-PROP-38 | `webhookId` nas tabelas de evento não tem chave estrangeira; o histórico é mantido quando o cadastro é removido | FDD-GAP-35 | Remover um cadastro apagaria entregas pendentes ou falharia |
| FDD-PROP-39 | Manter os padrões do Prisma na transação de `changeStatus` e medir a duração dela; definir `timeout` explícito só se a medição justificar | FDD-GAP-37 | Mudar o padrão sem medição alteraria o comportamento de uma transação existente |
| FDD-PROP-40 | O encerramento do worker aguarda o término do lote corrente antes de desconectar o Prisma | FDD-GAP-42 | Toda parada deixaria linhas em processamento até a retomada de FDD-PROP-03 |
| FDD-PROP-41 | O limite de 64KB aplica-se aos bytes do corpo JSON serializado em UTF-8 | FDD-GAP-43 | A checagem de tamanho feita pelo worker não teria o que medir |
| FDD-PROP-42 | Uma única migration cria as quatro tabelas novas | FDD-GAP-44 | A ordem de aplicação de migrations parciais precisaria ser coordenada entre ambientes |
| FDD-PROP-43 | O destino é validado no cadastro e a cada envio: o worker recusa os que resolvem para endereços privados, loopback ou link-local, conecta ao endereço validado (contra troca de DNS) e exige certificado TLS válido; entregas com destino recusado não têm resposta guardada | FDD-GAP-45 | A plataforma vira cliente HTTP de destinos escolhidos por qualquer usuário autenticado |
| FDD-PROP-44 | Até existir vínculo entre usuário e cliente, restringir os contratos de gerenciamento (FDD-CONTRATO-01 a FDD-CONTRATO-06) a `requireRole('ADMIN')` e checar a quem pertence o cadastro nas rotas por id. Reabre a decisão de `[09:37] Sofia`; não é requisito | FDD-GAP-46 | Sem isso, qualquer pessoa registrada lê eventos de pedidos de qualquer cliente |
| FDD-PROP-45 | Teto pequeno de cadastros por cliente, com número a fixar na revisão do FDD | FDD-GAP-47 | Sem teto, muitos cadastros lentos para um cliente multiplicam o trabalho do worker único |
