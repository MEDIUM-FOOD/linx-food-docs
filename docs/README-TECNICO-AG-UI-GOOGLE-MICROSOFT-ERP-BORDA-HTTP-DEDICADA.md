# Manual técnico por etapa: borda HTTP dedicada do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o boundary FastAPI do slice AG-UI. Ela é responsável por expor discovery público de capabilities, iniciar runs por SSE e reproduzir eventos persistidos por run ou thread.

Em termos operacionais, esta é a porta oficial do produto para AG-UI. Se a borda estiver errada, o resto do runtime nem chega a ser exercitado.

## 2. Superfícies públicas confirmadas

O router dedicado expõe quatro rotas:

- GET /ag-ui/capabilities
- POST /ag-ui/runs
- GET /ag-ui/runs/{run_id}/events
- GET /ag-ui/threads/{thread_id}/events

Todas usam a permissão AGENT_EXECUTE e rate limit da família agent.

## 3. Como o router trata discovery

GET /ag-ui/capabilities recebe executionKind opcional e delega a descoberta ao serviço de capabilities. O comportamento confirmado é falha fechada para executionKind desconhecido e resposta pública sem SQL, DSN ou segredos.

Na prática, isso separa claramente duas responsabilidades:

- discovery informa o que pode ser feito
- run executa o que foi escolhido

## 4. Como o router trata execução

POST /ag-ui/runs faz cinco validações centrais antes de abrir o stream:

1. exige autenticação e permissão
2. exige alguma fonte explícita de configuração
3. resolve correlation_id a partir do request ou gera um novo
4. converte o payload em AgUiRunContext
5. injeta um encoder SSE sobre o orquestrador

O retorno é StreamingResponse com media_type text/event-stream, header X-Correlation-Id e Cache-Control igual a no-cache.

## 5. Como o router trata replay

As rotas de replay recebem um AgUiEventStore canônico e devolvem AgUiReplayResponse já ordenado. O boundary não reimplementa ordenação, sanitização nem storage local. Ele apenas consome a porta de replay configurada pelo provider canônico.

Isso importa porque replay e execução continuam usando o mesmo contrato público, mas responsabilidades diferentes.

## 6. Decisões técnicas importantes

### 6.1. AG-UI usa rota própria

Executar AG-UI em /ag-ui/runs evita acoplamento com endpoints legados de agente e deixa explícito que o contrato é SSE tipado, não resposta JSON arbitrária.

### 6.2. Resume reutiliza o mesmo endpoint

O boundary não cria rota separada de continue. Isso simplifica a superfície pública e mantém todo run AG-UI no mesmo contrato.

### 6.3. Configuração é obrigatória e explícita

Sem yaml_config, yaml_inline_content ou encrypted_data, o router falha com 400. Isso elimina fallback implícito perigoso.

## 7. Erros típicos da borda

Os erros mais úteis desta etapa são:

- 400 quando falta fonte explícita de configuração
- 404 no discovery para executionKind inexistente
- falha de autenticação ou permissão antes de chegar ao orquestrador
- erro de consumo SSE no cliente quando o stream não respeita o contrato

## 8. Diagnóstico recomendado

Para investigar esta etapa:

1. confirme a permissão AGENT_EXECUTE no boundary
2. valide se o payload tem uma fonte de configuração explícita
3. confira o header X-Correlation-Id na abertura do stream
4. verifique se o replay consulta o provider canônico, não memória local improvisada

## 9. Evidências no código

- src/api/routers/ag_ui_router.py
  - Motivo: boundary HTTP do slice.
  - Comportamento confirmado: discovery, execução SSE e replay vivem em rotas próprias sob /ag-ui.
- src/api/routers/ag_ui_router.py
  - Motivo: validação de fonte de configuração.
  - Comportamento confirmado: o router falha com 400 quando o payload não traz yaml_config, yaml_inline_content ou encrypted_data.
- src/api/routers/ag_ui_router.py
  - Motivo: contrato do stream.
  - Comportamento confirmado: a execução responde como text event stream e devolve correlation id no header.
