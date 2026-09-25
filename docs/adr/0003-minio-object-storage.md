# ADR 0003: MinIO para armazenamento de vídeos e arquivos .zip

## Status

Aceito.

## Contexto

O requisito técnico pede persistência de dados e uma arquitetura escalável.
Vídeos originais e os `.zip` de frames extraídos são arquivos binários
potencialmente grandes — não são um bom encaixe para colunas `bytea` em
Postgres (infla o banco, degrada backups/replicação, e nenhum dos serviços
irmãos (`pos-*`, `project1`) já resolve esse problema, já que nenhum deles
lida com upload de arquivo).

## Decisão

Usar MinIO (armazenamento de objetos compatível com S3) como stack local via
Docker Compose, acessado pelos serviços via `aws-sdk-go-v2` apontado para o
endpoint do MinIO.

## Justificativa

- Object storage é o lugar correto para blobs — Postgres fica responsável
  apenas por metadados (nome, status, chave do objeto), não pelo conteúdo.
- Por ser compatível com a API S3, o mesmo código de cliente funciona sem
  alteração contra um S3/R2/Object Storage real em produção — só muda o
  endpoint e as credenciais via variável de ambiente.
- Upload/download usam streaming (o `video-upload-service` nunca carrega o
  vídeo inteiro em memória; usa upload multipart via
  `s3/feature/manager.Uploader`) e URLs pré-assinadas para download, o que
  mantém o próprio serviço de API fora do caminho crítico de transferência
  de dados grandes.

## Consequências

- Mais um componente na stack local (`minio` + container de inicialização
  `minio-init` para criar o bucket).
- Dois serviços (`upload-service`, `processing-service`) precisam de
  credenciais MinIO — mantidas como variáveis de ambiente compartilhadas no
  `docker-compose.yml` local, e como Secrets do Kubernetes nos charts Helm.
