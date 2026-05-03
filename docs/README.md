# Como Ler Esta Documentação

Esta documentação foi reorganizada para responder perguntas de arquitetura e operação, não para ensinar alguém a caçar arquivo. O objetivo é simples: permitir que quem trabalha com o projeto entenda a lógica do sistema, o papel de cada domínio, os contratos importantes e os pontos de diagnóstico sem depender de um mapa manual do repositório.

## 1. Qual problema este índice resolve

O problema anterior era de navegação conceitual. Havia muitos documentos, mas a entrada principal ainda induzia um comportamento ruim: escolher o próximo texto pelo nome do arquivo, e não pela dúvida real.

Este índice corrige isso organizando a leitura por intenção.

Se a necessidade for uma visão executiva da plataforma e um catálogo
mestre consolidado, use também o [README principal](../README.md). Este
índice interno da pasta docs foi mantido com outro objetivo: ajudar a
escolher a próxima leitura pela pergunta real, não só pelo nome do
arquivo.

- Entender a plataforma como sistema.
- Entender como o acervo é produzido.
- Entender como a plataforma consulta esse acervo.
- Entender como fluxos agentic são governados.
- Entender como operar, investigar e validar.

## 2. Se você quer entender a plataforma inteira

Comece pelo [README principal](../README.md) para visão executiva e por
[README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md](./README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md)
e
[README-TECNICO-ARQUITETURA-STACK-PROJETO.md](./README-TECNICO-ARQUITETURA-STACK-PROJETO.md)
para a mecânica real da plataforma. Esse par explica a separação entre
API, worker e scheduler, a stack confirmada, o papel do YAML, a
infraestrutura mandatória e a diferença entre produção de acervo,
consulta e execução assíncrona.

Depois leia [README-INGESTAO.md](./README-INGESTAO.md), [README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md), [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md), [README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md), [README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md), [README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md), [README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md), [README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md), [README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md), [README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md), [README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md), [README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md), [README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md), [README-RAG.md](./README-RAG.md), [README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md](./README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md) e [README-TECNICO-RAG-PIPELINE-COMPLETO.md](./README-TECNICO-RAG-PIPELINE-COMPLETO.md). Essa sequência é importante porque PDF, Excel, JSON, Confluence, Web Scraping e HTML são partes críticas da produção de acervo e porque os manuais especializados explicam os pipelines completos sob perspectivas complementares antes de entrar no consumo via RAG.

## 3. Se você quer entender produção de acervo

Use esta trilha.

1. [README-INGESTAO.md](./README-INGESTAO.md) para o fluxo documental oficial.
2. [README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md) para entender por que o PDF tem uma esteira própria, quais riscos ela reduz e como se compara ao estado da arte.
3. [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md) para seguir bootstrap, OCR, engines, multimodalidade, chunking, manifesto e troubleshooting do PDF.
4. [README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md) para entender por que planilhas exigem preservação tabular, quais decisões o pipeline toma e como o desenho se compara ao estado da arte workbook-first e dataframe-first.
5. [README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md) para seguir carga do workbook, análise estrutural, schema, stats, chunking row-aware e troubleshooting do Excel.
6. [README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md) para entender por que JSON semi-estruturado exige validação estrita, profiling estrutural, heurística de domínio e como o desenho atual se compara a parsing streaming, flattening tabular e parsers de alta performance.
7. [README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md) para seguir contrato YAML, decode, parse, schema summary, numeric stats, perfis, plugins de domínio e troubleshooting do JSON.
8. [README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md) para entender por que a ingestão de wiki corporativa exige escopo explícito, visibilidade, attachments, quality gating e comparação com o estado da arte de conectores Confluence.
9. [README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md) para seguir contrato YAML, transporte SDK ou HTTP, recursão, autorização, materialização HTML, attachments, multimodalidade e troubleshooting do Confluence.
10. [README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md) para entender por que ingestão web não é só parsing HTML, como a plataforma separa aquisição remota de chunking e como o desenho atual se compara ao estado da arte de browser automation, crawlers assíncronos e extração de conteúdo principal.
11. [README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md) para seguir contrato YAML, heurística de estratégia, proxy, autenticação, anexos, prefetch obrigatório, materialização HTML e troubleshooting do Web Scraping.
12. [README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md) para entender o slice HTML como tipo de conteúdo, a separação entre HTML base e Web Scraping e os limites atuais frente ao estado da arte de readability e boilerplate removal.
13. [README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md) para seguir inferência de `ContentType.HTML`, materialização canônica do caminho remoto, limpeza com BeautifulSoup, multimodalidade web e as lacunas reais do contrato local.
14. [README-ETL.md](./README-ETL.md) quando a dúvida for transformação estruturada e não ingestão documental.
15. [README-CONCEITUAL-ETL-COMPLETO.md](./README-CONCEITUAL-ETL-COMPLETO.md) para entender ETL como capacidade de plataforma, integrações, valor e limites.
16. [README-TECNICO-ETL-COMPLETO.md](./README-TECNICO-ETL-COMPLETO.md) para seguir request, worker, fan-out, Apify, schema metadata, transformação e troubleshooting do ETL.
17. [tutorial-101-processo-completo-de-ingestao-e-rag.md](./tutorial-101-processo-completo-de-ingestao-e-rag.md) quando a meta for seguir a história inteira, do pedido até a resposta.

