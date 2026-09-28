# Roteiro do vídeo de apresentação (≤ 10 min)

Roteiro falado, em português, para gravar a apresentação exigida pelo
enunciado (`project-docs-claude/enunciado.txt`): "vídeo de no máximo 10
minutos apresentando Documentação, Arquitetura escolhida e o projeto
funcionando". Ajuste o texto à sua própria fala — isto é um guia, não algo
para ler palavra por palavra. Trechos entre `[...]` são direções de tela
(o que mostrar), não fala.

Tempos são um orçamento, não um cronômetro exato — o importante é fechar
dentro de 10 minutos.

---

## 1. Abertura (0:00 – 0:30)

> "Oi, eu sou o Gabriel. Esse é o projeto que desenvolvi para o hackathon
> FIAP X: um sistema de processamento de vídeos. O desafio era pegar um
> projeto simples — que recebe um vídeo e devolve um `.zip` com os frames
> extraídos — e reconstruí-lo com as práticas de arquitetura que a gente
> viu no curso: microsserviços, mensageria, qualidade de software, CI/CD.
> Vou mostrar rapidamente a documentação, depois a arquitetura que
> escolhi e por quê, e por fim o sistema funcionando de verdade, ao vivo,
> incluindo o caminho de erro e um teste de carga."

`[Tela: nenhuma ainda, ou já abrir no README do repo project-5-hacka]`

---

## 2. Documentação (0:30 – 2:00)

`[Tela: repositório project-5-hacka no GitHub ou no editor]`

> "A documentação do projeto vive num repositório separado, o
> `project-5-hacka` — só documentação, sem código de serviço. Aqui está o
> `architecture.md`, com a visão geral do sistema, o diagrama de fluxo e a
> tabela completa do contrato de eventos entre os serviços."

`[Tela: abrir docs/architecture.md, mostrar o diagrama ASCII e a tabela de
eventos]`

> "Também tem um diagrama editável em draw.io — `architecture-diagram.html`
> — com os 4 serviços, os bancos de dados, o MinIO e o Mailhog, e cada seta
> já rotulada com o nome real do evento que ela representa."

`[Tela: abrir architecture-diagram.html no navegador, ou importar em
app.diagrams.net]`

> "Cada decisão de arquitetura tem seu próprio ADR — Architecture Decision
> Record — explicando o porquê, não só o quê. Por exemplo, o ADR 0001
> explica por que mantive um orquestrador de saga dedicado mesmo esse
> pipeline não precisando de nenhuma compensação, já que não há nada para
> desfazer se o ffmpeg falhar."

`[Tela: mostrar docs/adr/0001-orchestrated-saga.md rapidamente]`

> "E o esquema de banco de dados de cada serviço está resumido em
> `db-schema.md`, com os scripts de migração reais em cada repositório."

`[Tela: db-schema.md, ou um migrations/000001_init.up.sql]`

---

## 3. Arquitetura escolhida (2:00 – 4:00)

`[Tela: diagrama de arquitetura ou architecture.md]`

> "A arquitetura é orientada a eventos: 4 microsserviços Go independentes,
> cada um com seu próprio banco Postgres, coordenados por um orquestrador
> de saga, conversando exclusivamente via RabbitMQ. Nenhum serviço chama
> outro diretamente por HTTP — isso evita uma malha de chamadas síncronas
> e mantém os serviços fracamente acoplados."

> "O fluxo: o `upload-service` recebe o vídeo, grava no MinIO e publica o
> fato `video.uploaded`. O `saga-orchestrator` consome, cria a saga no
> estado `PROCESSING`, e manda o comando `video.process.requested` pro
> `processing-service`. Esse roda o ffmpeg, extrai os frames, gera o
> `.zip`, sobe pro MinIO, e publica `video.processing.completed` — ou
> `.failed`. O orquestrador então dispara dois comandos: atualiza o
> status no `upload-service` e manda o `notification-service` avisar o
> usuário por e-mail."

> "Todo serviço segue o padrão *transactional outbox*: a escrita no banco
> e o evento correspondente acontecem na mesma transação, e um processo
> separado — o `outbox-dispatcher` — publica esses eventos no RabbitMQ.
> Isso garante que, se o serviço cair entre gravar no banco e publicar, o
> evento não se perde: ao reiniciar, o dispatcher retoma de onde parou.
> Combinado com consumidores idempotentes — cada mensagem processada é
> registrada numa tabela `processed_events` antes de qualquer efeito — e
> retry com fila de mensagens mortas (DLQ) depois de 5 tentativas, isso é
> o que garante os dois requisitos centrais do desafio: processar mais de
> um vídeo ao mesmo tempo, e nunca perder uma requisição sob pico."

> "Autenticação é JWT com bcrypt real, embutida no próprio
> `upload-service` — decidi não criar um serviço de auth separado porque
> seria complexidade extra sem ganho real neste escopo (isso também está
> documentado num ADR). Armazenamento de vídeo e do `.zip` é no MinIO,
> compatível com S3."

