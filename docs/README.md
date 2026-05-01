# Como Ler Esta Documentação

Esta documentação foi reorganizada para responder perguntas de arquitetura e operação, não para ensinar alguém a caçar arquivo. O objetivo é simples: permitir que quem trabalha com o projeto entenda a lógica do sistema, o papel de cada domínio, os contratos importantes e os pontos de diagnóstico sem depender de um mapa manual do repositório.

## 1. Qual problema este índice resolve

O problema anterior era de navegação conceitual. Havia muitos documentos, mas a entrada principal ainda induzia um comportamento ruim: escolher o próximo texto pelo nome do arquivo, e não pela dúvida real.

Este índice corrige isso organizando a leitura por intenção.

- Entender a plataforma como sistema.
- Entender como o acervo é produzido.
- Entender como a plataforma consulta esse acervo.
- Entender como fluxos agentic são governados.
- Entender como operar, investigar e validar.

## 2. Se você quer entender a plataforma inteira

Comece por [README-ARQUITETURA.md](./README-ARQUITETURA.md). Esse documento explica a separação entre API, worker e scheduler, o papel do YAML, o uso de correlation_id e a diferença entre produção de acervo, consulta e execução assíncrona.

Depois leia [README-INGESTAO.md](./README-INGESTAO.md) e [README-RAG.md](./README-RAG.md). Essa sequência é importante porque RAG faz mais sentido quando você já entendeu como o acervo foi criado.

## 3. Se você quer entender produção de acervo

Use esta trilha.

1. [README-INGESTAO.md](./README-INGESTAO.md) para o fluxo documental oficial.
2. [README-ETL.md](./README-ETL.md) quando a dúvida for transformação estruturada e não ingestão documental.
3. [tutorial-101-processo-completo-de-ingestao-e-rag.md](./tutorial-101-processo-completo-de-ingestao-e-rag.md) quando a meta for seguir a história inteira, do pedido até a resposta.

O ponto conceitual aqui é este: ingestão produz acervo consultável; ETL transforma dados; nenhum dos dois deve ser entendido como sinônimo de consulta.

## 4. Se você quer entender consulta e resposta

Use esta trilha.

1. [README-RAG.md](./README-RAG.md) para entender runtime moderno, análise de pergunta, retrieval, fusão e geração.
2. [README-CACHING.md](./README-CACHING.md) se a dúvida for reaproveitamento de contexto, invalidação ou comportamento de runtime.
3. [tutorial-101-rag.md](./tutorial-101-rag.md) se você precisa de uma leitura didática e guiada antes do manual técnico completo.

## 5. Se você quer entender YAML, AST e governança agentic

Esse assunto precisa ser lido pela lógica de decisão, não por jargão.

- [README-AGENTIC-INICIANTES.md](./README-AGENTIC-INICIANTES.md) explica o modelo mental.
- [README-AST-AGENTIC-DESIGNER.md](./README-AST-AGENTIC-DESIGNER.md) explica por que a AST existe e como ela governa edição, validação e compilação.
- [README-DEEPAGENTS-SUPERVISOR.md](./README-DEEPAGENTS-SUPERVISOR.md) explica agentes governados.
- [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md) explica grafo determinístico e execução por nós.

## 6. Se você quer entender a API e a operação

A ordem mais produtiva costuma ser esta.

1. [README-SERVICE-API.md](./README-SERVICE-API.md) para entender o boundary HTTP real.
2. [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md) para entender como a configuração vira runtime executável.
3. [API-ENDPOINTS-SWAGGER.md](./API-ENDPOINTS-SWAGGER.md) para ver o contrato público atual.
4. [GUIA-DIDATICO-EXECUCAO-CANAIS.md](./GUIA-DIDATICO-EXECUCAO-CANAIS.md) se a dúvida envolver topologia operacional, filas e runners.
5. [README-LOGGING.md](./README-LOGGING.md) quando o problema for rastreabilidade e correlação.
6. [README-TESTS.MD](./README-TESTS.MD) quando a pergunta for validação oficial.

## 7. Se você quer entender UI agentic e pausa humana

