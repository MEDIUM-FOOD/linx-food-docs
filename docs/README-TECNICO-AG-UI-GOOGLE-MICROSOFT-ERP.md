# Manual tecnico, operacional e de uso: Generative UI local com AG-UI no projeto

## 1. O que esta implementado de fato

O slice executavel de Generative UI deste projeto e uma implementacao AG-UI local baseada em FastAPI no backend e HTML estatico com JavaScript puro no frontend. O boundary publico e a rota POST /ag-ui/runs. O request entra em um contrato Pydantic estrito, o backend resolve um contexto imutavel de execucao, delega para um adapter por executionKind e serializa os eventos em text/event-stream. No navegador, um cliente compartilhado consome o stream, aplica eventos em um store local e atualiza sidecar, timeline de tools e area principal da pagina.

Isso importa porque o comportamento real nao esta espalhado em varias rotas nem depende de uma SPA complexa. Ele esta concentrado em poucos componentes coesos: contrato de protocolo, router, orchestrator, adapter de dominio, materializador de dashboard e runtime compartilhado do frontend.

## 2. Contrato do request

A entrada canônica do slice esta em AgUiRunRequest. Os campos relevantes confirmados no codigo sao estes.

1. threadId: identidade logica da thread da interface.
2. runId: identidade do run atual.
3. executionKind: chave usada para escolher o adapter.
4. user_email: usuario operacional.
5. input: payload de negocio.
6. metadata: contexto adicional da tela.
7. yaml_config, yaml_inline_content ou encrypted_data: fonte explicita de configuracao.

O boundary falha fechado se nenhuma dessas fontes de configuracao for enviada. Isso e um comportamento importante: a UI AG-UI local nao executa com contexto implicito nem com fallback silencioso.

## 3. Borda HTTP e transporte

A rota dedicada fica em src/api/routers/ag_ui_router.py. O comportamento comprovado e este.

1. Prefixo /ag-ui.
2. Endpoint POST /runs.
3. Permissao exigida: agent_execute.
4. Rate limit herdado da camada de agente.
5. Resposta em StreamingResponse com media_type text/event-stream.
6. Header X-Correlation-Id devolvido ao frontend.

O router nao conhece o dominio. Ele apenas valida o request, resolve o correlation_id, monta AgUiRunContext e injeta AgUiSseEventEncoder no fluxo de resposta.

## 4. AgUiRunContext e isolamento do lifecycle

O contexto de execucao foi modelado como dataclass imutavel. Ele concentra correlation_id, thread_id, run_id, user_email, execution_kind, input, parent_run_id, metadata e as tres possibilidades de fonte de configuracao.

Esse desenho tem valor tecnico direto: depois que o router monta o contexto, as camadas seguintes nao ficam pescando dados soltos do request. Isso reduz acoplamento e ajuda a manter o orchestrator cego ao dominio.

## 5. Orquestrador do run

O orchestrator mora em src/api/services/ag_ui_run_orchestrator.py. Ele tem responsabilidade unica: governar o lifecycle AG-UI.

O comportamento confirmado no codigo e este.

1. Loga inicio do run com marker canônico.
2. Emite RUN_STARTED imediatamente.
3. Resolve o adapter por executionKind.
4. Repassa os eventos produzidos pelo adapter.
5. Considera RUN_FINISHED e RUN_ERROR como terminais.
6. Se o adapter nao emitir evento terminal, fecha automaticamente com RUN_FINISHED e outcome success.
7. Se nao houver adapter, retorna RUN_ERROR com AG_UI_ADAPTER_NOT_FOUND.
8. Se houver erro de dominio controlado, retorna RUN_ERROR com o code especifico.
9. Se houver erro inesperado, retorna RUN_ERROR com AG_UI_RUN_FAILED.

Esse desenho permite adicionar novos adapters sem alterar a logica de lifecycle.

## 6. Encoder SSE

O encoder mora em src/api/services/ag_ui_event_encoder.py e e validado por teste dedicado. A funcao dele e transformar cada BaseEvent em uma mensagem SSE independente, no formato event/data, terminada por linha em branco.

Em termos praticos, isso garante duas coisas.

1. O frontend recebe um payload completo por evento, nao fragmentos sem tipo.
2. O contrato de serializacao do protocolo fica centralizado, e nao espalhado pelo router ou pelo adapter.

## 7. Modelos de evento disponiveis

O arquivo src/api/schemas/ag_ui_models.py define os eventos oficiais. Os grupos mais relevantes para a interface local sao estes.

