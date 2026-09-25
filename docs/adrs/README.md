# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature de Webhooks de Notificação de Pedidos. Cada arquivo é nomeado `ADR-NNN-titulo-em-kebab-case.md` (ex.: `ADR-001-outbox-pattern-no-mysql.md`), no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).

## Índice

| ADR | Decisão |
|---|---|
| [ADR-001](ADR-001-outbox-pattern-no-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](ADR-002-worker-processo-separado-em-polling.md) | Worker em processo separado, em polling |
| [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md) | Retry com backoff exponencial e Dead Letter Queue |
| [ADR-004](ADR-004-autenticacao-hmac-sha256-por-endpoint.md) | Autenticação HMAC-SHA256 por endpoint |
| [ADR-005](ADR-005-garantia-at-least-once-com-x-event-id.md) | Garantia at-least-once com X-Event-Id |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões arquiteturais existentes do projeto |
| [ADR-007](ADR-007-payload-snapshot-na-insercao-do-evento.md) | Payload como snapshot no momento da inserção do evento |
