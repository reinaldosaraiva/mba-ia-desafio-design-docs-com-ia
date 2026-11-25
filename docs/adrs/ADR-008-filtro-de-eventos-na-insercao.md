# ADR-008: Filtro de eventos aplicado na inserção na outbox

| Campo | Valor |
|---|---|
| ID | ADR-008 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Bruno (fecha a decisão), Diego (concorda) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) |

## Status

Aceito. Decisão fechada na reunião em `[09:34] Bruno`, com concordância de `[09:34] Diego`.

## Contexto

1. No cadastro de webhook, o cliente informa a lista de status que quer receber (`[09:31] Marcos`).
2. A escolha dos eventos é feita por endpoint: "Por endpoint a gente pode escolher quais eventos receber" (`[09:33] Bruno`). Marcos confirma que o filtro de eventos é uma lista dos status que o webhook quer ouvir (`[09:33] Marcos`).
3. O exemplo dado é o do cliente que diz "só quero saber quando vira SHIPPED e DELIVERED". Na mesma fala, Marcos já situa o filtro "na hora de inserir na outbox" (`[09:33] Marcos`).
4. Diego pergunta se o filtro ocorre na inserção da outbox ou na hora de mandar (`[09:34] Diego`), o que abre a escolha registrada neste ADR.
5. Bruno aponta o ganho de filtrar na inserção: "Economiza linha na tabela" (`[09:34] Bruno`).
6. A inserção na outbox acontece dentro da mesma transação que muda o status do pedido (`[09:06] Diego`, `[09:40] Bruno`), conforme o [ADR-001](./ADR-001-outbox-no-mysql.md). Essa transação já é descrita como pesada (`[09:04] Bruno`).
7. No código atual, o status de destino é conhecido dentro dessa transação. Em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), `to` vem de `input.toStatus` e é validado por `canTransition(from, to)`. Essa função consulta a tabela `transitions` de `src/modules/orders/order.status.ts`.

## Decisão

Decisão ADR-008, fechada em `[09:34] Bruno` com concordância de `[09:34] Diego`:

1. Nós guardamos, em cada cadastro de webhook, a lista de status que ele quer ouvir (`[09:31] Marcos`, `[09:33] Marcos`).
2. Nós aplicamos o filtro no momento da inserção na outbox, e não no momento do envio (`[09:33] Marcos`, `[09:34] Bruno`).
3. Se nenhum webhook do cliente do pedido assina o status de destino da mudança, nenhuma linha é inserida na outbox (`[09:34] Bruno`).
4. (consequência técnica inferida a partir do código) Os valores da lista são os do enum `OrderStatus`, importado de `@prisma/client`. É o mesmo enum que define a tabela `transitions` em `src/modules/orders/order.status.ts`.
5. A validação desses valores em schema Zod segue a convenção existente `z.nativeEnum(OrderStatus)`, usada em `updateOrderStatusSchema` e `listOrdersQuerySchema` de `src/modules/orders/order.schemas.ts`. Isso está em linha com o reuso dos padrões de schemas Zod (`[09:30] Larissa`) registrado no [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md).

**Interpretação a confirmar (qual status é comparado):**

- **Leitura adotada.** O item 3 afirma que a lista é comparada com o status de destino da mudança. Essa leitura se sustenta no exemplo "quando vira SHIPPED e DELIVERED" (`[09:33] Marcos`).
- **Outra leitura possível.** A fala que fecha a decisão diz apenas "aquele status" (`[09:34] Bruno`), sem indicar se é o status de origem ou o de destino. A outra leitura seria comparar a lista com o status de origem. Esse status também está disponível na inserção, porque a função proposta por Bruno para inserir o evento recebe os dois status (`[09:41] Bruno`).
- **Efeito da outra leitura.** Nela, o exemplo de `DELIVERED` nunca casaria. `DELIVERED` não tem transições de saída na tabela `transitions` de `src/modules/orders/order.status.ts`.

## Alternativas Consideradas

### ADR-008-ALT-1: Filtrar no momento do envio, e não na inserção

- **O que era:** inserir o evento na outbox e decidir, só na hora de mandar, se cada webhook deve recebê-lo.
- **Quem levantou:** Diego, com a pergunta "Filtra na inserção do outbox ou na hora de mandar?" (`[09:34] Diego`).
- **Por que foi descartada:** Bruno responde "Na inserção". Ele justifica que, se nenhum webhook do customer quer aquele status, nem insere, e assim "Economiza linha na tabela" (`[09:34] Bruno`). Diego concorda (`[09:34] Diego`).
- **Trade-off:** a reunião não registrou argumento a favor desta alternativa além da própria pergunta.
  - Uma vantagem possível seria avaliar a lista vigente do webhook no momento do envio, refletindo cadastros e edições feitos depois da mudança de status (consequência técnica inferida).
  - O custo apontado na reunião são linhas na tabela para eventos que nenhum webhook do cliente quer (`[09:34] Bruno`).

## Consequências

### Positivas

- Mudanças para status que nenhum webhook do cliente assina não geram linha na outbox: "Economiza linha na tabela" (`[09:34] Bruno`).
- O worker lê os pendentes em batch pequeno (`[09:08] Diego`) a cada 2 segundos (`[09:09] Diego`). Com o filtro na inserção, ele não busca nem descarta eventos sem destinatário (consequência técnica inferida).
- O worker não precisa consultar a lista de status dos webhooks na hora de mandar, porque toda linha pendente já passou pelo filtro (consequência técnica inferida).

### Negativas e trade-off explícito

- Um webhook cadastrado depois de uma mudança de status não recebe o evento dessa mudança. Nenhuma linha foi inserida para ele naquele momento (consequência técnica inferida).
- Uma edição da lista de status via PATCH (`[09:33] Bruno`) só vale para mudanças de status posteriores. Linhas já inseridas não são reavaliadas contra a lista nova (consequência técnica inferida).
- A transação de mudança de status, que já é pesada (`[09:04] Bruno`), passa a consultar os cadastros de webhook do cliente para decidir se insere (consequência técnica inferida).
- Na leitura por status de destino, incluir `PENDING` na lista não gera eventos vindos de `OrderService.changeStatus` (consequência técnica inferida):
  - nenhuma entrada da tabela `transitions` tem `PENDING` como destino;
  - o pedido nasce `PENDING` em `OrderService.create`, fora de `changeStatus`.

**Trade-off aceito:** aceitamos filtrar os eventos na hora de inserir na outbox (`[09:33] Marcos`), e não na hora de mandar (`[09:34] Diego`), em troca de economizar linha na tabela (`[09:34] Bruno`).

## Referências

- **Transcrição:** `[09:04] Bruno`, `[09:06] Diego`, `[09:08] Diego`, `[09:09] Diego`, `[09:30] Larissa`, `[09:31] Marcos`, `[09:33] Bruno`, `[09:33] Marcos`, `[09:34] Diego`, `[09:34] Bruno`, `[09:40] Bruno`, `[09:41] Bruno`.
- **Código existente:**
  - `src/modules/orders/order.status.ts`: enum `OrderStatus` (importado de `@prisma/client`), tabela `transitions` e função `canTransition`.
  - `src/modules/orders/order.schemas.ts`: uso de `z.nativeEnum(OrderStatus)` em `updateOrderStatusSchema` e `listOrdersQuerySchema`.
  - `src/modules/orders/order.service.ts`: métodos `OrderService.changeStatus` e `OrderService.create`.
- **Documentos:** [RFC](../RFC.md) (seção de decisões relacionadas) · [FDD](../FDD.md) (detalhamento de implementação desta decisão) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md).
