# Status — FIAP X Video Processing (retomar aqui)

Última atualização: 2026-09-26.

## O que é isto

Hackathon FIAP X (`project-docs-claude/enunciado.txt`): sistema que recebe
upload de vídeo, extrai frames de forma assíncrona (ffmpeg) e disponibiliza
um `.zip` para download. Decisão de arquitetura (confirmada com o usuário):
seguir o padrão completo dos repositórios irmãos `pos-*` — microsserviços
separados, saga orquestrada, transactional outbox, consumidores
idempotentes, Helm, testcontainers — com MinIO para storage de vídeo/zip e
JWT embutido no upload-service (sem serviço de auth separado). Detalhes
completos e o porquê de cada decisão: `docs/architecture.md` + `docs/adr/`.

## Repositórios (todos criados, com git init + commits, NADA pushado ainda)

Todos em `/Users/franz/development/pos/`, como diretórios irmãos:

- `fiapx-video-upload-service` — auth JWT (bcrypt real, não mockado), upload → MinIO, listagem de status, URL de download presigned.
- `fiapx-video-processing-service` — worker ffmpeg (extração de frames + zip), MinIO.
- `fiapx-video-notification-service` — worker SMTP (Mailhog local), sem outbox (consumidor terminal).
- `fiapx-saga-orchestrator` — máquina de estados `UPLOADED→PROCESSING→COMPLETED|FAILED`, dono do `deploy/local/docker-compose.yml` compartilhado (19 serviços).
- `project-5-hacka` (este repo) — documentação: `docs/architecture.md`, `docs/adr/000{1,2,3,4}-*.md`, `docs/db-schema.md`, `docs/runbook.md`.
- `../fiapx-hackathon.code-workspace` — workspace raiz linkando os 5 acima.

## O que já está PRONTO e verificado

- **Golden path rodado de ponta a ponta de verdade via `docker compose up`** (não só estaticamente): registro → login → upload → `UPLOADED`→`PROCESSING`→`COMPLETED` (~1.7s, 3 frames extraídos) → saga history correta (`GET /api/v1/sagas/:video_id` no saga-orchestrator, porta 8084) → e-mail de conclusão chegou no Mailhog → download URL presigned funciona do host e o `.zip` baixado contém os 3 frames + `original.mp4`.
- **Caminho de falha também verificado de ponta a ponta**: upload de um arquivo inválido (`.txt` renomeado `.mp4`) → ffmpeg falha → status `FAILED` com `error_message` real do ffmpeg → e-mail de falha chegou no Mailhog.
- Todas as filas do RabbitMQ (management UI, porta 15672) dreram limpo após o fluxo — 0 mensagens nas DLQs, confirmando outbox + idempotência funcionando.
- Os 4 serviços Go: `gofmt`/`go vet`/`go build`/`go test` — verde em cada repo. Teste de integração real do ffmpeg (`fiapx-video-processing-service/tests/integration/extract_frames_test.go`, tag `integration`) agora **roda de verdade** (fixture `tests/fixtures/sample.mp4` gerada, ffmpeg instalado localmente via brew) e passa.
- Contrato de eventos conferido campo-a-campo nos 4 saltos — nomes de evento e campos JSON batem exatamente entre produtor e consumidor (conferido via grep direto no código).
- `amqp.go` é byte-a-byte idêntico nos 4 repos.
- Helm charts das 4 repos: `helm lint` passa limpo em todos.

### Bugs reais encontrados e corrigidos (sessão 2026-09-25, revisão estática)
1. `go.mod` resolvido para Go 1.25/1.26 mas `Dockerfile` apontava `golang:1.23-*` — corrigido para `golang:1.26-*` nos 4 repos.
2. `docker-compose.yml` com build context do notification-service faltando `-video-` no nome do diretório — corrigido.

