# FAQ de Onboarding (Consultor Júnior)

Esta FAQ parte do código atual do repositório.
Ela foi organizada para quem precisa alterar YAML, publicar tools,
acionar ingestão, ETL, RAG e entender o boundary HTTP real.

## 1) Onde mexer

### 1

Pergunta:
Onde eu começo quando preciso entender a plataforma toda?
Resposta:
Comece pelo app HTTP e pelos runners.
É ali que fica claro quem sobe API, worker e scheduler.
Isso evita confundir rota HTTP com execução assíncrona.
Na prática, primeiro entenda o contêiner que recebe a chamada e só
então siga para o service certo.
Onde no código:

- src/api/service_api.py
- app/runners/api_runner.py
- app/runners/worker_runner.py
- app/runners/scheduler_runner.py

### 2

Pergunta:
Onde eu mexo quando a dúvida é endpoint, rota ou payload?
Resposta:
Leia primeiro os routers da família certa.
O router mostra o contrato HTTP real e para onde a chamada segue.
Ele também deixa visível quando a operação é síncrona ou assíncrona.
Na prática, router é o melhor ponto de partida para não adivinhar o
comportamento pelo nome do endpoint.
Onde no código:

- src/api/routers/agent_router.py
- src/api/routers/workflow_router.py
- src/api/routers/rag_router.py
- src/api/routers/streaming_router.py

### 3

Pergunta:
Onde eu mexo quando a dúvida é YAML?
Resposta:
O ponto principal é a fábrica de configuração.
Ela resolve placeholders, injeta catálogo builtin e valida partes
importantes antes da execução.
Isso significa que o YAML não vai direto do arquivo para o runtime.
Na prática, mudança de YAML sem entender essa etapa costuma gerar erro
semântico difícil de diagnosticar.
Onde no código:

- src/config/config_cli/configuration_factory.py
- src/api/routers/config_resolution.py

### 4

Pergunta:
Onde eu mexo quando a dúvida é tool?
Resposta:
Comece pelo builder, pelo loader e pela factory de tools.
Esses três pontos explicam como a tool é descoberta, resolvida e
instanciada.
Sem isso, é fácil confundir nome documental com nome que o runtime
realmente entende.
Na prática, a tool só existe de verdade quando essas camadas concordam.
Onde no código:

- src/agentic_layer/tools/tools_library_builder.py
- src/agentic_layer/supervisor/tool_loader.py
- src/agentic_layer/supervisor/tools_factory.py

### 5

Pergunta:
Onde eu mexo quando a dúvida é a UI administrativa?
Resposta:
A UI está no mesmo app HTTP da API.
As páginas usam cliente JavaScript compartilhado para chamar os
endpoints administrativos.
Isso é importante porque a UI não fala com um backend paralelo.
Na prática, bug de tela muitas vezes é bug de contrato HTTP do mesmo
app principal.
Onde no código:

- src/api/routers/ui_router.py
- app/ui/static/js/shared/admin-api-client.js
- app/ui/static/js/admin-assembly-ast.js
- src/api/service_api.py

## 2) Modelagem e validações

### 6

Pergunta:
O YAML pode trazer tools_library preenchido?
Resposta:
Não.
O fluxo atual exige a chave tools_library presente e vazia para que a
injeção do catálogo builtin aconteça no runtime.
Se vier preenchida, o código falha fechado.
Na prática, isso impede que o cliente tente burlar o catálogo publicado.
Onde no código:

- src/config/config_cli/configuration_factory.py

### 7

Pergunta:
Como o sistema transforma linguagem natural em YAML governado?
Resposta:
O fluxo oficial passa pelo assembly agentic.
Ele expõe draft, validate, confirm e objective-to-yaml.
O endpoint de geração por linguagem natural é
/config/assembly/objective-to-yaml.
Isso separa geração, validação e confirmação final.
Na prática, a plataforma tenta evitar que YAML sensível seja montado
sem checagem estrutural.
Onde no código:

- src/api/routers/config_assembly_router.py
- src/config/agentic_assembly/assembly_service.py

### 8

Pergunta:
Por que às vezes /config/assembly responde como se não existisse?
Resposta:
Porque a feature pode estar desligada.
O router verifica a flag FEATURE_AGENTIC_AST_ENABLED antes de seguir.
Isso não significa necessariamente que a rota sumiu do código.
Na prática, antes de abrir bug você precisa conferir se a feature está
habilitada no ambiente.
Onde no código:

