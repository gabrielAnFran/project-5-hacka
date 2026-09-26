# Runbook — demonstração local e roteiro do vídeo

## Subir a stack

```bash
cd ../fiapx-saga-orchestrator/deploy/local
docker compose up --build
```

Aguardar todos os healthchecks (Postgres ×4, RabbitMQ, MinIO). UIs úteis:

- RabbitMQ management: http://localhost:15672 (guest/guest)
- MinIO console: http://localhost:9001 (minioadmin/minioadmin)
- Mailhog (caixa de entrada de e-mail local): http://localhost:8025

## Roteiro golden path (curl)

```bash
# 1. Registrar usuário
curl -s -X POST localhost:8081/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"demo@fiapx.local","password":"senha123"}'

# 2. Login
TOKEN=$(curl -s -X POST localhost:8081/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"demo@fiapx.local","password":"senha123"}' | jq -r .token)

# 3. Upload de vídeo
curl -s -X POST localhost:8081/api/v1/videos \
  -H "Authorization: Bearer $TOKEN" \
  -F "video=@sample.mp4"

# 4. Poll de status (repetir até status == COMPLETED)
curl -s localhost:8081/api/v1/videos -H "Authorization: Bearer $TOKEN" | jq

# 5. Download
curl -s localhost:8081/api/v1/videos/<video_id>/download -H "Authorization: Bearer $TOKEN" | jq

# 6. Conferir e-mail recebido em http://localhost:8025

# 7. (opcional) inspecionar o estado da saga
curl -s localhost:8084/api/v1/sagas/<video_id> | jq
```

Para demonstrar o caminho de falha, subir um arquivo que não é um vídeo
válido (ex: um `.txt` renomeado para `.mp4`) — o `ffmpeg` falhará, o status
do vídeo ficará `FAILED` com `error_message` preenchido, e um e-mail de
falha chegará no Mailhog.

## Roteiro sugerido para o vídeo de apresentação (≤10min)

1. (~2min) Documentação: mostrar `docs/architecture.md`, o diagrama de
   fluxo e a tabela de contrato de eventos.
2. (~2min) Arquitetura escolhida: explicar o porquê do orquestrador de saga
   mesmo sem compensação ([ADR 0001](adr/0001-orchestrated-saga.md)), e o
   padrão outbox + idempotência para garantir zero perda de requisição sob
   pico.
3. (~5min) Projeto funcionando: `docker compose up`, rodar o roteiro golden
   path acima ao vivo, mostrar o e-mail chegando no Mailhog, mostrar as
   filas no RabbitMQ management, mostrar o objeto no MinIO console.
4. (~1min) Caminho de falha + notificação de erro.