O ponto conceitual aqui é este: ingestão produz acervo consultável; ETL transforma dados; nenhum dos dois deve ser entendido como sinônimo de consulta.

## 4. Se você quer entender consulta e resposta

Use esta trilha.

1. [README-RAG.md](./README-RAG.md) para a visão unificada do tema RAG no catálogo principal.
2. [README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md](./README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md) para entender conceito, valor, decisões, limites e comparação com estado da arte.
3. [README-TECNICO-RAG-PIPELINE-COMPLETO.md](./README-TECNICO-RAG-PIPELINE-COMPLETO.md) para seguir ingestão, retrieval, fusão, cache, ACL, geração e troubleshooting.
4. [README-CONCEITUAL-CACHING-SISTEMA-COMPLETO.md](./README-CONCEITUAL-CACHING-SISTEMA-COMPLETO.md) para entender por que a plataforma usa múltiplas camadas de cache, qual valor isso entrega e quais limites operacionais existem.
5. [README-TECNICO-CACHING-SISTEMA-COMPLETO.md](./README-TECNICO-CACHING-SISTEMA-COMPLETO.md) para seguir hash canônico, WarmResourcePool, PipelineCacheManager, sessão efêmera, invalidação global e operação administrativa.
6. [tutorial-101-rag.md](./tutorial-101-rag.md) se você precisa de uma leitura didática e guiada antes do manual técnico completo.

## 5. Se você quer entender YAML, AST e governança agentic

Esse assunto precisa ser lido pela lógica de decisão, não por jargão.

- [README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](./README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md) explica o que a plataforma realmente configura por YAML, o que pode ser feito sem programação e onde termina esse limite.
- [README-TECNICO-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](./README-TECNICO-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md) mostra o ciclo técnico de carga, AST, seleção de alvos, tools_library e ETL declarativo.
- [README-CONCEITUAL-MCP-INTEGRACAO-USO-SISTEMA.md](./README-CONCEITUAL-MCP-INTEGRACAO-USO-SISTEMA.md) explica o MCP como capability governada por YAML, o valor de negócio da integração por protocolo e os limites reais do desenho atual.
- [README-TECNICO-MCP-INTEGRACAO-USO-SISTEMA.md](./README-TECNICO-MCP-INTEGRACAO-USO-SISTEMA.md) descreve merge por escopo, catálogo efetivo, proxy stdio em /mcp, autenticação, cache e troubleshooting operacional do MCP.
- [README-AGENTIC-INICIANTES.md](./README-AGENTIC-INICIANTES.md) explica o modelo mental.
- [README-AST-AGENTIC-DESIGNER.md](./README-AST-AGENTIC-DESIGNER.md) explica por que a AST existe e como ela governa edição, validação e compilação.
- [README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md](./README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md) explica o DeepAgent Supervisor como capacidade governada, seu valor, seus limites e sua relação com HIL, memória e subagentes.
- [README-TECNICO-DEEPAGENT-SUPERVISOR-COMPLETO.md](./README-TECNICO-DEEPAGENT-SUPERVISOR-COMPLETO.md) descreve AST, validator, runtime, middlewares, memória Redis, execução, continuação HIL e operação do DeepAgent Supervisor.
- [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md) explica grafo determinístico e execução por nós.
- [README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md](./README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md) aprofunda conceito, valor, estratégia e limites do agente workflow.
- [README-TECNICO-AGENTE-WORKFLOW-COMPLETO.md](./README-TECNICO-AGENTE-WORKFLOW-COMPLETO.md) detalha sintaxe, AST, runtime, nodes, HIL, API e troubleshooting do agente workflow.
- [README-AGENTIC-BACKGROUND-EXECUTION.md](./README-AGENTIC-BACKGROUND-EXECUTION.md) explica como solicitações agentic executam em background com schedules, runs, worker oficial e APIs administrativas.
- [README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md) explica o valor de negócio e o modelo mental do agendamento agentic com prompt em linguagem natural, scheduler universal, HIL durável e Generative UI compartilhada.
- [README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md) descreve o caminho técnico entre tool de schedule, scheduler universal, worker background, decisão HIL por API ou canal e uso da Generative UI agnóstica.

