# Manual técnico, operacional e de uso: AG-UI com referencial Google e Microsoft aplicado ao ERP

## 1. O que é esta feature

No plano técnico, o slice AG-UI deste projeto é uma implementação dedicada de interface agentic por eventos sobre a rota POST /ag-ui/runs. O backend recebe um request tipado, cria um contexto imutável de execução, resolve um adapter por executionKind, emite eventos AG-UI ordenados e serializa tudo em Server-Sent Events. O frontend em JavaScript puro consome esse stream, atualiza store compartilhado, renderiza sidecar, linha do tempo de tools, snapshots de estado e áreas principais de telas ERP.

## 2. Que problema ela resolve

Tecnicamente, a feature resolve o problema de sincronizar backend agentic e interface sem polling, sem payloads ad hoc por página e sem vazamento da lógica de domínio para o navegador. Ela também resolve um problema de governança: o frontend não escolhe SQL, não cria correlation_id e não materializa dashboard por HTML arbitrário. Tudo isso fica controlado pelo backend e pelo contrato tipado.

## 3. Referencial externo: AG-UI oficial, Microsoft e Google

O referencial externo lido na documentação oficial do protocolo descreve AG-UI como padrão aberto, leve e orientado por eventos para conectar aplicações voltadas ao usuário a backends agentic, com suporte documentado a integrações first-party para Microsoft Agent Framework e Google ADK. Esse ponto é importante para leitura técnica correta do projeto.

O slice atual não usa Microsoft Agent Framework nem Google ADK no código. Não há import, SDK ou adapter desses frameworks na implementação AG-UI local lida. A aproximação com o ecossistema oficial acontece no nível de arquitetura e contrato: lifecycle do run, eventos tipados, sincronização de estado, deltas, tool events, interrupções e transporte HTTP em streaming. Em termos práticos, o projeto implementa sua própria ponte compatível com a lógica do protocolo, adaptada à stack FastAPI + HTML estático + JavaScript puro.

## 4. Como a feature funciona por dentro

O caminho executável real tem cinco etapas.

### 4.1. Recepção do run

O router dedicado recebe um AgUiRunRequest, valida permissão de execução agentic, exige uma fonte explícita de configuração entre yaml_config, yaml_inline_content e encrypted_data, resolve correlation_id e monta AgUiRunContext.

### 4.2. Início do lifecycle

O AgUiRunOrchestrator emite RUN_STARTED assim que recebe o contexto. Isso garante que o frontend saiba imediatamente que a execução começou, mesmo antes de qualquer tool ou resposta textual.

### 4.3. Resolução do adapter

O orchestrator escolhe um adapter a partir de executionKind. No slice atual, o router registra apenas retail_demo. Se não existir adapter compatível, a execução termina com RUN_ERROR e código AG_UI_ADAPTER_NOT_FOUND.

### 4.4. Tradução do domínio

O adapter de varejo interpreta o input, valida capability, bloqueia chaves de SQL livre, resolve configurações do ambiente PDV, escolhe a query governada ou o caminho de dashboard dinâmico e começa a emitir eventos AG-UI.

### 4.5. Encerramento terminal

Se o adapter não emitir um evento terminal próprio, o orchestrator fecha a execução com RUN_FINISHED e outcome de sucesso. Se o adapter lançar AgUiExecutionError, o orchestrator envia RUN_ERROR com o código de domínio. Se houver exceção inesperada, envia AG_UI_RUN_FAILED.

## 5. Submódulos técnicos relevantes

### 5.1. Modelos de protocolo

Essa camada está em ag_ui_models. Ela define request, resume input, mensagens, deltas JSON Patch, outcomes de sucesso e interrupção, eventos de texto, eventos de tool, snapshots de estado, snapshots de atividade, custom e raw events.

O comportamento confirmado relevante é este.

1. Os modelos são estritos e rejeitam campos extras.
2. A serialização usa aliases camelCase oficiais.
3. TEXT_MESSAGE_CONTENT exige delta não vazio.
4. RUN_FINISHED suporta outcome interrupt com lista de interrupts.
5. STATE_DELTA usa lista de operações JSON Patch.

