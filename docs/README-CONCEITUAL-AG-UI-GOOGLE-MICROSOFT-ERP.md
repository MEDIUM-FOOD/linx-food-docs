# Manual conceitual, executivo, comercial e estrategico: Generative UI local com AG-UI no projeto

## 1. O que e esta feature

A Generative UI implementada neste projeto e o slice AG-UI: uma interface agentic orientada por eventos, feita para ligar uma tela de negocio a um backend que executa capacidades governadas e devolve progresso, estado, timeline de ferramentas, mensagens e resultado final no mesmo fluxo. Ela nao e um chat generico, nao e apenas streaming de texto e nao e um frontend isolado inventando contrato proprio por pagina. O que existe aqui e um protocolo de interface reutilizavel, aplicado a telas de ERP e analytics, com o backend controlando o que pode ser pedido e o frontend reagindo a eventos estruturados.

Na pratica, isso significa que a tela nao precisa saber como executar SQL, como rastrear correlation_id, como decidir se um dashboard e seguro ou como serializar o lifecycle de um run. Ela so precisa declarar a intencao de negocio, como vendas, checkout, catalogo ou dashboard dinamico, e consumir eventos AG-UI. O backend transforma essa intencao em execucao governada e conta a historia da execucao de ponta a ponta.

## 2. Que problema ela resolve

Sem uma camada como essa, cada tela rica do produto tenderia a reinventar a propria infraestrutura de progresso, loading, erro, streaming, atualizacao parcial de estado e rastreabilidade. Isso cria dois problemas graves. O primeiro e tecnico: interfaces diferentes passam a falar protocolos diferentes com o backend, o que aumenta acoplamento, duplicacao e fragilidade. O segundo e operacional: o usuario final ve uma experiencia inconsistente e o suporte perde capacidade de diagnosticar o que aconteceu em cada execucao.

O slice AG-UI resolve isso criando uma lingua comum entre interface e runtime. Em vez de cada pagina montar um mini-backend proprio no navegador, todas passam a usar o mesmo contrato de run, o mesmo stream de eventos e a mesma reconstruicao de estado. Isso reduz custo de evolucao, aumenta governanca e permite que a plataforma trate Generative UI como uma capacidade de produto, nao como um truque de uma pagina especifica.

## 3. Visao executiva

Para lideranca, o valor principal e transformar automacao agentic em algo observavel, governado e reutilizavel. Em vez de uma resposta final sem contexto, a interface mostra quando a execucao comecou, qual capacidade foi acionada, que ferramenta governada foi chamada, como o estado foi montado e como terminou. Isso reduz a sensacao de caixa-preta, melhora suporte e acelera demonstracoes internas e comerciais.

Tambem existe ganho claro de escala. Uma vez que o protocolo de interface esta pronto, novas telas podem nascer em cima da mesma base. O investimento deixa de ser reconstruir plumbing de streaming e passa a ser construir capacidades de negocio. Isso melhora time-to-market e reduz custo marginal de novas superficies operacionais.

## 4. Visao comercial

Comercialmente, esta feature permite vender experiencia agentic aplicada ao negocio, nao apenas backend inteligente. O cliente nao recebe so um chat que responde perguntas. Ele recebe uma tela operacional que mostra progresso, explica o que esta acontecendo, materializa resultado governado e, quando necessario, monta um dashboard seguro em tempo real.

No slice atual, isso aparece em historias comerciais concretas.

1. Cockpit executivo de vendas com leitura assistida de KPI do periodo.
2. Radar de checkout e funil com apoio operacional para identificar gargalos.
3. Central de oportunidades de catalogo para estoque, trade e pricing.
4. Canvas dinamico de dashboard com widgets e fontes de dados validadas.

O diferencial comercial nao esta em prometer IA irrestrita. Esta em combinar interface progressiva, governanca e dados operacionais aprovados. Isso responde melhor a objeções de seguranca, risco e previsibilidade.

## 5. Visao estrategica

Estrategicamente, o slice AG-UI fortalece a plataforma em quatro frentes.

1. Separa claramente interface agentic do backend de negocio, usando um contrato estavel em vez de acoplamento direto entre pagina e dominio.
2. Cria uma base comum para futuras telas agentic, reduzindo duplicacao de cliente, sidecar, timeline, estado e tratamento de interrupcao.
3. Mantem o controle critico no servidor, preservando governanca de capability, correlacao, tool call e validacao estrutural.
4. Aproxima a plataforma de um modelo de protocolo aberto de Generative UI, sem forcar troca de stack para React ou SDKs externos.

