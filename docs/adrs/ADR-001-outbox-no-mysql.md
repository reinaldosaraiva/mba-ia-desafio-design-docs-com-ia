# ADR-001: Outbox no MySQL para eventos de mudança de status

| Campo | Valor |
|---|---|
| ID | ADR-001 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-002](./ADR-002-worker-separado-com-polling.md) · [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) · [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) · [ADR-008](./ADR-008-filtro-de-eventos-na-insercao.md) |

## Status

Aceito. A decisão foi fechada na reunião em `[09:08] Larissa` ("Tá decidido então: outbox em MySQL."), reafirmada em `[09:40] Bruno` e `[09:41] Diego` e confirmada no resumo final em `[09:48] Larissa`.

## Contexto

1. Os clientes B2B querem ser notificados quando o status dos seus pedidos muda, em vez de consultar o `GET /orders` periodicamente (`[09:00] Marcos`).
2. A transação de mudança de status já é pesada: ela atualiza `orders`, insere em `order_status_history` e mexe no estoque (`[09:04] Bruno`). No código, essa transação é o callback de `this.prisma.$transaction` em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). Ela executa `tx.order.update` e `tx.orderStatusHistory.create` e faz o débito ou a reposição de estoque quando `shouldDebitStock` ou `shouldReplenishStock` indicam.
3. Bruno apontou dois problemas de uma chamada HTTP dentro dessa transação (`[09:04] Bruno`):
   - qualquer cliente lento travaria a mudança de status de outros pedidos;
   - um cliente fora do ar levaria à pergunta "dá rollback na mudança de status?", cuja resposta foi "Não dá".
4. É necessária a seguinte garantia (`[09:06] Diego`):
   - se a transação principal fez commit, o evento foi registrado;
   - se ela fez rollback, o evento some junto;
   - não há inconsistência possível.
5. O time é pequeno, e subir infraestrutura nova para isso seria overengineering (`[09:07] Larissa`, `[09:07] Diego`). O banco existente é MySQL, conforme o `datasource db` de `prisma/schema.prisma` (`provider = "mysql"`, `url = env("DATABASE_URL")`).
6. Bruno levantou uma preocupação com o crescimento: se muitos eventos se acumularem na tabela, o worker pode ficar lento (`[09:07] Bruno`). Diego respondeu (`[09:08] Diego`):
   - a tabela tem índice no campo de status e em `created_at`;
   - o worker lê só os pendentes, em batch pequeno;
   - as linhas entregues seriam arquivadas depois de 30 dias "ou assim", fora do escopo desta feature.

   O modo de leitura é tratado no ADR-002.
7. Regra de falha: se a inserção na outbox falhar, a transação sofre rollback. "Não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno`). Se a inserção ficar fora da transação, "perde a garantia toda" (`[09:41] Diego`).

## Decisão

**ADR-001.** Nós usamos o padrão outbox no MySQL existente para os eventos de mudança de status do pedido (`[09:06] Diego`, `[09:08] Larissa`):

1. **Registro na outbox.** Nós registramos o evento de mudança de status em uma tabela de outbox no próprio MySQL. O registro acontece dentro da mesma transação SQL que atualiza o pedido e grava o histórico de status (`[09:06] Diego`, `[09:40] Bruno`).
   - `webhook_outbox` é o nome citado na reunião (`[09:06] Diego`) para essa tabela.
   - A tabela ainda não existe em `prisma/schema.prisma` e será criada.
2. **Ponto de encaixe.** A inserção na outbox entra na transação de `OrderService.changeStatus`, em `src/modules/orders/order.service.ts` (`[09:40] Bruno`).
   - Ela fica no mesmo callback de `this.prisma.$transaction` que executa `tx.order.update` e `tx.orderStatusHistory.create`.
   - A função chamada nesse ponto é objeto do ADR-006.
3. **Falha na inserção.** Se a inserção na outbox falhar, a transação inteira sofre rollback e o status do pedido não muda (`[09:40] Bruno`, `[09:41] Diego`).
4. **Worker separado.** Um worker separado lê a outbox e faz as chamadas HTTP. Nenhuma chamada HTTP acontece dentro da transação de mudança de status (`[09:06] Diego`). O modo como o worker lê a tabela é decidido no ADR-002.
5. **Conteúdo e filtro.** O conteúdo de cada linha segue o ADR-007 (snapshot na inserção). A decisão de inserir ou não segue o ADR-008 (filtro na inserção).

**Interpretação a confirmar (escopo do ponto de encaixe).**
- Leitura adotada: apenas `OrderService.changeStatus` insere na outbox, porque é o único método citado em `[09:40] Bruno`.
- Outra leitura possível: o código mostra que `OrderService.create` também grava histórico (`fromStatus: null`, `toStatus: PENDING`) na sua própria transação. Como `[09:00] Marcos` e `[09:06] Diego` falam em "status muda", a criação do pedido também poderia gerar evento.
- A reunião não tratou desse caso.

