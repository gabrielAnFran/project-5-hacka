# ADR 0001: Orquestração via saga dedicada, mesmo sem compensação

## Status

Aceito.

## Contexto

O pipeline deste sistema é linear: `upload → processamento (ffmpeg) →
notificação`. Diferente do sistema irmão `pos-saga-orchestrator` (oficina
mecânica), que precisa de compensações reais — estornar pagamento, cancelar
orçamento, cancelar ordem de serviço — quando uma etapa falha no meio de
uma transação distribuída, aqui não existe nada para "desfazer": se o
`ffmpeg` falhar ao processar um vídeo, a única ação necessária é reportar a
falha ao usuário. Não há orçamento gerado, pagamento cobrado ou execução
iniciada que precise ser revertida.

## Decisão

Mesmo sem necessidade de compensação, mantivemos um serviço orquestrador
dedicado (`fiapx-saga-orchestrator`) com uma máquina de estados
(`UPLOADED → PROCESSING → COMPLETED|FAILED`), em vez de deixar o
`processing-service` publicar diretamente os comandos para o
`upload-service` e o `notification-service`.

## Justificativa

1. **Consistência com o restante do sistema.** Os demais três serviços já
   seguem o padrão outbox + consumidor idempotente; ter um único ponto que
   decide "o que acontece depois" mantém a mesma disciplina arquitetural em
   vez de introduzir uma exceção só porque o domínio é mais simples.
2. **Trilha de auditoria durável.** `saga_instances` + `saga_history`
   registram cada transição com o evento que a causou — útil para debugar
   "por que este vídeo ficou preso em PROCESSING" sem precisar correlacionar
   logs de 3 serviços diferentes.
3. **Costura para extensões futuras.** Se o sistema crescer para incluir
   verificação de vírus, geração de thumbnail, moderação de conteúdo ou
   qualquer outra etapa intermediária, isso vira apenas novas linhas na
   tabela de transição do orquestrador — nenhum serviço existente precisa
   aprender a chamar um novo vizinho diretamente. Sem o orquestrador, cada
   nova etapa aumentaria o acoplamento em O(n²) entre serviços.

## Consequências

- Um serviço a mais para operar (mais um banco, mais um conjunto de
  binários) do que o estritamente necessário para o fluxo atual.
- Em compensação, nenhum serviço de domínio (`upload`, `processing`,
  `notification`) precisa conhecer a existência dos outros — cada um só
  conhece os eventos que consome e publica, nunca um endereço de outro
  serviço.
