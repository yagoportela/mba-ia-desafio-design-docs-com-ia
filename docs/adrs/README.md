# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada em um arquivo individual,
nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`.

- [ADR-001 — Outbox transacional no MySQL](./ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker separado em polling](./ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Política de retry, backoff e DLQ](./ADR-003-politica-de-retry-backoff-e-dlq.md)
- [ADR-004 — Autenticação HMAC-SHA256 e rotação de secret](./ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md)
- [ADR-005 — Entrega at-least-once e idempotência via event-id](./ADR-005-entrega-at-least-once-e-idempotencia-via-event-id.md)
- [ADR-006 — Reuso dos padrões arquiteturais existentes](./ADR-006-reuso-dos-padroes-arquiteturais-existentes.md)
- [ADR-007 — Snapshot de payload na inserção da outbox](./ADR-007-snapshot-de-payload-na-insercao-da-outbox.md)