**Interpretação a confirmar (rollback e filtro).**
- Conflito: `[09:40] Bruno` diz que "não pode ter caso de status mudar e evento não sair". Já `[09:34] Bruno` decidiu não inserir nada quando nenhum webhook do customer quer aquele status (ADR-008).
- Leitura adotada: o rollback vale para falha na inserção, e não para a ausência de inserção causada pelo filtro.
- Outra leitura possível: a leitura literal de `[09:40] Bruno` exigiria um evento para toda mudança de status, o que contradiz `[09:34] Bruno`.

## Alternativas Consideradas

### ADR-001-ALT-1: Disparo síncrono no service de pedidos

Larissa levantou esta opção como primeira pergunta: disparar a notificação de forma síncrona no service de orders quando o status muda (`[09:03] Larissa`).

A alternativa foi rejeitada pelos seguintes motivos (`[09:04] Bruno`):
- a chamada HTTP entraria numa transação que já é pesada;
- um cliente lento travaria a mudança de status de outros pedidos;
- um cliente fora do ar não justificaria o rollback da mudança de status.

Larissa concordou (`[09:04] Larissa`) e Diego a declarou "fora de questão" (`[09:06] Diego`).

Trade-off que levou ao descarte: a mudança de status ficaria acoplada à latência e à disponibilidade do endpoint do cliente.

### ADR-001-ALT-2: Redis Streams / Redis Cluster como fila

Larissa levantou esta opção como alternativa à outbox: usar Redis Streams "ou alguma coisa parecida", com a ressalva de que "a gente acabaria precisando subir mais infra" (`[09:07] Larissa`).

Diego a descartou (`[09:07] Diego`):
- o time é pequeno;
- subir Redis Cluster para isso é overengineering;
- a outbox no MySQL existente resolve.

Trade-off que levou ao descarte: a opção exigiria infraestrutura nova para um time pequeno operar, sem necessidade, já que o MySQL existente atende.

## Consequências

### Positivas

- **Atomicidade.** A mudança de status e o registro do evento são atômicos: o commit registra o evento e o rollback o descarta, sem inconsistência possível (`[09:06] Diego`).
- **Sem infraestrutura nova.** A outbox usa o MySQL já configurado em `prisma/schema.prisma` (`[09:07] Larissa`, `[09:07] Diego`).
- **Isolamento do cliente.** A latência e a disponibilidade do endpoint do cliente deixam de afetar a transação de mudança de status, porque a chamada HTTP sai para o worker separado (`[09:04] Bruno`, `[09:06] Diego`).

### Negativas e trade-off explícito

- **Dependência da outbox.** A mudança de status passa a depender da inserção na outbox: se a inserção falhar, o status não muda (`[09:40] Bruno`, `[09:41] Diego`).
- **Erro propagado ao chamador.** Quem chama `OrderService.changeStatus` passa a receber erro também quando a falha está na inserção da outbox. Um erro lançado dentro do callback de `this.prisma.$transaction` desfaz a transação e sobe ao chamador, como já ocorre nesse método com `NotFoundError` e `InvalidStatusTransitionError` (consequência técnica inferida).
- **Transação mais pesada.** A transação, que já é pesada (`[09:04] Bruno`), ganha a escrita na outbox (consequência técnica inferida).
- **Crescimento da tabela.** A tabela cresce a cada evento inserido (`[09:07] Bruno`). O arquivamento de linhas entregues depois de 30 dias ficou fora do escopo desta feature (`[09:08] Diego`).
- **Entrega assíncrona.** A entrega deixa de acontecer no momento do commit e passa a depender do ciclo do worker descrito no ADR-002 (consequência técnica inferida).

**Trade-off aceito:** aceitamos que a mudança de status sofra rollback quando a inserção na outbox falhar e que a tabela cresça sem arquivamento nesta feature, em troca de atomicidade entre a mudança de status e o registro do evento e de não subir infraestrutura nova (`[09:06] Diego`, `[09:07] Diego`, `[09:08] Diego`, `[09:40] Bruno`).

## Referências

- **Transcrição:** `[09:00] Marcos`, `[09:03] Larissa`, `[09:04] Bruno`, `[09:04] Larissa`, `[09:06] Diego`, `[09:07] Larissa`, `[09:07] Diego`, `[09:07] Bruno`, `[09:08] Diego`, `[09:08] Larissa`, `[09:34] Bruno`, `[09:40] Bruno`, `[09:41] Diego`, `[09:48] Larissa`.
- **Código existente:**
  - `src/modules/orders/order.service.ts`:
    - `OrderService.changeStatus`: callback de `this.prisma.$transaction` com `tx.order.update` e `tx.orderStatusHistory.create`, além de `debitStock`/`replenishStock`, condicionados por `shouldDebitStock`/`shouldReplenishStock`;
    - `OrderService.create`: histórico inicial citado na interpretação a confirmar.
  - `prisma/schema.prisma`:
    - `datasource db` (`provider = "mysql"`);
    - models `Order` (`@@map("orders")`) e `OrderStatusHistory` (`@@map("order_status_history")`).
- **Documentos:**
  - [RFC](../RFC.md) (seção de decisões relacionadas);
  - [FDD](../FDD.md) (detalhamento de implementação, fora da altitude deste ADR);
  - [ADR-002](./ADR-002-worker-separado-com-polling.md), [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md), [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) e [ADR-008](./ADR-008-filtro-de-eventos-na-insercao.md).