### 7.1. Lifecycle do run

1. RUN_STARTED.
2. RUN_FINISHED.
3. RUN_ERROR.

### 7.2. Etapas observaveis

1. STEP_STARTED.
2. STEP_FINISHED.

### 7.3. Mensagens incrementais

1. TEXT_MESSAGE_START.
2. TEXT_MESSAGE_CONTENT.
3. TEXT_MESSAGE_END.
4. TEXT_MESSAGE_CHUNK.

### 7.4. Timeline de tools

1. TOOL_CALL_START.
2. TOOL_CALL_ARGS.
3. TOOL_CALL_END.
4. TOOL_CALL_RESULT.
5. TOOL_CALL_CHUNK.

### 7.5. Estado e atividade

1. STATE_SNAPSHOT.
2. STATE_DELTA.
3. MESSAGES_SNAPSHOT.
4. ACTIVITY_SNAPSHOT.
5. ACTIVITY_DELTA.

### 7.6. Extensibilidade

1. RAW.
2. CUSTOM.

### 7.7. Interrupcao humana

RUN_FINISHED suporta outcome interrupt com lista de interrupts. Isso evita criar um evento inventado separado para HIL. O contrato ja trata pausa terminal como parte do encerramento do run.

## 8. Adapter de dominio atual: retail_demo

O unico adapter registrado por padrao no router e RetailDemoAgUiAdapter.default(). Ele esta em src/api/services/ag_ui_retail_demo_adapter.py.

Essa classe e o coracao da implementacao de negocio do slice atual. Ela converte o input do navegador em uma de duas rotas.

1. Consulta governada por dyn_sql.
2. Materializacao dinamica de dashboard.

O adapter nao aceita qualquer payload. Ele trabalha com capability fechada. As capabilities confirmadas no catalogo sao estas.

1. sales_summary.
2. checkout_funnel.
3. catalog_opportunities.
4. customer_segments.
5. dashboard_dynamic.

## 9. Regras de seguranca no adapter

O adapter implementa protecoes importantes que precisam aparecer em qualquer documentacao tecnica seria.

### 9.1. Bloqueio de SQL livre

O input e percorrido recursivamente. Se aparecer chave como sql, raw_sql, sql_query ou statement, a execucao falha com AG_UI_RETAIL_FREE_SQL_BLOCKED.

### 9.2. Capability obrigatoria

Se o payload nao trouxer capability valida, a execucao falha com erro especifico de contrato.

### 9.3. Parametros escalares e controlados

Os parametros da capability sao validados contra o catalogo da query aprovada. Faltas, extras ou valores complexos como lista e objeto sao recusados.

### 9.4. Configuracao de banco obrigatoria

O adapter depende de DATABASE_VAREJO_DSN e DATABASE_VAREJO_SCHEMA. Se faltarem, a execucao falha com AG_UI_RETAIL_CONFIG_MISSING. O schema tambem precisa ser identificador simples, sem formato arbitrario.

## 10. Catalogo governado de queries

O catalogo local mapeia capability para query aprovada. Cada entrada tem capability, query_id, descricao, SQL, parametros, fetch_mode e result_format. Antes do uso, cada query passa por policy de somente leitura baseada em sqlparse.

As resolucoes confirmadas sao estas.

1. sales_summary -> pdv_vendas_kpis_periodo.
2. checkout_funnel -> pdv_checkout_funil_status.
3. catalog_opportunities -> pdv_catalogo_estoque_oportunidades.
4. customer_segments -> pdv_clientes_segmentacao.

O ganho tecnico disso e central: a interface pede intencao, nao comando de banco. O backend preserva governanca e ainda usa a fabrica canonica de dyn_sql do projeto.

## 11. Fluxo de consulta governada

Quando a capability nao e dashboard_dynamic, o adapter segue este fluxo.

1. Valida o input como objeto com capability e parameters.
2. Resolve configuracao do ambiente PDV.
3. Monta o catalogo fechado de queries.
4. Resolve a query correspondente a capability.
5. Valida parametros.
6. Emite STEP_STARTED.
7. Emite TOOL_CALL_START.
8. Emite TOOL_CALL_ARGS com JSON dos parametros.
9. Executa dyn_sql pela factory canônica.
10. Emite TOOL_CALL_END.
11. Emite TOOL_CALL_RESULT com o retorno bruto.
12. Emite STATE_SNAPSHOT com retailDemo.capability, queryId, toolName e result.
13. Emite mensagem textual curta de conclusao.
14. Emite STEP_FINISHED.
15. O orchestrator encerra com RUN_FINISHED se nenhum terminal foi enviado antes.

