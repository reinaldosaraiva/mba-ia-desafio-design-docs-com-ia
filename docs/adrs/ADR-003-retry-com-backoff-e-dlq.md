# ADR-003: Retry com backoff e fila de mortos (DLQ)

| Campo | Valor |
|---|---|
| ID | ADR-003 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Larissa (Tech Lead), Diego (Engenheiro Sênior, time de Plataforma) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-002](./ADR-002-worker-separado-com-polling.md) · [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) |

## Status

Aceito. Teto e intervalos foram fechados em `[09:17] Larissa`; a tabela separada foi sugerida por `[09:18] Diego`, com concordância de `[09:18] Bruno`; o endpoint de replay foi proposto em `[09:18] Diego` e anotado por `[09:19] Larissa`. Tudo foi confirmado no resumo de `[09:48] Larissa`. O resumo de `[09:48] Larissa` reconfirmou a decisão, e `[09:49] Diego` respondeu "Tá fechado".

## Contexto

Precisávamos definir o que acontece quando uma entrega de webhook não é concluída. As forças levantadas na reunião foram:

1. **Cliente offline.** O tema foi aberto com a pergunta "Se o cliente tá offline, o que a gente faz?" (`[09:14] Larissa`).
2. **Cliente lento também conta como falha.** Um cliente que não responde em 10 segundos é tratado como falha e marcado para retry (`[09:42] Diego`).
3. **Evento pendurado para sempre.** Sem teto de tentativas, existe "o problema de evento ficar pendurado pra sempre se o cliente sumiu" (`[09:15] Diego`).
4. **Indisponibilidades reais longas.** "Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada" (`[09:16] Diego`). Com um teto curto, "a gente retentaria três vezes em 30 minutos e mataria" (`[09:16] Diego`).
5. **Janela aceitável para o produto.** "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável." (`[09:17] Marcos`).
6. **Leitura da outbox e evidência.** O objetivo é deixar "mais limpa a leitura da outbox principal" e manter os eventos esgotados "como evidence pra debug e reprocessamento" (`[09:18] Diego`).
7. **Base já decidida.** A outbox no MySQL ([ADR-001](./ADR-001-outbox-no-mysql.md)) já prevê o estado pendente (`[09:08] Diego`). As entregas são feitas pelo worker descrito no [ADR-002](./ADR-002-worker-separado-com-polling.md).

## Decisão

**ADR-003.** Tratamos falhas de entrega com novas tentativas em intervalos crescentes, um teto fixo de tentativas e uma fila de mortos em tabela própria:

1. **Teto.** Fazemos no máximo 5 novas tentativas após o envio inicial. Esse é o teto de "5 tentativas" da reunião; a contagem é explicada em "Interpretação a confirmar", abaixo (`[09:15] Diego`, `[09:17] Larissa`).
2. **Intervalos.** O intervalo entre uma falha e a nova tentativa seguinte cresce nesta ordem: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, ou seja, 1m/5m/30m/2h/12h (`[09:17] Diego`, `[09:17] Larissa`).
3. **Fila de mortos.** Esgotado o teto, consideramos a falha permanente e movemos o evento para a tabela separada `webhook_dead_letter`. Essa tabela guarda o payload, o motivo da falha e o timestamp (`[09:15] Diego`, `[09:18] Diego`).
4. **Reprocessamento.** O reprocessamento da DLQ é apenas manual, pelo endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente (`[09:18] Diego`, `[09:19] Larissa`, `[09:35] Diego`). O acesso exige role ADMIN e registra quem fez o replay, para auditoria (`[09:36] Sofia`, `[09:36] Larissa`). O detalhamento fica no FDD.
5. **Convenções do schema.** A tabela `webhook_dead_letter` segue as convenções de `prisma/schema.prisma`:
   - model mapeado com `@@map` para um nome em snake_case, como `order_status_history`;
   - chave primária `String @id @default(uuid()) @db.Char(36)`, como em `User`, `Order` e `OrderStatusHistory` ("Tudo é uuid", resposta de `[09:51] Larissa` à pergunta de `[09:51] Diego` sobre o id da outbox; aplicar o UUID às demais tabelas novas segue o padrão do código e é interpretação a confirmar);
   - datas em `DateTime` com `@default(now())`, como `createdAt` e `changedAt`.

   Os campos da tabela ficam no FDD.

**Precisão sobre "backoff exponencial".** A transcrição chama o mecanismo de "backoff exponencial" (`[09:15] Diego`, `[09:48] Larissa`). Porém, a sequência 1/5/30/120/720 minutos não tem razão constante entre intervalos consecutivos. O que foi decidido é a tabela de intervalos 1m/5m/30m/2h/12h, e não uma fórmula. Neste ADR, "backoff" designa essa tabela fixa.

**Interpretação a confirmar (número de envios).** A transcrição fala em "5 tentativas" (`[09:15] Diego`, `[09:17] Larissa`, `[09:48] Larissa`) e, ao mesmo tempo, lista cinco intervalos (`[09:17] Diego`). Adotamos a leitura "1 envio inicial + 5 novas tentativas (6 envios no total)", por dois motivos:

1. Cada intervalo antecede uma nova tentativa. Portanto, cinco intervalos correspondem a cinco novas tentativas depois do envio inicial (`[09:17] Diego`).
2. Diego fala em "quase 15 horas entre primeira falha e última tentativa" (`[09:17] Diego`). A soma dos intervalos (1 minuto + 5 minutos + 30 minutos + 2 horas + 12 horas) dá 14 horas e 36 minutos, e esse total só é alcançado se o intervalo de 12 horas for usado.

