# Manual conceitual, executivo, comercial e estratégico: AG-UI com referencial Google e Microsoft aplicado ao ERP

## 1. O que é esta feature

AG-UI é o protocolo de interface agentic usado para ligar uma aplicação voltada ao usuário a um backend de agente que precisa trabalhar em tempo real, emitir eventos intermediários, atualizar estado, mostrar ferramentas acionadas e encerrar com sucesso, erro ou interrupção humana. No ecossistema oficial do protocolo, ele aparece como uma camada aberta e orientada por eventos, com integrações documentadas para Microsoft Agent Framework e Google ADK. No projeto, porém, ele não foi implementado através desses SDKs. O que existe aqui é uma implementação própria e governada do mesmo modelo conceitual, baseada em FastAPI, SSE, modelos Pydantic estritos e frontend em HTML estático com JavaScript puro.

Isso importa porque evita duas leituras erradas. A primeira seria achar que o projeto consome diretamente um kit AG-UI da Microsoft ou do Google. O código lido não confirma isso. A segunda seria achar que o projeto inventou um streaming qualquer sem relação com o protocolo externo. Também não é isso. O slice atual implementa um contrato de eventos, lifecycle e sincronização de estado que conversa com a lógica do AG-UI oficial, mas foi adaptado à arquitetura real do repositório: API própria, adapters governados, DeepAgent/YAML e telas administrativas estáticas.

## 2. Que problema ela resolve

O problema real é que telas agentic de negócio não cabem bem em request/response simples. Um cockpit comercial, um radar de checkout ou um canvas de dashboard precisam mostrar que a execução começou, qual capability foi escolhida, qual consulta governada foi disparada, quando o estado mudou, quando um widget apareceu e se o processo terminou ou pediu revisão humana. Sem um protocolo como AG-UI, cada tela tenderia a criar um mini-contrato próprio de progresso, mensagens e erros.

No projeto, AG-UI resolve isso criando uma fronteira única entre backend e frontend. A tela abre um único canal para a rota dedicada, o backend continua sendo a autoridade da execução e a interface recebe um stream tipado de eventos reconstituíveis. Isso reduz acoplamento entre tela e motor interno e permite que várias superfícies ERP reutilizem o mesmo protocolo com objetivos diferentes.

## 3. Visão executiva

Executivamente, AG-UI importa porque transforma execução agentic em ativo operacional observável. Em vez de depender apenas de uma resposta final, o sistema passa a mostrar trabalho em andamento, trilha de ferramenta, montagem incremental de dashboard e correlação da execução. Isso reduz a sensação de caixa-preta e melhora suporte, demonstração, governança e confiança do usuário final.

Também reduz custo de evolução. Quando a plataforma quer abrir uma nova tela operacional para vendas, checkout ou catálogo, ela não precisa reinventar toda a camada de streaming e estado. O investimento vai para a capability de negócio e para o adapter governado, não para plumbing repetido de transporte.

## 4. Visão comercial

Comercialmente, AG-UI ajuda a vender experiência agentic aplicada ao ERP, não apenas backend inteligente. O cliente vê uma tela que recebe progresso, explica o que o agente está fazendo, mostra resultado governado e, quando necessário, prepara uma interface dinâmica segura. Isso é diferente de um chat genérico ou de um dashboard estático sem contexto operacional.

No cenário lido no código, isso aparece em quatro histórias vendáveis.

1. Cockpit executivo de vendas, que cruza período, KPIs e leitura assistida sem expor SQL ao navegador.
2. Radar de checkout e UCP, que acompanha gargalos do funil e explica eventos de abandono, cancelamento e conversão.
3. Central de oportunidades de catálogo, que cruza estoque, oferta e vendas para priorizar ação comercial.
4. Canvas dinâmico de dashboard, que materializa widgets e fontes de dados de forma progressiva e segura.

Essas histórias são fortes porque combinam dados governados, interface progressiva e explicação agentic no mesmo fluxo.

## 5. Visão estratégica

Estrategicamente, o slice AG-UI fortalece a plataforma em cinco pontos.

1. Cria um contrato estável de interface agentic separado do WebChat e dos endpoints legados.
2. Permite reaproveitar a mesma base de backend para várias telas ERP com experiências diferentes.
3. Mantém o controle crítico no servidor: correlation_id, policy de capability, escolha da tool e validação de dashboard.
4. Aproxima o projeto do ecossistema aberto do AG-UI sem forçar dependência imediata de SDK externo.
5. Prepara terreno para evoluções futuras como HIL mais rico, composição de subagentes, estado compartilhado mais amplo e integrações adicionais.

## 6. Conceitos necessários para entender

### 6.1. AG-UI como protocolo aberto