## 6. Se você quer entender a API e a operação

A ordem mais produtiva costuma ser esta.

1. [README-SERVICE-API.md](./README-SERVICE-API.md) para entender o boundary HTTP real.
2. [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md) para entender como a configuração vira runtime executável.
3. [README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](./README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md) quando a dúvida for até onde a plataforma realmente permite configurar agentes, workflows e ETL sem programação.
4. [README-TECNICO-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](./README-TECNICO-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md) quando a dúvida for o caminho técnico real entre template, assembly AST, tools_library e extract_transform_load.
5. [API-ENDPOINTS-SWAGGER.md](./API-ENDPOINTS-SWAGGER.md) para ver o contrato público atual.
6. [GUIA-DIDATICO-EXECUCAO-CANAIS.md](./GUIA-DIDATICO-EXECUCAO-CANAIS.md) se a dúvida envolver topologia operacional, filas e runners.
7. [README-LOGGING.md](./README-LOGGING.md) quando o problema for rastreabilidade e correlação.
8. [README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md](./README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md) para entender a arquitetura de logging como capacidade de plataforma, com correlation_id, arquivo local e providers remotos.
9. [README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md](./README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md) para seguir middleware, handlers, CloudWatch, providers administrativos e troubleshooting.
10. [README-TESTS.MD](./README-TESTS.MD) quando a pergunta for validação oficial.
11. [README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md) quando a dúvida for o valor e os limites do agendamento agentic com prompt em NL, aprovação humana e interface generativa compartilhada.
12. [README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md) quando a dúvida for o fluxo ponta a ponta entre scheduler, worker, run, HIL e superfície AG-UI reutilizável.

## 7. Se você quer entender UI agentic e pausa humana

Esses dois assuntos se cruzam, mas não são a mesma coisa.

- [README-AG-UI.md](./README-AG-UI.md) cobre o protocolo de interface orientado a eventos e o sidecar compartilhado.
- [README-CONCEITUAL-AG-UI-GOOGLE-MICROSOFT-ERP.md](./README-CONCEITUAL-AG-UI-GOOGLE-MICROSOFT-ERP.md) explica o AG-UI com referencial oficial do ecossistema, a implementação local e o valor em telas ERP.
- [README-TECNICO-AG-UI-GOOGLE-MICROSOFT-ERP.md](./README-TECNICO-AG-UI-GOOGLE-MICROSOFT-ERP.md) detalha endpoint, lifecycle, adapter, SSE, telas ERP e troubleshooting do slice AG-UI.
- [README-HUMAN-IN-THE-LOOP.md](./README-HUMAN-IN-THE-LOOP.md) cobre a pausa humana, o envelope de aprovação e a retomada governada.
- [README-CONCEITUAL-HIL-APIS-WHATSAPP.md](./README-CONCEITUAL-HIL-APIS-WHATSAPP.md) explica como a pausa humana aparece no boundary de APIs e no uso com WhatsApp sob perspectiva de valor e governança.
- [README-TECNICO-HIL-APIS-WHATSAPP.md](./README-TECNICO-HIL-APIS-WHATSAPP.md) detalha o fluxo técnico de HIL em APIs e na continuidade operacional com WhatsApp.
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

