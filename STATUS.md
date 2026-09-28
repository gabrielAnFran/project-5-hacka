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

Todos os 5 repos commitados e com working tree limpo neste momento (exceto um arquivo solto e não relacionado, `project-docs-claude/pos.code-workspace`, deixado como está).

### CI workflows (sessão 2026-09-26, depois do golden path)

`.github/workflows/ci.yml` criado nos 4 repos de serviço, adaptado de `pos-os-service/.github/workflows/ci.yml` (jobs `lint`/`test`/`build`/`sonar`), com `go-version: "1.26"` e a lista de `TARGET`s de cada serviço (notification-service só tem `server`+`worker`, os outros 3 têm `server`+`worker`+`outbox-dispatcher`). Todos os 4 jobs `lint`/`test`/`build` foram exercitados localmente (não só lidos) antes de commitar:

- **Bug real encontrado**: `.golangci.yml` nos 4 repos estava no formato v1, mas `golangci-lint-action@v6` instala a versão `latest` (v2), que recusa carregar config v1 — o job `lint` teria quebrado 100% das vezes. Corrigido com `golangci-lint migrate` (ferramenta oficial) nos 4 repos.
- A migração revelou achados reais de lint que também teriam quebrado o CI: 2 falsos-positivos do `gosec` (G101 "hardcoded credential" numa constante de issuer JWT e num teste comparando a URL default `guest:guest@localhost` do RabbitMQ — não são segredos reais), permissões de arquivo de teste (`0o644`→`0o600`), e 2 avisos de depreciação do `staticcheck` em `processing-service` (`manager.NewUploader`/`Upload` do aws-sdk-go-v2 v1.23.10 agora deprecados a favor de `feature/s3/transfermanager` — migração maior, fora de escopo aqui, suprimida com `//nolint:staticcheck` e justificativa). Todos suprimidos com `//nolint` pontual e comentário do porquê, não com exclusão ampla de regra.

### Testcontainers reais (sessão 2026-09-26, depois da CI)

Testes de integração reais com `testcontainers-go` (v0.34.0, mesma versão do `pos-os-service`) escritos e verificados (rodando de verdade, não só compilando) nos 3 repos que ainda tinham placeholders — o ffmpeg do processing-service já tinha sido feito antes:

- `fiapx-video-upload-service/tests/integration/` — Postgres + RabbitMQ + MinIO (`bitnamilegacy/minio`, mesma imagem do compose). Cobre UserRepository, VideoRepository (upsert de status, outbox, paginação/filtro), OutboxRepository, ProcessedEventRepository, `messaging.Conn` (publish/consume/retry/DLQ), e `S3Client` contra MinIO real — inclusive baixando de verdade via HTTP a partir da URL presigned, o que exercita o fix de `MINIO_PUBLIC_ENDPOINT` desta mesma sessão. 21 testes, ~26s.
- `fiapx-video-notification-service/tests/integration/` — Postgres + RabbitMQ + Mailhog. Cobre NotificationRepository, ProcessedEventRepository, `messaging.Conn`, e um round-trip real de SMTP via `email.Sender` contra o Mailhog, checando a entrega pela API HTTP dele (destinatário, assunto, corpo) em vez de só "sem erro". 12 testes, ~17-20s.
- `fiapx-saga-orchestrator/tests/integration/` — Postgres + RabbitMQ. Cobre SagaRepository/OutboxRepository/ProcessedEventRepository, `messaging.Conn`, e principalmente um teste de fluxo completo que liga um `messaging.Conn` real ao `HandleEvent` real (igual ao `cmd/worker/main.go`) e verifica de ponta a ponta: publicar `video.uploaded` → saga real vai pra `PROCESSING` no Postgres → outbox real com `video.process.requested` → reentrega duplicada é no-op (idempotência) → `video.processing.completed`/`.failed` levam ao estado terminal certo com os 2 eventos de outbox esperados (`video.status.*` + `video.notify.requested`). 12 testes, ~13-15s.

`go test ./...` (sem tags) continua rápido e sem dependências externas nos 4 repos — os testes de integração ficam atrás de `//go:build integration` e só rodam com `go test -tags=integration ./...`.

Todos os 4 repos com working tree limpo depois desses commits.

### Load-spike smoke test (sessão 2026-09-28)

`fiapx-saga-orchestrator/scripts/load_spike_test.sh` escrito e verificado contra a stack real: dispara N uploads concorrentes (default 30), confere que todos são aceitos (202) e que todos chegam a `COMPLETED` dentro de um timeout. Rodado com N=20, 50 e 100 — todos aceitos e concluídos, 0 mensagens em qualquer DLQ. Documentado em `docs/runbook.md`.

Nota curiosa: a primeira tentativa com N=50 reportou 30 vídeos "travados" — não era um bug do sistema, era o próprio script não passando `?limit=` no polling contra `GET /api/v1/videos`, que tem paginação com página default de 20 (`fiapx-video-upload-service/internal/infrastructure/db/video_repository_gorm.go:195`). Corrigido no script.

## O que NÃO está feito ainda

1. **Nada foi pushado para o GitHub** — todos os 5 repos são só locais, sem remote configurado. Entrega explícita do hackathon; requer confirmação explícita do usuário antes de criar repos/push (ação pública). **Nota**: os workflows de CI só vão rodar de verdade depois desse push (GitHub Actions não roda em repos locais), e o job `test` de cada um agora inclui os testes de integração reais com testcontainers — vale conferir que rodam certo lá também (containers dentro de `ubuntu-latest` devem funcionar sem configuração extra, já que GitHub-hosted runners têm Docker).
2. **Vídeo de apresentação (≤10min)** — roteiro em `docs/runbook.md`, vídeo em si não gravado.

## Para retomar, nesta ordem sugerida

1. Decidir sobre GitHub push (item 1) — perguntar ao usuário antes. Depois do push, conferir se os workflows de CI passam de verdade no GitHub Actions (só foram validados localmente, rodando os mesmos comandos que os jobs executam, incluindo os testes de integração com testcontainers).
2. Gravar o vídeo (item 2) — agora que o golden path, o caminho de falha, E o teste de carga estão comprovadamente funcionando ao vivo, dá pra gravar seguindo o roteiro de `docs/runbook.md` sem medo de travar no meio. Único item restante do checklist do hackathon além do push pro GitHub.

## Notas úteis para retomar a stack

- Docker Desktop precisa estar aberto (`open -a Docker`).
- Buildar sequencial para não estourar memória: `cd fiapx-saga-orchestrator/deploy/local && COMPOSE_PARALLEL_LIMIT=1 docker compose build`.
- Subir: `docker compose up -d`.
- UIs: RabbitMQ http://localhost:15672 (guest/guest), MinIO console http://localhost:9001 (minioadmin/minioadmin), Mailhog http://localhost:8025.
- Rota da saga é `GET localhost:8084/api/v1/sagas/<video_id>` (já corrigido no runbook).
