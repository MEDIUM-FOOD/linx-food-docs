# Documentação da Plataforma

Este índice organiza a leitura principal da documentação com base no
runtime atual.
A regra prática é simples: escolha o documento dono do assunto e trate
o código como fonte de verdade quando houver dúvida.

## Por onde começar

- README-ARQUITETURA.md para visão macro do sistema.
- GUIA-COMERCIAL-PLATAFORMA.md para visão estratégica do produto e dos
  módulos principais.
- API-ENDPOINTS-SWAGGER.md para famílias de rota e leitura do OpenAPI.
- README-SERVICE-API.md para o boundary consolidado da FastAPI.
- README-INGESTAO.md para ingestão e leitura operacional.
- README-ETL.md para ETL dedicado.
- README-RAG.md para consulta e runtime moderno de QA.
- README-AG-UI.md para protocolo de interface agentic, endpoint SSE,
  sidecar e telas demo PDV.
- README-HUMAN-IN-THE-LOOP.md para contrato de pausa humana, envelope
  `hil`, componente compartilhado de revisão e retomada governada.
- tutorial-101-generative-ui.md para onboarding guiado do AG-UI atual,
  do endpoint `/ag-ui/runs` até as telas estáticas da demo.
- tutorial-101-human-in-the-loop.md para onboarding guiado de HIL
  Generative UI entre DeepAgent, WebChats e sidecar AG-UI.
- tutorial-101-processo-completo-de-ingestao-e-rag.md para uma visão
  guiada, ponta a ponta, do fluxo combinado de ingestão e RAG.
- README-INTEGRACOES-GOVERNADAS.md para catálogo governado de integrações.
- README-CACHING.md para cache de infraestrutura, sessão e runtime.
- README-TESTS.MD para a suíte oficial e seus artefatos.
- metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md para onboarding de
  desenvolvimento assistido por Copilot, agentes, instruções e validação.
- GUIA-USUARIO-TOOLS.md para catálogo e famílias de tools.
- faq-onboarding.md para perguntas rápidas de onboarding.

## Sequência recomendada de leitura técnica

- Primeiro README-ARQUITETURA.md para entender processos, fronteiras e
  infraestrutura comum.
- Depois README-INGESTAO.md e README-ETL.md para entender os domínios
  assíncronos do worker.
- Depois README-RAG.md para entender o runtime moderno de consulta que
  roda sobre o acervo já preparado.
- Em seguida, tutorial-101-processo-completo-de-ingestao-e-rag.md se a
  meta for seguir a história inteira do request até o efeito final.

Em linguagem simples: a ordem mais segura é plataforma primeiro,
produção do acervo depois, consulta por último.

## Documentos donos por assunto

- Arquitetura: README-ARQUITETURA.md
- Estratégia e posicionamento: GUIA-COMERCIAL-PLATAFORMA.md
- API e OpenAPI: API-ENDPOINTS-SWAGGER.md
- Service API: README-SERVICE-API.md
- Ingestão: README-INGESTAO.md
- ETL: README-ETL.md
- RAG: README-RAG.md
- AG-UI: README-AG-UI.md
- Human in the Loop: README-HUMAN-IN-THE-LOOP.md
- AG-UI tutorial guiado: tutorial-101-generative-ui.md
- HIL tutorial guiado: tutorial-101-human-in-the-loop.md
- Integrações governadas: README-INTEGRACOES-GOVERNADAS.md
- Tools: GUIA-USUARIO-TOOLS.md
- YAML: README-CONFIGURACAO-YAML.md
- Caching: README-CACHING.md
- Testes: README-TESTS.MD
- Logging: README-LOGGING.md
- Scheduler: README-SCHEDULER.md
- Autenticação: README-SISTEMA-AUTENTICACAO.md
- MFA web: README-AUTENTICACAO-MFA.md
- Autorização: README-AUTORIZACAO.md
- Governança do Copilot: copilot-instructions-framework.md
- Metodologia de desenvolvimento assistido por Copilot:
  metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md
- AST agentic: README-AST-AGENTIC-DESIGNER.md
- Supervisor: README-AGENTE-SUPERVISOR.md
- DeepAgent: README-DEEPAGENTS-SUPERVISOR.md
- Workflow: README-AGENTE-WORKFLOW.md
- Schema de banco: README-SCHEMA-BANCO.md

## Como escolher a leitura certa

- Se o problema for rota, payload ou permissão, leia
  API-ENDPOINTS-SWAGGER.md.
- Se o problema for wiring da aplicação FastAPI, middlewares ou startup,
  leia README-SERVICE-API.md.
- Se o problema for worker, scheduler ou topologia do runtime, leia
  README-ARQUITETURA.md.
- Se o problema for ingestão, fan-out, status ou runs, leia
  README-INGESTAO.md.
- Se o problema for ETL, leia README-ETL.md.
- Se o problema for retrieval ou resposta ruim do QA, leia
  README-RAG.md.
- Se o problema for uma tela recebendo eventos de agente, sidecar,
  dashboard dinâmico ou endpoint `/ag-ui/runs`, leia README-AG-UI.md.
- Se o problema for pausa humana, envelope `hil`, `thread_id`,
  `resume_endpoint`, `HilContract` ou `HilReviewPanel`, leia
  README-HUMAN-IN-THE-LOOP.md.
- Se o problema for onboarding rápido do slice AG-UI, navegação entre
  telas demo ou primeiros passos locais, leia
  tutorial-101-generative-ui.md.