### Bugs reais encontrados e corrigidos (sessão 2026-09-26, execução real)
3. **`minio/minio` e `minio/mc` não pull mais** (MinIO foi source-only a partir de 2025-10-15 e puxou suas imagens do Docker Hub/quay.io) — trocado para `bitnamilegacy/minio` / `bitnamilegacy/minio-client` em `fiapx-saga-orchestrator/deploy/local/docker-compose.yml`, healthcheck do minio trocado de `mc ready local` para `curl -f http://localhost:9000/minio/health/live` (a imagem bitnami não pré-configura o alias `local` do `mc` que a imagem oficial tinha).
4. **Build paralelo dos 12 binários Go via `docker compose build` estourava a memória da VM do Docker Desktop** (só 3.8GB alocados) — `go build` do pgx morria com `signal: killed` (OOM). Contorno: `COMPOSE_PARALLEL_LIMIT=1 docker compose build` (sequencial). Considerar aumentar a memória do Docker Desktop nas configurações se isso incomodar.
5. **Upload de vídeo retornava 500 sempre**, causa raiz em `fiapx-video-upload-service/go.mod`: `aws-sdk-go-v2/service/internal/checksum` e `.../s3shared` estavam fixados em versões mais novas (v1.11.5/v1.20.4) do que as que `service/s3 v1.75.0` e `feature/s3/manager v1.17.0` foram de fato construídos contra (v1.5.3/v1.18.10) — o mismatch quebrava a montagem do middleware stack do `PutObject` com erro `not found: S3100Continue`, falhando 100% client-side antes de qualquer I/O de rede (por isso não relacionado ao MinIO/bitnami, que foi descartado como causa via reprodução isolada). Corrigido com `go get .../checksum@v1.5.3 .../s3shared@v1.18.10` + `go mod tidy`. Também foi adicionado um `slog.Error` no handler de upload (estava engolindo o erro real, por isso o bug ficou invisível nos logs).
6. **URL de download presigned vinha com o hostname interno `minio:9000`** (o nome do serviço no docker-compose), que não resolve do host e não pode ser trocado por `localhost:9000` depois porque a assinatura SigV4 é calculada sobre o header `Host`. Corrigido criando um segundo cliente S3 (`presignClient`) usado só para `PresignGetObject`, apontado para um novo endpoint configurável via `MINIO_PUBLIC_ENDPOINT` (default = `MINIO_ENDPOINT`, então nada quebra onde não precisa do split); setado para `localhost:9000` no compose. Ver `internal/infrastructure/storage/s3_client.go` e `internal/infrastructure/config/config.go`.
7. **`docs/runbook.md` documentava a rota errada da saga** (`/sagas/<video_id>` em vez de `/api/v1/sagas/<video_id>`) — corrigido.

Todos os 5 repos com working tree **sujo** neste momento (mudanças acima ainda não commitadas — ver "Para retomar", passo 0).

## O que NÃO está feito ainda

1. **CI workflows completos** (`.github/workflows/ci.yml` por repositório) — ainda não criados em nenhum dos 4 repos. Template de referência já lido e pronto para reaproveitar: `/Users/franz/development/pos/pos-os-service/.github/workflows/ci.yml` (jobs: `lint` golangci-lint+govet+gofmt, `test` com cobertura, `build` docker para cada `TARGET`, `sonar` — SonarQube efêmero via container na própria action). Ajustar `go-version` para `1.26` e a lista de `TARGET`s por serviço.
2. **Testes de integração reais (testcontainers-go)** — ainda placeholders/`t.Skip` em 3 dos 4 repos (só o ffmpeg do processing-service roda de verdade agora):
   - `fiapx-video-upload-service/tests/integration/stub_test.go`
   - `fiapx-video-notification-service/tests/integration/placeholder_test.go`
   - `fiapx-saga-orchestrator/tests/integration/saga_flow_test.go`
   Referência: `pos-os-service/tests/integration/*.go` (Postgres/RabbitMQ via testcontainers-go, atrás de `//go:build integration`).
3. **Teste de carga (load-spike smoke test)** — script disparando N uploads concorrentes contra a stack rodando. Ainda não escrito.
4. **Nada foi pushado para o GitHub** — todos os 5 repos são só locais, sem remote configurado. Entrega explícita do hackathon; requer confirmação explícita do usuário antes de criar repos/push (ação pública).
5. **Vídeo de apresentação (≤10min)** — roteiro em `docs/runbook.md`, vídeo em si não gravado.

## Para retomar, nesta ordem sugerida

0. **Commitar as correções da sessão 2026-09-26** nos 4 repos afetados (`fiapx-video-upload-service`, `fiapx-video-processing-service` — só o fixture novo —, `fiapx-saga-orchestrator`, `project-5-hacka`) antes de continuar — ver lista de bugs 3-7 acima. Perguntar ao usuário se quer revisar antes ou já commitar.
1. CI workflows (item 1) — mecânico, rápido, baixo risco.
2. Testcontainers restantes (item 2).
3. Load-spike test (item 3).
4. Decidir sobre GitHub push (item 4) — perguntar ao usuário antes.
5. Gravar o vídeo (item 5) — agora que o golden path E o caminho de falha estão comprovadamente funcionando ao vivo, dá pra gravar seguindo o roteiro de `docs/runbook.md` sem medo de travar no meio.

## Notas úteis para retomar a stack

- Docker Desktop precisa estar aberto (`open -a Docker`).
- Buildar sequencial para não estourar memória: `cd fiapx-saga-orchestrator/deploy/local && COMPOSE_PARALLEL_LIMIT=1 docker compose build`.
- Subir: `docker compose up -d`.
- UIs: RabbitMQ http://localhost:15672 (guest/guest), MinIO console http://localhost:9001 (minioadmin/minioadmin), Mailhog http://localhost:8025.
- Rota da saga é `GET localhost:8084/api/v1/sagas/<video_id>` (já corrigido no runbook).
