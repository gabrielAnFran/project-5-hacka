# Entregável — Hackathon FIAP X: Sistema de Processamento de Vídeos

Documento único com todos os links exigidos pelo enunciado
(`project-docs-claude/enunciado.txt`), para envio ao Portal do Aluno.

## Documentação

- ✅ **Documentação da arquitetura proposta**:
  [`docs/architecture.md`](docs/architecture.md) — visão geral, diagrama de
  fluxo, contrato de eventos, garantias de escalabilidade e não-perda de
  requisição.
  - Diagrama editável (draw.io):
    [`docs/architecture-diagram.html`](docs/architecture-diagram.html)
  - Decisões de arquitetura (ADRs): [`docs/adr/`](docs/adr/)
    - [0001 — Saga orquestrada mesmo sem compensação](docs/adr/0001-orchestrated-saga.md)
    - [0002 — Auth JWT embutida no upload-service](docs/adr/0002-embedded-jwt-auth.md)
    - [0003 — MinIO como object storage](docs/adr/0003-minio-object-storage.md)
    - [0004 — Repositórios separados por serviço](docs/adr/0004-separate-repos-per-service.md)
- ✅ **Script de criação do banco de dados**: resumido em
  [`docs/db-schema.md`](docs/db-schema.md); scripts reais de migração em
  `migrations/*.sql` em cada um dos 4 repositórios de serviço (links abaixo).

## Código — Repositórios GitHub

| Repositório | Papel | Link |
|---|---|---|
| `project-5-hacka` | Documentação e arquitetura (este repositório) | https://github.com/gabrielAnFran/project-5-hacka |
| `fiapx-video-upload-service` | Auth JWT, upload, listagem de status, download | https://github.com/gabrielAnFran/fiapx-video-upload-service |
| `fiapx-video-processing-service` | Extração de frames (ffmpeg) + zip | https://github.com/gabrielAnFran/fiapx-video-processing-service |
| `fiapx-video-notification-service` | Notificação por e-mail | https://github.com/gabrielAnFran/fiapx-video-notification-service |
| `fiapx-saga-orchestrator` | Orquestração da saga + stack local de demonstração | https://github.com/gabrielAnFran/fiapx-saga-orchestrator |

Todos os repositórios são públicos, com CI (GitHub Actions) rodando
lint + testes (unitários e de integração com testcontainers, cobertura
≥80%) + build + análise de qualidade em cada push.

## Apresentação

- ⏳ **Vídeo (≤10min)** apresentando documentação, arquitetura escolhida e o
  projeto funcionando: **TODO — adicionar o link aqui depois de gravar e
  publicar** (YouTube não listado, Google Drive, ou similar).
  - Roteiro sugerido: [`docs/runbook.md`](docs/runbook.md)
  - Script de demonstração ao vivo (golden path + caminho de falha + teste
    de carga, tudo automatizado):
    [`fiapx-saga-orchestrator/scripts/demo.sh`](https://github.com/gabrielAnFran/fiapx-saga-orchestrator/blob/main/scripts/demo.sh)