- [../README.md](../README.md): visão executiva da plataforma e catálogo mestre consolidado.
- [README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md](./README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md): manual conceitual, executivo, comercial e estratégico da arquitetura macro e da stack do projeto.
- [README-TECNICO-ARQUITETURA-STACK-PROJETO.md](./README-TECNICO-ARQUITETURA-STACK-PROJETO.md): manual técnico e operacional da topologia multiprocesso, infraestrutura obrigatória e stack confirmada do projeto.
- [README-SERVICE-API.md](./README-SERVICE-API.md): boundary HTTP, responsabilidades da API e como o runtime expõe capacidades para clientes e operação.
- [README-LOGGING.md](./README-LOGGING.md): rastreabilidade, correlação e leitura operacional de logs.
- [README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md](./README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md): manual conceitual, executivo, comercial e estratégico da arquitetura de logging com correlation_id, arquivo local e providers remotos.
- [README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md](./README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md): manual técnico e operacional da arquitetura de logging, incluindo arquivo local, CloudWatch e leitura administrativa por provider.
- [README-TESTS.MD](./README-TESTS.MD): estratégia oficial de validação automatizada.
- [README-SCHEDULER.md](./README-SCHEDULER.md): processamento agendado, responsabilidades do scheduler e relação com o restante do runtime.
- [README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md): visão conceitual, executiva, comercial e estratégica do agendamento agentic em background com HIL e Generative UI compartilhada.
- [README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md): manual técnico e operacional do fluxo entre scheduler universal, execução background, comunicação HIL e superfície AG-UI agnóstica.

### 9.2 Produção de acervo, ingestão e ETL