> "E cada serviço é stateless — o estado vive só no banco — então
> escalar é só subir mais réplicas do worker; o RabbitMQ distribui as
> mensagens entre os consumidores concorrentes na mesma fila."

---

## 4. O projeto funcionando (4:00 – 8:30)

`[Tela: terminal, na pasta fiapx-saga-orchestrator/deploy/local]`

> "Vamos ver funcionando de verdade. Aqui eu já tenho a stack local no ar
> com `docker compose up` — são 18 containers: os 4 serviços (cada um com
> server, worker e dispatcher), Postgres para cada um, RabbitMQ, MinIO e
> Mailhog para simular e-mail."

`[Tela: docker compose ps, mostrando os containers healthy]`

### 4.1 Golden path

> "Primeiro o caminho feliz: registro de usuário, login, upload de um
> vídeo real."

`[Tela: terminal, rodar os comandos do runbook.md — register, login,
upload com um sample.mp4]`

> "A resposta já volta 202 — aceito — porque o upload só grava no banco e
> no MinIO e devolve na hora; o processamento pesado do ffmpeg é
> assíncrono, então não trava a requisição HTTP. Agora eu consulto o
> status..."

`[Tela: GET /api/v1/videos, mostrando status]`

> "Completou em menos de 2 segundos — extraiu os frames, gerou o zip. Se
> eu consultar o estado da saga no orquestrador, vejo o histórico completo
> da transição: `UPLOADED` para `PROCESSING`, `PROCESSING` para
> `COMPLETED`."

`[Tela: GET /api/v1/sagas/<video_id>]`

> "O e-mail de conclusão já chegou no Mailhog..."

`[Tela: abrir http://localhost:8025 no navegador]`

> "E o link de download é uma URL pré-assinada direto pro MinIO, válida
> por um tempo limitado."

`[Tela: baixar o zip pela URL, abrir e mostrar os frames + o vídeo
original dentro]`

### 4.2 Caminho de falha

> "Agora o caminho de erro, que é um dos requisitos: subir um arquivo que
> não é um vídeo válido."

`[Tela: fazer upload de um .txt renomeado para .mp4]`

> "O ffmpeg falha ao processar, o status vai para `FAILED` com a mensagem
> de erro real do ffmpeg gravada, e o usuário recebe um e-mail de falha —
> não silenciosamente perdido."

`[Tela: mostrar o status FAILED com error_message, e o e-mail de falha no
Mailhog]`

### 4.3 Pico de carga (não perder requisição)

> "E o requisito mais importante para picos: o sistema não pode perder
> requisição. Escrevi um script que dispara N uploads concorrentes contra
> o `upload-service` e confirma que todos são aceitos e todos terminam."

`[Tela: rodar fiapx-saga-orchestrator/scripts/load_spike_test.sh 50 ou 100]`

> "Cem uploads simultâneos: os cem foram aceitos na hora, e o worker do
> `processing-service` — que consome a fila um vídeo por vez — deu conta
> de processar todos em poucos segundos, sem perder nenhuma mensagem. Se
> eu olhar o RabbitMQ, todas as filas de mensagens mortas estão zeradas."

`[Tela: RabbitMQ management UI, http://localhost:15672, mostrando as
filas e DLQs em zero]`

---

## 5. Qualidade e CI/CD (8:30 – 9:30)

`[Tela: terminal ou GitHub Actions]`

> "Sobre qualidade: cada serviço tem testes unitários e testes de
> integração reais com testcontainers — sobem Postgres, RabbitMQ, MinIO e
> Mailhog reais em containers Docker, sem mocks, para validar
> repositórios, mensageria, upload real pro MinIO e até um envio de
> e-mail de ponta a ponta contra o Mailhog."

`[Tela: rodar go test -tags=integration ./... num dos repos, ou mostrar o
código de um teste]`

> "E cada repositório tem CI no GitHub Actions: lint, testes, build de
> cada binário Docker, e uma verificação de qualidade com SonarQube
> efêmero — tudo isso já rodando de verdade agora que os 5 repositórios
> estão no GitHub."

`[Tela: aba Actions de um dos repositórios, mostrando o workflow verde]`

---

## 6. Encerramento (9:30 – 10:00)

> "Resumindo: arquitetura orientada a eventos com 4 microsserviços e um
> orquestrador de saga, outbox transacional e idempotência garantindo zero
> perda de requisição sob pico, autenticação JWT, persistência em
> Postgres e MinIO, testes reais com testcontainers, e CI/CD completo nos
> 5 repositórios no GitHub. Os links de todos os repositórios estão na
> descrição. Obrigado!"

`[Tela: lista dos 5 links do GitHub]`

---

## Links dos repositórios (para a descrição do vídeo)

- https://github.com/gabrielAnFran/project-5-hacka (documentação)
- https://github.com/gabrielAnFran/fiapx-video-upload-service
- https://github.com/gabrielAnFran/fiapx-video-processing-service
- https://github.com/gabrielAnFran/fiapx-video-notification-service
- https://github.com/gabrielAnFran/fiapx-saga-orchestrator
