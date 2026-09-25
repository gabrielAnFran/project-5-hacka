# ADR 0002: Autenticação JWT embutida no upload-service

## Status

Aceito.

## Contexto

O requisito funcional exige que o sistema seja "protegido por usuário e
senha". O sistema irmão `project1` tem um precedente de serviço de auth
separado (`azion-auth-function`, uma edge function que emite JWT a partir de
consulta de CPF/CNPJ em outro serviço).

## Decisão

Implementar registro/login com senha (hash bcrypt) e emissão/validação de
JWT diretamente dentro do `video-upload-service`, sem criar um serviço de
auth separado.

## Justificativa

O caso de uso de `azion-auth-function` existe porque múltiplos serviços do
sistema `project1` precisavam validar identidade de clientes por
CPF/CNPJ consultando uma fonte de verdade compartilhada. Aqui não há esse
cenário: o único consumidor de identidade é o próprio `upload-service` (os
outros três serviços nunca recebem requisições HTTP de usuários finais,
apenas eventos internos). Introduzir um serviço de auth separado
adicionaria uma dependência de rede síncrona sem nenhum ganho de
desacoplamento real.

## Consequências

- Mais simples de operar e demonstrar no vídeo de apresentação do
  hackathon (menos um serviço no `docker compose up`).
- Se no futuro outro serviço precisar validar tokens de usuário
  diretamente (hoje nenhum precisa — os workers só confiam em eventos
  assinados pelo próprio broker interno), a extração para um serviço de
  auth dedicado é um refactor localizado, não uma reescrita.
