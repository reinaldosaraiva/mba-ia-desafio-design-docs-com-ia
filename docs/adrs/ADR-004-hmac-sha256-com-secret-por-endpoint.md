# ADR-004: Assinatura HMAC-SHA256 com secret por endpoint

| Campo | Valor |
|---|---|
| ID | ADR-004 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Sofia (Engenheira de Segurança), Marcos (Product Manager), Larissa (Tech Lead) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) · [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) |

## Status

Aceito. Decisão fechada na reunião em `[09:22] Sofia`. A geração da secret pela plataforma foi definida em `[09:31] Marcos`. A decisão foi repetida no resumo de encerramento em `[09:48] Larissa`.

## Contexto

1. A feature é de webhook de saída. Os eventos saem da plataforma para o cliente, e o cliente apenas recebe (`[09:02] Marcos`, `[09:03] Sofia`). Quem assina é a plataforma, e quem verifica é o cliente, do lado dele (`[09:20] Sofia`).
2. Os eventos carregam dados de pedidos e vão para um endpoint fora da infraestrutura da plataforma. O cliente precisa conseguir validar duas coisas (`[09:19] Sofia`):
   - a origem: a requisição veio realmente da plataforma;
   - a integridade: ninguém adulterou o payload no caminho.
3. Sofia propôs HMAC: a plataforma assina com uma secret compartilhada com o cliente e envia a assinatura num header (`[09:20] Sofia`).
4. Bruno perguntou qual algoritmo usar (`[09:20] Bruno`). A resposta foi SHA-256, porque HMAC-SHA256 é o padrão de mercado e todo cliente sério tem biblioteca para isso (`[09:20] Sofia`).
5. O vazamento de secret não é hipotético. A equipe já teve um cliente que vazou a secret em log de aplicação dele (`[09:22] Diego`). Diego trouxe esse caso para reforçar a importância da secret por endpoint e da rotação, propostas em `[09:21] Sofia`.
6. O módulo de webhooks reaproveita o logger Pino que já está no projeto (`[09:29] Bruno`; ver [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md)).
   - Hoje a lista de redação desse logger (`redactPaths` em `src/shared/logger/index.ts`) cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`.
   - Os valores dessas entradas são substituídos por `[REDACTED]`.
   - Nenhuma entrada se refere à secret de webhook.

## Decisão

**ADR-004.** Nós assinamos as entregas de webhook com HMAC-SHA256, nestes termos:

1. Nós calculamos a assinatura de cada entrega com HMAC-SHA256 sobre o corpo do request (`[09:22] Sofia`). Usamos uma secret compartilhada entre a plataforma e o cliente (`[09:20] Sofia`).
2. Nós enviamos a assinatura no header `X-Signature` (`[09:20] Sofia`). O nome foi confirmado na lista de headers do request em `[09:44] Diego`. O cliente verifica a assinatura do lado dele (`[09:20] Sofia`).
3. Nós usamos uma secret única por endpoint de webhook. Não existe secret global da plataforma (`[09:21] Sofia`).
4. Nós geramos a secret na plataforma e a devolvemos ao cliente na criação do cadastro do webhook (`[09:31] Marcos`).
5. O cliente pode pedir uma nova secret pela API. Após a rotação, a secret antiga permanece válida por 24 horas em paralelo com a nova, para que o cliente migre os sistemas dele. Depois disso, a antiga deixa de valer (`[09:21] Sofia`).

Os termos "payload" (`[09:20] Sofia`, `[09:48] Larissa`) e "corpo do request" (`[09:22] Sofia`) são tratados como o mesmo conteúdo. Este ADR usa a forma fechada em `[09:22] Sofia`.

**Interpretação a confirmar:** em `[09:21] Sofia`, "a antiga fica válida por 24 horas em paralelo" tem por objetivo declarado dar tempo para o cliente migrar os sistemas dele. Como a feature é só de saída (`[09:02] Marcos`), o efeito dessa validade depende de qual secret a plataforma usa para assinar as entregas no intervalo. A reunião não disse isso. As duas leituras possíveis são:

- a plataforma continua assinando com a secret antiga até o fim das 24 horas e só então passa a usar a nova;
- a plataforma passa a assinar com a nova logo na rotação, e o cliente passa a aceitar as duas.

Este ADR não escolhe entre elas. A proposta de trabalho e a decisão ficam no [FDD](../FDD.md), para a revisão do documento de design com Bruno e Diego (`[09:50] Larissa`), antes da revisão de segurança de Sofia (`[09:46] Sofia`).

**Não definido neste ADR:** dois pontos não foram decididos na reunião e não recebem decisão aqui. Ficam como lacunas a tratar no [FDD](../FDD.md):

- o formato da assinatura (codificação, prefixo);
- a proteção da secret em repouso (a reunião decidiu que a tabela de configuração guarda a secret: `[09:21] Bruno`, `[09:21] Sofia`; não decidiu como protegê-la).

## Alternativas Consideradas

### ADR-004-ALT-1: Secret global da plataforma

Nesta alternativa, uma única secret da plataforma assinaria as entregas de todos os endpoints de webhook. Sofia mencionou a opção e a descartou em `[09:21] Sofia`: "Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo."

O motivo do descarte é o alcance de um vazamento:

- com secret global, o vazamento compromete a assinatura de todos os endpoints;
- com secret por endpoint, o vazamento fica restrito ao endpoint afetado.

A transcrição não registra nenhuma vantagem defendida para esta opção.

## Consequências

### Positivas

- O cliente passa a ter como validar a origem da requisição e a integridade do payload (`[09:19] Sofia`, `[09:20] Sofia`).
- O algoritmo é de fácil adoção pelo cliente: HMAC-SHA256 é o padrão de mercado, e todo cliente sério tem biblioteca para isso (`[09:20] Sofia`).
- O vazamento da secret de um endpoint não expõe os demais (`[09:21] Sofia`).
- Num vazamento como o que já ocorreu em log de aplicação de cliente (`[09:22] Diego`), o cliente pode pedir nova secret pela API. Ele tem então 24 horas, com as duas secrets em paralelo, para migrar os sistemas dele (`[09:21] Sofia`).

### Negativas e trade-off explícito

- A plataforma passa a gerenciar uma secret por endpoint. Durante cada rotação, gerencia duas secrets válidas ao mesmo tempo por 24 horas (`[09:21] Sofia`).
- A proteção depende do cliente, porque é ele quem verifica a assinatura do lado dele (`[09:20] Sofia`).
- (consequência técnica inferida) A secret passa a ser um dado sensível que pode acabar registrada em log da plataforma. O caso citado em `[09:22] Diego` ocorreu no log da aplicação de um cliente e serviu de argumento para a rotação; quanto aos logs da plataforma, hoje a lista `redactPaths` de `src/shared/logger/index.ts` não cobre a secret.
- (consequência técnica inferida) A lista `redactPaths` precisará ser estendida para que a secret não seja registrada em log. Dois fatos sustentam essa inferência: a secret é devolvida na criação do cadastro (`[09:31] Marcos`), e o módulo usa o mesmo logger (`[09:29] Bruno`). A proposta concreta fica para o [FDD](../FDD.md).
- O código de HMAC e de geração de secret precisa passar pela revisão de segurança da Sofia antes do deploy, com pelo menos dois dias úteis reservados (`[09:46] Sofia`).

**Trade-off aceito:** aceitamos gerenciar uma secret única por endpoint, com duas secrets válidas em paralelo por 24 horas a cada rotação (`[09:21] Sofia`). Em troca, o vazamento de uma secret não vaza tudo, e o cliente tem tempo para migrar os sistemas dele (`[09:21] Sofia`).

## Referências

- Transcrição: `[09:02] Marcos`, `[09:03] Sofia`, `[09:19] Sofia`, `[09:20] Sofia`, `[09:20] Bruno`, `[09:21] Sofia`, `[09:22] Diego`, `[09:22] Sofia`, `[09:29] Bruno`, `[09:31] Marcos`, `[09:44] Diego`, `[09:46] Sofia`, `[09:48] Larissa`.
- Código existente: `src/shared/logger/index.ts`, constante `redactPaths`, usada em `createLogger` como `redact.paths` com censura `[REDACTED]`.
- Documentos:
  - [RFC](../RFC.md): seção de decisões relacionadas.
  - [FDD](../FDD.md): detalhamento de implementação desta decisão e proposta de extensão de `redactPaths`.
  - [ADR-005](./ADR-005-at-least-once-com-x-event-id.md): header `X-Event-Id`, enviado junto com `X-Signature`.
  - [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md): reuso do logger Pino.