O resultado pratico e importante: o projeto ganha um eixo de evolucao para UI agentic sem precisar reescrever o produto inteiro. Isso prepara terreno para experiencias futuras mais ricas, inclusive com mais capacidades, mais adapters e outros dominios alem do varejo demo atual.

## 6. Conceitos necessarios para entender

### 6.1. Generative UI

Generative UI, neste contexto, nao significa gerar HTML livre no navegador. Significa que a interface e montada ou atualizada a partir de eventos estruturados produzidos pela execucao agentic. O backend nao devolve so um texto. Ele devolve estado, atividade, timeline, mensagens, deltas e marcos de renderizacao. A tela usa isso para construir uma experiencia viva.

### 6.2. AG-UI como contrato de interface

No projeto, AG-UI e o contrato que organiza essa conversa. Ele define inicio de run, fim de run, erro, mensagens incrementais, tool calls, snapshots de estado, deltas em JSON Patch, eventos customizados e resultado terminal. Em termos simples, ele padroniza como a tela acompanha o que o backend esta fazendo.

### 6.3. Capability governada

A tela nao pede SQL livre nem manda uma consulta arbitraria. Ela pede uma capability de negocio. Isso e importante porque desloca a responsabilidade da execucao para o servidor. O navegador diz a intencao. O backend decide a execucao real. Esse modelo reduz risco de seguranca, facilita auditoria e torna a interface previsivel.

### 6.4. Snapshot e delta

Snapshot e uma fotografia completa do estado. Delta e uma alteracao incremental. Os dois juntos permitem uma UI progressiva: a tela pode nascer vazia, receber um estado inicial e depois ser enriquecida sem recarregar tudo a cada mudanca.

### 6.5. Sidecar agentic

O sidecar e o painel auxiliar que acompanha a tela principal. Ele mostra mensagens, status, correlation_id, timeline de ferramentas e possiveis interrupcoes humanas. Isso evita jogar tudo dentro da area principal da pagina e permite que a interface explique o que esta fazendo sem quebrar o layout operacional.

### 6.6. DashboardSpec segura

No caso de dashboard dinamico, o backend nao manda HTML ou JavaScript arbitrario. Ele valida uma especificacao fechada, com tipos de widget, layout, fontes de dados governadas e travas de seguranca explicitas. Isso e o oposto de renderizar markup livre vindo da IA.

## 7. Como a feature funciona por dentro

A tela monta um pedido de run com threadId, runId, executionKind, usuario operacional, input de negocio, metadata da tela e uma fonte explicita de configuracao. O backend recebe esse pedido na rota dedicada, registra o correlation_id, escolhe o adapter adequado, emite o inicio do run e comeca a produzir eventos.

Se a capability for de consulta governada, o adapter resolve a query aprovada, executa a ferramenta dyn_sql canonicamente, publica o resultado como snapshot de estado e envia uma mensagem curta de conclusao. Se a capability for de dashboard dinamico, o fluxo muda: em vez de chamar dyn_sql diretamente, o backend valida uma DashboardSpec segura e vai materializando o painel por eventos, incluindo snapshot inicial, deltas de data sources, deltas de widgets e estado final pronto para renderizacao.

O frontend nao precisa conhecer o dominio interno. Ele so recebe eventos, atualiza o store, renderiza o sidecar e preenche a area principal da tela. Esse desenho e o que torna a feature reutilizavel por outras telas e por terceiros.

## 8. Divisao em etapas ou submodulos

### 8.1. Fronteira de protocolo

E a camada que define o que uma tela pode pedir e que tipo de evento ela pode receber. Ela existe para impedir contratos diferentes por pagina e para falhar fechado quando o payload sai do combinado.

### 8.2. Borda HTTP dedicada

E a camada que recebe o run AG-UI. Ela autentica, exige fonte explicita de configuracao e devolve SSE. Ela existe para separar esta experiencia das rotas legadas de agente e para garantir que Generative UI tenha um boundary proprio.

### 8.3. Orquestracao de lifecycle

E a camada que nao conhece varejo, SQL ou dashboard. Ela conhece apenas o lifecycle do run. Seu papel e iniciar a execucao, delegar ao adapter certo e garantir que sempre exista um encerramento coerente, seja de sucesso ou erro.