A outra leitura inclui o envio inicial nas "5 tentativas", leitura que o "total 5 tentativas" de `[09:48] Larissa` favorece. Nesse caso, seriam 5 envios no total e apenas quatro intervalos. O intervalo de 12 horas ficaria sem uso, o que contraria as "quase 15 horas" de `[09:17] Diego`. Este ponto deve ser confirmado na revisão do FDD que Larissa disse que vai marcar com Bruno e Diego (`[09:50] Larissa`).

## Alternativas Consideradas

### ADR-003-ALT-1: Retry indefinido com backoff

**O que era:** continuar retentando com intervalos crescentes, sem teto; o evento nunca seria dado como falha permanente.

**Quem levantou:** Diego, ao citar que "algumas pessoas defendem" essa abordagem (`[09:15] Diego`).

**Por que foi descartada:** "isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu" (`[09:15] Diego`). A alternativa foi preterida em favor do teto de 5 tentativas (`[09:17] Larissa`).

### ADR-003-ALT-2: 3 tentativas (mais agressivo)

**O que era:** um teto menor de tentativas, proposto com a pergunta "3 não é melhor? Mais agressivo." (`[09:16] Bruno`).

**Por que foi descartada:** Diego respondeu que "3 é pouco": em uma indisponibilidade de manhã, "a gente retentaria três vezes em 30 minutos e mataria". Ele lembrou também a indisponibilidade de duas horas em manutenção planejada (`[09:16] Diego`). Larissa fechou com "Cinco fica bom." (`[09:16] Larissa`).

**Trade-off:** a falha permanente chegaria mais cedo, mas ao custo de mandar para a DLQ eventos de clientes que voltariam depois de uma manutenção como essa.

### ADR-003-ALT-3: Marcar como "failed" na própria outbox

**O que era:** manter o evento esgotado na própria outbox, apenas marcado como falho, em vez de usar uma tabela separada. A opção veio da pergunta "Faz numa tabela separada ou marca como 'failed' na própria outbox?" (`[09:17] Larissa`).

**Por que foi preterida:** Diego escolheu a tabela separada `webhook_dead_letter`, que deixa "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento" (`[09:18] Diego`). Bruno concordou: "Faz sentido." (`[09:18] Bruno`).

**Trade-off:** a alternativa evitaria uma tabela a mais, mas misturaria eventos esgotados com eventos ativos na leitura da outbox.

## Consequências

### Positivas

- Nenhum evento fica pendurado para sempre, porque o teto encerra as tentativas automáticas (`[09:15] Diego`).
- A janela de quase 15 horas cobre indisponibilidades como a de duas horas em manutenção planejada (`[09:16] Diego`, `[09:17] Diego`).
- A leitura da outbox principal fica mais limpa, e a `webhook_dead_letter` serve de evidência para debug e reprocessamento (`[09:18] Diego`).
- O evento esgotado não se perde: ele pode voltar à outbox como pendente pelo replay (`[09:18] Diego`).

### Negativas e trade-off explícito

- Um cliente indisponível por mais tempo que a janela fica sem o evento até que alguém com role ADMIN faça o replay manual. Essa janela foi aceita pelo produto (`[09:17] Marcos`, `[09:18] Diego`, `[09:36] Sofia`).
- O schema ganha uma tabela a mais, e o fluxo do evento passa a envolver duas tabelas: a outbox e a `webhook_dead_letter` (`[09:18] Diego`).
- (consequência técnica inferida) Como o timeout de 10 segundos conta como falha e vai para retry (`[09:42] Diego`), uma nova tentativa pode repetir uma entrega que o cliente já tinha recebido. Esse caso é coberto pela garantia at-least-once do [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) (`[09:24] Diego`).

**Trade-off aceito:** aceitamos que o evento de um cliente indisponível por mais que a janela de quase 15 horas saia do fluxo automático e dependa de replay manual pelo endpoint admin (`[09:17] Diego`, `[09:17] Marcos`, `[09:18] Diego`). Em troca, nenhum evento fica pendurado para sempre (`[09:15] Diego`) e a leitura da outbox principal fica mais limpa (`[09:18] Diego`).

## Referências

- **Transcrição:** `[09:08] Diego`, `[09:14] Larissa`, `[09:15] Diego`, `[09:16] Bruno`, `[09:16] Diego`, `[09:16] Larissa`, `[09:17] Diego`, `[09:17] Marcos`, `[09:17] Larissa`, `[09:18] Diego`, `[09:18] Bruno`, `[09:19] Larissa`, `[09:24] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa`, `[09:42] Diego`, `[09:48] Larissa`, `[09:49] Diego`, `[09:50] Larissa`, `[09:51] Larissa`.
- **Código existente:** `prisma/schema.prisma`:
  - `@@map` em snake_case, como `order_status_history` no model `OrderStatusHistory`;
  - `id String @id @default(uuid()) @db.Char(36)` nos models `User`, `Customer`, `Product`, `Order`, `OrderItem` e `OrderStatusHistory` (exceção: `OrderNumberSequence`, com `id Int @id @default(1)`);
  - datas `createdAt DateTime @default(now())`, `updatedAt DateTime @updatedAt` e `changedAt DateTime @default(now())`.
- **Documentos:** [RFC](../RFC.md) (seção de decisões relacionadas) · [FDD](../FDD.md) (seções sobre retry, dead letter e endpoint de replay, incluindo role ADMIN e auditoria) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-002](./ADR-002-worker-separado-com-polling.md) · [ADR-005](./ADR-005-at-least-once-com-x-event-id.md).