Esse fluxo e o padrao da interface AG-UI para paginas fixas de vendas, checkout e catalogo.

## 12. Materializacao dinamica de dashboard

Quando a capability e dashboard_dynamic, o adapter nao executa dyn_sql diretamente. Ele desvia para DashboardMaterializationService.

A materializacao segue este contrato.

1. Detecta capability dashboard_dynamic.
2. Extrai dashboardSpec ou dashboard_spec do input.
3. Cria um dashboardId baseado no run.
4. Emite CUSTOM retail.dashboard.spec.started.
5. Emite STATE_SNAPSHOT inicial com status materializing, spec nula, widgets vazios, dataSources vazios e errors vazios.
6. Valida a DashboardSpec.
7. Se a spec for invalida, emite custom de falha, muda status para validation_failed, publica errors estruturados e envia mensagem textual curta.
8. Se a spec for valida, emite custom de validacao, deltas de substituicao da spec, deltas de inclusao de data sources, deltas de inclusao de widgets, custom de render pronto e delta final com status ready.

Esse desenho e importante porque transforma o dashboard em um processo progressivo e auditavel, nao em um blob pronto sem historia.

## 13. Contrato da DashboardSpec

A fronteira do dashboard fica em src/api/schemas/ag_ui_dashboard_models.py. O contrato confirmado e fechado e versionado. Os elementos principais sao estes.

1. version = 1.0.
2. title.
3. layout em grid, com columns, rowHeight e gap.
4. filters opcionais.
5. widgets.
6. dataSources.
7. narrative.
8. refreshPolicy.
9. safety.

### 13.1. Tipos permitidos de widget

1. kpi.
2. line_chart.
3. bar_chart.
4. donut_chart.
5. table.
6. insight_card.
7. alert.
8. timeline.
9. ranking.

### 13.2. Fonte de dados permitida

A fonte de dados confirmada hoje e dyn_sql, com queryId governada e allowedParameters declarados.

### 13.3. Safety obrigatoria

O contrato exige cinco travas explicitas com valor False.

1. htmlAllowed.
2. scriptAllowed.
3. freeSqlAllowed.
4. secretsAllowed.
5. correlationIdAllowed.

Isso deixa claro que o dashboard dinamico nao e um caminho para burlar governanca.

## 14. Validador de DashboardSpec

A validacao existe em duas camadas no repositório.

1. Camada Python no backend.
2. Camada JavaScript segura para a UI dinamica.

As regras confirmadas incluem.

1. Rejeicao de chaves proibidas como html, script, sql, query, dsn, secret e correlationId.
2. Rejeicao de HTML, JavaScript e SQL livre em strings.
3. Rejeicao de campos fora do contrato.
4. Rejeicao de widget que referencia data source inexistente.
5. Rejeicao de parametro nao declarado em allowedParameters.
6. Rejeicao de layout impossivel ou sobreposto.

Na pratica, isso significa que a flexibilidade do dashboard continua subordinada a um contrato seguro.

## 15. Cliente compartilhado do frontend

O cliente mora em app/ui/static/js/shared/ag-ui-client.js. Ele implementa consumo de SSE por POST, nao por EventSource tradicional. O fluxo confirmado e este.

1. Resolve a URL do endpoint.
2. Monta headers com Content-Type JSON e Accept text/event-stream.
3. Inclui X-API-Key se ela existir.
4. Nao gera correlation_id no browser.
5. Faz POST com o payload do run.
6. Captura X-Correlation-Id da resposta.
7. Faz parse incremental de blocos SSE.
8. Reconstrui cada evento a partir das linhas event: e data:.
9. Suporta retry explicito com maxReconnectAttempts e reconnectDelayMs.

Esse cliente e a principal peca de reuso para terceiros no lado web.

## 16. Store de estado AG-UI

O store mora em app/ui/static/js/shared/ag-ui-state-store.js. Ele converte eventos em estado local mutavel e notificavel. O estado inicial contem run, messages, tools, state, activities, steps, interrupts, rawEvents, customEvents e lastEvent.

Comportamentos importantes confirmados.

