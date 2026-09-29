# Hackathon FIAP X — Documento de Entrega

**PÓS TECH — Arquitetura de Software**
**Data de Entrega:** 29/09/2026
**Versão:** 1

---

## Dados do Grupo

**Integrantes:**

| Nome Completo | RM |
|---|---|
| Gabriel Antunes França | RM369678 |

---

## 1. Vídeo de Demonstração

**Link:** <https://drive.google.com/file/d/1TjqH0rwqwOp3uPZniKSsamxJZ7FZxIsp/view?usp=sharing>

**O vídeo demonstra:**

- Documentação da arquitetura (visão geral, diagrama, contrato de eventos, ADRs).
- Arquitetura escolhida: microsserviços orientados a eventos, saga orquestrada, outbox
  transacional, idempotência.
- O projeto funcionando ao vivo: golden path completo (registro → login → upload →
  processamento assíncrono → download), caminho de falha (upload inválido → `FAILED` com
  motivo real do erro → notificação), e um teste de carga com uploads concorrentes provando
  que o sistema não perde requisição sob pico.
- Toda a demonstração ao vivo é automatizada por
  [`fiapx-saga-orchestrator/scripts/demo.sh`](https://github.com/gabrielAnFran/fiapx-saga-orchestrator/blob/main/scripts/demo.sh),
  que registra usuário, faz upload, acompanha o status, consulta a saga, baixa o resultado,
  repete o fluxo com um arquivo inválido, e dispara o teste de carga — explicando cada passo e
  mostrando o retorno real de cada chamada.

---

## 2. Repositórios (5 repos)

### Repo 1 — Documentação e Arquitetura
**URL:** https://github.com/gabrielAnFran/project-5-hacka
**Descrição:** Repositório sem código de serviço — documentação da arquitetura, ADRs, esquema
de banco de dados, runbook de execução local, diagrama editável (draw.io) e este documento de
entrega.

### Repo 2 — Video Upload Service
**URL:** https://github.com/gabrielAnFran/fiapx-video-upload-service
**Descrição:** Autenticação (JWT, bcrypt real), recebe o upload do vídeo (streaming direto
para o MinIO), grava o registro com status `UPLOADED` e o evento `video.uploaded` no outbox
na mesma transação, expõe listagem de status paginada e URL de download pré-assinada. Go
1.26, Gin, GORM/PostgreSQL, aws-sdk-go-v2 (MinIO).

CI/CD:

- `.github/workflows/ci.yml` — `go vet`, `gofmt`, **golangci-lint v2** (staticcheck, errcheck,
  gosec, bodyclose — bloqueante), testes unitários + de integração reais (testcontainers-go:
  Postgres, RabbitMQ e MinIO reais, não mockados) rodando juntos via
  `go test -tags=integration ./... -coverpkg=$PKGS` (com `cmd/*` excluído do cálculo — código
  de wiring de `main()`, sem lógica própria) — cobertura real de **84,7%** (seção 3.1), build
  Docker multi-stage (distroless, non-root) para os 3 binários (`server`, `worker`,
  `outbox-dispatcher`), e um job **SonarQube** (instância efêmera self-hosted na própria
  pipeline, quality gate real via API — informativo, não bloqueante; o gate bloqueante é o
  golangci-lint).

### Repo 3 — Video Processing Service
**URL:** https://github.com/gabrielAnFran/fiapx-video-processing-service
**Descrição:** Worker que baixa o vídeo do MinIO, executa `ffmpeg` extraindo um frame por
segundo, compacta o resultado em `.zip`, envia para o MinIO, e publica o fato
`video.processing.completed` ou `.failed`. Go 1.26, GORM/PostgreSQL, ffmpeg via `exec`.

CI/CD:

- `.github/workflows/ci.yml` — mesmo padrão de qualidade (golangci-lint v2 bloqueante),
  testes unitários + de integração reais via testcontainers (Postgres, RabbitMQ, MinIO) **e**
  um teste com o binário `ffmpeg` real (instalado no runner via `apt-get`) contra um vídeo de
  amostra real — cobertura de **82,2%** (seção 3.1), build Docker (imagem final com ffmpeg via
  Alpine).

### Repo 4 — Video Notification Service
**URL:** https://github.com/gabrielAnFran/fiapx-video-notification-service
**Descrição:** Consumidor terminal (sem outbox próprio) que envia e-mail ao usuário quando o
processamento termina, com sucesso ou falha. Go 1.26, GORM/PostgreSQL, `net/smtp`.

CI/CD:

- `.github/workflows/ci.yml` — mesmo padrão de qualidade, testes de integração reais
  (Postgres, RabbitMQ, e um SMTP real via **Mailhog** em container — o teste confirma a entrega
  do e-mail consultando a API HTTP do Mailhog, não apenas "sem erro" do cliente SMTP) —
  cobertura de **80,6%** (seção 3.1).

### Repo 5 — Saga Orchestrator
**URL:** https://github.com/gabrielAnFran/fiapx-saga-orchestrator
**Descrição:** Coordena o fluxo via máquina de estados própria
(`UPLOADED → PROCESSING → COMPLETED|FAILED`), com `saga_instances`/`saga_history` como fonte
de verdade auditável. Go 1.26, Gin, GORM/PostgreSQL. Também hospeda o `docker-compose.yml`
local (`deploy/local/`) que sobe a stack completa dos 4 serviços + infraestrutura (18
containers) para desenvolvimento e demonstração, o script de teste de carga
(`scripts/load_spike_test.sh`) e o script de demonstração ao vivo (`scripts/demo.sh`).

CI/CD:

- `.github/workflows/ci.yml` — mesmo padrão de qualidade, testes de integração reais
  (Postgres, RabbitMQ), incluindo um teste de fluxo completo que liga um consumidor real ao
  `HandleEvent` real (como o `cmd/worker` faz de verdade) e verifica de ponta a ponta:
  publicar um evento → saga real transiciona no Postgres → outbox real com o comando
  esperado — cobertura de **82,9%** (seção 3.1, já estava acima de 80% antes desta rodada).

---

## 3. Infraestrutura

| Componente | Status | Detalhes |
|---|---|---|
| Docker Compose local | ✅ Concluído | `fiapx-saga-orchestrator/deploy/local/docker-compose.yml` sobe os 4 serviços (cada um com `server`/`worker`/`outbox-dispatcher`, exceto o notification-service que não tem dispatcher) + 4 PostgreSQL + RabbitMQ + MinIO + Mailhog — 18 containers, verificado rodando de ponta a ponta (não só revisão estática) |
| Mensageria | ✅ Concluído | RabbitMQ 3, exchange topic `video.events`, exchange de retry `video.retry`, DLQ por serviço após 5 tentativas. Contrato de eventos conferido campo-a-campo nos 4 saltos — nomes de evento e payload batem exatamente entre produtor e consumidor |
| Bancos de dados | ✅ Concluído | PostgreSQL 16, um banco por serviço (`upload_service`, `processing_service`, `notification_service`, `saga_orchestrator`) — nenhum serviço acessa banco de outro |
| Object Storage | ✅ Concluído | MinIO (compatível com S3) — bucket `fiapx-videos`, prefixos `raw/` (vídeo original) e `processed/` (zip resultante); download via URL pré-assinada |
| Dockerfiles | ✅ Concluído | Multi-stage em todos os 4 repos; imagem final distroless/non-root (upload, notification, saga-orchestrator) ou Alpine com ffmpeg (processing-service) |
| Helm charts | ✅ Concluído | `charts/<serviço>/` em cada um dos 4 repos — `helm lint` passa limpo em todos. Não aplicados a nenhum cluster Kubernetes real (fora do escopo desta entrega; o enunciado aceita Docker Compose como alternativa válida à stack de containers) |
| Verificação de qualidade no CI | ✅ Concluído (4/4 repos) | **golangci-lint v2** (staticcheck, errcheck, gosec, bodyclose) bloqueante no job `lint` de todos os 4 repositórios. Job **SonarQube** (instância efêmera self-hosted, quality gate real via API, informativo) também presente nos 4 |
| Cobertura de testes | ✅ **≥80% em todos os 4 repositórios** | Ver seção 3.1 — medida repo-wide, não só da lógica de negócio |
| Testes de integração reais | ✅ Concluído (4/4 repos) | testcontainers-go (Postgres, RabbitMQ, MinIO, Mailhog reais via Docker) em todos os 4 repos — nenhum mock de infraestrutura |
| Teste de carga | ✅ Concluído | `scripts/load_spike_test.sh` — verificado com 20, 50 e 100 uploads simultâneos: todos aceitos (202) e concluídos, 0 mensagens em qualquer fila de mensagens mortas (DLQ) |
| Observabilidade (métricas/dashboards) | ⏳ Pendente | Logs estruturados (`log/slog`) presentes; Prometheus/Grafana ou stack equivalente (sugestão do enunciado, não obrigatória) não instrumentados nesta entrega |
| Deploy automatizado (CD) | ⏳ Não aplicável a esta entrega | O enunciado não exige deploy em nuvem; a demonstração é local via Docker Compose |

### 3.1. Cobertura de testes (medida, não estimada)

Medição real: `go test -tags=integration ./... -coverpkg=$PKGS -coverprofile=coverage.out`,
com `$PKGS` = `go list ./...` **excluindo pacotes `cmd/*`** (wiring de `main()`, sem lógica
própria — mesma convenção usada nos repositórios irmãos `pos-*`). Roda testes unitários e
testes de integração (testcontainers-go, Postgres/RabbitMQ/MinIO/Mailhog reais) juntos, numa
única execução — é o comando que os 4 pipelines de CI agora usam de verdade (job `test`), com
um passo adicional que **falha o build se a cobertura cair abaixo de 80%**.

| Serviço | Antes desta rodada | Depois | Δ |
|---|---|---|---|
| Video Upload Service | 62,1% | **84,7%** | +22,6 pp |
| Video Processing Service | 43,0% | **82,2%** | +39,2 pp |
| Video Notification Service | 72,9% | **80,6%** | +7,7 pp |
| Saga Orchestrator | 82,9% | **82,9%** | já estava acima de 80% |

**Todos os 4 repositórios atingiram ≥80% de cobertura repo-wide.**

**Achado relevante — Video Processing Service:** este repositório nunca tinha recebido uma
suíte de testcontainers (só existia o teste com o `ffmpeg` real). Seu repositório Postgres,
cliente RabbitMQ e cliente MinIO estavam em 0% de cobertura — a causa raiz de sua cobertura
total (43,0%) estar bem abaixo dos demais. Corrigido escrevendo a suíte completa (Postgres +
RabbitMQ + MinIO), no mesmo padrão dos outros 3 repositórios.

**Achados relevantes — bugs reais encontrados durante o trabalho de cobertura e execução
(não cosméticos):**

1. **Upload de vídeo retornava 500 sempre** (Video Upload Service): `go.mod` tinha uma
   incompatibilidade de versões entre `aws-sdk-go-v2/service/internal/checksum`/`s3shared` e
   `service/s3`, quebrando a montagem do cliente S3 com o erro `not found: S3100Continue` —
   falha 100% client-side, antes de qualquer I/O de rede. Só foi descoberto ao rodar o golden
   path de verdade contra a stack real, não na revisão estática de código.
2. **URL de download pré-assinada inutilizável fora da rede do Docker**: a URL vinha assinada
   com o hostname interno do docker-compose (`minio:9000`), que não resolve do host, e a
   assinatura SigV4 impede trocar o hostname depois. Corrigido com um segundo cliente S3
   dedicado à assinatura, apontado para um endpoint público configurável.
3. **Imagens `minio/minio`/`minio/mc` pararam de existir no Docker Hub** (MinIO tornou-se
   "source-only" a partir de 15/10/2025) — a stack local não subia mais do zero. Corrigido
   trocando para `bitnamilegacy/minio`/`minio-client`.

---

## 4. Diagramas

Diagrama de arquitetura (4 microsserviços, bancos, RabbitMQ, MinIO, Mailhog, com cada seta
rotulada pelo nome real do evento/comando) disponível em duas formas:

- Versão em texto/ASCII, no corpo de
  [`docs/architecture.md`](docs/architecture.md#visão-geral).
- Versão editável em [draw.io](https://app.diagrams.net):
  [`docs/architecture-diagram.html`](docs/architecture-diagram.html) (abrir no navegador, ou
  importar/arrastar em app.diagrams.net para editar).

```
   usuário ──HTTP──► video-upload-service ──fato: video.uploaded──► saga-orchestrator
                            ▲                                              │
                            │ comandos: video.status.completed|failed     │ comando:
                            │                                              │ video.process.requested
                            │                                              ▼
                     saga-orchestrator ◄──fatos: video.processing.completed|failed── video-processing-service
                            │
                            └─comando: video.notify.requested──► video-notification-service
```

---

## 5. Decisões Arquiteturais

Registradas como ADRs completos em
[`docs/adr/`](https://github.com/gabrielAnFran/project-5-hacka/tree/main/docs/adr):

### 5.1. [ADR 0001 — Saga orquestrada mesmo sem compensação](https://github.com/gabrielAnFran/project-5-hacka/blob/main/docs/adr/0001-orchestrated-saga.md)

O pipeline (`upload → processamento → notificação`) é linear — se o `ffmpeg` falhar, não há
nada para desfazer (sem orçamento gerado ou pagamento cobrado, diferente do sistema irmão
`pos-saga-orchestrator`). Ainda assim, mantivemos um orquestrador dedicado com máquina de
estados própria, em vez de deixar o `processing-service` publicar diretamente os comandos
para os outros dois serviços — pela consistência com o padrão outbox + idempotência já
adotado nos demais serviços, e porque um ponto único de decisão evita uma malha de
dependências entre os 3 serviços de domínio.

### 5.2. [ADR 0002 — Autenticação JWT embutida no upload-service](https://github.com/gabrielAnFran/project-5-hacka/blob/main/docs/adr/0002-embedded-jwt-auth.md)

Decisão de **não** criar um serviço de autenticação separado: JWT com bcrypt real, embutido no
próprio `upload-service`. Um quinto serviço só para autenticação seria complexidade adicional
sem ganho real neste escopo — não há múltiplos serviços de domínio que precisem compartilhar
identidade entre si além da validação do token.

### 5.3. [ADR 0003 — MinIO como object storage](https://github.com/gabrielAnFran/project-5-hacka/blob/main/docs/adr/0003-minio-object-storage.md)

MinIO (compatível com S3) para vídeo original e `.zip` resultante, em vez de armazenar bytes
no Postgres ou num volume compartilhado entre containers — separa dados binários grandes do
banco relacional e permite URLs de download pré-assinadas sem que o `upload-service` precise
fazer streaming do arquivo de volta para o cliente.

### 5.4. [ADR 0004 — Repositórios separados por serviço](https://github.com/gabrielAnFran/project-5-hacka/blob/main/docs/adr/0004-separate-repos-per-service.md)

Cada microsserviço em seu próprio repositório Git (em vez de um monorepo), seguindo o padrão
já validado nos repositórios irmãos `pos-*` — deploy e versionamento independentes por
serviço, CI/CD isolado por repositório.

### 5.5. Escolhas de tecnologia

- **Go 1.26 / Gin / GORM** em todos os serviços.
- **RabbitMQ** (não Kafka) — operação mais simples, DLX/retry nativos, throughput mais que
  suficiente para o volume esperado.
- **PostgreSQL** — um banco por serviço, sem schema compartilhado.
- **Padrão Transactional Outbox** em todos os serviços com escrita própria (exceto o
  notification-service, consumidor terminal sem produção de eventos) — garante que a
  atualização de estado e a publicação do evento correspondente nunca fiquem inconsistentes
  entre si, mesmo em caso de falha do broker.
- **Idempotência** via tabela `processed_events`, checada antes de qualquer processamento —
  necessário porque RabbitMQ garante *at-least-once delivery*.

---

## 6. Fluxo do Sistema (golden path e caminho de falha)

**Golden path:**
```
POST /api/v1/auth/register, /api/v1/auth/login → POST /api/v1/videos (202, streaming pro MinIO)
→ video.uploaded → saga PROCESSING → video.process.requested
→ ffmpeg extrai frames + zip → video.processing.completed
→ saga COMPLETED → video.status.completed (upload-service) + video.notify.requested (e-mail)
→ GET /api/v1/videos mostra COMPLETED → GET /api/v1/videos/{id}/download retorna URL pré-assinada
```
Verificado ao vivo contra a stack real: do upload ao `COMPLETED`, menos de 2 segundos (3
frames extraídos de um vídeo de amostra), e-mail de conclusão recebido no Mailhog, `.zip`
baixado com sucesso.

**Caminho de falha:**
```
POST /api/v1/videos (arquivo inválido, ex: .txt renomeado para .mp4) → 202 aceito
→ ffmpeg falha ao processar → video.processing.failed (com error_message real do ffmpeg)
→ saga FAILED → video.status.failed + video.notify.requested (tipo FAILED)
→ GET /api/v1/videos mostra FAILED com error_message → e-mail de falha recebido no Mailhog
```
Verificado ao vivo da mesma forma, com o `error_message` sendo a saída real do `ffmpeg`, não
uma mensagem genérica.

**Teste de carga (pico de requisições):** `fiapx-saga-orchestrator/scripts/load_spike_test.sh` dispara N uploads
concorrentes e confirma que todos são aceitos e todos completam — verificado com N=20, 50 e
100, sem nenhuma requisição perdida ou rejeitada, e 0 mensagens em qualquer DLQ em qualquer
execução.

---

## 7. Como Executar Localmente

```bash
# Clonar os 5 repositórios como diretórios irmãos
mkdir fiapx-hackathon && cd fiapx-hackathon
git clone https://github.com/gabrielAnFran/project-5-hacka
git clone https://github.com/gabrielAnFran/fiapx-video-upload-service
git clone https://github.com/gabrielAnFran/fiapx-video-processing-service
git clone https://github.com/gabrielAnFran/fiapx-video-notification-service
git clone https://github.com/gabrielAnFran/fiapx-saga-orchestrator

# Subir a stack completa (4 serviços + Postgres x4 + RabbitMQ + MinIO + Mailhog)
cd fiapx-saga-orchestrator/deploy/local
# COMPOSE_PARALLEL_LIMIT=1 evita estouro de memória do Docker Desktop ao compilar os binários Go
COMPOSE_PARALLEL_LIMIT=1 docker compose build
docker compose up -d

# Rodar a demonstração completa (golden path + caminho de falha + teste de carga)
# de uma vez, com explicação de cada passo:
../../scripts/demo.sh
```

Roteiro manual passo a passo (curl) em
[`project-5-hacka/docs/runbook.md`](https://github.com/gabrielAnFran/project-5-hacka/blob/main/docs/runbook.md).