- src/api/routers/config_assembly_router.py
- src/config/settings.py

### 9

Pergunta:
Como o sistema valida uso de tools em supervisor e workflow?
Resposta:
Há validação semântica no fluxo agentic.
Ela verifica referências, consistência estrutural e uso de tools no
modelo governado.
Isso reduz erro de configuração antes da execução real.
Na prática, não confie só no nome da tool no YAML; deixe a validação
oficial dizer se o contrato fecha.
Onde no código:

- src/config/agentic_assembly/validators/tools_semantic_validator.py
- src/config/agentic_assembly/validators/supervisor_semantic_validator.py

### 10

Pergunta:
Como eu descubro o contrato HTTP que o cliente deve respeitar?
Resposta:
Use o OpenAPI do runtime e o schema Pydantic dos routers.
O Swagger mostra campos, tipos e respostas públicas.
Mas o comportamento operacional continua no router e no service.
Na prática, payload e retorno vêm do OpenAPI; semântica de execução vem
na leitura do código.
Onde no código:

- src/api/service_api.py
- src/api/routers/rag_router.py
- src/api/routers/agent_router.py
- src/api/routers/workflow_router.py

## 3) Execução, erros e logs

### 11

Pergunta:
Como o correlation_id entra no fluxo?
Resposta:
O boundary HTTP cria ou reaproveita esse identificador.
Depois ele segue com a request, a resposta e o restante da trilha
operacional.
Isso é o elo entre API, worker e investigação.
Na prática, nunca tente depurar execução distribuída sem esse ID.
Onde no código:

- src/api/service_api.py
- src/core/log_origin_metadata.py

### 12

Pergunta:
Como eu acompanho uma task assíncrona?
Resposta:
Use polling ou SSE nas rotas de status.
Essas rotas são a fachada pública para acompanhar jobs vivos.
Também existe compatibilidade em /api/v1/status.
Na prática, task aceita não significa task concluída; você precisa olhar
o status até o estado terminal.
Onde no código:

- src/api/routers/streaming_router.py

### 13

Pergunta:
Como diferenciar erro do endpoint de erro do worker?
Resposta:
Se a API falha antes de enfileirar, o problema está no boundary HTTP,
na autenticação ou no payload.
Se a API aceita e depois o job morre, a investigação vai para a fila,
worker ou dependência externa.
Na prática, task_id e correlation_id separam bem essas duas etapas.
Onde no código:

- src/api/routers/rag_router.py
- src/api/services/worker_process_runtime.py
- app/runners/worker_runner.py

### 14

Pergunta:
Por que algumas rotas longas respondem 202 e outras continuam em 200?
Resposta:
Porque as famílias de endpoint não seguem um único padrão.
Ingestão e ETL publicam job e devolvem aceitação assíncrona.
Já /agent pode responder 200 para resultado final, HIL ou envelope de
continuidade.
Na prática, você precisa ler o contrato da família, não assumir padrão
único para toda rota demorada.
Onde no código:

- src/api/routers/rag_router.py
- src/api/routers/agent_router.py
- src/api/routers/rag_ingestion_router.py
- src/api/routers/rag_etl_router.py

### 15

Pergunta:
O Swagger fica público por padrão?
Resposta:
Não.
A aplicação aplica controle específico de acesso à documentação.
Além disso, o app já sobe com middlewares globais de permissão.
Na prática, conseguir abrir /docs depende de credencial e permissão,
não só de a rota existir.
Onde no código:

- src/api/service_api.py
- src/api/security/permissions.py

## 4) Ingestão e ETL

### 16

Pergunta:
Qual é o caminho mínimo para iniciar uma ingestão?
Resposta:
Você chama a rota de ingestão com YAML válido e recebe um task_id.
Depois acompanha o andamento no status e, quando necessário, nas rotas
operacionais de ingestão.
Isso separa aceite HTTP de processamento pesado.
Na prática, sucesso do endpoint sozinho não prova indexação concluída.
Onde no código:

- src/api/routers/rag_ingestion_router.py
- src/api/routers/streaming_router.py
- src/api/routers/ingestion_runs_router.py

### 17

Pergunta:
Quem coordena a ingestão depois que o router aceita o pedido?
Resposta:
O service de ingestão monta o pedido operacional e aciona o fluxo real.
Quando há fan-out, a coordenação passa pela camada dedicada a esse tipo
de paralelismo.
Isso mantém o router fino e o comportamento pesado fora do HTTP.
Na prática, regra de ingestão mora no service e no coordenador, não no
endpoint.
Onde no código:

- src/services/ingestion_service.py
- src/services/document_fanout_coordinator.py

### 18

Pergunta:
Como eu acompanho a ingestão depois que ela começou?
Resposta:
Além do status efêmero da task, o projeto expõe leitura operacional de
runs de ingestão.
Essa leitura serve para histórico, detalhes e visão consolidada.
Isso é diferente do polling simples de uma task viva.
Quando existe fan-out, o contrato `fanout_overview` resume filhos,
contagens, throughput, ETA e estado operacional do lote.
Na prática, use status para o agora e ingestion-runs para a visão mais
operacional.
Onde no código:

- src/api/routers/streaming_router.py
- src/api/routers/ingestion_runs_router.py

### 19

Pergunta:
ETL é a mesma coisa que ingestão?
Resposta:
Não.
ETL tem rota própria, service próprio e orquestração própria.
O objetivo é transformar e carregar fontes específicas, não repetir a
mesma trilha documental da ingestão.
Na prática, tratar ETL como sinônimo de ingestão leva a documentação e
operação erradas.
Onde no código:

- src/api/routers/rag_etl_router.py
- src/services/etl_service.py
- src/etl_layer/orchestrator.py

### 20

Pergunta:
Quais pipelines ETL aparecem com evidência no código atual?
Resposta:
O código mostra pipelines ligados a Booking, Hotels.com, TripAdvisor e
schema metadata.
Eles não existem só na documentação; aparecem em módulos reais do ETL.
Isso é importante para separar pipeline implementado de pipeline apenas
planejado.
Na prática, só trate como suportado o que existe em provider e
orquestrador reais.
Onde no código:

- src/etl_layer/orchestrator.py
- src/etl_layer/providers/apify/
- src/etl_layer/providers/data/table_schema_metadata_processor.py

## 5) RAG

### 21

Pergunta:
Qual é o endpoint principal do RAG hoje?
Resposta:
O ponto central de execução é /rag/execute.
Ao redor dele existem rotas complementares para ingestão, ETL,
reindexação e reconstrução de auto_config.
Isso mostra que o módulo RAG não é só pergunta e resposta.
Na prática, para consulta você começa em /rag/execute, não nas rotas de
manutenção.
Onde no código:

- src/api/routers/rag_router.py
- src/api/routers/rag_operations_router.py

### 22

Pergunta:
Quem monta o runtime de perguntas e respostas?
Resposta:
A fachada principal é o sistema de QA.
Ele garante o pipeline moderno e delega o restante para a montagem de
runtime e componentes especializados.
Isso desacopla API e motor de consulta.
Na prática, o router não responde sozinho; ele chama a camada de QA.
Onde no código:

- src/qa_layer/content_qa_system.py
- src/orchestrators/qa_runtime_assembly.py

### 23

Pergunta:
Como o sistema escolhe a estratégia de retrieval?
Resposta:
Existe uma camada de orquestração inteligente para isso.
Ela pode direcionar a pergunta para estratégias diferentes de busca e
combinação.
Isso significa que duas perguntas parecidas podem seguir caminhos
internos diferentes.
Na prática, retrieval ruim nem sempre é culpa do prompt final.
Onde no código:

- src/qa_layer/rag_engine/intelligent_orchestrator.py
- src/qa_layer/rag_engine/retrieval_engine.py

### 24

Pergunta:
Como eu depuro quando o RAG não trouxe contexto?
Resposta:
Primeiro confirme se a ingestão indexou conteúdo útil.
Depois verifique a estratégia de retrieval e os filtros aplicados.
Só depois vale mexer na geração.
Na prática, a maior parte dos problemas de contexto nasce antes do LLM
responder.
Onde no código:

- src/qa_layer/rag_engine/retrieval_engine.py
- src/api/routers/rag_ingestion_router.py
- src/api/routers/ingestion_runs_router.py

### 25

Pergunta:
Existe caminho especializado para planilhas e Excel?
Resposta:
Sim.
O runtime atual mantém caminho especializado para cenários tabulares.
Isso evita tratar planilha como texto livre desde o início.
Na prática, perguntas sobre Excel podem seguir uma lógica diferente da
consulta textual comum.
Onde no código:

- src/qa_layer/json_rag/specialized_rag_excel.py
- src/qa_layer/content_qa_system.py

## 6) Tools avançadas e linguagem natural

### 26

Pergunta:
Como o catálogo builtin de tools é gerado?
Resposta:
O builder descobre funções marcadas com decorators próprios e sincroniza
o catálogo persistido.
Depois o runtime usa esse catálogo como base da injeção em YAML.
Isso evita depender de lista manual espalhada em vários arquivos.
Na prática, adicionar tool no código sem passar por esse fluxo não fecha
o contrato da plataforma.
Onde no código:

- src/agentic_layer/tools/tools_library_builder.py
- src/config/config_cli/configuration_factory.py

### 27

Pergunta:
Quando eu uso dyn_sql?
Resposta:
Use dyn_sql quando a query já existe e o agente só precisa preencher
parâmetros aprovados.
A resolução tenta primeiro o YAML e depois o registro governado por
tenant.
Isso é execução de consulta conhecida, não geração livre de SQL.
Na prática, dyn_sql é o caminho certo para consulta homologada.
Onde no código:

- src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py
- src/agentic_layer/tools/domain_tools/dynamic_tool_registry_resolver.py
- src/agentic_layer/supervisor/tool_loader.py

### 28

Pergunta:
Quando eu uso dyn_api?
Resposta:
Use dyn_api quando o endpoint HTTP já está aprovado e descrito por
contrato.
A resolução também tenta primeiro o YAML e depois o registro governado
por tenant.
No caminho persistido atual, o protocolo aceito é rest_json.
Na prática, dyn_api não é um atalho para qualquer chamada HTTP solta.
Onde no código:

- src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py
- src/agentic_layer/tools/domain_tools/dynamic_tool_registry_resolver.py
- src/agentic_layer/supervisor/tool_loader.py

### 29

Pergunta:
Quando eu uso schema_rag_sql?
Resposta:
Use schema_rag_sql quando a entrada é linguagem natural e a geração de
SQL depende de metadados de schema.
Esse caminho não substitui dyn_sql porque resolve outro problema.
Um executa consulta conhecida; o outro ajuda a montar SQL guiada por
estrutura do banco.
Na prática, misturar os dois cenários costuma piorar o desenho.
Onde no código:

- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
- src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py

### 30

Pergunta:
Recommend-tools e objective-to-yaml são tools comuns?
Resposta:
Não.
Essas superfícies ajudam na autoria governada e na escolha de composição.
Elas usam o catálogo efetivo e devolvem diagnóstico ou proposta, mas não
são a mesma coisa que a tool executada pelo agente.
Na prática, servem para desenhar e validar antes de publicar o YAML.
Onde no código:

- src/api/routers/config_assembly_router.py
- src/config/agentic_assembly/assembly_service.py
- src/agentic_layer/supervisor/tools_factory.py

### 31

Pergunta:
Onde eu mexo quando a dúvida é HIL Generative UI?
Resposta:
Comece pelo contrato HIL e depois siga para o frontend compartilhado.
O backend publica a pausa humana no envelope `hil`.
O frontend normaliza esse envelope e renderiza a revisão com um
componente reaproveitável.
Se a tela for AG-UI, o sidecar entra só como transporte e apresentação
da pendência.
Na prática, HIL Generative UI aqui não é HTML gerado pelo agente; é uma
revisão humana montada a partir de dados estruturados autorizados pelo
backend.
Onde no código:

- src/api/routers/agent_router.py
- app/ui/static/js/shared/ui-webchat-hil-contract.js
- app/ui/static/js/shared/hil-review-panel.js
- app/ui/static/js/shared/ag-ui-sidecar-chat.js

### 32

Pergunta:
O AG-UI já continua um HIL sozinho?
Resposta:
Não.
O sidecar AG-UI já mostra a pendência e já entrega a decisão humana por
callback.
Mas a retomada formal continua sendo responsabilidade da interface dona
do fluxo, que precisa chamar o endpoint correto de continue.
Isso evita esconder regra crítica dentro do sidecar e mantém a fronteira
de backend explícita.
Na prática, ver o painel de aprovação na tela não prova que a retomada
já está ligada no servidor.
Onde no código:

- app/ui/static/js/shared/ag-ui-sidecar-chat.js
- src/api/schemas/ag_ui_models.py
- src/api/routers/agent_router.py
- src/api/routers/workflow_router.py