No referencial oficial, AG-UI é descrito como protocolo leve e orientado por eventos que conecta aplicações voltadas ao usuário a backends agentic. A documentação oficial também posiciona o protocolo dentro de uma pilha maior de padrões agentic, ao lado de MCP para tools e A2A para comunicação entre agentes. Isso ajuda a entender o papel do slice local: ele não substitui integração com dados ou com outros agentes; ele trata da interface do usuário com a execução.

### 6.2. Google ADK e Microsoft Agent Framework como referencial, não como dependência local

O site oficial do protocolo lista Microsoft Agent Framework e Google ADK como integrações first-party suportadas. Isso significa que o ecossistema AG-UI já se apresenta como compatível com frameworks desses fornecedores. No entanto, o código lido neste repositório não mostra SDK Microsoft Agent Framework nem Google ADK no slice AG-UI. O projeto implementa o contrato de forma própria, usando sua arquitetura existente. Em termos simples: o repositório segue a ideia do protocolo, mas não incorpora essas stacks como runtime local do AG-UI atual.

### 6.3. SSE como transporte atual

O protocolo oficial admite diferentes transportes. A implementação local escolheu Server-Sent Events na rota dedicada. Isso simplifica consumo em telas administrativas e mantém o browser em um modelo unidirecional de recebimento de eventos enquanto o POST continua sendo o gatilho da execução.

### 6.4. Adapter de domínio

Adapter é a peça que traduz um domínio de negócio para o protocolo AG-UI. O orquestrador não sabe nada sobre varejo, SQL ou dashboard. Ele sabe apenas iniciar run, delegar ao adapter certo e garantir evento terminal coerente.

### 6.5. Capability governada

Capability é a ação de negócio permitida ao frontend pedir. No slice atual, a UI não manda SQL livre nem monta query arbitrária. Ela pede capabilities como sales_summary, checkout_funnel, catalog_opportunities e dashboard_dynamic. O backend valida essa capability, escolhe a query aprovada ou a materialização segura correspondente e então emite eventos.

### 6.6. Snapshot, delta e custom event

Snapshot é o estado inteiro. Delta é a mudança incremental, no caso local via JSON Patch. Custom event é um evento adicional documentado pela própria aplicação, usado aqui para marcos de materialização do dashboard. Esses três elementos são a base do comportamento progressivo das telas ERP do projeto.

## 7. Como a feature funciona por dentro

Conceitualmente, o caminho é simples. A tela ERP não conversa com o domínio interno. Ela monta um pedido AG-UI, aponta para a rota dedicada e fica ouvindo eventos. O backend recebe esse pedido, cria o contexto do run, escolhe um adapter compatível com o tipo de execução, transforma a intenção de negócio em ferramentas ou materialização segura e transmite a história da execução como stream.

No slice atual, esse contrato foi moldado para varejo e analytics de ERP. Por isso a ponte concreta passa por um adapter de demonstração de varejo. Esse adapter resolve capabilities fechadas de vendas, checkout e catálogo para queries dyn_sql aprovadas, e resolve a capability de dashboard dinâmico para uma esteira de validação e materialização de DashboardSpec. O frontend então reconstitui o estado em sidecar, timeline de tools, cards de resultado e canvas dinâmico.

## 8. Divisão em etapas ou submódulos

### 8.1. Contrato de protocolo

É a camada que define o formato aceito pelo frontend e o tipo oficial dos eventos. Ela existe para impedir que cada tela invente seu próprio payload.

### 8.2. Borda HTTP dedicada

É a camada que recebe o run AG-UI, valida permissão, exige fonte explícita de configuração YAML e devolve SSE. Ela existe para isolar AG-UI dos endpoints legados.

### 8.3. Orquestração de lifecycle

É a camada que emite início, delega ao adapter certo e garante fechamento terminal consistente. Ela existe para desacoplar transporte do domínio.

### 8.4. Adapter de domínio varejo

É a camada que interpreta capability e transforma intenção ERP em consulta governada ou dashboard dinâmico. Ela existe para impedir regra de negócio no frontend e SQL livre vindo do navegador.

### 8.5. Runtime de frontend compartilhado

É a camada que recebe eventos, reconstrói estado, renderiza sidecar e atualiza a página principal. Ela existe para evitar duplicação de cliente SSE e store por tela.

## 9. Como usar em telas de um sistema ERP

O uso correto em ERP, segundo o desenho comprovado, segue um padrão repetível.

Primeiro, cada tela define uma intenção de negócio clara, não uma query livre. Depois, a tela monta um contexto mínimo com screenId, título, período e capability. Em seguida, ela usa o cliente SSE compartilhado para abrir um POST em /ag-ui/runs. A resposta vai atualizando sidecar, status operacional e área principal da tela conforme os eventos chegam.