### 8.4. Adapter de dominio

E a camada que traduz a intencao de negocio em execucao concreta. No slice atual, esse adapter conhece o dominio PDV demo. Ele decide qual capability e permitida, qual query aprovada sera usada e quando o fluxo deve ir para materializacao de dashboard.

### 8.5. Materializacao dinamica de dashboard

E a camada que transforma especificacao de dashboard em eventos reconstituiveis. Ela existe para permitir flexibilidade visual sem abrir risco de HTML, JavaScript, SQL livre, segredos ou correlation_id vazando para o canvas.

### 8.6. Runtime compartilhado do frontend

E a camada que transforma eventos em experiencia de pagina. Ela inclui o cliente SSE por POST, o sidecar, o store de estado e o controller compartilhado das telas. Ela existe para que cada nova tela reaproveite o mesmo mecanismo sem duplicar infraestrutura.

## 9. Pipeline ou fluxo principal

O fluxo principal do slice AG-UI pode ser entendido assim.

1. A pagina abre e monta contexto operacional, como screenId, capability, periodo e YAML carregado no layout mestre.
2. O usuario dispara a execucao e a tela envia um POST para /ag-ui/runs.
3. O backend valida autenticacao e configuracao, resolve o correlation_id e inicia o run.
4. O orchestrator emite RUN_STARTED e entrega a execucao ao adapter registrado para aquele executionKind.
5. O adapter decide se a capability vira consulta governada ou materializacao de dashboard.
6. O backend passa a emitir eventos AG-UI: tool calls, mensagens, snapshots, deltas e custom events.
7. O cliente JavaScript aplica esses eventos no store compartilhado.
8. O sidecar mostra progresso, correlation_id, mensagens e timeline de ferramentas.
9. A tela principal renderiza o resultado governado ou o canvas dinamico.
10. O run termina com RUN_FINISHED ou RUN_ERROR.

A ordem importa porque ela separa responsabilidade de execucao, narracao da execucao e renderizacao da interface. Isso melhora diagnostico e reduz acoplamento.

## 10. Decisoes tecnicas e trade-offs

### 10.1. Implementacao local em vez de SDK externo

O projeto escolheu implementar o slice com FastAPI, Pydantic, SSE e JavaScript puro. O ganho e aderencia imediata a arquitetura existente. O custo e nao herdar automaticamente a infraestrutura pronta de um SDK externo. O efeito pratico e positivo: a plataforma controla o contrato e evolui no proprio ritmo.

### 10.2. Capabilities fechadas em vez de payload livre

O ganho e seguranca, governanca e previsibilidade operacional. O custo e exigir evolucao do catalogo de capabilities sempre que uma nova experiencia de negocio for criada. Essa escolha favorece produto enterprise, onde controle importa mais que liberdade irrestrita no navegador.

### 10.3. Dashboard por especificacao segura em vez de HTML gerado

O ganho e reduzir risco estrutural de injecao, DOM inseguro e vazamento de segredo. O custo e limitar a liberdade visual ao contrato permitido. A consequencia pratica e correta para o contexto do projeto: flexibilidade suficiente para dashboards ricos, sem perder controle.

### 10.4. HTML estatico e JavaScript puro em vez de React

O ganho e compatibilidade total com a UI atual do repositório. O custo e menos ecossistema pronto de componentes agentic. Em troca, o projeto prova que uma Generative UI governada nao depende de React para existir.

## 11. Como pode ser utilizado por terceiros e outras implementacoes

O ponto mais importante para terceiros e este: o slice local nao depende do layout exato das paginas demo. O contrato reutilizavel esta em tres blocos.

1. A rota POST /ag-ui/runs com payload tipado.
2. O stream de eventos AG-UI via SSE.
3. O cliente e store compartilhados do frontend.

Um terceiro pode reutilizar isso de duas formas.

A primeira e consumir o backend existente, criando uma tela nova que mande executionKind, capability, input contextual e fonte explicita de configuracao, e depois reconstrua o estado a partir dos eventos.

A segunda e implementar um novo backend compativel com o mesmo contrato, desde que respeite o lifecycle de eventos, o formato dos snapshots e a disciplina de nao empurrar logica insegura para o navegador. Em ambos os casos, a interface nao precisa ser igual a das demos atuais. O que precisa ser igual e o contrato.