### 5.2. Borda HTTP

Essa camada está em ag_ui_router. Ela existe para separar AG-UI dos endpoints legados de agente e streaming. O endpoint real é POST /ag-ui/runs e responde com media type text/event-stream.

O comportamento confirmado relevante é este.

1. Sem autenticação, o endpoint falha fechado.
2. Sem fonte explícita de configuração, falha com 400.
3. O correlation_id vem do request state ou é gerado no backend.
4. O header X-Correlation-Id volta na resposta.

### 5.3. Encoder SSE

Essa camada está em ag_ui_event_encoder. Ela serializa cada evento como par event/data e fecha cada mensagem com linha em branco, no formato esperado por consumidores SSE. Isso simplifica o cliente de frontend e reforça a ideia de que o backend transmite um BaseEvent completo por mensagem.

### 5.4. Orquestrador de run

Essa camada está em ag_ui_run_orchestrator. Ela é propositalmente cega ao domínio. Seu único trabalho é controlar o lifecycle AG-UI.

O comportamento confirmado relevante é este.

1. Emite RUN_STARTED antes de chamar o adapter.
2. Considera RUN_FINISHED e RUN_ERROR como eventos terminais.
3. Fecha automaticamente com RUN_FINISHED quando o adapter não encerra explicitamente.
4. Converte ausência de adapter e erro de domínio em RUN_ERROR tipado.

### 5.5. Adapter de varejo

Essa camada está em ag_ui_retail_demo_adapter. Ela concentra a implementação de negócio do slice atual.

O comportamento confirmado relevante é este.

1. Capabilities aprovadas são sales_summary, checkout_funnel, catalog_opportunities, customer_segments e dashboard_dynamic.
2. Não há execução de SQL livre vinda do browser.
3. Cada capability resolve uma query_id aprovada em catálogo fechado.
4. O executor usa a factory canônica de dyn_sql.
5. dashboard_dynamic não usa executor SQL direto; passa pela materialização de DashboardSpec.

Observação importante: customer_segments existe no catálogo do adapter, mas o slice lido de telas estáticas não confirmou uma página ERP específica usando essa capability hoje.

### 5.6. Materialização de dashboard

Essa camada está em ag_ui_dashboard_materialization e em ag_ui_dashboard_models. Ela valida a spec e a converte em eventos reconstituíveis para o frontend.

O comportamento confirmado relevante é este.

1. Snapshot inicial nasce com status materializing.
2. Em caso de spec válida, o backend publica evento de validação, deltas de spec, data sources e widgets, e depois marca status ready.
3. Em caso de spec inválida, o backend troca status para validation_failed, devolve erros estruturados e emite mensagem textual curta.

### 5.7. Runtime compartilhado do frontend

Essa camada está em ag-ui-client, ag-ui-sidecar-chat, ag-ui-retail-demo-page e páginas HTML estáticas do admin. Ela permite que várias telas ERP consumam o mesmo protocolo sem reescrever cliente SSE e sidecar.

O comportamento confirmado relevante é este.

1. O browser faz POST para /ag-ui/runs, não GET EventSource clássico.
2. O cliente lê blocos SSE incrementais do body.
3. O browser não gera correlation_id.
4. O sidecar adapta interrupções AG-UI ao painel de HIL compartilhado.
5. O controller base monta payload com screenId, capability, período, metadata e fonte de configuração vinda do layout mestre.

## 6. Contrato operacional do request

O request real aceito pelo endpoint tem estes campos principais.

1. threadId: identidade lógica da thread da tela.
2. runId: identidade do run específico.
3. executionKind: chave usada para resolver o adapter.
4. user_email: usuário operacional da execução.
5. input: payload de negócio.
6. metadata: contexto de tela.
7. yaml_config ou yaml_inline_content ou encrypted_data: fonte explícita de configuração.

No runtime local das telas ERP, o controller base monta payloads com executionKind igual a retail_demo, metadata contendo screenId, screenTitle, yamlPath e inputMode, e input contendo capability, parameters, context e, no caso do dashboard, dashboardSpec.

