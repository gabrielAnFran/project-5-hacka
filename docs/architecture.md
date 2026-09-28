# Arquitetura — FIAP X Video Processing

## Visão geral

Sistema de processamento de vídeos assíncrono, composto por 4 microsserviços
Go independentes (cada um em seu próprio repositório, com seu próprio banco
de dados), coordenados por um orquestrador de saga e comunicando-se
exclusivamente via eventos em um exchange RabbitMQ (`video.events`).

Diagrama editável (importar em [draw.io](https://app.diagrams.net) via
File → Open ou arrastando o arquivo): [`architecture-diagram.html`](architecture-diagram.html).

```
                         ┌────────────────────┐
   usuário  ──HTTP──────▶│  video-upload-      │
                         │  service (8081)     │◀───┐
                         │  auth JWT, upload,   │    │ comandos:
                         │  listagem, download  │    │ video.status.completed
                         └─────────┬────────────┘    │ video.status.failed
                                   │ fato:            │
                                   │ video.uploaded    │
                                   ▼                  │
                         ┌────────────────────┐       │
                         │  saga-orchestrator  │───────┘
                         │  (8084)             │
                         │  UPLOADED→PROCESSING│───────┐
                         │  →COMPLETED|FAILED   │       │ comando:
                         └─────────┬────────────┘       │ video.notify.requested
                                   │ comando:            │
                                   │ video.process.       ▼
                                   │ requested    ┌────────────────────┐
                                   ▼              │ video-notification- │
                         ┌────────────────────┐   │ service (8083)      │
                         │ video-processing-   │   │ SMTP (Mailhog local)│
                         │ service (8082)      │   └────────────────────┘
                         │ ffmpeg + zip, MinIO │
                         └─────────┬────────────┘
                                   │ fatos:
                                   │ video.processing.completed
                                   │ video.processing.failed
                                   └──────────────▶ saga-orchestrator
```

Todos os serviços são stateless (exceto pelo seu próprio banco Postgres) e
publicam/consomem eventos via `RabbitMQ`. Nenhum serviço chama outro
diretamente por HTTP — o `saga-orchestrator` é o único ponto que decide o
que acontece a seguir e publica os comandos correspondentes. Isso mantém os
4 serviços fracamente acoplados e evita uma malha de chamadas síncronas
N×N.

## Repositórios

| Repositório | Papel | Porta | Banco |
|---|---|---|---|
| [`fiapx-video-upload-service`](../../fiapx-video-upload-service) | Auth (JWT), upload para MinIO, listagem de status, URL de download | 8081 | `upload_service` |
| [`fiapx-video-processing-service`](../../fiapx-video-processing-service) | Extração de frames via ffmpeg, zip, upload do resultado | 8082 | `processing_service` |
| [`fiapx-video-notification-service`](../../fiapx-video-notification-service) | Envio de e-mail (SMTP) em caso de sucesso/falha | 8083 | `notification_service` |
| [`fiapx-saga-orchestrator`](../../fiapx-saga-orchestrator) | Máquina de estados da saga, coordenação via eventos, stack local de desenvolvimento | 8084 | `saga_orchestrator` |

Cada serviço segue a mesma estrutura (`cmd/{server,worker,outbox-dispatcher}`,
`internal/{domain,application,infrastructure,presentation}`), o padrão
**transactional outbox** (a escrita no domínio e o evento correspondente
acontecem na mesma transação de banco; um binário `outbox-dispatcher`
separado publica os eventos pendentes no RabbitMQ) e **consumidores
idempotentes** (tabela `processed_events`, checada antes de qualquer
processamento). Ver `pos-os-service` e `pos-saga-orchestrator` (sistema
irmão do mesmo autor) para a origem desses padrões.

## Contrato de eventos (`video.events`, topic exchange)

**Fatos** (produzidos pelo dono do dado; consumidos apenas pelo orquestrador):

| Evento | Produtor | Consumidor | Payload principal |
|---|---|---|---|
| `video.uploaded` | upload-service | saga-orchestrator | `video_id, user_id, user_email, original_filename, content_type, size_bytes, source_bucket, source_object_key, uploaded_at` |
| `video.processing.completed` | processing-service | saga-orchestrator | `video_id, user_id, job_id, zip_bucket, zip_object_key, frame_count, completed_at` |
| `video.processing.failed` | processing-service | saga-orchestrator | `video_id, user_id, job_id, error_code, error_message, failed_at` |

**Comandos** (produzidos exclusivamente pelo orquestrador):

| Evento | Produtor | Consumidor | Payload principal |
|---|---|---|---|
| `video.process.requested` | saga-orchestrator | processing-service | `video_id, user_id, saga_id, source_bucket, source_object_key, frame_interval_seconds` |
| `video.status.completed` | saga-orchestrator | upload-service | `video_id, zip_bucket, zip_object_key, frame_count, completed_at` |
| `video.status.failed` | saga-orchestrator | upload-service | `video_id, error_code, error_message, failed_at` |
| `video.notify.requested` | saga-orchestrator | notification-service | `video_id, user_id, recipient_email, notification_type (COMPLETED\|FAILED), original_filename, error_message?` |

Máquina de estados: `UPLOADED → PROCESSING → COMPLETED\|FAILED` (linear,
sem compensação — ver [ADR 0001](adr/0001-orchestrated-saga.md)).

## Fluxo (golden path)

1. Usuário registra conta e faz login (`POST /auth/register`, `POST /auth/login` no upload-service) → recebe JWT.
2. `POST /api/v1/videos` (multipart, Bearer JWT) — o vídeo é enviado (streamed) direto para o MinIO, uma linha `videos` é gravada com status `UPLOADED` e um evento `video.uploaded` é gravado no outbox, na mesma transação. Resposta `202 Accepted`.
3. O `outbox-dispatcher` do upload-service publica `video.uploaded` no RabbitMQ.
4. O `saga-orchestrator` consome, cria uma `saga_instances` (estado `PROCESSING`) e publica `video.process.requested`.
5. O `processing-service` consome, baixa o vídeo do MinIO, roda `ffmpeg` extraindo um frame por segundo, compacta em `.zip`, envia o zip para o MinIO, e publica `video.processing.completed` (ou `.failed`).
6. O `saga-orchestrator` consome o fato, transiciona a saga para `COMPLETED`/`FAILED`, e publica dois comandos: `video.status.completed|failed` (para o upload-service atualizar o status do vídeo) e `video.notify.requested` (para o notification-service enviar o e-mail).
7. Usuário consulta `GET /api/v1/videos` e vê o status atualizado; quando `COMPLETED`, `GET /api/v1/videos/{id}/download` retorna uma URL pré-assinada do MinIO válida por ~15 minutos.

## Escalabilidade e garantia de não perder requisições

- `POST /videos` só grava no banco + outbox e faz streaming para o MinIO — devolve `202` sem nunca chamar `ffmpeg` de forma síncrona, então não há acúmulo de conexões HTTP presas sob pico.
- O outbox é o limite de durabilidade: se o processo cair entre o commit no banco e a publicação no RabbitMQ, o `outbox-dispatcher` retoma do ponto onde parou ao reiniciar — nenhum evento se perde. Mensagens são publicadas com `delivery_mode=2` (persistente) em filas duráveis.
- "Processar mais de um vídeo simultaneamente" é resolvido escalando o **número de réplicas** do worker do processing-service — o RabbitMQ distribui round-robin entre consumidores concorrentes na mesma fila. O prefetch desse worker é mantido baixo (1–2) porque cada entrega é pesada em CPU/memória (ffmpeg), ao contrário do worker de notificação que pode ter prefetch mais alto.
- Entrega *at-least-once* + idempotência via `processed_events` + retry/DLQ (header `x-retry-count`, até 5 tentativas, depois fila morta) garante que falhas transitórias (worker reiniciado no meio do ffmpeg, banco indisponível por um instante) resultam em reentrega, não em perda silenciosa.
- Todos os binários são stateless → cada um vira um `Deployment` Kubernetes independente, escalável via `HPA` (CPU/memória no dia 1; escalonamento por profundidade de fila via KEDA é uma extensão futura natural).

## Armazenamento de objetos (MinIO)

Bucket único `fiapx-videos`, prefixos:
- `raw/{user_id}/{video_id}/original.<ext>` — vídeo original enviado pelo usuário
- `processed/{user_id}/{video_id}/frames.zip` — resultado da extração de frames

Ver [ADR 0003](adr/0003-minio-object-storage.md).