Esses dois assuntos se cruzam, mas não são a mesma coisa.

- [README-AG-UI.md](./README-AG-UI.md) cobre o protocolo de interface orientado a eventos e o sidecar compartilhado.
- [README-HUMAN-IN-THE-LOOP.md](./README-HUMAN-IN-THE-LOOP.md) cobre a pausa humana, o envelope de aprovação e a retomada governada.
- [tutorial-101-generative-ui.md](./tutorial-101-generative-ui.md) e [tutorial-101-human-in-the-loop.md](./tutorial-101-human-in-the-loop.md) ajudam quando você precisa primeiro de uma história guiada antes da visão técnica completa.

## 8. Ordem recomendada para onboarding técnico

Se você está chegando agora no projeto, a ordem mais segura é esta.

1. Arquitetura.
2. Ingestão.
3. RAG.
4. Assembly agentic.
5. API e operação.
6. Tutoriais 101 especializados.

Essa ordem reduz um erro comum: tentar entender um slice muito específico sem antes entender o contrato macro da plataforma.

Se a necessidade for uma entrada rápida em linguagem simples, complemente essa trilha com [faq-onboarding.md](./faq-onboarding.md).

## 9. Catálogo completo de READMEs e tutoriais 101

Além das trilhas por intenção, este índice também precisa cumprir uma função operacional: servir como catálogo rastreável dos manuais principais e dos tutoriais 101 da pasta docs. Isso evita dois problemas práticos: documento importante esquecido fora da navegação e leitura errada guiada só por busca textual.

### 9.1 Plataforma, arquitetura e operação

- [README-ARQUITETURA.md](./README-ARQUITETURA.md): visão macro da plataforma, separação entre API, worker, scheduler, YAML-first e contratos transversais.
- [README-SERVICE-API.md](./README-SERVICE-API.md): boundary HTTP, responsabilidades da API e como o runtime expõe capacidades para clientes e operação.
- [README-LOGGING.md](./README-LOGGING.md): rastreabilidade, correlação e leitura operacional de logs.
- [README-TESTS.MD](./README-TESTS.MD): estratégia oficial de validação automatizada.
- [README-SCHEDULER.md](./README-SCHEDULER.md): processamento agendado, responsabilidades do scheduler e relação com o restante do runtime.

### 9.2 Produção de acervo, ingestão e ETL

- [README-INGESTAO.md](./README-INGESTAO.md): pipeline oficial de ingestão documental.
- [README-ETL.md](./README-ETL.md): transformação estruturada de dados quando o problema não é ingestão documental.
- [tutorial-101-ingestao.md](./tutorial-101-ingestao.md): introdução guiada à ingestão.
- [tutorial-101-ingestao-pdf.md](./tutorial-101-ingestao-pdf.md): leitura 101 do pipeline de ingestão de PDF.
- [tutorial-101-ingestao-excel-e-rag-de-excel.md](./tutorial-101-ingestao-excel-e-rag-de-excel.md): ingestão e consulta sobre planilhas.
- [tutorial-101-etl.md](./tutorial-101-etl.md): tutorial didático sobre ETL no contexto da plataforma.
- [tutorial-101-processo-completo-de-ingestao-e-rag.md](./tutorial-101-processo-completo-de-ingestao-e-rag.md): narrativa ponta a ponta do pedido até a resposta.
- [tutorial-101-auto-config-e-bm25-no-pipeline-de-ingestao-e-rag.md](./tutorial-101-auto-config-e-bm25-no-pipeline-de-ingestao-e-rag.md): ajuste de comportamento de retrieval dentro do pipeline.
- [tutorial-101-bm25-vs-auto-config-dnit.md](./tutorial-101-bm25-vs-auto-config-dnit.md): comparação guiada de estratégias de recuperação em um caso real.

### 9.3 Consulta, RAG e caching

