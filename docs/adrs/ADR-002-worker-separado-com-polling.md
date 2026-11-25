# ADR-002: Worker em processo separado com polling de 2 segundos

| Campo | Valor |
|---|---|
| ID | ADR-002 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Larissa (Tech Lead) e Diego (Engenheiro Sênior, time de Plataforma) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) · [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) |

## Status

Aceito. A decisão foi fechada na reunião nos seguintes momentos:

- `[09:10] Larissa`: polling de 2 segundos.
- `[09:11] Diego` e `[09:11] Larissa`: processo separado e entry-point próprio.
- `[09:12] Diego` e `[09:13] Larissa`: worker único e limitação de ordenação.

A decisão foi repetida no resumo final em `[09:48] Larissa` ("Worker separado em polling de 2 segundos").

## Contexto

1. **Latência percebida.** Para os clientes B2B, "qualquer coisa abaixo de 10 segundos já é 'tempo real'". O importante é que a notificação não fique pendurada nem exija atualização manual (`[09:02] Marcos`).
2. **Origem dos eventos.** Um worker separado lê a tabela de outbox e dispara as chamadas HTTP (`[09:06] Diego`). A outbox em MySQL foi decidida em `[09:08] Larissa` e está registrada em [ADR-001](./ADR-001-outbox-no-mysql.md). Este ADR trata apenas de como essa tabela é lida.
3. **Sem listener nativo no MySQL.** O MySQL não tem listener nativo como o NOTIFY/LISTEN do Postgres. A trigger existe, mas só executa SQL e não notifica processo externo (`[09:09] Diego`).
4. **Índices e lote.** Bruno perguntou se o worker ficaria lento com o acúmulo de eventos (`[09:07] Bruno`). A resposta foi que a outbox tem índice no campo de status (pendente, processando, falhou, entregue) e em `created_at`, e que o worker lê só os pendentes, em lote pequeno (`[09:08] Diego`).
5. **Mesma base e mesma stack.** O worker precisa conectar no mesmo banco (`[09:11] Bruno`), com a mesma stack, mas sem ser o mesmo processo da API (`[09:11] Diego`). No código, a URL de conexão vem de `env("DATABASE_URL")` em `prisma/schema.prisma`; `src/config/env.ts` apenas a valida em `envSchema`, e o cliente é criado por `createPrismaClient` em `src/config/database.ts`.
6. **Ponto de partida no código.** A API tem como entry-point `src/server.ts`. Ele é executado em desenvolvimento pelo script `dev` e, após o build, pelo script `start` de `package.json`.

## Decisão

A decisão ADR-002 compreende:

1. **Polling de 2 segundos.** Nós entregamos os eventos por um worker que lê a outbox por polling em loop. A cada 2 segundos, ele busca os eventos pendentes mais antigos, em lote pequeno, processa e marca (`[09:08] Diego`, `[09:09] Diego`). A decisão foi registrada em `[09:10] Larissa`: "Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos." O intervalo foi validado em `[09:10] Marcos`.

   > **Interpretação a confirmar:** a frase de `[09:10] Larissa` combina "mínima" com "pior caso". Há uma leitura alternativa. Com coleta a cada 2 segundos (`[09:09] Diego`), a espera até o worker pegar o evento varia de 0 a 2 segundos, conforme o instante do commit em relação ao ciclo de polling. Nessa leitura, 2 segundos é o limite superior dessa espera, e não um piso.

2. **Processo separado.** Nós executamos o worker em processo separado, nunca dentro da mesma instância da API (`[09:11] Diego`). Ele tem entry-point próprio, `src/worker.ts` (novo), iniciado pelo script `npm run worker` (novo) (`[09:11] Larissa`; reforçado em `[09:28] Bruno`).
3. **Modelo de inicialização e encerramento.** Nós usamos o padrão de `src/server.ts` como modelo do novo entry-point (`[09:11] Larissa`). Esse padrão tem:
   - uma função assíncrona `bootstrap` com tratamento de falha (`bootstrap_failed`, `process.exit(1)`);
   - uma função `shutdown` acionada por `SIGINT` e `SIGTERM`, que chama `prisma.$disconnect()` antes de encerrar.
4. **Instância própria de `PrismaClient`.** Nós conectamos o worker ao mesmo banco, com a mesma `DATABASE_URL`, em uma instância própria de `PrismaClient`. O motivo é que o `PrismaClient` é por processo e o worker é outro processo Node (`[09:30] Bruno`). (consequência técnica inferida a partir do código) Pela convenção do código, a instância vem de `src/config/database.ts`: o singleton `prisma` é criado por `createPrismaClient()` no carregamento do módulo. Assim, cada processo que importa o arquivo tem a sua própria instância. A `DATABASE_URL` continua validada por `envSchema` em `src/config/env.ts`.
5. **Script e build.** Nós registramos o script em `package.json` tomando como referência os scripts `dev` (`tsx watch --env-file=.env src/server.ts`) e `start` (`node --env-file=.env dist/server.js`), que carregam a configuração via `--env-file=.env`. O `include` de `tsconfig.build.json` (`src/**/*.ts`) já abrange o novo entry-point no build. A linha de comando exata fica para o detalhamento de implementação no [FDD](../FDD.md).
6. **Worker único.** Nós operamos com um único worker, que processa na ordem de `created_at` da outbox. Isso dá ordenação implícita por `order_id` (`[09:12] Diego`). Trata-se de limitação conhecida: não há garantia de ordenação global, só por `order_id` e enquanto houver um único worker (`[09:13] Larissa`).