1. RUN_STARTED muda status para running.
2. RUN_FINISHED seta outcome e lista de interrupts quando houver interrupt.
3. RUN_ERROR marca estado de erro.
4. Eventos de texto montam mensagem incremental por messageId.
5. Eventos de tool alimentam timeline compartilhada.
6. STATE_SNAPSHOT substitui state inteiro.
7. STATE_DELTA aplica JSON Patch simples.
8. ACTIVITY_SNAPSHOT e ACTIVITY_DELTA atualizam atividades em andamento.

O ponto forte aqui e que qualquer tela nova que consuma AG-UI pode reaproveitar exatamente o mesmo mecanismo de reconstruicao de estado.

## 17. Sidecar compartilhado

O sidecar mora em app/ui/static/js/shared/ag-ui-sidecar-chat.js. Ele e um componente de pagina estatica, nao uma dependencia de framework pesado.

As responsabilidades confirmadas sao estas.

1. Montar o DOM do sidecar.
2. Mostrar status, correlation_id, contexto, mensagens, timeline de tools e interrupcoes.
3. Reaproveitar o store de estado.
4. Abrir e fechar o painel.
5. Enviar mensagem para o backend via buildRunPayload configuravel.
6. Adaptar interrupts AG-UI ao painel HIL compartilhado.

Isso faz do sidecar uma infraestrutura de interface pronta para outras telas, nao um widget colado apenas na demo atual.

## 18. Controller compartilhado das telas fixas

O controller fica em app/ui/static/js/shared/ag-ui-retail-demo-page.js. Ele mostra claramente como uma nova tela pode ser criada sem reimplementar o protocolo.

O comportamento confirmado e este.

1. Recebe screenId, screenTitle, capability e defaultPrompt.
2. Resolve API key a partir do layout mestre.
3. Resolve YAML inline ou encrypted_data a partir do layout mestre.
4. Exige userEmail operacional.
5. Monta payload com executionKind retail_demo.
6. Inclui metadata com screenId, screenTitle, yamlPath e inputMode.
7. Monta input com capability, parameters, message e context.
8. Atualiza status da pagina conforme chegam RUN_STARTED, RUN_ERROR, STATE_SNAPSHOT e RUN_FINISHED.

Essa classe e a melhor prova pratica de como terceiros internos podem criar outra tela AG-UI sem reinventar a infraestrutura.

## 19. Como criar uma nova tela AG-UI neste projeto

O caminho tecnico recomendado, baseado no codigo atual, e este.

1. Definir a capability de negocio que a tela precisa.
2. Se ela pertencer ao dominio atual, incluir a capability no catalogo governado do adapter de varejo. Se pertencer a outro dominio, criar novo adapter e novo executionKind.
3. Criar uma pagina HTML estatica com host para status, sidecar, filtros e area principal.
4. Reusar AgUiRetailDemoPageController ou criar controller equivalente.
5. Fazer a area principal reagir a STATE_SNAPSHOT e, se necessario, a STATE_DELTA.
6. Nao colocar SQL, segredo ou correlation_id no DOM como fonte de verdade.
7. Validar com testes unitarios e Playwright.

## 20. Como terceiros podem consumir o protocolo

Terceiros podem integrar de duas formas.

### 20.1. Consumir o backend local

Nesse caso, basta implementar um cliente HTTP capaz de fazer POST em /ag-ui/runs e consumir SSE. O cliente precisa entender event/data, ler JSON e reagir aos tipos de evento.

### 20.2. Reproduzir o contrato em outro backend

Nesse caso, o backend terceiro precisa emitir o mesmo tipo de lifecycle: RUN_STARTED, eventos intermediarios, snapshots ou deltas e um terminal coerente. O frontend pode continuar sendo o mesmo cliente e o mesmo store, desde que o contrato seja respeitado.

O aprendizado tecnico central e este: o reaproveitamento esta no protocolo, nao nas paginas demo.

## 21. Contrato real das paginas demo atuais

As paginas confirmadas por leitura e testes sao estas.

1. Cockpit de vendas.
2. Radar de checkout.
3. Central de catalogo.
4. Dashboard dinamico.

As tres primeiras usam o fluxo de consulta governada. A quarta usa o fluxo de DashboardSpec materializada.

## 22. O que acontece em caso de sucesso

### 22.1. Páginas fixas de varejo

No caminho feliz, a pagina dispara o run, recebe RUN_STARTED, exibe correlation_id, mostra tool call governada, recebe STATE_SNAPSHOT com retailDemo.result, renderiza o resultado na area principal e fecha com RUN_FINISHED.

