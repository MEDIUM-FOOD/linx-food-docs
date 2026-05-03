# Manual detalhado da etapa: Borda HTTP dedicada do AG-UI

## 1. O que esta etapa faz

Esta etapa expõe o boundary HTTP proprio do AG-UI. Ela existe para separar discovery, execucao e replay das rotas legadas de agente, workflow ou telas administrativas.

Em linguagem simples: e a porta oficial pela qual uma interface AG-UI entra no sistema.

## 2. Onde ela entra no fluxo

No codigo lido, essa camada fica no router com prefixo /ag-ui. E ela que recebe o request autenticado, resolve correlation_id, monta o contexto de execucao, aciona o orchestrator e devolve o stream SSE.

## 3. O que entra e o que sai

Entradas confirmadas:

- GET /ag-ui/capabilities com executionKind opcional
- POST /ag-ui/runs com AgUiRunRequest
- GET /ag-ui/runs/{run_id}/events
- GET /ag-ui/threads/{thread_id}/events

Saidas confirmadas:

- menu publico de capabilities AG-UI
- stream text/event-stream com eventos AG-UI
- replay de eventos persistidos por run ou por thread
- header X-Correlation-Id no stream de execucao

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. o router e declarado com prefixo /ag-ui;
2. todas as rotas usam permissao AGENT_EXECUTE e rate limit canônico;
3. list_ag_ui_capabilities consulta o service de discovery e falha fechado para execution_kind desconhecido;
4. run_ag_ui rejeita o request se nao houver fonte explicita de configuracao;
5. _resolve_correlation_id herda o correlation_id do request state ou gera um novo;
6. _build_context monta AgUiRunContext com usuario, tenant, execution kind, input, metadata e resume;
7. o orchestrator e executado e cada evento vira SSE pelo encoder;
8. os endpoints de replay consultam o provider canônico do event store e devolvem resposta tipada.

O detalhe mais importante e que o router nao tenta deduzir configuracao nem redirecionar para rotas antigas. O boundary AG-UI existe justamente para ser unico e explicito.

## 5. Decisoes tecnicas importantes

### 5.1. Discovery e execucao estao no mesmo boundary

Capabilities, run e replay convivem no mesmo slice. Isso simplifica integracao para a UI e deixa claro onde comeca e termina o contrato AG-UI.

### 5.2. Falha fechada para configuracao ausente

POST /ag-ui/runs exige yaml_config, yaml_inline_content ou encrypted_data. O backend nao inventa fonte de YAML nem reaproveita estado oculto.

### 5.3. Correlation ID sai no header do stream

O router devolve X-Correlation-Id ja na resposta SSE. Isso ajuda suporte e frontend a amarrarem a execucao ao replay e aos logs.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- executionKind desconhecido em /capabilities devolve erro explicito;
- request sem fonte de configuracao e rejeitado com 400;
- falta de permissao impede acesso ao boundary;
- se o event store canônico estiver mal configurado, o replay nao funciona;
- frontends que tratem AG-UI como GET SSE simples falham, porque aqui a execucao nasce via POST.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- log de capabilities solicitado com execution_kind e usuario;
- log de run recebido com execution_kind, thread_id, run_id e user;
- ausencia do header X-Correlation-Id quando o problema esta antes do stream abrir;
- erro 400 dizendo que faltou yaml_config, yaml_inline_content ou encrypted_data;
- erro 404 ao filtrar capabilities por execution_kind inexistente.

Em linguagem simples: se a UI nao consegue nem entrar direito no slice, o problema costuma estar nesta borda.

## 8. Exemplo pratico guiado

Cenario: uma pagina PDV quer abrir um assistente governado.

1. a pagina chama /ag-ui/capabilities para descobrir o menu publico disponivel;
2. escolhe retail_demo e monta payload compativel com a capability;
3. envia POST para /ag-ui/runs;
4. recebe SSE com X-Correlation-Id;
5. se precisar auditar depois, consulta /ag-ui/runs/{run_id}/events.

O valor desta etapa e transformar AG-UI em surface oficial de produto, e nao em adaptacao improvisada sobre endpoints antigos.

## 9. Evidencias no codigo

- src/api/routers/ag_ui_router.py
  - Simbolo relevante: router = APIRouter(prefix="/ag-ui")
  - Comportamento confirmado: boundary HTTP dedicado do slice AG-UI.
- src/api/routers/ag_ui_router.py
  - Simbolo relevante: list_ag_ui_capabilities
  - Comportamento confirmado: discovery com falha fechada para execution_kind desconhecido.
- src/api/routers/ag_ui_router.py
  - Simbolo relevante: run_ag_ui
  - Comportamento confirmado: rejeicao de run sem fonte explicita de configuracao e stream SSE dedicado.
- src/api/routers/ag_ui_router.py
  - Simbolo relevante: replay_ag_ui_run_events e replay_ag_ui_thread_events
  - Comportamento confirmado: replay de auditoria por run e por thread.