O projeto já mostra quatro variações concretas desse padrão.

### 9.1. Cockpit executivo de vendas

É a tela para gestor comercial que quer receita, ticket médio e leitura assistida do período. O backend traduz isso para a capability sales_summary e para a query governada pdv_vendas_kpis_periodo.

### 9.2. Radar checkout e UCP

É a tela para operação digital que quer enxergar sessões, cancelamentos, conversão e gargalos do funil. O backend traduz isso para checkout_funnel e para a query governada pdv_checkout_funil_status.

### 9.3. Central de oportunidades de catálogo

É a tela para comercial e estoque que quer cruzar saldo, preço, oferta e vendas. O backend traduz isso para catalog_opportunities e para a query governada pdv_catalogo_estoque_oportunidades.

### 9.4. Canvas dinâmico de dashboard

É a tela para gestão que quer montar um painel progressivo com widgets seguros e data sources aprovadas. O backend traduz isso para dashboard_dynamic e para a validação e materialização de DashboardSpec, sem liberar HTML, JavaScript, segredo ou correlation_id para dentro da spec.

## 10. Casos de uso e exemplos de negócio

### 10.1. Reunião diária de vendas

Um gerente abre o cockpit AG-UI e pede leitura do período. O sistema dispara a capability de vendas, mostra a chamada da tool governada, entrega o snapshot com o resultado e usa o sidecar para explicar o indicador. O valor de negócio é acelerar leitura executiva sem planilha paralela nem SQL manual.

### 10.2. Sala de guerra de checkout

Uma operação digital suspeita aumento de abandono. A tela de checkout usa AG-UI para mostrar o funil em tempo real e o sidecar explica os gargalos. O valor de negócio é identificar rapidamente onde a conversão caiu sem expor o banco a consultas improvisadas do navegador.

### 10.3. Prioridade comercial de estoque e oferta

O time de catálogo precisa entender quais itens têm estoque alto e baixa tração. A central de oportunidades pede a capability apropriada, recebe resultado governado e usa o sidecar para contextualizar prioridades. O valor de negócio é priorizar ação comercial usando dados confiáveis e narrativa assistida.

### 10.4. Montagem de painel executivo seguro

Uma liderança quer um dashboard composto por KPI, série temporal, ranking e mix. Em vez de renderizar HTML livre vindo do agente, o projeto usa DashboardSpec validado, eventos progressivos de materialização e canvas sob controle da aplicação. O valor de negócio é ganhar flexibilidade visual sem abrir risco de segurança na interface.

## 11. Decisões técnicas e trade-offs

### 11.1. Implementar AG-UI localmente em vez de depender de SDK Microsoft ou Google

Ganho: aderência imediata à arquitetura do repositório, sem trocar stack frontend nem runtime backend.

Custo: o projeto não recebe automaticamente todos os recursos e middlewares dos SDKs do ecossistema oficial.

Conseqüência prática: a implementação local é mais controlada, mas cobre um subconjunto governado do espaço AG-UI.

### 11.2. Usar HTML estático e JavaScript puro

Ganho: integração direta com a UI atual do projeto e menor custo de stack.

Custo: menos ecossistema pronto do que um cliente React dedicado.

Conseqüência prática: o projeto comprova que AG-UI pode servir ERP estático sem depender de React ou CopilotKit.

### 11.3. Capabilities fechadas em vez de payload livre

Ganho: segurança, governança e previsibilidade.

Custo: menos flexibilidade para experimentação espontânea pelo navegador.

Conseqüência prática: a evolução da tela depende do catálogo de capabilities, não de SQL ou markup soltos.

## 12. O que acontece em caso de sucesso

No caminho feliz, a tela envia um run com fonte explícita de configuração, o backend devolve RUN_STARTED, o adapter executa a capability governada ou materializa o dashboard, a UI recebe snapshots, deltas e mensagens, e a execução termina com RUN_FINISHED. O operador percebe isso como tela viva, sidecar aberto, resultado principal preenchido e correlation_id disponível para rastreabilidade.

## 13. O que acontece em caso de erro

Os erros confirmados no slice lido caem em quatro famílias.

1. Erro de autenticação ou permissão na borda.
2. Erro por ausência de fonte explícita de configuração YAML.
3. Erro de adapter não registrado para o executionKind.
4. Erro de negócio governado, como capability inválida, SQL livre bloqueado, parâmetro inválido ou configuração PDV ausente.

## 14. Limites e pegadinhas

O slice atual não comprova integração direta com Microsoft Agent Framework nem Google ADK. Eles entram aqui como referencial oficial do ecossistema AG-UI, não como dependência do código local.

