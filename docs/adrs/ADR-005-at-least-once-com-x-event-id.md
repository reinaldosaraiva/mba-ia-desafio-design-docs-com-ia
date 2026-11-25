# ADR-005: Entrega at-least-once com `X-Event-Id` para deduplicação

| Campo | Valor |
|---|---|
| ID | ADR-005 |
| Status | Aceito |
| Data da decisão | Reunião técnica transcrita em `TRANSCRICAO.md` (a transcrição não informa a data calendário) |
| Decisores | Larissa (Tech Lead, fechou a decisão), Diego (propôs o modelo de entrega) e Marcos (assumiu a documentação no portal do desenvolvedor) |
| Documentos relacionados | [RFC](../RFC.md) · [FDD](../FDD.md) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) · [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md) |

## Status

Aceito. Decisão fechada na reunião em `[09:26] Larissa`. O compromisso de documentação foi assumido em `[09:26] Marcos`. A decisão foi reafirmada no resumo final em `[09:48] Larissa` ("Idempotência por X-Event-Id, garantia at-least-once").

## Contexto

1. Os eventos de mudança de status entram na outbox e são lidos por um worker separado, que dispara as chamadas HTTP ([ADR-001](./ADR-001-outbox-no-mysql.md), `[09:06] Diego`). Falhas de entrega passam por retry e DLQ, e o replay manual da DLQ "recoloca na outbox como pendente" ([ADR-003](./ADR-003-retry-com-backoff-e-dlq.md), `[09:15] Diego`, `[09:18] Diego`).
2. **Possibilidade de duplicata.** Diego afirma que a entrega será at-least-once e que "pode acontecer de o cliente receber o mesmo evento duas vezes". Segundo ele, o cliente "tem que estar preparado" (`[09:24] Diego`).
3. **Como o cliente diferencia.** Bruno pergunta como o cliente reconhece a duplicata (`[09:25] Bruno`). Diego responde que cada evento leva um `event_id` no header `X-Event-Id`. Esse valor é um UUID gerado quando o evento entra na outbox, é único por evento e permite ao cliente deduplicar do lado dele (`[09:25] Diego`).
4. **Objeção.** Sofia aponta que isso "joga responsabilidade pro cliente" (`[09:25] Sofia`).
5. **Resposta à objeção.** Diego reconhece que joga, mas argumenta em três pontos (`[09:25] Diego`):
   - é o padrão de mercado ("Stripe faz assim, GitHub faz assim");
   - garantir exactly-once "exigiria coordenação dos dois lados e fica muito mais complexo";
   - at-least-once com `event_id` "resolve 99% dos casos".
6. **Código existente.** A dependência `uuid` já está declarada em `package.json`. Ela já é usada em `src/middlewares/request-logger.middleware.ts` para gerar o `X-Request-Id`.

## Decisão

**ADR-005.** Nós adotamos entrega at-least-once, com identificador de evento para deduplicação do lado do cliente (`[09:26] Larissa`):

1. Nós garantimos entrega at-least-once. O cliente pode receber o mesmo evento mais de uma vez e deve estar preparado para isso (`[09:24] Diego`).
2. Nós atribuímos a cada evento um UUID no momento em que ele entra na outbox. O valor é único por evento (`[09:25] Diego`).
3. Nós enviamos esse UUID no header `X-Event-Id` (`[09:25] Diego`). Esse header acompanha os demais headers do request, entre eles o `X-Signature` definido no [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md) (`[09:44] Diego`).
4. O cliente deduplica pelo `X-Event-Id` do lado dele (`[09:25] Diego`, `[09:26] Larissa`).
5. Nós documentamos esse comportamento com destaque no portal do desenvolvedor (`[09:26] Marcos`).

**Interpretação a confirmar:**

- **Estabilidade do id entre reenvios.**
  - Leitura adotada: o `X-Event-Id` identifica o evento e se mantém em todas as entregas dele, inclusive após replay de DLQ. Diego o descreve como "único por evento" e como a chave da deduplicação (`[09:25] Diego`).
  - Outra leitura: o UUID é gerado na entrada na outbox (`[09:25] Diego`) e o replay "recoloca na outbox como pendente" (`[09:18] Diego`). Nesse caso, o replay produziria um id novo, e o cliente não reconheceria o evento reprocessado como duplicata.
  - Essa outra leitura ganha força se o `X-Event-Id` for o próprio id UUID da linha da outbox (`[09:51] Larissa`). A transcrição não estabelece esse vínculo.
- **Relação com o `event_id` do payload.**
  - Leitura adotada: o campo `event_id` do payload JSON (`[09:43] Diego`) carrega o mesmo valor do `X-Event-Id`. Diego chama o valor do header de `event_id` (`[09:25] Diego`).
  - Outra leitura: são identificadores distintos. A transcrição não afirma explicitamente que são o mesmo valor.

## Alternativas Consideradas

### ADR-005-ALT-1: Garantia exactly-once

A alternativa era garantir que cada evento chegasse ao cliente exatamente uma vez, sem que ele precisasse deduplicar. Nenhum participante a propôs como solução. Diego a citou como contraponto ao responder à objeção de que at-least-once "joga responsabilidade pro cliente" (`[09:25] Sofia`).

Ela atenderia a essa preocupação de Sofia. Foi descartada em `[09:25] Diego` porque "exigiria coordenação dos dois lados e fica muito mais complexo". A opção por at-least-once foi fechada em `[09:26] Larissa`.

## Consequências

### Positivas

- Simplicidade: a entrega não depende da coordenação dos dois lados que exactly-once exigiria (`[09:25] Diego`).
- O modelo se alinha ao padrão de mercado citado na reunião, com Stripe e GitHub como exemplos (`[09:25] Diego`).
- O cliente dispõe de um identificador único por evento para reconhecer duplicatas (`[09:25] Diego`).

### Negativas e trade-off explícito

- A responsabilidade de deduplicar fica com o cliente (`[09:25] Sofia`), ponto reconhecido por Diego ("Joga") em `[09:25] Diego`.
- O cliente precisa estar preparado para receber o mesmo evento mais de uma vez (`[09:24] Diego`).
- O comportamento precisa estar documentado com destaque no portal do desenvolvedor, compromisso assumido em `[09:26] Marcos`.

**Trade-off aceito:** aceitamos que a responsabilidade de deduplicar pelo `X-Event-Id` fique com o cliente (`[09:25] Sofia`, `[09:25] Diego`) em troca da simplicidade de não depender da coordenação dos dois lados que exactly-once exigiria (`[09:25] Diego`).

## Referências

- Transcrição: `[09:06] Diego`, `[09:15] Diego`, `[09:18] Diego`, `[09:24] Diego`, `[09:25] Bruno`, `[09:25] Diego`, `[09:25] Sofia`, `[09:26] Larissa`, `[09:26] Marcos`, `[09:43] Diego`, `[09:44] Diego`, `[09:48] Larissa`, `[09:51] Larissa`.
- Código existente:
  - `package.json`: dependência `uuid` em `dependencies`.
  - `src/middlewares/request-logger.middleware.ts`: middleware `requestLogger`, que usa `uuidv4` (importado de `uuid`) para gerar o `X-Request-Id`.
- Documentos: [RFC](../RFC.md) (seção de decisões relacionadas) · [FDD](../FDD.md) · [ADR-001](./ADR-001-outbox-no-mysql.md) · [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) · [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md).