### 22.2. Dashboard dinamico

No caminho feliz, o canvas comeca vazio, recebe snapshot inicial materializing, passa a receber spec, dataSources e widgets por deltas, exibe historico de construcao e termina com status ready e RUN_FINISHED.

## 23. O que acontece em caso de erro

Os cenarios de erro confirmados no codigo incluem estes.

1. 401 por ausencia de autenticacao.
2. 400 por ausencia de yaml_config, yaml_inline_content ou encrypted_data.
3. RUN_ERROR com AG_UI_ADAPTER_NOT_FOUND quando executionKind nao esta registrado.
4. RUN_ERROR com AG_UI_RETAIL_CAPABILITY_REQUIRED quando capability esta ausente.
5. RUN_ERROR com AG_UI_RETAIL_CAPABILITY_NOT_ALLOWED quando a capability nao esta no catalogo.
6. RUN_ERROR com AG_UI_RETAIL_INVALID_PARAMETERS quando faltam parametros, sobram parametros ou algum valor nao e escalar.
7. RUN_ERROR com AG_UI_RETAIL_FREE_SQL_BLOCKED quando o payload tenta trazer SQL livre.
8. RUN_ERROR com AG_UI_RETAIL_CONFIG_MISSING quando o ambiente PDV esta incompleto.
9. validation_failed no dashboard quando a DashboardSpec viola o contrato.

## 24. Observabilidade e diagnostico

A investigacao de problemas nesse slice deve seguir esta ordem.

1. Confirmar se o request chegou a /ag-ui/runs.
2. Confirmar autenticacao e permissao de agent_execute.
3. Ler o X-Correlation-Id devolvido ao frontend.
4. Verificar executionKind e capability enviados.
5. Ver se o erro ocorreu antes do adapter, dentro do adapter ou na materializacao do dashboard.
6. Diferenciar falta de configuracao de erro de dominio.
7. Conferir no frontend se houve RUN_STARTED sem STATE_SNAPSHOT, porque isso costuma separar falha de execucao de falha de renderizacao.

Os logs do orchestrator e da materializacao usam markers canônicos, o que ajuda a rastrear o fluxo com correlation_id.

## 25. Testes que comprovam o comportamento

O slice AG-UI tem evidencias de teste relevantes.

### 25.1. Contrato de protocolo

O teste test_ag_ui_protocol_contract confirma alias oficiais, rejeicao de campos extras, deltas JSON Patch e outcome interrupt em RUN_FINISHED.

### 25.2. Boundary HTTP

O teste test_ag_ui_router confirma autenticacao obrigatoria, exigencia de configuracao explicita, resposta SSE e coexistencia com rotas antigas /agent/execute, /agent/continue e /status/stream/{task_id}.

### 25.3. Browser das paginas fixas

O teste Playwright das telas fixas confirma sidecar abrindo, correlation_id aparecendo, DOM sendo atualizado e ausencia de correlation_id dentro do payload enviado.

### 25.4. Browser do dashboard dinamico

O teste Playwright do dashboard confirma canvas vazio no inicio, materializacao de quatro widgets via stream, historico da construcao e safety com correlationIdAllowed false.

## 26. Troubleshooting

### 26.1. Recebo 400 dizendo que falta configuracao

Causa provavel: a pagina nao carregou YAML inline nem payload criptografado no layout mestre.

Como confirmar: inspecionar o payload enviado e verificar ausencia de yaml_inline_content, yaml_config e encrypted_data.

### 26.2. A UI abre o sidecar mas nao mostra resultado

Causa provavel: houve RUN_STARTED, mas nao houve STATE_SNAPSHOT ou houve RUN_ERROR antes do snapshot.

Como confirmar: inspecionar os eventos recebidos no cliente e o estado final do store.

### 26.3. O dashboard fica vazio

Causa provavel: a spec foi recusada ou a pagina nao reagiu aos deltas.

Como confirmar: verificar se chegou retail.dashboard.validation.failed ou se o state.retailDashboard.status nao saiu de materializing.

### 26.4. O backend recusa a capability

Causa provavel: capability nao cadastrada no catalogo ou payload com nome diferente do esperado.

Como confirmar: comparar a capability enviada com o catalogo do adapter.

### 26.5. O correlation_id nao aparece na tela

Causa provavel: o cliente nao capturou o header X-Correlation-Id ou o backend falhou antes de responder.

Como confirmar: inspecionar headers da resposta e callback onCorrelationId do cliente.