O slice atual também não comprova WebSocket, voz, anexos multimodais, frontend tool calls arbitrárias nem retomada formal de resume input na rota dedicada. O contrato de modelos tem elementos para continuidade e interrupção, mas a superfície executável lida nesta tarefa está centrada em POST com SSE e outcome de interrupção no encerramento do run.

Também é erro tratar AG-UI como sinônimo de chat. O caso mais rico lido no projeto é justamente um canvas de dashboard materializado por eventos.

## 15. Explicação 101

Se fosse para explicar de forma simples: o protocolo AG-UI é como a língua que a tela fala com o agente. Google e Microsoft aparecem no ecossistema oficial como frameworks que podem falar essa língua. O projeto, porém, construiu seu próprio tradutor local dessa língua. Quando uma tela ERP pede um cockpit de vendas ou um radar de checkout, ela não fala direto com SQL, nem com o agente bruto. Ela manda um pedido padronizado e vai recebendo a história do que está acontecendo até o resultado final.

## 16. Checklist de entendimento

- Entendi o que é AG-UI no ecossistema oficial.
- Entendi por que Microsoft Agent Framework e Google ADK entram aqui como referencial externo.
- Entendi que o projeto não usa esses SDKs diretamente no slice atual.
- Entendi como a implementação local foi desenhada com FastAPI, SSE, adapters e JS puro.
- Entendi como reutilizar o protocolo em telas ERP de vendas, checkout, catálogo e dashboard.
- Entendi os principais casos de uso e exemplos de negócio.
- Entendi os limites do que está implementado hoje.

## 17. Evidências no código

- src/api/schemas/ag_ui_models.py
  - Motivo da leitura: confirmar contrato tipado de request, eventos, deltas e outcome interrupt.
  - Símbolo relevante: AgUiRunRequest, AgUiRunStartedEvent, AgUiRunFinishedEvent, AgUiStateDeltaEvent.
  - Comportamento confirmado: fronteira estrita e aliases oficiais do protocolo.

- src/api/routers/ag_ui_router.py
  - Motivo da leitura: confirmar endpoint dedicado, exigência de config explícita e retorno SSE.
  - Símbolo relevante: POST /ag-ui/runs.
  - Comportamento confirmado: AG-UI roda em rota própria e devolve X-Correlation-Id.

- src/api/services/ag_ui_run_orchestrator.py
  - Motivo da leitura: confirmar lifecycle do run e resolução por executionKind.
  - Símbolo relevante: AgUiRunOrchestrator.
  - Comportamento confirmado: emite RUN_STARTED, delega ao adapter e garante terminal RUN_FINISHED ou RUN_ERROR.

- src/api/services/ag_ui_retail_demo_adapter.py
  - Motivo da leitura: confirmar tradução de capabilities ERP para dyn_sql governado e dashboard dinâmico.
  - Símbolo relevante: RetailDemoAgUiAdapter, RetailDemoQueryCatalog.
  - Comportamento confirmado: browser não envia SQL livre; capability define query aprovada.

- src/api/services/ag_ui_dashboard_materialization.py
  - Motivo da leitura: confirmar materialização progressiva de DashboardSpec.
  - Símbolo relevante: DashboardMaterializationService.
  - Comportamento confirmado: snapshot inicial, deltas, custom events e estado ready ou validation_failed.

- app/ui/static/js/shared/ag-ui-client.js
  - Motivo da leitura: confirmar cliente SSE real do frontend.
  - Símbolo relevante: createAgUiSseClient.
  - Comportamento confirmado: POST + text/event-stream + retry explícito sem correlation_id inventado no browser.

- app/ui/static/ui-admin-plataforma-ag-ui-vendas-cockpit.html
  - Motivo da leitura: confirmar tela ERP de vendas.
  - Comportamento confirmado: cockpit executivo de vendas com filtros, sidecar e área de resultado governado.

- app/ui/static/ui-admin-plataforma-ag-ui-checkout-radar.html
  - Motivo da leitura: confirmar tela ERP de checkout.
  - Comportamento confirmado: radar de checkout e UCP com filtros e resultado governado.

- app/ui/static/ui-admin-plataforma-ag-ui-catalogo-central.html
  - Motivo da leitura: confirmar tela ERP de catálogo.
  - Comportamento confirmado: central de oportunidades com sidecar e resultado governado.

- app/ui/static/ui-admin-plataforma-ag-ui-dashboard-dinamico.html
  - Motivo da leitura: confirmar tela ERP de canvas dinâmico.
  - Comportamento confirmado: workbench de dashboard com histórico, canvas e ações de reset e retry.