- [README-RAG.md](./README-RAG.md): runtime moderno de pergunta e resposta com retrieval e geração.
- [README-CACHING.md](./README-CACHING.md): reaproveitamento de contexto, invalidação e impacto operacional do cache.
- [README_RAG_DNIT.md](./README_RAG_DNIT.md): leitura aplicada do caso DNIT.
- [tutorial-101-rag.md](./tutorial-101-rag.md): introdução 101 ao funcionamento de RAG.
- [tutorial-101-status-feature-geracao-sql-linguagem-natural.md](./tutorial-101-status-feature-geracao-sql-linguagem-natural.md): estado atual e limites da geração SQL por linguagem natural.

### 9.4 YAML, AST, assembly e fluxos agentic

- [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md): ciclo de vida da configuração até o runtime.
- [README-AGENTIC-INICIANTES.md](./README-AGENTIC-INICIANTES.md): modelo mental inicial do ecossistema agentic.
- [README-AST-AGENTIC-DESIGNER.md](./README-AST-AGENTIC-DESIGNER.md): AST como fonte tipada de verdade do assembly agentic.
- [README-AGENTIC-CONTRATO-COMUM.md](./README-AGENTIC-CONTRATO-COMUM.md): contrato compartilhado entre espinhas dorsais agentic.
- [README-AGENTE-SUPERVISOR.md](./README-AGENTE-SUPERVISOR.md): supervisor clássico e seus contratos.
- [README-DEEPAGENTS-SUPERVISOR.md](./README-DEEPAGENTS-SUPERVISOR.md): deepagents governados por supervisor.
- [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md): workflow determinístico baseado em grafo.
- [tutorial-101-agentes.md](./tutorial-101-agentes.md): explicação introdutória do conceito de agentes.
- [tutorial-101-deepagents.md](./tutorial-101-deepagents.md): leitura 101 sobre deepagents.
- [tutorial-101-workflow.md](./tutorial-101-workflow.md): tutorial guiado sobre workflows.
- [tutorial-101-criacao-de-agentes-via-yaml.md](./tutorial-101-criacao-de-agentes-via-yaml.md): criação orientada por YAML.
- [tutorial-101-nl2yaml.md](./tutorial-101-nl2yaml.md): tradução de linguagem natural para YAML.
- [tutorial-101-tecnica-nl-para-yaml-e-dsl.md](./tutorial-101-tecnica-nl-para-yaml-e-dsl.md): fundamentos da técnica de NL para YAML e DSL.
- [tutorial-101-configuracao-yaml-execucao-local-e-nl-para-sql.md](./tutorial-101-configuracao-yaml-execucao-local-e-nl-para-sql.md): combinação entre configuração local e NL para SQL.

### 9.5 Tools, schemas, integrações e superfícies especializadas

- [README-DYNAMIC-API-TOOLS.md](./README-DYNAMIC-API-TOOLS.md): tools dinâmicas orientadas a APIs.
- [README-DYNAMIC-SQL-TOOLS.md](./README-DYNAMIC-SQL-TOOLS.md): tools dinâmicas orientadas a SQL.
- [README-SCHEMA-BANCO.md](./README-SCHEMA-BANCO.md): organização e papel do schema do banco.
- [README-SQL-SCHEMA-RAG-TOOL.md](./README-SQL-SCHEMA-RAG-TOOL.md): tool especializada no schema SQL do RAG.
- [README-GOOGLE-UCP.md](./README-GOOGLE-UCP.md): integração e contrato do fluxo Google UCP.
- [README-MCP.md](./README-MCP.md): capacidades ligadas ao ecossistema MCP.
- [README-INTEGRACOES-GOVERNADAS.md](./README-INTEGRACOES-GOVERNADAS.md): integração com controles e governança explícita.
- [README-EXEMPLOS-INTEGRACAO-API.md](./README-EXEMPLOS-INTEGRACAO-API.md): documento explicitamente voltado a exemplos completos de uso de APIs.
- [tutorial-101-exemplos-api-deepagent-hil-execute-continue.md](./tutorial-101-exemplos-api-deepagent-hil-execute-continue.md): tutorial de exemplos completos de API para deepagent com pausa humana.

### 9.6 Interface, pausa humana e canais externos