Para outras implementacoes internas, o padrao recomendado e criar um novo adapter de dominio e registrá-lo por executionKind, sem alterar o orchestrator. Isso mantem o mesmo protocolo de interface e troca apenas a traducao de negocio.

## 12. Recursos avancados confirmados no codigo

Os recursos mais relevantes confirmados no slice atual sao estes.

1. Timeline de ferramentas com eventos de inicio, argumentos, fim e resultado.
2. Snapshot completo de estado para reconstruir a interface.
3. Deltas incrementais em JSON Patch para atualizar partes do estado sem refazer tudo.
4. Sidecar reutilizavel com mensagens, status, correlation_id e contexto da tela.
5. Adaptacao de interrupcoes para o painel HIL compartilhado.
6. Materializacao progressiva de dashboard por custom events e state deltas.
7. Retry explicito no cliente SSE do navegador.
8. Falha fechada para campos fora do contrato ou configuracao ausente.

Esses recursos mostram que a feature vai alem de streaming textual. Ela implementa uma interface agentic realmente dirigida por estado e eventos.

## 13. O que acontece em caso de sucesso

No caminho feliz, o usuario dispara a tela, o backend aceita o run, o sidecar abre, o correlation_id aparece, a ferramenta governada e executada ou o dashboard e materializado, a area principal recebe estado pronto e a execucao termina com sucesso. O usuario percebe isso como uma tela viva e explicativa, nao como um spinner opaco.

## 14. O que acontece em caso de erro

Os erros confirmados se dividem em tres grupos.

1. Erros de boundary, como falta de autenticacao ou falta de fonte de configuracao.
2. Erros de orquestracao, como executionKind sem adapter registrado.
3. Erros de dominio governado, como capability invalida, parametro fora do contrato, SQL livre bloqueado, configuracao PDV ausente ou DashboardSpec recusada.

Em todos esses casos, o desenho evita fallback silencioso. O erro aparece como erro explicito de contrato ou de execucao.

## 15. Impacto tecnico

Tecnicamente, a feature encapsula complexidade de streaming, estado, ferramentas, materializacao dinamica e HIL em um conjunto coeso de contratos reutilizaveis. Isso reduz acoplamento entre pagina e dominio, facilita testes, melhora observabilidade e cria uma base clara para novas telas agentic.

## 16. Impacto executivo

Executivamente, a feature reduz risco operacional de interfaces caixa-preta, melhora previsibilidade de execucao e cria uma capacidade clara de demonstrar automacao assistida com governanca. Isso ajuda lideranca a confiar mais no uso operacional da IA dentro do produto.

## 17. Impacto comercial

Comercialmente, a feature permite mostrar IA aplicada a problemas reais do cliente com uma interface visivel, rastreavel e segura. O valor tangivel esta em acelerar leitura operacional, reduzir dependencia de planilhas paralelas e transformar dados governados em experiencia interativa.

Promessa que pode ser feita: a plataforma entrega UI agentic governada com estado progressivo e integracao a capacidades aprovadas.

Promessa que nao pode ser feita com base no codigo lido: liberdade irrestrita para qualquer frontend executar qualquer acao arbitraria ou gerar interface livre sem contrato.

## 18. Impacto estrategico

Estrategicamente, esta implementacao fortalece a plataforma porque cria um pilar reutilizavel para novas experiencias agentic. Ela prepara o produto para crescer em torno de um contrato comum de interface, em vez de multiplicar telas isoladas com protocolos diferentes. Isso melhora governanca, reaproveitamento e velocidade futura.

## 19. Exemplos praticos guiados

### 19.1. Gestor comercial lendo o periodo

Cenario: um gerente quer resumir o desempenho de vendas do mes.

Entrada: a tela envia capability sales_summary com periodo e contexto da tela.

Processamento: o backend resolve a query aprovada, executa dyn_sql e publica o resultado como snapshot.

Saida: a pagina mostra o resultado governado e o sidecar explica a execucao.

### 19.2. Operacao digital monitorando checkout

Cenario: a equipe suspeita aumento de abandono.

Entrada: a tela envia capability checkout_funnel.

Processamento: o adapter traduz isso para uma query governada de funil.

Saida: a interface mostra o status do funil e a narrativa assistida no sidecar.

### 19.3. Lideranca pedindo um painel dinamico

