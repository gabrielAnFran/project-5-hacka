# ADR 0004: Um repositório Git por serviço

## Status

Aceito.

## Contexto

O sistema irmão `pos-*` (mesmo autor) já estabelece o precedente de um
repositório por microsserviço (`pos-os-service`, `pos-billing-service`,
`pos-production-service`, `pos-saga-orchestrator`), unidos por um arquivo
`.code-workspace` na raiz. A alternativa seria um monorepo com os 4
serviços em subpastas.

## Decisão

Seguir o mesmo padrão: 4 repositórios Git independentes
(`fiapx-video-upload-service`, `fiapx-video-processing-service`,
`fiapx-video-notification-service`, `fiapx-saga-orchestrator`), mais este
repositório (`project-5-hacka`) para documentação e o arquivo de workspace
raiz `fiapx-hackathon.code-workspace`.

## Justificativa

- Cada serviço tem seu próprio pipeline de CI, seu próprio versionamento
  semântico e seu próprio histórico de commits — sem risco de um commit em
  um serviço acionar CI/CD de outro serviço não relacionado.
- Reforça o desacoplamento arquitetural: não existe a tentação de importar
  um pacote Go de outro serviço diretamente (cada repositório tem sua
  própria cópia de `internal/infrastructure/messaging/amqp.go`, por
  exemplo) — a única forma de comunicação entre serviços é via evento no
  RabbitMQ, nunca via import de código.
- Consistente com a decisão de arquitetura de seguir o padrão `pos-*``
  completo (ver contexto do hackathon).

## Consequências

- Sem um módulo Go compartilhado, arquivos como `amqp.go`, `config.go` e
  `postgres.go` são duplicados entre os 4 repositórios (mudança em um não
  se propaga automaticamente para os outros — é uma escolha deliberada de
  trade-off, não um descuido).
- Rodar o sistema completo localmente exige clonar os 4 repositórios de
  serviço como diretórios irmãos deste repositório de documentação (ver
  `docs/runbook.md` e `fiapx-saga-orchestrator/deploy/local/README.md`).