- [README-AG-UI.md](./README-AG-UI.md): protocolo e arquitetura de interface agentic.
- [README-HUMAN-IN-THE-LOOP.md](./README-HUMAN-IN-THE-LOOP.md): pausa humana, aprovação e retomada governada.
- [README-INSTAGRAM-PROVISIONING.md](./README-INSTAGRAM-PROVISIONING.md): provisionamento e contratos do canal Instagram.
- [README-WHATSAPP-PROVISIONING.md](./README-WHATSAPP-PROVISIONING.md): provisionamento e contratos do canal WhatsApp.
- [tutorial-101-generative-ui.md](./tutorial-101-generative-ui.md): leitura guiada de UI generativa.
- [tutorial-101-human-in-the-loop.md](./tutorial-101-human-in-the-loop.md): tutorial introdutório de pausa humana.
- [tutorial-101-provisionamento-instagram.md](./tutorial-101-provisionamento-instagram.md): visão 101 do provisionamento Instagram.
- [tutorial-101-provisionamento-whatsapp.md](./tutorial-101-provisionamento-whatsapp.md): visão 101 do provisionamento WhatsApp.

### 9.7 Segurança e controle de acesso

- [README-AUTENTICACAO-MFA.md](./README-AUTENTICACAO-MFA.md): autenticação com múltiplos fatores.
- [README-SISTEMA-AUTENTICACAO.md](./README-SISTEMA-AUTENTICACAO.md): visão do sistema de autenticação como capacidade de plataforma.
- [README-AUTORIZACAO.md](./README-AUTORIZACAO.md): regras e impacto de autorização.

### 9.8 Metodologia de desenvolvimento

- [metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md](./metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md): ponto de entrada da metodologia.
- [metodologia-desenv/README-METODOLOGIA-DESENV-GOVERNANCA-COPILOT.md](./metodologia-desenv/README-METODOLOGIA-DESENV-GOVERNANCA-COPILOT.md): governança de uso do Copilot.
- [metodologia-desenv/README-METODOLOGIA-DESENV-ARTEFATOS-GITHUB.md](./metodologia-desenv/README-METODOLOGIA-DESENV-ARTEFATOS-GITHUB.md): artefatos e disciplina de trabalho no GitHub.
- [metodologia-desenv/README-METODOLOGIA-DESENV-MELHORIA-CONTINUA.md](./metodologia-desenv/README-METODOLOGIA-DESENV-MELHORIA-CONTINUA.md): melhoria contínua aplicada ao fluxo do time.
- [metodologia-desenv/README-METODOLOGIA-DESENV-SUITE-TESTES.md](./metodologia-desenv/README-METODOLOGIA-DESENV-SUITE-TESTES.md): estratégia de suíte oficial dentro da metodologia.
- [metodologia-desenv/README-METODOLOGIA-DESENV-ONBOARDING.md](./metodologia-desenv/README-METODOLOGIA-DESENV-ONBOARDING.md): onboarding do processo de desenvolvimento.
- [metodologia-desenv/README-METODOLOGIA-DESENV-FLUXOS-TRABALHO.md](./metodologia-desenv/README-METODOLOGIA-DESENV-FLUXOS-TRABALHO.md): fluxo operacional do trabalho de engenharia.

## 10. O que este índice evita por design

Este índice foi escrito para evitar quatro antipadrões.

- Catálogo de arquivos sem explicação.
- Lista de links sem critério de leitura.
- Mistura entre documento conceitual e inventário operacional.
- Falsa sensação de entendimento baseada só em nomes de módulos.

## 11. Explicação 101

Se você resumir este índice para alguém novo, a mensagem é esta: não escolha leitura pelo nome do arquivo. Escolha pela pergunta que você está tentando responder sobre o sistema.

## 12. Evidências no código

- [src/api/service_api.py](../src/api/service_api.py): boundary HTTP da plataforma.
- [src/services/ingestion_service.py](../src/services/ingestion_service.py): serviço de ingestão usado pelo caminho oficial.
- [src/qa_layer/content_qa_system.py](../src/qa_layer/content_qa_system.py): fachada principal do runtime de QA.
- [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py): orquestração do assembly agentic.
