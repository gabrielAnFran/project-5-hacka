# FIAP X — Sistema de Processamento de Vídeos

Repositório de documentação e arquitetura do projeto de hackathon FIAP X: um
sistema que recebe upload de vídeos, extrai frames de forma assíncrona e
disponibiliza o resultado como um `.zip` para download.

O enunciado original está em
[`project-docs-claude/enunciado.txt`](project-docs-claude/enunciado.txt).

## Arquitetura

Ver [`docs/architecture.md`](docs/architecture.md) para a visão geral,
diagrama de fluxo, contrato de eventos e a justificativa de escalabilidade.
Decisões de arquitetura individuais estão em [`docs/adr/`](docs/adr/).
Esquema de banco de dados por serviço em [`docs/db-schema.md`](docs/db-schema.md).

## Repositórios do sistema

Este repositório não contém código de serviço — apenas documentação. O
código vive em 4 repositórios irmãos (clonar todos lado a lado com este):

- [`fiapx-video-upload-service`](../fiapx-video-upload-service) — auth, upload, listagem, download
- [`fiapx-video-processing-service`](../fiapx-video-processing-service) — extração de frames (ffmpeg) + zip
- [`fiapx-video-notification-service`](../fiapx-video-notification-service) — notificação por e-mail
- [`fiapx-saga-orchestrator`](../fiapx-saga-orchestrator) — coordenação via saga + stack local de desenvolvimento

O arquivo `../fiapx-hackathon.code-workspace` (na raiz de
`/Users/franz/development/pos/`) abre os 5 repositórios juntos no editor.

## Rodando localmente

```bash
cd ../fiapx-saga-orchestrator/deploy/local
docker compose up --build
```

Ver [`fiapx-saga-orchestrator/deploy/local/README.md`](../fiapx-saga-orchestrator/deploy/local/README.md)
para o roteiro completo de demonstração (registro → login → upload → status → download).

## Entregáveis do hackathon

- ✅ Documentação da arquitetura — `docs/architecture.md` + `docs/adr/`
- ✅ Scripts de criação de banco de dados — `migrations/*.sql` em cada repositório de serviço, resumidos em `docs/db-schema.md`
- ✅ Código versionado no GitHub — 4 repositórios de serviço + este
- ✅ Vídeo de apresentação (≤10min) — link em [`ENTREGAVEL.md`](ENTREGAVEL.md#1-vídeo-de-demonstração)
