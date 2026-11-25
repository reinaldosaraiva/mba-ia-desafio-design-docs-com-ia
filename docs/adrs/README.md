# Architectural Decision Records

Este diretório guarda os ADRs (Architecture Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**. Cada arquivo registra uma decisão arquitetural fechada na reunião técnica transcrita em [`TRANSCRICAO.md`](../../TRANSCRICAO.md), no formato `ADR-NNN-titulo-em-kebab-case.md` (variante MADR, em português).

Os ADRs de 001 a 006 cobrem as seis decisões principais da reunião; 007 e 008 registram duas decisões complementares em que uma alternativa real foi descartada. A origem de cada item está no [Tracker](../TRACKER.md). A proposta que os reúne está no [RFC](../RFC.md) e o detalhamento de implementação no [FDD](../FDD.md).

| ID | Título | Status | Fechada em |
|---|---|---|---|
| ADR-001 | [Outbox no MySQL para eventos de mudança de status](ADR-001-outbox-no-mysql.md) | Aceito | `[09:08] Larissa` |
| ADR-002 | [Worker em processo separado com polling de 2 segundos](ADR-002-worker-separado-com-polling.md) | Aceito |  |
| ADR-003 | [Retry com backoff e fila de mortos (DLQ)](ADR-003-retry-com-backoff-e-dlq.md) | Aceito | `[09:17] Larissa` |
| ADR-004 | [Assinatura HMAC-SHA256 com secret por endpoint](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | Aceito | `[09:22] Sofia` |
| ADR-005 | [Entrega at-least-once com `X-Event-Id` para deduplicação](ADR-005-at-least-once-com-x-event-id.md) | Aceito | `[09:26] Larissa` |
| ADR-006 | [Reuso dos padrões existentes do projeto](ADR-006-reuso-dos-padroes-do-projeto.md) | Aceito | `[09:30] Larissa` |
| ADR-007 | [Snapshot do payload no momento da inserção](ADR-007-snapshot-do-payload-na-insercao.md) | Aceito | `[09:52] Bruno` |
| ADR-008 | [Filtro de eventos aplicado na inserção na outbox](ADR-008-filtro-de-eventos-na-insercao.md) | Aceito | `[09:34] Bruno` |