- [README-INGESTAO.md](./README-INGESTAO.md): pipeline oficial de ingestão documental.
- [README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do pipeline PDF ponta a ponta.
- [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md): manual técnico e operacional do pipeline PDF ponta a ponta.
- [README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do pipeline Excel ponta a ponta.
- [README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md): manual técnico e operacional do pipeline Excel ponta a ponta.
- [README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do pipeline JSON ponta a ponta.
- [README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md): manual técnico e operacional do pipeline JSON ponta a ponta.
- [README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do pipeline Confluence ponta a ponta.
- [README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md): manual técnico e operacional do pipeline Confluence ponta a ponta.
- [README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do pipeline Web Scraping ponta a ponta.
- [README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md): manual técnico e operacional do pipeline Web Scraping ponta a ponta.
- [README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md](./README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do slice HTML da ingestão, separado do pipeline de Web Scraping.
- [README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md](./README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md): manual técnico e operacional do slice HTML da ingestão, incluindo materialização, limpeza, multimodalidade web e lacunas de contrato.
- [README-ETL.md](./README-ETL.md): visão geral do tema ETL e porta de entrada para a trilha especializada.
- [README-CONCEITUAL-ETL-COMPLETO.md](./README-CONCEITUAL-ETL-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do ETL completo.
- [README-TECNICO-ETL-COMPLETO.md](./README-TECNICO-ETL-COMPLETO.md): manual técnico e operacional do ETL completo, incluindo Apify, schema metadata e fan-out.
- [tutorial-101-ingestao.md](./tutorial-101-ingestao.md): introdução guiada à ingestão.
- [tutorial-101-ingestao-pdf.md](./tutorial-101-ingestao-pdf.md): leitura 101 do pipeline de ingestão de PDF, complementar aos dois manuais completos.
- [tutorial-101-ingestao-excel-e-rag-de-excel.md](./tutorial-101-ingestao-excel-e-rag-de-excel.md): ingestão e consulta sobre planilhas.
- [tutorial-101-etl.md](./tutorial-101-etl.md): tutorial didático sobre ETL no contexto da plataforma.
- [tutorial-101-processo-completo-de-ingestao-e-rag.md](./tutorial-101-processo-completo-de-ingestao-e-rag.md): narrativa ponta a ponta do pedido até a resposta.
- [tutorial-101-auto-config-e-bm25-no-pipeline-de-ingestao-e-rag.md](./tutorial-101-auto-config-e-bm25-no-pipeline-de-ingestao-e-rag.md): ajuste de comportamento de retrieval dentro do pipeline.
- [tutorial-101-bm25-vs-auto-config-dnit.md](./tutorial-101-bm25-vs-auto-config-dnit.md): comparação guiada de estratégias de recuperação em um caso real.

### 9.3 Consulta, RAG e caching

- [README-RAG.md](./README-RAG.md): visão unificada do tema RAG no catálogo principal.
- [README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md](./README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do pipeline RAG ponta a ponta.
- [README-TECNICO-RAG-PIPELINE-COMPLETO.md](./README-TECNICO-RAG-PIPELINE-COMPLETO.md): manual técnico e operacional do pipeline RAG ponta a ponta.
- [README-CONCEITUAL-CACHING-SISTEMA-COMPLETO.md](./README-CONCEITUAL-CACHING-SISTEMA-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do sistema de caching completo.
- [README-TECNICO-CACHING-SISTEMA-COMPLETO.md](./README-TECNICO-CACHING-SISTEMA-COMPLETO.md): manual técnico e operacional do sistema de caching completo, incluindo invalidação, snapshots e contratos administrativos.
- [tutorial-101-bm25-vs-auto-config-dnit.md](./tutorial-101-bm25-vs-auto-config-dnit.md): leitura aplicada do caso DNIT com foco em vocabulário lexical e auto_config.
- [tutorial-101-rag.md](./tutorial-101-rag.md): introdução 101 ao funcionamento de RAG.
- [tutorial-101-status-feature-geracao-sql-linguagem-natural.md](./tutorial-101-status-feature-geracao-sql-linguagem-natural.md): estado atual e limites da geração SQL por linguagem natural.

### 9.4 YAML, AST, assembly e fluxos agentic

- [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md): ciclo de vida da configuração até o runtime.
- [README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](./README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md): manual conceitual, executivo, comercial e estratégico sobre o que a plataforma realmente configura por YAML sem programação.
- [README-TECNICO-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](./README-TECNICO-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md): manual técnico e operacional sobre carga do YAML, assembly AST, seleção de alvos agentic e ETL declarativo.
- [README-CONCEITUAL-NL2YAML-COMPLETO.md](./README-CONCEITUAL-NL2YAML-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do fluxo NL2YAML governado por AST.
- [README-TECNICO-NL2YAML-COMPLETO.md](./README-TECNICO-NL2YAML-COMPLETO.md): manual técnico e operacional do fluxo objective-to-yaml, incluindo AST, validação, dry-run, drift e publicação segura.
- [README-AGENTIC-INICIANTES.md](./README-AGENTIC-INICIANTES.md): modelo mental inicial do ecossistema agentic.
- [README-AST-AGENTIC-DESIGNER.md](./README-AST-AGENTIC-DESIGNER.md): AST como fonte tipada de verdade do assembly agentic.
- [README-AGENTIC-CONTRATO-COMUM.md](./README-AGENTIC-CONTRATO-COMUM.md): contrato compartilhado entre espinhas dorsais agentic.
- [README-AGENTIC-BACKGROUND-EXECUTION.md](./README-AGENTIC-BACKGROUND-EXECUTION.md): execução agentic em background baseada em solicitação de prompt, com schedules, runs, HIL e APIs administrativas.
- [README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md): leitura especializada para entender o pedido em linguagem natural, a agenda canônica, o valor do HIL e o papel da interface generativa compartilhada.
- [README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md): leitura especializada para seguir o fluxo técnico entre tool, scheduler, worker, finalização HIL e componentes reutilizáveis de AG-UI.
- [README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md](./README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do DeepAgent Supervisor completo.
- [README-TECNICO-DEEPAGENT-SUPERVISOR-COMPLETO.md](./README-TECNICO-DEEPAGENT-SUPERVISOR-COMPLETO.md): manual técnico e operacional do DeepAgent Supervisor completo, incluindo AST, runtime, HIL e memória persistente.
- [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md): workflow determinístico baseado em grafo.
- [README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md](./README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md): manual conceitual, executivo, comercial e estratégico do agente workflow completo.
- [README-TECNICO-AGENTE-WORKFLOW-COMPLETO.md](./README-TECNICO-AGENTE-WORKFLOW-COMPLETO.md): manual técnico, operacional e de sintaxe do agente workflow completo.
- [README-CONCEITUAL-TELEMETRIA-INTERACOES-AGENTE.md](./README-CONCEITUAL-TELEMETRIA-INTERACOES-AGENTE.md): visão conceitual, executiva e estratégica da tabela interaction_runs como histórico monitorável das respostas do agente.
- [README-TECNICO-TELEMETRIA-INTERACOES-AGENTE.md](./README-TECNICO-TELEMETRIA-INTERACOES-AGENTE.md): fluxo técnico de gravação, consulta e anotação humana das interações persistidas do agente.
- [tutorial-101-agentes.md](./tutorial-101-agentes.md): explicação introdutória do conceito de agentes.
- [tutorial-101-deepagents.md](./tutorial-101-deepagents.md): leitura 101 sobre deepagents.
- [tutorial-101-workflow.md](./tutorial-101-workflow.md): tutorial guiado sobre workflows.
- [tutorial-101-criacao-de-agentes-via-yaml.md](./tutorial-101-criacao-de-agentes-via-yaml.md): criação orientada por YAML.
- [tutorial-101-nl2yaml.md](./tutorial-101-nl2yaml.md): tradução de linguagem natural para YAML.
- [tutorial-101-tecnica-nl-para-yaml-e-dsl.md](./tutorial-101-tecnica-nl-para-yaml-e-dsl.md): fundamentos da técnica de NL para YAML e DSL.
- [tutorial-101-configuracao-yaml-execucao-local-e-nl-para-sql.md](./tutorial-101-configuracao-yaml-execucao-local-e-nl-para-sql.md): combinação entre configuração local e NL para SQL.

### 9.5 Tools, schemas, integrações e superfícies especializadas

- [README-DYNAMIC-API-TOOLS.md](./README-DYNAMIC-API-TOOLS.md): tools dinâmicas orientadas a APIs.
- [README-CONCEITUAL-DYNAMIC-API-TOOLS.md](./README-CONCEITUAL-DYNAMIC-API-TOOLS.md): manual conceitual, executivo, comercial e estratégico do dyn_api como capability governada.
- [README-TECNICO-DYNAMIC-API-TOOLS.md](./README-TECNICO-DYNAMIC-API-TOOLS.md): manual técnico e operacional do dyn_api, incluindo runtime, auth profiles, registro persistido, retry e boundaries administrativos.
- [README-DYNAMIC-SQL-TOOLS.md](./README-DYNAMIC-SQL-TOOLS.md): tools dinâmicas orientadas a SQL.
- [README-CONCEITUAL-DYNAMIC-SQL-TOOLS.md](./README-CONCEITUAL-DYNAMIC-SQL-TOOLS.md): manual conceitual, executivo, comercial e estratégico do dyn_sql como capability governada.
- [README-TECNICO-DYNAMIC-SQL-TOOLS.md](./README-TECNICO-DYNAMIC-SQL-TOOLS.md): manual técnico e operacional do dyn_sql, incluindo catálogo builtin, AST, registro persistido, retry e cache.
- [README-SCHEMA-BANCO.md](./README-SCHEMA-BANCO.md): organização e papel do schema do banco.
- [README-SQL-SCHEMA-RAG-TOOL.md](./README-SQL-SCHEMA-RAG-TOOL.md): tool especializada no schema SQL do RAG.
- [README-CONCEITUAL-SCHEMA-METADATA-PRE-REQUISITO-NL2SQL.md](./README-CONCEITUAL-SCHEMA-METADATA-PRE-REQUISITO-NL2SQL.md): manual conceitual, executivo, comercial e estratégico do pipeline de schema metadata como base operacional do NL2SQL.
- [README-TECNICO-SCHEMA-METADATA-PRE-REQUISITO-NL2SQL.md](./README-TECNICO-SCHEMA-METADATA-PRE-REQUISITO-NL2SQL.md): manual técnico e operacional do encadeamento ETL -> exportação -> ingestão -> runtime para schema metadata.
- [README-GOOGLE-UCP.md](./README-GOOGLE-UCP.md): integração e contrato do fluxo Google UCP.
- [README-CONCEITUAL-MCP-INTEGRACAO-USO-SISTEMA.md](./README-CONCEITUAL-MCP-INTEGRACAO-USO-SISTEMA.md): manual conceitual, executivo, comercial e estratégico do MCP como capability governada e expansível por YAML.
- [README-TECNICO-MCP-INTEGRACAO-USO-SISTEMA.md](./README-TECNICO-MCP-INTEGRACAO-USO-SISTEMA.md): manual técnico e operacional do MCP, incluindo resolução por escopo, proxy stdio em /mcp, autenticação, cache e troubleshooting.
- [README-INTEGRACOES-GOVERNADAS.md](./README-INTEGRACOES-GOVERNADAS.md): integração com controles e governança explícita.
- [README-EXEMPLOS-INTEGRACAO-API.md](./README-EXEMPLOS-INTEGRACAO-API.md): documento explicitamente voltado a exemplos completos de uso de APIs.
- [tutorial-101-exemplos-api-deepagent-hil-execute-continue.md](./tutorial-101-exemplos-api-deepagent-hil-execute-continue.md): tutorial de exemplos completos de API para deepagent com pausa humana.

### 9.6 Interface, pausa humana e canais externos

- [README-AG-UI.md](./README-AG-UI.md): protocolo e arquitetura de interface agentic.
- [README-CONCEITUAL-AG-UI-GOOGLE-MICROSOFT-ERP.md](./README-CONCEITUAL-AG-UI-GOOGLE-MICROSOFT-ERP.md): manual conceitual, executivo, comercial e estratégico do AG-UI com foco em Google, Microsoft e uso em ERP.
- [README-TECNICO-AG-UI-GOOGLE-MICROSOFT-ERP.md](./README-TECNICO-AG-UI-GOOGLE-MICROSOFT-ERP.md): manual técnico, operacional e de uso do AG-UI com foco em Google, Microsoft e telas ERP.
- [README-HUMAN-IN-THE-LOOP.md](./README-HUMAN-IN-THE-LOOP.md): pausa humana, aprovação e retomada governada.
- [README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md): mostra o papel do HIL quando a execução já saiu do chat e virou run agendado.
- [README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md): mostra como a decisão HIL chega por POST seguro ou por canal e sincroniza o run background.
- [README-CONCEITUAL-WHATSAPP-AGENTE-ONBOARDING.md](./README-CONCEITUAL-WHATSAPP-AGENTE-ONBOARDING.md): capacidade conceitual e impacto do WhatsApp como canal conversacional de agentes.
- [README-TECNICO-WHATSAPP-AGENTE-ONBOARDING.md](./README-TECNICO-WHATSAPP-AGENTE-ONBOARDING.md): onboarding, webhook, cadastro do canal e conversa via WhatsApp com exemplos completos.
- [README-CONCEITUAL-INSTAGRAM-AGENTE-COMENTARIOS-DM.md](./README-CONCEITUAL-INSTAGRAM-AGENTE-COMENTARIOS-DM.md): capacidade conceitual do Instagram como canal de agente para inbox, comentario e mencao.
- [README-TECNICO-INSTAGRAM-AGENTE-COMENTARIOS-DM.md](./README-TECNICO-INSTAGRAM-AGENTE-COMENTARIOS-DM.md): onboarding, callback, DM, comentario publico e configuracao tecnica do canal Instagram.
- [README-CONCEITUAL-HIL-APIS-WHATSAPP.md](./README-CONCEITUAL-HIL-APIS-WHATSAPP.md): capacidade conceitual da pausa humana aplicada a APIs e ao fluxo com WhatsApp.
- [README-TECNICO-HIL-APIS-WHATSAPP.md](./README-TECNICO-HIL-APIS-WHATSAPP.md): comportamento técnico e operacional da pausa humana nas APIs e no fluxo com WhatsApp.
- [README-INSTAGRAM-PROVISIONING.md](./README-INSTAGRAM-PROVISIONING.md): provisionamento e contratos do canal Instagram.
- [README-WHATSAPP-PROVISIONING.md](./README-WHATSAPP-PROVISIONING.md): provisionamento e contratos do canal WhatsApp.
- [ROTEIRO-IMPLANTACAO-VENDAS-MULTICANAL.md](./ROTEIRO-IMPLANTACAO-VENDAS-MULTICANAL.md): roteiro de implantação orientado a operação comercial multicanal.
- [PROPOSTA-PROJETO-VENDAS-WHATSAPP-INSTAGRAM.md](./PROPOSTA-PROJETO-VENDAS-WHATSAPP-INSTAGRAM.md): proposta comercial e funcional de uso combinado de WhatsApp e Instagram.
- [PLANO-IMPLEMENTACAO-INSTAGRAM-COMMENTS-DM.md](./PLANO-IMPLEMENTACAO-INSTAGRAM-COMMENTS-DM.md): plano detalhado de implantação do slice de comentários e DM no Instagram.
- [tutorial-101-generative-ui.md](./tutorial-101-generative-ui.md): leitura guiada de UI generativa.
- [tutorial-101-human-in-the-loop.md](./tutorial-101-human-in-the-loop.md): tutorial introdutório de pausa humana.
- [tutorial-101-provisionamento-instagram.md](./tutorial-101-provisionamento-instagram.md): visão 101 do provisionamento Instagram.
- [tutorial-101-provisionamento-whatsapp.md](./tutorial-101-provisionamento-whatsapp.md): visão 101 do provisionamento WhatsApp.

### 9.7 Segurança e controle de acesso

- [README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md](./README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md): visão conceitual, executiva, comercial e estratégica da autenticação humana do projeto, incluindo Google, sessão web, login local e MFA TOTP.
- [README-TECNICO-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md](./README-TECNICO-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md): manual técnico e operacional dos endpoints, contratos, middleware, sessão federada, integração Google e segundo fator TOTP.
- [README-AUTENTICACAO-MFA.md](./README-AUTENTICACAO-MFA.md): autenticação com múltiplos fatores.
- [README-SISTEMA-AUTENTICACAO.md](./README-SISTEMA-AUTENTICACAO.md): visão do sistema de autenticação como capacidade de plataforma.
- [README-CONCEITUAL-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md](./README-CONCEITUAL-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md): visão conceitual, executiva, comercial e estratégica da autorização, das permissões e do controle de acesso nas APIs e no runtime.
- [README-TECNICO-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md](./README-TECNICO-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md): manual técnico e operacional do catálogo de permissões, enforcement HTTP, grants humanos, ACL de documentos e governança administrativa.

### 9.8 Metodologia de desenvolvimento

- [metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md](./metodologia-desenv/README-METODOLOGIA-DESENV-INDICE.md): ponto de entrada da metodologia.
- [metodologia-desenv/README-METODOLOGIA-DESENV-GOVERNANCA-COPILOT.md](./metodologia-desenv/README-METODOLOGIA-DESENV-GOVERNANCA-COPILOT.md): governança de uso do Copilot.
- [metodologia-desenv/README-METODOLOGIA-DESENV-ARTEFATOS-GITHUB.md](./metodologia-desenv/README-METODOLOGIA-DESENV-ARTEFATOS-GITHUB.md): artefatos e disciplina de trabalho no GitHub.
- [metodologia-desenv/README-METODOLOGIA-DESENV-MELHORIA-CONTINUA.md](./metodologia-desenv/README-METODOLOGIA-DESENV-MELHORIA-CONTINUA.md): melhoria contínua aplicada ao fluxo do time.
- [metodologia-desenv/README-METODOLOGIA-DESENV-SUITE-TESTES.md](./metodologia-desenv/README-METODOLOGIA-DESENV-SUITE-TESTES.md): estratégia de suíte oficial dentro da metodologia.
- [metodologia-desenv/README-METODOLOGIA-DESENV-ONBOARDING.md](./metodologia-desenv/README-METODOLOGIA-DESENV-ONBOARDING.md): onboarding do processo de desenvolvimento.
- [metodologia-desenv/README-METODOLOGIA-DESENV-FLUXOS-TRABALHO.md](./metodologia-desenv/README-METODOLOGIA-DESENV-FLUXOS-TRABALHO.md): fluxo operacional do trabalho de engenharia.
- [metodologia-desenv/copilot-instructions-framework.md](./metodologia-desenv/copilot-instructions-framework.md): framework de instruções e disciplina de uso do Copilot no contexto metodológico do projeto.

### 9.9 Catálogos de tools e guias funcionais

- [GUIA-COMERCIAL-PLATAFORMA.md](./GUIA-COMERCIAL-PLATAFORMA.md): narrativa comercial e posicionamento de plataforma.
- [GUIA-DIDATICO-EXECUCAO-CANAIS.md](./GUIA-DIDATICO-EXECUCAO-CANAIS.md): visão operacional de canais, runners e execução.
- [GUIA-USUARIO-TOOLS.md](./GUIA-USUARIO-TOOLS.md): explicação funcional do uso de tools para operação e consultoria.
- [tools/alfabetica.md](./tools/alfabetica.md): catálogo de tools em ordem alfabética.
- [tools/api_dinamica.md](./tools/api_dinamica.md): catálogo especializado nas tools dinâmicas orientadas a API.
- [tools/esportes.md](./tools/esportes.md): catálogo temático de tools ligadas ao domínio de esportes.
- [tools/por_finalidade.md](./tools/por_finalidade.md): catálogo de tools por intenção de uso.
- [tools/sql_dinamico.md](./tools/sql_dinamico.md): catálogo especializado nas tools dinâmicas orientadas a SQL.
- [tools/varejo.md](./tools/varejo.md): conjunto de tools orientadas a varejo.
- [tools/social.md](./tools/social.md): conjunto de tools voltadas a social e canais digitais.
- [tools/whatsapp_business.md](./tools/whatsapp_business.md): catálogo temático de tools ligadas a WhatsApp Business.

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