Cenario: a area quer um dashboard com KPI, serie temporal, ranking e mix.

Entrada: a tela envia capability dashboard_dynamic com DashboardSpec segura.

Processamento: o backend valida a spec, rejeita conteudo proibido e materializa widgets por eventos.

Saida: o canvas nasce vazio e termina pronto, sem HTML livre vindo do agente.

## 20. Explicacao 101

Pense na feature como um tradutor entre a tela e o backend agentic. A tela diz o que quer fazer. O backend decide como fazer com seguranca. Durante a execucao, ele vai narrando o processo com eventos. A tela usa esses eventos para mostrar progresso, mensagens, ferramentas usadas e resultado final. Isso e a Generative UI local do projeto: uma interface que nasce e evolui a partir da execucao, mas sempre dentro de um contrato seguro.

## 21. Limites e pegadinhas

O slice atual ainda tem limites claros.

1. O adapter registrado por padrao e apenas retail_demo.
2. O transporte comprovado e SSE por POST, nao WebSocket.
3. O projeto nao prova, neste slice, uso direto de SDK Microsoft Agent Framework ou Google ADK.
4. A interface nao aceita SQL livre, HTML livre, JavaScript livre, segredos nem correlation_id vindo de DashboardSpec.
5. Nem toda capability imaginavel ja existe; a extensao correta passa por novo adapter ou novo catalogo governado.

## 22. Checklist de entendimento

- Entendi que a Generative UI local do projeto e o slice AG-UI.
- Entendi qual problema ela resolve.
- Entendi por que a interface trabalha por capability e eventos.
- Entendi como a tela e o backend se conectam.
- Entendi como terceiros podem reutilizar o contrato.
- Entendi quais recursos avancados ja existem.
- Entendi os limites atuais do slice.

## 23. Evidencias no codigo

- src/api/schemas/ag_ui_models.py
  - Motivo da leitura: confirmar o contrato dos eventos, do run e dos deltas.
  - Simbolo relevante: AgUiRunRequest, AgUiRunFinishedEvent, AgUiStateDeltaEvent.
  - Comportamento confirmado: a interface agentic local e governada por modelos estritos e JSON Patch.

- src/api/routers/ag_ui_router.py
  - Motivo da leitura: confirmar a fronteira publica da feature.
  - Simbolo relevante: POST /ag-ui/runs.
  - Comportamento confirmado: a Generative UI local tem rota dedicada, exige configuracao explicita e devolve SSE.

- src/api/services/ag_ui_run_orchestrator.py
  - Motivo da leitura: confirmar o lifecycle do run.
  - Simbolo relevante: AgUiRunOrchestrator.
  - Comportamento confirmado: o orchestrator emite RUN_STARTED e garante terminal de sucesso ou erro.

- src/api/services/ag_ui_retail_demo_adapter.py
  - Motivo da leitura: confirmar como a intencao de negocio vira execucao governada.
  - Simbolo relevante: RetailDemoAgUiAdapter.
  - Comportamento confirmado: o backend traduz capability em dyn_sql aprovado ou em materializacao de dashboard.

- src/api/services/ag_ui_dashboard_materialization.py
  - Motivo da leitura: confirmar o canvas dinamico por eventos.
  - Simbolo relevante: DashboardMaterializationService.
  - Comportamento confirmado: snapshot inicial, deltas, custom events e status final ready ou validation_failed.

- app/ui/static/js/shared/ag-ui-client.js
  - Motivo da leitura: confirmar o contrato reutilizavel do lado do navegador.
  - Simbolo relevante: createAgUiSseClient.
  - Comportamento confirmado: o cliente faz POST SSE, captura correlation_id e suporta retry explicito.

- app/ui/static/js/shared/ag-ui-retail-demo-page.js
  - Motivo da leitura: confirmar o padrao de tela reutilizavel.
  - Simbolo relevante: AgUiRetailDemoPageController.
  - Comportamento confirmado: uma nova pagina pode reutilizar o mesmo fluxo mudando screenId, capability e contexto.

- app/ui/static/js/shared/ag-ui-sidecar-chat.js
  - Motivo da leitura: confirmar sidecar e HIL compartilhados.
  - Simbolo relevante: createAgUiSidecarChat.
  - Comportamento confirmado: mensagens, tools, correlation_id e interrupcoes ficam desacoplados da area principal da tela.