- Se o problema for onboarding rápido do HIL atual, reaproveitamento do
  review panel entre WebChat/Admin/AG-UI ou a ponte entre renderização e
  continue formal, leia tutorial-101-human-in-the-loop.md.
- Se o problema for importação Swagger, testes seguros ou catálogo
  técnico e funcional de integrações, leia
  README-INTEGRACOES-GOVERNADAS.md.
- Se o problema for invalidação, Redis, cache de sessão ou cache de
  runtime, leia README-CACHING.md.
- Se o problema for validação oficial, artefatos da suíte ou status do
  repositório, leia README-TESTS.MD.
- Se o problema for como desenvolver, corrigir, testar ou documentar usando
  Copilot e os agentes do repositório, leia
  metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md.
- Se o problema for gerar SQL proposta a partir de linguagem natural,
  dyn_sql, proc_sql ou schema_rag_sql, leia
  README-DYNAMIC-SQL-TOOLS.md.
- Se o problema for dyn_sql, dyn_api ou schema_rag_sql, leia
  GUIA-USUARIO-TOOLS.md.
- Se o problema for login web, access key, sessão ou conta pessoal, leia
  README-SISTEMA-AUTENTICACAO.md.
- Se o problema for desafio TOTP ou política de segundo fator, leia
  README-AUTENTICACAO-MFA.md.
- Se o problema for autoria governada por linguagem natural, leia
  README-AST-AGENTIC-DESIGNER.md.

## Rotas rápidas por assunto

Use estas sequências quando você já sabe o tema e quer navegar entre os
documentos principais sem ficar caçando link isolado.

### Arquitetura e operação

- Comece em [README-ARQUITETURA.md](./README-ARQUITETURA.md)
- Depois vá para [README-SERVICE-API.md](./README-SERVICE-API.md)
- Feche com [GUIA-DIDATICO-EXECUCAO-CANAIS.md](./GUIA-DIDATICO-EXECUCAO-CANAIS.md)

### Ingestão até resposta RAG

- Comece em [README-INGESTAO.md](./README-INGESTAO.md)
- Depois vá para [README-RAG.md](./README-RAG.md)
- Feche com [tutorial-101-processo-completo-de-ingestao-e-rag.md](./tutorial-101-processo-completo-de-ingestao-e-rag.md)

### Supervisor, DeepAgent e Workflow

- Comece em [README-AGENTE-SUPERVISOR.md](./README-AGENTE-SUPERVISOR.md)
- Depois compare com [README-DEEPAGENTS-SUPERVISOR.md](./README-DEEPAGENTS-SUPERVISOR.md)
- Feche com [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md)

### YAML agentic e montagem governada

- Comece em [README-AGENTIC-INICIANTES.md](./README-AGENTIC-INICIANTES.md)
- Depois vá para [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md)
- Feche com [README-AST-AGENTIC-DESIGNER.md](./README-AST-AGENTIC-DESIGNER.md)

### Human in the Loop e Generative UI de revisão

- Comece em [README-HUMAN-IN-THE-LOOP.md](./README-HUMAN-IN-THE-LOOP.md)
- Depois vá para [tutorial-101-human-in-the-loop.md](./tutorial-101-human-in-the-loop.md)
- Feche com [README-AG-UI.md](./README-AG-UI.md)
  se a dúvida envolver sidecar, eventos ou renderização da pausa humana

### AG-UI e telas agentic

- Comece em [README-AG-UI.md](./README-AG-UI.md)
- Depois vá para [tutorial-101-generative-ui.md](./tutorial-101-generative-ui.md)
- Feche com [README-HUMAN-IN-THE-LOOP.md](./README-HUMAN-IN-THE-LOOP.md)
  ou [tutorial-101-human-in-the-loop.md](./tutorial-101-human-in-the-loop.md)
  se a dúvida envolver pausa humana, review panel ou continue formal

### Metodologia de desenvolvimento com Copilot

- Comece em [metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md](./metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md)
- Depois vá para [metodologia-desenv/README-METODOLOGIA-DESENV-ONBOARDING.md](./metodologia-desenv/README-METODOLOGIA-DESENV-ONBOARDING.md)
- Feche com [metodologia-desenv/README-METODOLOGIA-DESENV-FLUXOS-TRABALHO.md](./metodologia-desenv/README-METODOLOGIA-DESENV-FLUXOS-TRABALHO.md)
  quando a dúvida for qual agente acionar em cada tipo de tarefa

## Como usar esta documentação

1. Consulte primeiro o documento dono do assunto.
2. Confirme no código qualquer decisão de contrato sensível.
3. Use o OpenAPI do runtime antes de integrar um endpoint.
4. Em YAML agentic, prefira o fluxo AST quando a mudança afetar
   workflows, multi_agents, selected_workflow, selected_supervisor ou
   tools_library.

## Como rodar e validar

1. Suba o ambiente local com os papéis necessários.
2. Acesse /docs com uma credencial autorizada.
3. Valide um endpoint curto e um endpoint assíncrono.
4. Correlacione request, task_id e correlation_id antes de concluir.

## Evidência no código

- src/api/service_api.py
- src/api/routers/
- src/api/routers/config_resolution.py
- src/config/config_cli/configuration_factory.py
- src/config/agentic_assembly/
- src/services/ingestion_service.py
- src/services/etl_service.py
- src/qa_layer/content_qa_system.py

## Lacunas no código

Não encontrado no código.

Onde deveria estar:

- um gerador automático de índice Markdown a partir do OpenAPI e do
  catálogo de tools;
- uma exportação administrativa pronta dos documentos donos e suas
  superfícies reais do runtime.
