# RFC — Sistema de Webhooks de Notificação de Pedidos

## 1. Metadados

| Campo | Valor |
|---|---|
| Título | Sistema de Webhooks de Notificação de Pedidos |
| Autor (RFC-META-01) | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | 2026-11-25 |
| Revisores | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| Documentos relacionados | [PRD](PRD.md) · [FDD](FDD.md) · ADR-001 a ADR-008 (seção 8) |

RFC-META-01: Larissa conduziu a reunião, registrou o resumo das decisões e disse que abriria o documento de design da feature (`[09:48] Larissa`, `[09:50] Larissa`).

## 2. Resumo executivo (TL;DR)

- Outbox no MySQL existente, gravada na mesma transação da mudança de status (RFC-DEC-01, [ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
- Worker em processo separado da API, com polling de 2 segundos (RFC-DEC-02, [ADR-002](adrs/ADR-002-worker-separado-com-polling.md)).
- 5 tentativas nos intervalos 1m/5m/30m/2h/12h e, depois delas, fila de mortos com reprocessamento manual por administradores (RFC-DEC-03, [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).
- Assinatura HMAC-SHA256 com secret por endpoint e rotação. Na rotação, a secret antiga continua válida por 24 horas em paralelo com a nova (RFC-DEC-04, [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- Entrega at-least-once, com identificador de evento para que o cliente faça a deduplicação (RFC-DEC-05, [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
- Módulo igual aos demais, com reuso dos padrões existentes do projeto (RFC-DEC-06, [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).
- Evento gravado como snapshot no momento da inserção, apenas para os status assinados por algum webhook do cliente (RFC-DEC-07, RFC-DEC-08, [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md), [ADR-008](adrs/ADR-008-filtro-de-eventos-na-insercao.md)).

## 3. Contexto e problema

- **RFC-CTX-01.** Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para ser notificados em tempo real quando o status dos seus pedidos muda. Hoje eles consultam a listagem de pedidos de tempos em tempos, o que deixa a integração lenta e cara (`[09:00] Marcos`).
- **RFC-CTX-02.** Para esses clientes, qualquer tempo abaixo de 10 segundos já é "tempo real". O importante é que a notificação não fique pendurada e que eles não precisem atualizar manualmente (`[09:02] Marcos`).
- **RFC-CTX-03.** O alcance é só notificação de saída: os clientes recebem e não enviam nada para a plataforma (`[09:02] Marcos`, `[09:03] Sofia`).
- **RFC-CTX-04.** Hoje não existe módulo de notificação, evento, fila ou webhook. Os módulos montados em `src/app.ts` e `src/routes/index.ts` são auth, users, customers, products e orders.
- **RFC-CTX-05.** A transação de mudança de status já é pesada: ela atualiza o pedido, grava o histórico de status e decrementa o estoque dos produtos do pedido (`[09:04] Bruno`; no código, o débito ocorre na transição `PENDING` para `PAID` e há reposição no cancelamento a partir de `PAID` ou `PROCESSING` — `src/modules/orders/order.status.ts`).
- **RFC-CTX-06.** Há risco comercial: a Atlas sugeriu que pode migrar para o concorrente se a feature não for entregue até o fim do trimestre (`[09:00] Marcos`).
- **RFC-CTX-07.** Marcos informou que a Atlas quer a entrega para o fim de novembro (`[09:45] Marcos`). *Interpretação a confirmar:* a reunião não esclarece se essa data substitui o "fim do trimestre" de `[09:00] Marcos` ou se são marcos distintos.

## 4. Proposta técnica

### 4.1 Visão geral

Propomos notificar os clientes por webhooks de saída, com a entrega desacoplada da mudança de status. Quando o status de um pedido muda, a mesma transação que altera o pedido registra o evento numa outbox no MySQL existente, já com o estado do pedido naquele instante e só se algum webhook do cliente assina o novo status (RFC-DEC-01, RFC-DEC-07, RFC-DEC-08): se a transação sofre rollback, o evento é descartado junto (`[09:06] Diego`). Um worker em processo separado lê a outbox em polling e faz as chamadas HTTP (RFC-DEC-02), de modo que um cliente lento ou fora do ar não trava nem desfaz mudanças de status (`[09:04] Bruno`). O MySQL existente foi preferido a infraestrutura nova, por ser um time pequeno (`[09:07] Diego`). As falhas temporárias são absorvidas por novas tentativas, e as persistentes ficam guardadas para reprocessamento (RFC-DEC-03). Cada entrega é assinada (RFC-DEC-04), e a garantia é de entrega ao menos uma vez, com deduplicação feita pelo cliente (RFC-DEC-05). Tudo entra como um módulo igual aos demais (RFC-DEC-06). O detalhamento de implementação está no [FDD](FDD.md).

Elementos da solução:

1. Registro do evento em uma tabela de outbox, na mesma transação da mudança de status (RFC-DEC-01).
2. Worker separado, que lê a outbox em polling (RFC-DEC-02).
3. Novas tentativas com intervalos crescentes e fila de mortos para falha persistente (RFC-DEC-03).
4. Assinatura das entregas, com secret por cadastro e rotação (RFC-DEC-04).
5. Entrega com garantia de ao menos uma vez e identificador de evento para o cliente deduplicar (RFC-DEC-05).
6. API de gerenciamento com cadastro, edição, remoção, listagem, filtro por status, histórico de entregas e rotação de secret (RFC-DEC-04, RFC-DEC-06, RFC-DEC-08; requisitos no [PRD](PRD.md)).
7. Reprocessamento manual da fila de mortos, restrito a administradores (RFC-DEC-03).

### 4.2 Decisões que sustentam a proposta

| ID | Decisão | Fechada em | ADR |
|---|---|---|---|
| RFC-DEC-01 | Outbox no MySQL existente, gravada na mesma transação da mudança de status. Uma falha na gravação desfaz a mudança (`[09:40] Bruno`, `[09:41] Diego`). *Interpretação a confirmar (alcance da emissão e convívio com o filtro):* registrada no ADR-001. | `[09:08] Larissa` | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| RFC-DEC-02 | Worker em processo separado, com entry-point próprio, lendo a outbox em polling de 2 segundos. O worker é único, e a ordenação é garantida só por pedido, como limitação conhecida. | `[09:10] Larissa` (polling de 2 segundos); `[09:11] Diego` e `[09:11] Larissa` (processo separado e entry-point); `[09:13] Larissa` (worker único) | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) |
| RFC-DEC-03 | Teto de 5 tentativas, com intervalos 1m/5m/30m/2h/12h. Esgotado o teto, o evento vai para uma fila de mortos em tabela separada. O reprocessamento é só manual, restrito à role ADMIN, e registra quem o fez. *Interpretação a confirmar:* RFC-QA-06. | `[09:17] Larissa` (teto e intervalos); `[09:18] Diego` (tabela separada e replay); `[09:36] Sofia` e `[09:36] Larissa` (role ADMIN e registro de quem fez) | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| RFC-DEC-04 | HMAC-SHA256 sobre o corpo do request, com secret única por endpoint. A plataforma gera a secret no cadastro, e o cliente pode rotacioná-la pela API. Após a rotação, a secret antiga vale por 24 horas em paralelo com a nova. | `[09:22] Sofia`; `[09:31] Marcos` (secret gerada pela plataforma) | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| RFC-DEC-05 | Entrega at-least-once, com identificador único por evento, atribuído na entrada na outbox, para deduplicação no cliente. O comportamento é documentado no portal do desenvolvedor. | `[09:25] Diego` (UUID gerado na entrada na outbox); `[09:26] Larissa` (decisão); `[09:26] Marcos` (portal do desenvolvedor) | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| RFC-DEC-06 | Módulo igual aos demais, reusando os padrões existentes: erros, logger Pino, middleware de erro centralizado, schemas Zod, códigos de erro e controle de role. O worker usa uma instância própria do cliente Prisma no mesmo banco. | `[09:30] Larissa`; `[09:30] Bruno` (instância própria de `PrismaClient`); `[09:36] Larissa` (reuso do `requireRole`) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| RFC-DEC-07 | Payload renderizado na inserção (snapshot), refletindo o pedido no instante da mudança de status. | `[09:52] Larissa` (preferência); `[09:52] Bruno` (fechamento) | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| RFC-DEC-08 | Cada cadastro guarda os status que quer ouvir, e o filtro é aplicado na inserção. Se nenhum webhook do cliente assina o status de destino, nada é gravado. *Interpretação a confirmar:* RFC-QA-08. | `[09:33] Marcos` (lista de status por cadastro); `[09:34] Bruno` (filtro aplicado na inserção) | [ADR-008](adrs/ADR-008-filtro-de-eventos-na-insercao.md) |

## 5. Alternativas consideradas

Todas foram discutidas e descartadas na reunião.

| ID | Alternativa | Trade-off que levou ao descarte | Origem | ADR |
|---|---|---|---|---|
| RFC-ALT-01 | Disparo síncrono no service de pedidos | Dispensaria outbox e worker (vantagem inferida), mas poria chamada HTTP numa transação já pesada: cliente lento travaria a mudança de status de outros pedidos; cliente fora do ar não justifica rollback (`[09:04] Bruno`). | `[09:03] Larissa` | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| RFC-ALT-02 | Redis Streams (ou similar) como fila | Daria fila dedicada (vantagem inferida), mas Redis Cluster é overengineering para time pequeno; a outbox no MySQL existente resolve (`[09:07] Diego`). | `[09:07] Larissa`, prevendo "subir mais infra" | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| RFC-ALT-03 | Trigger do banco acionando o worker | No MySQL, trigger só executa SQL; avisar o worker exigiria improviso. Polling de 2 segundos já atende o requisito "abaixo de 10 segundos" (`[09:09] Diego`). | `[09:09] Bruno`, para ser "mais reativo" | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) |
| RFC-ALT-04 | Novas tentativas sem limite | Não desistiria do evento (vantagem inferida), mas o deixaria "pendurado pra sempre se o cliente sumiu" (`[09:15] Diego`); prevaleceu o teto de 5 (`[09:17] Larissa`). | Citada por Diego | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| RFC-ALT-05 | Limite de 3 tentativas | Indisponibilidade de manhã esgotaria as tentativas em 30 minutos; já houve cliente indisponível por duas horas em manutenção planejada (`[09:16] Diego`). | `[09:16] Bruno`, como "mais agressivo" | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| RFC-ALT-06 | Entrega exatamente uma vez | "Exigiria coordenação dos dois lados e fica muito mais complexo" (`[09:25] Diego`). | Citada por Diego ante a objeção de que at-least-once "joga responsabilidade pro cliente" (`[09:25] Sofia`) | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| RFC-ALT-07 | Secret global da plataforma | Sem vantagem defendida registrada; pesou o alcance de um vazamento: "se vaza uma, vaza tudo" (`[09:21] Sofia`). | Mencionada e descartada por Sofia | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| RFC-ALT-08 | Id auto incremental na outbox | Dispensaria gerar UUID na aplicação (vantagem inferida); Larissa escolheu UUID, padrão do resto do projeto ("Tudo é uuid", `[09:51] Larissa`). | `[09:51] Diego`, ao perguntar | [FDD](FDD.md), modelo de dados |

## 6. Questões em aberto

### 6.1 Levantadas na reunião e não decididas

| ID | Questão | Onde ficou | Encaminhamento |
|---|---|---|---|
| RFC-QA-01 | Limite de envio por cliente: 50 pedidos mudando de status num minuto geram 50 chamadas? | Fora do escopo (`[09:39] Diego`) | Observar e decidir depois (`[09:39] Larissa`) |
| RFC-QA-02 | Escala para múltiplos workers em paralelo | "Problema do futuro" (`[09:13] Diego`) | Documentar o worker único como limitação conhecida (`[09:13] Larissa`) |
| RFC-QA-03 | Arquivamento das linhas já entregues, "depois de 30 dias ou assim" | Fora do escopo (`[09:08] Diego`) | Nenhuma ação nesta feature |
| RFC-QA-04 | E-mail ao cliente cujo webhook está com problema | Fora de escopo desta fase (`[09:37] Larissa`) | Talvez na próxima fase, após medir o impacto ("futuro", `[09:38] Marcos`) |
| RFC-QA-05 | Endurecimento da autorização do cadastro de webhooks | Por enquanto, qualquer role autenticada (`[09:37] Sofia`), sob a premissa de `[09:32] Marcos`, que não vale no código (ver RFC-RISCO-07) | "Mais pra frente a gente pode endurecer", sem prazo |

### 6.2 Interpretações adotadas a confirmar na revisão

- **RFC-QA-06 — "5 tentativas".** Adotada: envio inicial mais 5 novas tentativas, uma por intervalo (`[09:17] Diego`). Outra: 5 envios no total (`[09:48] Larissa`), deixando sem uso o intervalo de 12 horas ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).
- **RFC-QA-07 — Secret que assina nas 24 horas da rotação** (`[09:21] Sofia`). Leituras: a plataforma segue com a antiga até o fim do prazo, ou passa à nova e o cliente aceita as duas. Nenhuma leitura foi adotada; proposta de trabalho e decisão ficam no [FDD](FDD.md), e encaminhar o ponto à revisão de Sofia é proposta deste documento ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- **RFC-QA-08 — Status comparado pelo filtro.** Adotada: o de destino, pelo exemplo "quando vira SHIPPED e DELIVERED" (`[09:33] Marcos`). Outra: "aquele status" (`[09:34] Bruno`) admitiria também o de origem ([ADR-008](adrs/ADR-008-filtro-de-eventos-na-insercao.md)).
- **RFC-QA-09 — Identificador do evento no replay.** Adotada: é mantido, por ser "único por evento" (`[09:25] Diego`). Outra: ele nasce na entrada da outbox e o replay "recoloca na outbox como pendente" (`[09:18] Diego`), logo geraria um novo ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).

### 6.3 Lacunas identificadas ao escrever este documento

Pontos **não** discutidos na reunião. Os trechos citados em cada item são vizinhos do assunto, não uma decisão sobre ele. Seguem para a seção de lacunas do [FDD](FDD.md):

- **RFC-GAP-01** — Destino de evento pendente quando o cadastro é removido ou desativado (`[09:21] Bruno`, `[09:33] Bruno`).
- **RFC-GAP-02** — Qual resposta do cliente conta como sucesso; só o timeout de 10 segundos, como falha, foi definido (`[09:42] Diego`).
- **RFC-GAP-03** — Destino dos eventos em processamento quando o worker cai (`[09:08] Diego`).
- **RFC-GAP-04** — Se a criação de pedido também gera evento; só a mudança de status foi tratada (`[09:40] Bruno`).
- **RFC-GAP-05** — Formato da assinatura e armazenamento da secret em repouso (`[09:21] Sofia`).
- **RFC-GAP-06** — Se a URL sem `https` gera o erro de validação comum ou código próprio do módulo (`[09:23] Sofia`, `[09:28] Bruno`).
- **RFC-GAP-07** — Validação do destino das entregas: a URL é informada pelo cliente e o worker faz a requisição, o que expõe a plataforma a pedidos para endereços internos (`[09:31] Marcos`, `[09:06] Diego`).

## 7. Impacto e riscos

### 7.1 Impacto

- **RFC-IMP-01** — Tabelas novas no MySQL existente, sem infraestrutura nova: outbox (`[09:06] Diego`), fila de mortos (`[09:18] Diego`), cadastro de webhooks (`[09:21] Sofia`).
- **RFC-IMP-02** — Worker em processo novo, fora da API; entry-point `src/worker.ts` (novo) (`[09:11] Diego`, `[09:11] Larissa`).
- **RFC-IMP-03** — Service de pedidos registra o evento na transação da mudança de status; falha desfaz a mudança (`[09:40] Bruno`).
- **RFC-IMP-04** — Um módulo novo, no padrão dos existentes (`[09:27] Bruno`).
- **RFC-IMP-05** — Revisão de segurança pré-deploy, com ao menos dois dias úteis reservados (`[09:46] Sofia`).
- **RFC-IMP-06** — Documentação no portal do desenvolvedor, destacando at-least-once (`[09:26] Marcos`, `[09:40] Marcos`).
- **RFC-IMP-07** — Estimativa: três sprints, revisão de Sofia incluída (`[09:46] Larissa`, `[09:47] Larissa`).

### 7.2 Riscos

Probabilidade e impacto são avaliação deste documento; a reunião não os classificou.

| ID | Risco | Probabilidade | Impacto | Mitigação | Origem |
|---|---|---|---|---|---|
| RFC-RISCO-01 | Cliente indisponível, até por horas | Alta | Média | Retentativas, fila de mortos e replay manual | `[09:16] Diego` |
| RFC-RISCO-02 | Vazamento de secret; precedente em log de aplicação de cliente | Média | Alta | Secret por endpoint e rotação | `[09:22] Diego` |
| RFC-RISCO-03 | Cliente sem deduplicação processa evento duplicado | Média | Média | Identificador único por evento; destaque no portal | `[09:24] Diego` |
| RFC-RISCO-04 | Falha no registro bloqueia a mudança de status | Baixa | Alta | Rollback, como decidido; sem chamada HTTP na transação | `[09:40] Bruno` |
| RFC-RISCO-05 | Perda da Atlas para o concorrente sem entrega no trimestre | Média | Alta | Confirmação do prazo com a Atlas, pendente (RFC-PROX-03) | `[09:00] Marcos` |
| RFC-RISCO-06 | (inferência deste documento) Três sprints excederem o fim de novembro pedido pela Atlas | Média | Alta | Confirmação do prazo com a Atlas (RFC-PROX-03) | `[09:45] Marcos` |
| RFC-RISCO-07 | (inferência deste documento a partir do código) O registro de usuário é público e não há vínculo entre usuário e cliente; com o acesso amplo aceito em `[09:37] Sofia`, qualquer pessoa pode cadastrar um webhook e passar a receber os eventos de pedidos de qualquer cliente. A leitura desses pedidos já é possível hoje pela API; a feature acrescenta o envio automático e o poder de alterar cadastros alheios | Alta | Alta | Bloqueante de deploy: reexaminar o acesso amplo na revisão de segurança (RFC-PROX-02); detalhe no FDD (FDD-RISCO-05) | `src/modules/auth/auth.routes.ts` |

## 8. Decisões relacionadas

Todos com status Aceito.

| ADR | Decisão |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Outbox no MySQL, na transação da mudança de status |
| [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) | Worker único e separado, polling de 2 s |
| [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) | 5 tentativas (1m/5m/30m/2h/12h), fila de mortos, replay manual |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com 24 h em paralelo |
| [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com identificador de evento |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões do projeto |
| [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload na inserção |
| [ADR-008](adrs/ADR-008-filtro-de-eventos-na-insercao.md) | Filtro de eventos na inserção |

### 8.1 Próximos passos

- **RFC-PROX-01** — Revisão do documento de design com Bruno e Diego, antes de codar (`[09:50] Larissa`). Proposta deste documento: levar a essa sessão RFC-QA-06, RFC-QA-08 e RFC-QA-09.
- **RFC-PROX-02** — Agendar a revisão de segurança de Sofia, pré-deploy (`[09:46] Sofia`, `[09:49] Sofia`). Proposta deste documento: levar a ela RFC-QA-07 e RFC-GAP-05.
- **RFC-PROX-03** — Marcos ficou de confirmar o prazo com a Atlas; confirmação pendente (`[09:47] Marcos`).