## 7. Lifecycle de eventos no slice atual

O fluxo mais comum das telas fixas de ERP segue esta ordem.

1. RUN_STARTED.
2. STEP_STARTED.
3. TOOL_CALL_START.
4. TOOL_CALL_ARGS.
5. TOOL_CALL_END.
6. TOOL_CALL_RESULT.
7. STATE_SNAPSHOT.
8. TEXT_MESSAGE_START.
9. TEXT_MESSAGE_CONTENT.
10. TEXT_MESSAGE_END.
11. STEP_FINISHED.
12. RUN_FINISHED.

No dashboard dinâmico, em vez de tool timeline simples, a execução ganha CUSTOM events de dashboard, STATE_SNAPSHOT inicial e várias STATE_DELTA para spec, dataSources, widgets e status final ready.

## 8. Como usar em telas ERP do projeto

### 8.1. Padrão de montagem de uma tela nova

O padrão comprovado no projeto é este.

1. Criar uma tela HTML com área de filtros, área principal de resultado e host do sidecar.
2. Reusar o controller compartilhado de varejo ou especializá-lo para a tela.
3. Definir screenId, screenTitle, capability e prompt padrão.
4. Reaproveitar o endpoint /ag-ui/runs.
5. Resolver a fonte de configuração a partir do layout mestre, não via segredo ou SQL no DOM.

### 8.2. Cockpit executivo de vendas

Uso prático: tela para diretoria comercial ou gerente de unidade consultar receita, ticket médio e leitura do período.

Implementação confirmada: capability sales_summary, query pdv_vendas_kpis_periodo, filtros de início e fim, sidecar para explicação adicional e área principal preenchida por snapshot governado.

### 8.3. Radar checkout e UCP

Uso prático: tela para operação digital monitorar funil, cancelamentos, abandono e sinais UCP.

Implementação confirmada: capability checkout_funnel, query pdv_checkout_funil_status, filtros de período, cards de indicador previstos e painel principal com resultado governado.

### 8.4. Central de oportunidades de catálogo

Uso prático: tela para trade marketing, pricing e estoque priorizarem oportunidades comerciais.

Implementação confirmada: capability catalog_opportunities, query pdv_catalogo_estoque_oportunidades, filtros de período e narrativa assistida no sidecar.

### 8.5. Canvas dinâmico de dashboard

Uso prático: tela para montar painel executivo seguro com widgets governados.

Implementação confirmada: capability dashboard_dynamic, DashboardSpec local validada no frontend antes do envio, safety com correlationIdAllowed false, três dataSources dyn_sql aprovadas e canvas que sai de vazio para quatro widgets.

## 9. Como usar em um ERP além das telas demo

Para usar AG-UI em telas de ERP fora da demo atual, a recomendação técnica baseada no código é esta.

1. Trabalhar por capability, nunca por SQL ou markup livre.
2. Deixar o frontend responsável só por contexto de tela, filtros e renderização.
3. Deixar o backend responsável por correlation_id, catálogo de queries, validação de input e estado terminal.
4. Reaproveitar o sidecar e o cliente SSE compartilhados.
5. Usar snapshots e deltas para a área principal da tela, e não apenas texto incremental.

Em termos práticos, uma tela ERP nova pode ser de financeiro, compras, logística, SLA ou atendimento, desde que siga a mesma disciplina: capability fechada, data sources aprovadas e renderização sob controle da aplicação.

## 10. Relação com Google e Microsoft no desenho técnico

O ponto tecnicamente importante não é fingir que o projeto já roda em cima de Microsoft Agent Framework ou Google ADK. O ponto correto é este.

1. O protocolo oficial AG-UI já se apresenta como compatível com esses frameworks.
2. O slice local implementa os mesmos blocos conceituais centrais: run, stream de eventos, snapshots, deltas, tools, estado, erro e interrupção.
3. Isso cria um caminho de evolução mais limpo caso o projeto deseje, no futuro, ligar um backend desses frameworks ao mesmo frontend AG-UI ou a uma adaptação próxima.

