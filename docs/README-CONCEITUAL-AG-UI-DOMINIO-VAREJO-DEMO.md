# Manual detalhado da etapa: Dominio varejo demo do AG-UI

## 1. O que esta etapa faz

Esta etapa prova AG-UI como superficie de negocio concreta. Ela transforma capabilities governadas de varejo e PDV em consultas aprovadas ou materializacao segura de dashboard, e devolve tudo isso como eventos de experiencia AG-UI.

Em linguagem simples: e a parte que mostra AG-UI deixando de ser so protocolo e virando produto utilizavel.

## 2. Onde ela entra no fluxo

No codigo lido, esse dominio aparece no RetailDemoAgUiAdapter, no catalogo fechado de queries PDV e no caminho alternativo de dashboard dinamico. Ele e acionado quando o execution_kind e retail_demo.

## 3. O que entra e o que sai

Entradas confirmadas:

- capability retail_demo dentro do input
- parameters escalares quando a capability e baseada em query
- dashboardSpec quando a capability e dashboard_dynamic
- configuracao de ambiente DATABASE_VAREJO_DSN e DATABASE_VAREJO_SCHEMA

Saidas confirmadas:

- eventos de passo, tool call, snapshot de estado e mensagem textual
- resultado da query aprovada ou do dashboard materializado
- bloqueio explicito quando o payload tenta sair do contrato governado

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. o adapter detecta se o payload pede materializacao de dashboard;
2. se for dashboard_dynamic, delega ao DashboardMaterializationService e encerra o fluxo;
3. se nao for, transforma o input em intent de varejo;
4. bloqueia SQL livre em qualquer parte do payload;
5. resolve DSN e schema a partir do ambiente;
6. monta um RetailDemoQueryCatalog fechado e validado como read-only;
7. escolhe a query aprovada pela capability;
8. valida parametros exatos e escalares;
9. cria dyn_sql pela factory canonica com secret_key governada;
10. executa a query e devolve eventos AG-UI com tool call, snapshot e mensagem final.

O detalhe mais importante e que o usuario final nunca recebe SQL livre nem DSN. A UI conversa com capabilities de negocio, enquanto o backend faz o binding seguro com dyn_sql.

## 5. Decisoes tecnicas importantes

### 5.1. Catalogo fechado de capabilities substitui SQL ad hoc

sales_summary, checkout_funnel, catalog_opportunities e customer_segments sao consultas aprovadas. Isso reduz risco operacional e facilita demonstracao comercial.

### 5.2. Read-only e validado no catalogo

Cada query aprovada passa por SqlReadOnlyPolicy usando sqlparse. Na pratica, a etapa se protege contra evolucao indevida para escrita no banco.

### 5.3. Dashboard dinamico e um caminho governado separado

Quando a capability e dashboard_dynamic, o fluxo muda para materializacao de DashboardSpec segura. Isso evita forcar toda experiencia visual a passar por query textual.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- DATABASE_VAREJO_DSN ou DATABASE_VAREJO_SCHEMA ausentes geram AG_UI_RETAIL_CONFIG_MISSING;
- schema invalido gera AG_UI_RETAIL_INVALID_SCHEMA;
- capability fora do catalogo gera AG_UI_RETAIL_CAPABILITY_NOT_ALLOWED;
- SQL livre no payload gera AG_UI_RETAIL_FREE_SQL_BLOCKED;
- parametros faltantes, extras ou nao escalares geram AG_UI_RETAIL_INVALID_PARAMETERS;
- se a tool dyn_sql nao for materializada, o adapter falha com AG_UI_RETAIL_TOOL_NOT_CREATED.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- erro de configuracao PDV ja no inicio da execucao;
- snapshot retailDemo com capability, queryId, toolName e result;
- tool_call_id formado por run_id mais query_id;
- diferenca entre capability esperada e capability realmente enviada no input;
- resultado vazio ou inesperado vindo da query aprovada, nao de SQL livre.

Em linguagem simples: se a tela PDV parece poderosa demais ou permissiva demais, esta etapa mostra que o desenho real e justamente o oposto, governado e fechado.

## 8. Exemplo pratico guiado

Cenario: a lideranca quer KPIs de vendas do periodo.

1. a UI envia capability sales_summary com p1 e p2;
2. o adapter valida que nao existe SQL livre no payload;
3. escolhe a query pdv_vendas_kpis_periodo no catalogo;
4. executa dyn_sql com DSN resolvido por segredo do ambiente;
5. emite STEP_STARTED, TOOL_CALL_START, TOOL_CALL_RESULT, STATE_SNAPSHOT e mensagem final;
6. o orchestrator fecha o run com success.

O valor desta etapa e permitir conversa com dados de negocio sem abrir mao de governanca.

## 9. Evidencias no codigo

- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: RetailDemoAgUiAdapter.execute
  - Comportamento confirmado: bifurcacao entre dashboard dynamic e queries aprovadas de varejo.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: RetailDemoQueryCatalog
  - Comportamento confirmado: catalogo fechado de capabilities PDV e queries read-only.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: EnvironmentRetailDemoSettingsProvider
  - Comportamento confirmado: obrigatoriedade de DATABASE_VAREJO_DSN e DATABASE_VAREJO_SCHEMA.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: DynamicSqlRetailDemoExecutor
  - Comportamento confirmado: execucao via factory canonica dyn_sql com secret_key governada.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: _contains_free_sql_key e _validate_parameters
  - Comportamento confirmado: bloqueio de SQL livre e validacao estrita de parametros.
