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

## Teste de carga (load-spike smoke test)

```bash
../fiapx-saga-orchestrator/scripts/load_spike_test.sh [N] [UPLOAD_BASE_URL]
```

Registra um usuário novo, dispara `N` uploads concorrentes (default 30)
contra o `upload-service`, e espera até todos chegarem a um estado
terminal. Falha se qualquer requisição não for aceita (202) ou se algum
vídeo não chegar a `COMPLETED` dentro do timeout — demonstrando que o
sistema absorve o pico via outbox + fila em vez de perder ou rejeitar
requisições, mesmo o worker do processing-service consumindo a fila um
vídeo por vez. Verificado localmente com 20, 50 e 100 uploads
simultâneos — todos aceitos e concluídos, 0 mensagens nas DLQs em
qualquer execução.

## Roteiro sugerido para o vídeo de apresentação (≤10min)

1. (~2min) Documentação: `docs/architecture.md`, diagrama draw.io
   ([`architecture-diagram.html`](architecture-diagram.html)), contrato de
   eventos, ADRs.
2. (~2min) Arquitetura escolhida: orquestrador de saga mesmo sem
   compensação ([ADR 0001](adr/0001-orchestrated-saga.md)), outbox +
   idempotência para garantir zero perda de requisição sob pico.
3. (~4:30) Projeto funcionando: golden path ao vivo, caminho de falha, e um
   teste de carga demonstrando que um pico de uploads concorrentes não
   perde nenhuma requisição.
4. (~1min) Testes com testcontainers reais e CI/CD nos 5 repositórios.
