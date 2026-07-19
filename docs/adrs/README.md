# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature
**Sistema de Webhooks de Notificação de Pedidos**, no formato MADR e na convenção
de nomes do desafio: `ADR-NNN-titulo-em-kebab-case.md`.

Todas as decisões foram tomadas na reunião técnica registrada em
[`TRANSCRICAO.md`](../../TRANSCRICAO.md); cada ADR cita o timestamp em que a
decisão foi fechada e a rastreabilidade completa está em [`docs/TRACKER.md`](../TRACKER.md).

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão outbox no MySQL para publicação de eventos de webhook |
| [ADR-002](ADR-002-worker-separado-polling.md) | Worker de entrega em processo separado, lendo a outbox por polling de 2 s |
| [ADR-003](ADR-003-retry-backoff-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h |
| [ADR-005](ADR-005-at-least-once-x-event-id.md) | Garantia at-least-once com deduplicação por `X-Event-Id` |
| [ADR-006](ADR-006-reuso-padroes-existentes.md) | Reuso dos padrões existentes do projeto para o módulo de webhooks |
| [ADR-007](ADR-007-snapshot-payload-na-insercao.md) | Snapshot do payload renderizado na inserção na outbox |
