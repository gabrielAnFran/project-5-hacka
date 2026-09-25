# Esquema de banco de dados

Cada serviço tem seu próprio banco Postgres, isolado — nenhum serviço lê ou
escreve diretamente no banco de outro. Os scripts SQL abaixo são apenas um
resumo; a fonte de verdade executável fica em `migrations/000001_init.up.sql`
dentro de cada repositório de serviço (linkado abaixo) para evitar
divergência entre este documento e o código real.

## upload_service — [`fiapx-video-upload-service/migrations`](../../fiapx-video-upload-service/migrations)

- `users(id, email UNIQUE, password_hash, created_at, updated_at)`
- `videos(id, user_id FK, original_filename, content_type, size_bytes, source_bucket, source_object_key, status CHECK IN (UPLOADED,PROCESSING,COMPLETED,FAILED), zip_bucket, zip_object_key, frame_count, error_message, created_at, updated_at, completed_at, failed_at)` — índice em `(user_id, status)` para a listagem por usuário.
- `outbox` — padrão transactional outbox (id, event_id UNIQUE, aggregate_id, event_name, payload JSONB, headers JSONB, created_at, published_at nullable).
- `processed_events(event_id PK, processed_at)` — idempotência do worker que consome `video.status.completed|failed`.

## processing_service — [`fiapx-video-processing-service/migrations`](../../fiapx-video-processing-service/migrations)

- `processing_jobs(id, video_id UNIQUE, user_id, source_bucket, source_object_key, status CHECK IN (RUNNING,COMPLETED,FAILED), frame_interval_seconds, frame_count, zip_bucket, zip_object_key, error_message, started_at, completed_at, created_at, updated_at)`.
- `outbox`, `processed_events` — mesmo padrão acima.

## notification_service — [`fiapx-video-notification-service/migrations`](../../fiapx-video-notification-service/migrations)

- `notification_log(id, video_id, user_id, recipient, channel DEFAULT 'email', type CHECK IN (COMPLETED,FAILED), status CHECK IN (SENT,FAILED), error_message, sent_at, created_at)`.
- `processed_events` — sem tabela `outbox`: este serviço é um consumidor terminal, nunca publica eventos.

## saga_orchestrator — [`fiapx-saga-orchestrator/migrations`](../../fiapx-saga-orchestrator/migrations)

- `saga_instances(id, saga_type, video_id UNIQUE, state, context JSONB, last_event_id, retry_count, created_at, updated_at)`.
- `saga_history(id, saga_id FK, from_state, to_state, event_name, event_id, error, at)` — trilha de auditoria append-only de cada transição.
- `outbox`, `processed_events` — mesmo padrão.

## Padrão comum: outbox + processed_events

Toda escrita de domínio que precisa emitir um evento grava a linha de
domínio E a linha de `outbox` na mesma transação Postgres. Um binário
`outbox-dispatcher` separado, em cada serviço (exceto notification, que não
publica nada), faz polling da tabela `outbox` (`WHERE published_at IS
NULL`), publica no RabbitMQ, e marca `published_at`. Isso garante que uma
falha entre o commit no banco e a publicação no broker nunca perde o
evento — o dispatcher retoma de onde parou.

Do lado do consumo, `processed_events` guarda o `event_id` de cada evento já
tratado; todo handler de evento checa essa tabela antes de fazer qualquer
trabalho, tornando o consumo seguro mesmo com entrega *at-least-once* do
RabbitMQ (reentregas por timeout, retry, ou reinício do worker não
duplicam efeitos colaterais).
