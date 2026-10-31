# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante deve ser registrada aqui em arquivos individuais,
nomeados sequencialmente no formato `ADR-NNN-titulo-em-kebab-case.md` (por exemplo `ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md`), com número de 3 dígitos e título curto em minúsculas, sem acento.

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md) | Usar padrão outbox no MySQL para registrar eventos de webhook | Aceito |
| [ADR-002](ADR-002-processar-outbox-com-worker-separado-em-polling.md) | Processar a outbox com um worker em processo separado, por polling a cada 2 segundos | Aceito |
| [ADR-003](ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md) | Operar com um único worker e garantir ordering apenas por order_id | Aceito |
| [ADR-004](ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md) | Fazer retry com backoff exponencial em 5 tentativas e mover falhas para uma DLQ em tabela separada | Aceito |
| [ADR-005](ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md) | Assinar webhooks com HMAC-SHA256 usando uma secret por endpoint, rotacionável com grace period de 24h | Aceito |
| [ADR-006](ADR-006-garantir-entrega-at-least-once-com-x-event-id.md) | Garantir entrega at-least-once com X-Event-Id para deduplicação no cliente | Aceito |
| [ADR-007](ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md) | Reusar os padrões existentes do projeto no módulo de webhooks | Aceito |
| [ADR-008](ADR-008-gravar-payload-como-snapshot-na-insercao.md) | Gravar o payload do evento como snapshot na inserção da outbox | Aceito |