## 27. Limites atuais

1. O router registra apenas retail_demo por padrao.
2. O transporte comprovado e SSE por POST.
3. O dominio atual documentado e PDV demo.
4. customer_segments existe no adapter, mas a leitura desta rodada nao confirmou tela estatica dedicada para essa capability.
5. A rota AG-UI atual nao prova, por si so, um fluxo de retomada dedicado na mesma superficie publica.

## 28. Explicacao 101

O backend AG-UI deste projeto funciona como uma esteira de eventos. A tela manda um pedido padronizado. O servidor valida o pedido, decide qual capacidade de negocio vai rodar e vai avisando a tela do que esta acontecendo. A tela nao precisa adivinhar o estado da execucao. Ela recebe esse estado pronto e vai desenhando a experiencia conforme os eventos chegam.

## 29. Checklist de entendimento

- Entendi a rota publica do slice.
- Entendi como o orchestrator isola o lifecycle.
- Entendi como o adapter traduz capability em execucao governada.
- Entendi como funciona a materializacao de dashboard.
- Entendi como o cliente, o store e o sidecar podem ser reutilizados.
- Entendi como criar uma nova tela ou um novo adapter.
- Entendi os testes e os limites atuais.

## 30. Evidencias no codigo

- src/api/routers/ag_ui_router.py
  - Motivo da leitura: confirmar endpoint dedicado, configuracao obrigatoria e SSE.
  - Comportamento confirmado: POST /ag-ui/runs devolve stream AG-UI com X-Correlation-Id.

- src/api/services/ag_ui_run_orchestrator.py
  - Motivo da leitura: confirmar lifecycle do run.
  - Comportamento confirmado: RUN_STARTED inicial, resolucao por executionKind e fechamento terminal.

- src/api/services/ag_ui_retail_demo_adapter.py
  - Motivo da leitura: confirmar governanca de capability, dyn_sql e dashboard.
  - Comportamento confirmado: SQL livre bloqueado, queries aprovadas e desvio para dashboard_dynamic.

- src/api/services/ag_ui_dashboard_materialization.py
  - Motivo da leitura: confirmar materializacao progressiva do canvas.
  - Comportamento confirmado: snapshot inicial, deltas estruturados e status final ready ou validation_failed.

- src/api/schemas/ag_ui_dashboard_models.py
  - Motivo da leitura: confirmar o contrato fechado de DashboardSpec.
  - Comportamento confirmado: layout, widgets, dataSources, refreshPolicy e safety obrigatoria.

- app/ui/static/js/shared/ag-ui-client.js
  - Motivo da leitura: confirmar cliente reutilizavel para SSE por POST.
  - Comportamento confirmado: parse incremental, callback de correlation_id e retry explicito.

- app/ui/static/js/shared/ag-ui-state-store.js
  - Motivo da leitura: confirmar reconstruicao de estado a partir de eventos.
  - Comportamento confirmado: snapshots, deltas, mensagens, tools e interrupts viram estado navegavel.

- app/ui/static/js/shared/ag-ui-sidecar-chat.js
  - Motivo da leitura: confirmar sidecar e adaptacao para HIL.
  - Comportamento confirmado: painel reutilizavel desacoplado da area principal da pagina.

- app/ui/static/js/shared/ag-ui-retail-demo-page.js
  - Motivo da leitura: confirmar como novas paginas podem reutilizar o slice.
  - Comportamento confirmado: payload padronizado e consumo integrado com status e sidecar.

- tests/unit/test_ag_ui_protocol_contract.py
  - Motivo da leitura: confirmar rigidez do contrato.
  - Comportamento confirmado: alias oficiais, falha fechada para campo extra e outcome interrupt.

- tests/unit/test_ag_ui_router.py
  - Motivo da leitura: confirmar o boundary HTTP.
  - Comportamento confirmado: autenticacao obrigatoria, configuracao explicita e coexistencia com rotas antigas.

- tests/playwright/test_ag_ui_varejo_demo_pages.py
  - Motivo da leitura: confirmar comportamento das paginas fixas no browser.
  - Comportamento confirmado: correlation_id, sidecar aberto, payload sem correlation_id e DOM atualizado.

- tests/playwright/test_ag_ui_dashboard_dinamico.py
  - Motivo da leitura: confirmar comportamento do dashboard dinamico no browser.
  - Comportamento confirmado: canvas vazio no inicio, quatro widgets materializados e safety com correlationIdAllowed false.