Hoje, porém, a integração comprovada é outra: FastAPI no backend, SSE no transporte, adapters governados no domínio e JavaScript puro no frontend.

## 11. O que acontece em caso de sucesso

No caminho feliz, o endpoint autentica, aceita o payload, devolve o header X-Correlation-Id, o cliente consome RUN_STARTED, a capability é resolvida com segurança, o estado da tela é preenchido progressivamente e a execução termina com RUN_FINISHED. Os testes confirmam esse comportamento tanto no boundary quanto nas páginas Playwright.

## 12. O que acontece em caso de erro

### 12.1. Sem autenticação

Sintoma: 401 logo na chamada.

Causa provável: ausência de X-API-Key ou sessão compatível.

### 12.2. Sem fonte explícita de configuração

Sintoma: 400 no endpoint.

Causa provável: ausência de yaml_config, yaml_inline_content e encrypted_data.

### 12.3. Adapter inexistente

Sintoma: RUN_ERROR com AG_UI_ADAPTER_NOT_FOUND.

Causa provável: executionKind sem registro no orchestrator.

### 12.4. SQL livre bloqueado

Sintoma: RUN_ERROR com AG_UI_RETAIL_FREE_SQL_BLOCKED.

Causa provável: payload do navegador trouxe chaves como sql, raw_sql ou equivalente.

### 12.5. Configuração PDV ausente

Sintoma: RUN_ERROR com AG_UI_RETAIL_CONFIG_MISSING.

Causa provável: DATABASE_VAREJO_DSN ou DATABASE_VAREJO_SCHEMA não configurados.

### 12.6. DashboardSpec inválida

Sintoma: status validation_failed no estado do dashboard e mensagem textual de recusa.

Causa provável: spec fora do contrato validado.

## 13. Observabilidade e diagnóstico

O diagnóstico do slice AG-UI costuma seguir esta ordem.

1. Confirmar se o request chegou ao endpoint /ag-ui/runs.
2. Ler o X-Correlation-Id devolvido ao frontend.
3. Confirmar qual executionKind foi usado.
4. Ver se o erro ocorreu antes do adapter ou dentro dele.
5. Em telas ERP, comparar status do sidecar, timeline de tools e snapshot principal.

Os logs do orchestrator e da materialização de dashboard são marcados com início, passo, erro e fim, o que ajuda a reconstruir a execução.

## 14. Limites atuais confirmados no código

1. Só há um adapter registrado por padrão: retail_demo.
2. O transporte comprovado é SSE via POST, não WebSocket.
3. Não há SDK Microsoft Agent Framework nem Google ADK no slice local.
4. Não há prova no slice lido de anexos multimodais, voz ou frontend tool calls genéricas.
5. Há suporte de contrato para interruption outcome, mas a retomada formal dedicada deste slice não foi confirmada na mesma rota AG-UI.

## 15. Troubleshooting

### Erro 401 ou 403

Revisar autenticação e permissão agent_execute.

### Erro 400 pedindo configuração

Verificar se a tela carregou YAML inline, yaml_config ou encrypted_data no contexto mestre.

### Sidecar abre mas resultado não preenche

Verificar se chegou STATE_SNAPSHOT ou apenas texto. Se só houver texto, o adapter pode não estar emitindo snapshot para aquela capability.

### Dashboard continua vazio

Verificar se o input saiu com capability dashboard_dynamic e se a DashboardSpec passou na validação local e remota.

### Tool errada ou inexistente

Verificar se a capability enviada bate com o catálogo fechado do adapter.

### Correlation ID ausente na UI

Verificar se a resposta do endpoint retornou o header X-Correlation-Id e se o cliente SSE o capturou.

## 16. Explicação 101

Pense assim: o protocolo AG-UI é o canal pelo qual a tela acompanha o trabalho do agente. Microsoft e Google aparecem no ecossistema oficial como formas de usar essa mesma ideia com outros frameworks. O projeto fez isso do seu jeito, com a própria API e a própria UI. A tela ERP pede uma capability, o backend decide o trabalho real e vai narrando o processo com eventos até montar o resultado. Isso é o que torna a interface agentic visível, governada e reutilizável.

