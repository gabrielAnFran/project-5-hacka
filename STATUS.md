# Status — FIAP X Video Processing (retomar aqui amanhã)

Última atualização: 2026-09-25.

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

- Os 4 serviços Go: escrito, `gofmt`/`go vet`/`go build` (todos os binários `cmd/{server,worker,outbox-dispatcher}`, exceto notification que não tem dispatcher)/`go test` — **tudo verde** em cada repo.
- Contrato de eventos conferido campo-a-campo nos 4 saltos (`video.uploaded` → saga → `video.process.requested` → processing → `video.processing.completed|failed` → saga → `video.status.completed|failed` + `video.notify.requested` → upload/notification) — **nomes de evento e campos JSON batem exatamente** entre produtor e consumidor em todos os casos, conferido via grep direto no código, não apenas confiando no relato dos agentes que escreveram cada serviço.
- `amqp.go` é byte-a-byte idêntico nos 4 repos (confirmado via `diff`).
- Helm charts das 4 repos: **`helm lint` passa limpo** em todos (só um aviso informativo de "icon is recommended").
- 2 bugs reais encontrados e corrigidos durante a revisão:
  1. `go.mod` de todos os 4 repos foi resolvido para Go 1.25/1.26 pelo `go mod tidy` (por causa do `aws-sdk-go-v2` e deps transitivas), mas os `Dockerfile` ainda apontavam `golang:1.23-*` no estágio de build — corrigido para `golang:1.26-*`/`golang:1.26-bookworm` nos 4 repos.
  2. `docker-compose.yml` (em `fiapx-saga-orchestrator/deploy/local/`) tinha o build context do notification-service apontando para `../../../fiapx-notification-service` (faltava `-video-`) — corrigido para `../../../fiapx-video-notification-service`.
- Todos os 5 repos com working tree limpo após essas correções.

## O que NÃO está feito ainda

1. **Execução end-to-end real via Docker.** O Docker Desktop não estava rodando nesta máquina — tudo acima foi verificado a nível de código/build, **nunca rodei `docker compose up` de verdade**. Próximo passo natural: abrir o Docker Desktop, `cd fiapx-saga-orchestrator/deploy/local && docker compose up --build`, seguir `project-5-hacka/docs/runbook.md` (registro → login → upload → poll status → download → conferir e-mail no Mailhog em http://localhost:8025), e corrigir o que quebrar (é bem provável que algo quebre na primeira tentativa — é a primeira vez que os 4 serviços conversam de verdade via RabbitMQ real).
2. **CI workflows completos** (`.github/workflows/ci.yml` por repositório) — ainda não criados em nenhum dos 4 repos. Template de referência já lido e pronto para reaproveitar: `/Users/franz/development/pos/pos-os-service/.github/workflows/ci.yml` (jobs: `lint` golangci-lint+govet+gofmt, `test` com cobertura, `build` docker para cada `TARGET`, `sonar` — SonarQube efêmero via container na própria action, sem precisar de conta/token externo). Ajustar `go-version` para `1.26` (ou o que cada `go.mod` pedir) e a lista de `TARGET`s por serviço (notification-service só tem `server`+`worker`, sem dispatcher).
3. **Testes de integração reais (testcontainers-go)** — hoje são placeholders/`t.Skip` nos 4 repos:
   - `fiapx-video-upload-service/tests/integration/stub_test.go`
   - `fiapx-video-processing-service/tests/integration/extract_frames_test.go` (já tem a estrutura pronta, só falta o fixture de vídeo real — ver item 4)
   - `fiapx-video-notification-service/tests/integration/placeholder_test.go`
   - `fiapx-saga-orchestrator/tests/integration/saga_flow_test.go`
   Referência de como os repos irmãos fazem isso: `pos-os-service/tests/integration/*.go` (Postgres/RabbitMQ via testcontainers-go, atrás de `//go:build integration`).
4. **Fixture de vídeo real para o teste de ffmpeg** — `ffmpeg` **não está instalado localmente** nesta máquina (`which ffmpeg` não encontrou nada), então não deu pra gerar o `sample.mp4` sintético agora. Opções para amanhã: `brew install ffmpeg` e gerar com `ffmpeg -f lavfi -i testsrc=duration=3:size=64x64:rate=10 tests/fixtures/sample.mp4`, ou gerar dentro de um container Docker (a imagem final do processing-service já tem ffmpeg via alpine).
5. **Teste de carga (load-spike smoke test)** — script disparando N uploads concorrentes contra a stack rodando, para demonstrar "não perde requisição sob pico". Ainda não escrito. Provavelmente um script simples em `fiapx-saga-orchestrator/scripts/` ou dentro do runbook.
6. **Nada foi pushado para o GitHub** — todos os 5 repos são só locais (`git init` + commits), sem remote configurado. Isso é entrega explícita do hackathon ("projeto deve ser versionado no Github"), então em algum momento precisa criar os repos no GitHub e dar push (ação que requer confirmação explícita antes de fazer, por ser uma ação visível/pública).
7. **Vídeo de apresentação (≤10min)** — roteiro já escrito em `docs/runbook.md`, mas o vídeo em si obviamente não foi gravado.

## Para retomar amanhã, nesta ordem sugerida

1. Rodar o golden path de verdade (item 1 acima) — é o que vai revelar se algo no contrato de eventos ou na config do compose ainda está errado apesar da revisão estática.
2. CI workflows (item 2) — mecânico, rápido, baixo risco.
3. Testcontainers (item 3) + fixture ffmpeg (item 4) — mais trabalhoso.
4. Load-spike test (item 5).
5. Decidir sobre GitHub push (item 6) — perguntar ao usuário antes.
6. Gravar o vídeo (item 7).