Retentativas, backoff e DLQ estão em [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md). O reuso dos demais padrões do projeto está em [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md).

## Alternativas Consideradas

### ADR-002-ALT-1: Worker dentro da mesma instância da API

- **O que era:** executar o loop de polling dentro do processo da API, junto com o servidor iniciado em `src/server.ts`.
- **Quem levantou:** Diego citou a opção e a descartou na mesma fala (`[09:11] Diego`).
- **Motivo do descarte:** o worker ficaria preso ao ciclo de vida da API ("se a API reinicia, perde o worker").
- **Observação:** a reunião não registrou vantagem defendida para esta opção.
- **Status:** descartada.

### ADR-002-ALT-2: Trigger do banco para o worker reagir imediatamente

- **O que era:** usar trigger do banco para tornar a leitura "mais reativa".
- **Quem levantou:** Bruno (`[09:09] Bruno`).
- **Motivo do descarte (`[09:09] Diego`):** no MySQL, a trigger só executa SQL e não notifica processo externo. Avisar o worker exigiria improvisar algo como escrever em arquivo ou bater num endpoint, o que "fica esquisito".
- **Trade-off:** a reatividade maior custaria um mecanismo improvisado, enquanto o polling de 2 segundos já atende o requisito de "abaixo de 10 segundos".
- **Status:** descartada.

### ADR-002-ALT-3: Múltiplos workers em paralelo

- **O que era:** escalar a entrega com vários workers em paralelo.
- **Quem levantou:** Diego, ao discutir ordenação (`[09:12] Diego`). Bruno retomou o tema em `[09:13] Bruno` ("E se algum dia a gente quiser escalar?").
- **Motivo de não adotar nesta fase:** com múltiplos workers em paralelo, perde-se a garantia de ordenação por `order_id` (`[09:12] Diego`).
- **Caminhos citados para o futuro:** particionar por `order_id` ou usar lock pessimista. Foram tratados como "problema do futuro, não agora" (`[09:13] Diego`) e não constituem requisito.
- **Registro:** o uso de um único worker fica documentado como limitação conhecida (`[09:13] Larissa`).
- **Status:** adiada.

## Consequências

### Positivas

- O intervalo de 2 segundos atende o requisito de "abaixo de 10 segundos" (`[09:09] Diego`, `[09:10] Marcos`).
- Um reinício da API não interrompe o worker, que tem ciclo de vida próprio (`[09:11] Diego`).
- O worker não introduz banco nem stack novos (`[09:11] Diego`). Ele reaproveita `src/config/env.ts` e `src/config/database.ts`.
- A ordenação por `order_id` sai sem coordenação adicional enquanto houver um único worker (`[09:12] Diego`). Isso cobre o que os clientes pediram, já que eles nunca pediram ordenação global (`[09:14] Marcos`).

### Negativas e trade-off explícito

- Cada evento aguarda a coleta pelo ciclo de polling, o que foi aceito explicitamente (`[09:10] Larissa`). Vale a ressalva da nota de interpretação na seção Decisão.
- Não há escala horizontal nesta fase: passar a vários workers quebra a ordenação por `order_id` (`[09:12] Diego`, `[09:13] Larissa`).
- Não há garantia de ordenação global (`[09:13] Larissa`).
- Há mais um processo para iniciar e encerrar, com entry-point, script e instância de `PrismaClient` próprios (`[09:11] Larissa`, `[09:30] Bruno`).

**Trade-off aceito:** aceitamos a espera do ciclo de polling de 2 segundos e a ordenação garantida apenas por `order_id` com um único worker, em troca de atender o requisito de "abaixo de 10 segundos" sem improvisar notificação a partir do MySQL e com a entrega independente de reinícios da API (`[09:09] Diego`, `[09:11] Diego`, `[09:13] Larissa`).

## Referências

- **Transcrição:** `[09:02] Marcos`, `[09:06] Diego`, `[09:07] Bruno`, `[09:08] Diego`, `[09:08] Larissa`, `[09:09] Bruno`, `[09:09] Diego`, `[09:10] Larissa`, `[09:10] Marcos`, `[09:11] Bruno`, `[09:11] Diego`, `[09:11] Larissa`, `[09:12] Diego`, `[09:13] Bruno`, `[09:13] Diego`, `[09:13] Larissa`, `[09:14] Marcos`, `[09:28] Bruno`, `[09:30] Bruno`, `[09:48] Larissa`.
- **Código existente:**
  - `src/server.ts`: `bootstrap`, `shutdown`, tratamento de `SIGINT`/`SIGTERM` e `prisma.$disconnect()`.
  - `src/config/database.ts`: `createPrismaClient` e o singleton `prisma`.
  - `src/config/env.ts`: `DATABASE_URL` em `envSchema`.
  - `package.json`: scripts `dev` e `start`.
  - `tsconfig.build.json`: `include`.
- **Documentos:** [RFC](../RFC.md) (seção de decisões relacionadas) · [FDD](../FDD.md) (detalhamento de implementação do worker) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) · [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md).