## 17. Checklist de entendimento

- Entendi a diferença entre o protocolo oficial e a implementação local.
- Entendi a superfície HTTP real do slice AG-UI.
- Entendi o papel do orchestrator, do adapter e do encoder SSE.
- Entendi como capabilities ERP são traduzidas para queries aprovadas ou dashboard seguro.
- Entendi como o frontend consome o stream com JavaScript puro.
- Entendi os erros, limites e pontos de troubleshooting.

## 18. Evidências no código

- src/api/routers/ag_ui_router.py
  - Motivo da leitura: confirmar entrada HTTP, autenticação, config source e SSE.
  - Comportamento confirmado: POST /ag-ui/runs é a rota executável do slice.

- src/api/services/ag_ui_run_orchestrator.py
  - Motivo da leitura: confirmar lifecycle do run.
  - Comportamento confirmado: RUN_STARTED inicial, resolução por executionKind e fechamento terminal coerente.

- src/api/services/ag_ui_event_encoder.py
  - Motivo da leitura: confirmar serialização SSE.
  - Comportamento confirmado: cada AgUiBaseEvent vira event/data com JSON oficial do payload.

- src/api/services/ag_ui_retail_demo_adapter.py
  - Motivo da leitura: confirmar catálogo fechado de capabilities e queries PDV.
  - Comportamento confirmado: SQL livre é bloqueado e capability vira dyn_sql governado.

- src/api/services/ag_ui_dashboard_materialization.py
  - Motivo da leitura: confirmar materialização progressiva do dashboard.
  - Comportamento confirmado: custom events e deltas constroem o canvas dinamicamente.

- app/ui/static/js/shared/ag-ui-client.js
  - Motivo da leitura: confirmar cliente SSE do frontend.
  - Comportamento confirmado: POST, leitura incremental do body, retry explícito e captura do X-Correlation-Id.

- app/ui/static/js/shared/ag-ui-sidecar-chat.js
  - Motivo da leitura: confirmar sidecar, store e adaptação de interrupções.
  - Comportamento confirmado: sidecar reutilizável e adaptação para painel HIL compartilhado.

- app/ui/static/js/shared/ag-ui-retail-demo-page.js
  - Motivo da leitura: confirmar padrão de tela ERP sobre AG-UI.
  - Comportamento confirmado: monta payload com capability, contexto de tela e fonte de configuração do layout mestre.

- app/ui/static/js/ag-ui-dashboard-dinamico.js
  - Motivo da leitura: confirmar geração local de DashboardSpec e consumo do canvas.
  - Comportamento confirmado: DashboardSpec segura usa apenas queryIds governadas e correlationIdAllowed false.

- tests/unit/test_ag_ui_protocol_contract.py
  - Motivo da leitura: confirmar regras fortes do contrato de protocolo.
  - Comportamento confirmado: aliases oficiais, deltas JSON Patch e outcome interrupt.

- tests/unit/test_ag_ui_router.py
  - Motivo da leitura: confirmar boundary HTTP e coexistência com rotas antigas.
  - Comportamento confirmado: rota AG-UI não substitui /agent/execute nem /agent/continue.

- tests/unit/test_ag_ui_retail_demo_adapter.py
  - Motivo da leitura: confirmar capabilities, bloqueio de SQL livre e dashboard dinâmico.
  - Comportamento confirmado: adapter termina em RUN_FINISHED no caminho feliz e RUN_ERROR no caminho bloqueado.

- tests/playwright/test_ag_ui_varejo_demo_pages.py
  - Motivo da leitura: confirmar telas ERP fixas no browser.
  - Comportamento confirmado: sidecar abre, correlation_id aparece e DOM recebe resultado governado.

- tests/playwright/test_ag_ui_dashboard_dinamico.py
  - Motivo da leitura: confirmar canvas dinâmico no browser.
  - Comportamento confirmado: canvas sai de vazio para quatro widgets via stream AG-UI.
