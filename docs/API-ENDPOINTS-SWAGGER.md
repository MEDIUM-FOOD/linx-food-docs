# API, Endpoints e Swagger

Este documento organiza a superfície HTTP real do projeto.
A fonte de verdade continua sendo o app FastAPI e o OpenAPI publicado
em /openapi.json.

## O que a API publica hoje

O mesmo app HTTP publica três coisas ao mesmo tempo.

- endpoints de produto e operação;
- páginas e assets da UI administrativa;
- OpenAPI em /docs, /redoc e /openapi.json.

## Proteção do Swagger

As rotas /docs, /redoc e /openapi.json passam pelo middleware global de
permissões e pelo controle específico de acesso ao Swagger.

Em linguagem simples: o schema existe no app, mas não fica aberto por
conveniência.

## Famílias principais de rota

- /agent para execução e continuidade de agentes.
- /workflow para execução de workflows.
- /rag para ingestão, ETL, reindex, rebuild de auto_config e
  dispatcher de execução.
- /status e /api/v1/status para polling e SSE.
- /config/assembly para draft, validate, confirm, schema, catalog,
  preflight, recommend-tools e objective-to-yaml.
- /config/nl2sql para geração assistida de SQL.
- /config/contract para validação de contrato YAML legado.
- /schema-metadata para leitura e ingestão de metadados de schema.
- /ingestion-runs e /interaction-runs para histórico operacional.
- /client-portal para portal do cliente.
- /channels para multicanal.
- /admin para administração.
- /api/auth para autenticação web e federada.
- /api/whatsapp, /api/instagram e /api/dnit para provisionamento e
  projetos.

## Contratos que merecem atenção

### /rag/execute

É o dispatcher unificado do RAG.
O mesmo endpoint pode entregar resultado síncrono ou aceitar execução
assíncrona, dependendo da operação.

### /rag/ingest e /rag/etl

Essas rotas respondem 202 quando o job é aceito para o worker oficial.
O trabalho pesado não roda no processo HTTP.

### /status e /api/v1/status

Essas rotas são a fachada pública de acompanhamento de task.
Elas servem para polling e SSE.

### /config/assembly/*

Esse grupo cobre o fluxo governado da AST agentic.
Os endpoints principais observados são:

- POST /config/assembly/draft
- POST /config/assembly/objective-to-yaml
- POST /config/assembly/validate
- POST /config/assembly/confirm
- GET /config/assembly/schema
- GET /config/assembly/catalog
- POST /config/assembly/preflight
- POST /config/assembly/recommend-tools

### /config/nl2sql/generate

É a superfície dedicada para NL2SQL.
Ela gera proposta de SQL com diagnósticos, mas não salva nem executa a
query automaticamente.

### /agent/execute e /agent/continue

Essas rotas tratam supervisor clássico e deepagent.
No contrato atual, /agent/execute pode responder HTTP 200 tanto para
resultado final quanto para pausa HIL ou envelope assíncrono.

## Como a autenticação entra no boundary

O boundary aceita mais de uma origem de credencial.

- X-API-Key no header.
- authentication.access_key no YAML resolvido.
- sessão federada no fluxo web.

Isso importa porque o mesmo app atende chamadas técnicas e audiência
humana.

## Respostas síncronas e assíncronas

Nem toda rota longa usa o mesmo padrão.

- RAG de ingestão e ETL usam 202 quando o job é enfileirado.
- /agent pode continuar em 200 mesmo em casos de HIL ou envelope de
  continuidade.

Quando houver execução assíncrona, os campos operacionais mais úteis
costumam ser:

- task_id;
- correlation_id;
- polling_url;
- stream_url;
- cancel_url, quando suportado.

## Erros que mais aparecem

- 400 para YAML ou payload inválido.
- 401 para credencial ausente.
- 403 para permissão insuficiente.
- 404 para recurso ausente ou feature desligada.
- 409 para cancelamento ou conflito de estado.
- 422 para validação Pydantic.
- 429 para rate limit.
- 500 e 503 para falha interna ou dependência indisponível.

## Como rodar e validar

1. Suba a API.
2. Acesse /openapi.json com credencial que tenha leitura do Swagger.
3. Valide uma rota curta, como /config/assembly/catalog.
4. Valide uma rota longa, como /rag/ingest ou /rag/etl.
5. Acompanhe o task_id em /status.
6. Confirme o mesmo correlation_id no retorno e nos logs.

## Evidência no código

- src/api/service_api.py
- src/api/routers/agent_router.py
- src/api/routers/workflow_router.py
- src/api/routers/rag_router.py
- src/api/routers/rag_ingestion_router.py
- src/api/routers/rag_etl_router.py
- src/api/routers/rag_operations_router.py
- src/api/routers/streaming_router.py
- src/api/routers/config_assembly_router.py
- src/api/routers/config_nl2sql_router.py
- src/api/routers/client_portal_router.py
- src/api/security/user_auth.py
- src/api/security/permissions.py

## Lacunas no código

Não encontrado no código.

Onde deveria estar:

- um manifesto administrativo único com rotas, permissões e features
  ativas do ambiente;
- uma exportação automática do OpenAPI enriquecida com permissões e
  papel operacional de cada família.
