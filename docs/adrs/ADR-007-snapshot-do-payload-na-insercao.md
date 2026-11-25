# ADR-007: Snapshot do payload no momento da inserção

| Campo | Valor |
|---|---|
| ID | ADR-007 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Bruno (Engenheiro Pleno, time de Pedidos; fecha a decisão), Larissa (Tech Lead), Diego (Engenheiro Sênior, time de Plataforma) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) |

## Status

Aceito. Decisão fechada na reunião em `[09:52] Bruno` ("Beleza, snapshot. Decidido."), sobre a preferência manifestada por `[09:52] Larissa` e a concordância de `[09:52] Diego`.

## Contexto

1. A linha da outbox é inserida na mesma transação de `OrderService.changeStatus` que muda o status do pedido (`[09:40] Bruno`, `[09:41] Diego`). A escolha da outbox e a atomicidade dessa inserção são tratadas no [ADR-001](./ADR-001-outbox-no-mysql.md).
2. O formato do payload já havia sido definido (`[09:43] Diego`, detalhado no FDD), mas não o momento em que ele é montado. Depois do resumo final, Bruno perguntou: "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?" (`[09:51] Bruno`).
3. Larissa respondeu: "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito." (`[09:52] Larissa`). A preocupação central é o pedido mudar depois da inserção do evento.
4. Pelo desenho fechado na reunião, há intervalo entre a inserção e o envio. O worker lê a outbox em polling a cada 2 segundos (`[09:09] Diego`, `[09:10] Larissa`). As falhas são retentadas com backoff 1m/5m/30m/2h/12h, em 5 tentativas (`[09:17] Larissa`). Eventos na DLQ podem ser recolocados na outbox por replay manual (`[09:18] Diego`). A reunião não ligou explicitamente esses intervalos a esta decisão; eles são citados aqui apenas como fatos do desenho.
5. Diego já havia descrito a tabela `webhook_dead_letter` com "a payload, motivo da falha e timestamp" (`[09:18] Diego`), o que é coerente com um payload materializado por evento.
6. No código existente, `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) carrega o pedido no início da transação com `include: { items: true }`.
   - Pelo modelo `Order` de `prisma/schema.prisma`, esse objeto já traz os campos do pedido usados no payload (`id`, `orderNumber`, `customerId`, `totalCents`, `status`). A lista de campos do payload está no FDD.
   - Nesse objeto, `status` ainda contém o status anterior (atribuído à variável `from`). O novo status vem de `input.toStatus` (variável `to`) e só é gravado pelo `tx.order.update` executado depois.
   - Esse `update` altera explicitamente apenas `status`. Os campos do pedido usados no payload (`id`, `orderNumber`, `customerId`, `totalCents`) permanecem válidos durante a transação; `updatedAt` é renovado pelo `@updatedAt`.

## Decisão

**ADR-007**: Nós gravamos, em cada linha da outbox, o payload do evento já renderizado no momento da inserção (snapshot), e não apenas o `order_id` para renderizar na hora do envio (`[09:52] Larissa`; fechamento em `[09:52] Bruno`).

A decisão teve a concordância de Diego, "Concordo, snapshot na inserção" (`[09:52] Diego`), e de Bruno, "Beleza, snapshot. Decidido." (`[09:52] Bruno`). O ADR-007 compreende:

1. A renderização ocorre na inserção da linha, dentro da transação de mudança de status (ver [ADR-001](./ADR-001-outbox-no-mysql.md)). Assim, o evento reflete o estado do pedido no instante em que o status mudou (`[09:52] Larissa`).
2. O envio usa o payload gravado. Não há nova renderização a partir do estado do pedido no momento do envio, que é o que distingue esta decisão da alternativa ADR-007-ALT-1 (`[09:51] Bruno`, `[09:52] Larissa`).
3. Se o pedido mudar depois da inserção, o evento já inserido continua refletindo o estado de quando o status mudou (`[09:52] Larissa`).
4. O formato e a lista de campos do payload não são objeto deste ADR. Eles estão no FDD.

**Interpretação a confirmar ("caso esquisito"):** Larissa não descreveu o "caso esquisito" (`[09:52] Larissa`).
- Leitura adotada: com renderização no envio, o payload poderia mostrar um estado do pedido posterior à mudança que gerou o evento. É o contrário do que ela afirma na frase anterior.
- Outra leitura: ela se referia a um cenário específico, como a sequência rápida PAID, PROCESSING, SHIPPED do mesmo pedido, levantada em `[09:12] Larissa`. A reunião não confirma qual das leituras vale.

**Interpretação a confirmar (timestamp do payload):** o payload tem um timestamp ISO 8601 (`[09:43] Diego`), sem indicação do instante a que se refere. Já o header X-Timestamp é explicitamente "o timestamp do envio" (`[09:44] Diego`).
- Leitura adotada: o timestamp do payload é fixado na renderização, ou seja, na inserção, o que é coerente com o snapshot. O instante do envio fica com o X-Timestamp, que não integra o snapshot.
- Outra leitura: o timestamp do payload seria o do envio. Isso exigiria alterar o payload gravado no momento do envio e contrariaria a decisão de `[09:52] Larissa`.

## Alternativas Consideradas

### ADR-007-ALT-1: Guardar só o `order_id` e renderizar o payload na hora do envio

- **O que era:** a linha da outbox guardaria apenas o `order_id`, e o payload seria montado na hora do envio (`[09:51] Bruno`).
- **Quem levantou:** Bruno, em forma de pergunta e sem defender a opção (`[09:51] Bruno`).
- **Por que foi descartada:** o payload refletiria o estado do pedido no envio, e não o do instante da mudança de status. Se o pedido mudasse depois da inserção, haveria o "caso esquisito" (`[09:52] Larissa`). Diego e Bruno acompanharam o snapshot (`[09:52] Diego`, `[09:52] Bruno`).
- **(consequência técnica inferida)** A favor da alternativa, não discutido na reunião: linhas menores na outbox e payload sempre no formato mais recente no envio.
- **(consequência técnica inferida)** `OrderService.delete` permite remover pedidos em PENDING ou CANCELLED. Com esta alternativa, um pedido removido antes do envio deixaria de ter dados a renderizar.

## Consequências

### Positivas

- O evento reflete o estado do pedido no instante em que o status mudou, mesmo que o pedido mude depois (`[09:52] Larissa`).
- Evita o "caso esquisito" apontado no descarte da alternativa ADR-007-ALT-1 (`[09:52] Larissa`).
- **(consequência técnica inferida)** Retentativas, reentregas da garantia at-least-once ([ADR-005](./ADR-005-at-least-once-com-x-event-id.md)) e replays da DLQ transmitem o mesmo conteúdo da primeira tentativa. Isso é coerente com a deduplicação pelo X-Event-Id, gerado quando o evento entra na outbox (`[09:25] Diego`).
- **(consequência técnica inferida)** O conteúdo do evento já inserido não depende de o pedido ainda existir, nem de seu estado, no momento do envio.
- **(consequência técnica inferida)** Montar o snapshot não exige leitura extra em `OrderService.changeStatus`, porque os campos do pedido já estão no objeto carregado no início da transação.

### Negativas e trade-off explícito

- O evento não acompanha mudanças posteriores do pedido. Ele registra o estado do instante da mudança de status (`[09:52] Larissa`). Para detalhes, a reunião já previa que o cliente consulte GET /orders/:id (`[09:43] Diego`).
- **(consequência técnica inferida)** Cada linha da outbox armazena o payload inteiro, e não só o `order_id`, o que aumenta o espaço ocupado pela tabela.
  - O aumento é atenuado pelo payload enxuto, sem items (`[09:43] Diego`, `[09:44] Bruno`), e pelo limite de 64KB por payload (`[09:24] Larissa`).
  - O arquivamento de linhas entregues ("depois de 30 dias ou assim") ficou fora do escopo desta feature (`[09:08] Diego`).
- **(consequência técnica inferida)** Mudanças futuras no formato do payload não alcançam linhas já inseridas. Eventos pendentes, em retentativa ou recolocados na outbox por replay da DLQ (`[09:18] Diego`) saem no formato vigente na inserção.

**Trade-off aceito:** aceitamos guardar na outbox o payload já renderizado, em vez de só o `order_id` (`[09:51] Bruno`), em troca de o evento refletir o estado de quando o status mudou, mesmo que o pedido mude depois (`[09:52] Larissa`).

## Referências

- Transcrição: `[09:08] Diego`, `[09:09] Diego`, `[09:10] Larissa`, `[09:12] Larissa`, `[09:17] Larissa`, `[09:18] Diego`, `[09:24] Larissa`, `[09:25] Diego`, `[09:40] Bruno`, `[09:41] Diego`, `[09:43] Diego`, `[09:44] Diego`, `[09:44] Bruno`, `[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`, `[09:52] Bruno`.
- Código existente: `src/modules/orders/order.service.ts` (classe `OrderService`, métodos `changeStatus` e `delete`) · `prisma/schema.prisma` (modelo `Order`).
- Documentos: [RFC](../RFC.md) (seção de decisões relacionadas) · [FDD](../FDD.md) (formato e lista de campos do payload) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-005](./ADR-005-at-least-once-com-x-event-id.md).
