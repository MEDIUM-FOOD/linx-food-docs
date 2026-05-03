# Manual técnico, executivo, comercial e estratégico: SQL Schema RAG Tool

## 1. O que é esta feature

SQL Schema RAG Tool é a capacidade de transformar uma pergunta em linguagem natural em uma proposta de SQL usando recuperação semântica de documentos de schema já indexados em vector store.

O nome canônico dessa capacidade no catálogo interno é `schema_rag_sql`. No runtime, ela aparece em duas superfícies diferentes que precisam ser entendidas separadamente.

A primeira superfície é a tool em si, criada pela factory `create_sql_schema_rag_tool`. Essa tool recebe apenas a pergunta do usuário, busca documentos `schema_metadata`, monta contexto e pede ao LLM uma SQL.

A segunda superfície é o endpoint administrativo dedicado de NL2SQL. Ele usa a mesma engine de recuperação e geração, mas adiciona resolução de YAML, contrato HTTP estável, diagnósticos estruturados e o guardrail central de somente leitura.

Em linguagem simples, a feature não tenta adivinhar uma query do nada. Ela primeiro procura um mapa semântico do banco, reduz o universo para o pedaço mais promissor e só então pede a SQL ao modelo.

## 2. Que problema ela resolve

O problema real aqui não é escrever SQL. O problema real é transformar intenção de negócio em consulta revisável quando o schema é grande, técnico, legado e pouco legível.

Sem essa feature, o operador costuma cair em um dos dois extremos ruins.

O primeiro extremo é abrir o banco manualmente, navegar tabela por tabela e depender de alguém que já conhece siglas, nomes legados e joins históricos.

O segundo extremo é mandar a pergunta direto para um modelo sem contexto técnico controlado. Isso costuma gerar SQL com tabelas erradas, colunas inventadas, joins frágeis ou comandos perigosos.

SQL Schema RAG Tool reduz esse risco porque trabalha com cinco freios estruturais.

- busca apenas documentos semanticamente parecidos com a pergunta;
- filtra somente documentos marcados como `schema_metadata`;
- limita o volume de contexto enviado ao LLM;
- fixa o dialeto SQL aceito pelo runtime;
- no caso do endpoint dedicado, valida a proposta com guardrail somente leitura antes de devolvê-la como SQL utilizável.

## 3. Visão conceitual

Conceitualmente, a feature é uma cadeia de quatro problemas técnicos resolvidos em ordem.

### 3.1. Resolver o alvo semântico

O runtime precisa saber em qual vector store procurar o metadata do schema. Esse alvo vem de `schema_metadata.vectorstore_id`.

### 3.2. Recuperar contexto útil

A tool consulta o vector store com `top_k` controlado e rejeita qualquer documento que não seja do tipo `schema_metadata`. Isso impede misturar schema com conteúdo irrelevante do acervo.

### 3.3. Transformar documentos em contexto legível

Os documentos recuperados são reorganizados em um contexto textual com cabeçalho por tabela e limite operacional de tamanho. Isso prepara o material para o LLM sem explodir o prompt.

### 3.4. Gerar uma proposta de SQL

Com o contexto pronto, o runtime chama o LLM com baixa temperatura, pede uma única consulta no dialeto explícito e tenta extrair uma SQL limpa da resposta.

## 4. Visão tática

Taticamente, essa feature é ideal para três situações.

- exploração inicial de um schema pouco conhecido;
- desenho de consulta quando os nomes físicos do banco não ajudam;
- apoio assistido a times que precisam de uma proposta inicial antes da revisão humana.

Ela é menos adequada quando a consulta já está aprovada e precisa ser executada repetidamente. Nesse caso, a geração dinâmica deixa de ser o centro do problema e a governança da query persistida passa a importar mais.

## 5. Visão técnica

Tecnicamente, o coração da feature está em quatro elementos do código lido.

### 5.1. Factory da tool

`SqlSchemaRagToolFactory` valida o YAML mínimo, resolve logger, `vectorstore_id`, dialeto, `top_k`, retry externo, factory do vector store e limite de contexto.

### 5.2. Recuperação vetorial

A factory tenta usar `similarity_search` e, se o backend não oferecer esse método, tenta `search_similar`. Ambas as chamadas passam pelo helper central de retry externo.

### 5.3. Preparação do contexto

O runtime mantém apenas documentos com `document_type=schema_metadata`, reorganiza o conteúdo por tabela e trunca o material acima de 12000 caracteres.

### 5.4. Superfície HTTP dedicada

`Nl2SqlService` envolve a tool para montar resposta estruturada, propagar `correlation_id`, produzir diagnósticos estáveis e aplicar `SqlReadOnlyGuardrail` antes de devolver a SQL como proposta revisável.

## 6. Visão executiva

Para liderança, o valor principal é acelerar análise de dados sem abrir mão de controle. A plataforma não entrega execução automática de SQL gerada por IA. Ela entrega uma proposta assistida, rastreável e com barreira explícita de revisão.

Isso reduz dependência de especialistas em schema nas etapas iniciais e diminui o risco de automação precipitada em uma área sensível.

## 7. Visão comercial

Comercialmente, a dor atendida é direta: o cliente sabe o que quer perguntar, mas não consegue traduzir isso rapidamente para um banco complexo e pouco amigável.

O diferencial real não é IA que escreve SQL. O diferencial real é IA que recebe contexto semântico controlado do schema e devolve uma proposta mais governável.

Essa distinção importa porque ela muda a promessa comercial. O produto não promete acerto mágico. Ele promete redução de atrito, melhor ponto de partida técnico e menor risco operacional.

## 8. Visão estratégica

Estrategicamente, essa feature fortalece a plataforma em quatro frentes.

- transforma metadados de schema em ativo operacional de IA;
- desacopla descoberta de consulta da memória humana sobre o banco;
- reaproveita a infraestrutura de vector store, retry e tool catalog já existente;
- cria uma base sólida para evoluir de assistente de schema para fluxos mais amplos de analytics assistido.

## 9. Por que isso é ideal para bancos enormes e nomes pouco significativos

O código lido mostra por que esse desenho é mais adequado para bancos grandes do que uma geração ingênua.

### 9.1. O runtime não manda o schema inteiro ao modelo

O contexto vem de busca vetorial com `top_k` limitado. Isso reduz ruído e custo.

### 9.2. A tool aceita apenas `schema_metadata`

Mesmo que o vector store tenha outros tipos de documento, a factory filtra o conjunto e descarta o que não for schema. Isso reduz contaminação do contexto.

### 9.3. O contexto tem limite operacional

Quando o material recuperado cresce demais, ele é truncado e o endpoint dedicado registra isso como diagnóstico. Isso é importante em bases grandes porque impede prompts sem controle.

### 9.4. O problema vira qual pedaço do schema parece relevante

Essa troca é decisiva. Em vez de depender do conhecimento completo do banco, o runtime tenta recuperar o subconjunto mais promissor primeiro.

### 9.5. Ainda existe dependência da qualidade do metadata

Essa arquitetura lida melhor com nomes ruins, mas não elimina a necessidade de bons documentos de schema. Se o metadata estiver pobre, a SQL também tende a perder qualidade.

## 10. Conceitos necessários para entender

### 10.1. `schema_rag_sql`

É o identificador canônico da family de tool no catálogo interno.

### 10.2. `schema_metadata`

É o tipo de documento que representa metadados de schema no acervo pesquisável. A tool só aceita esse tipo na etapa de recuperação.

### 10.3. `vectorstore_id`

É o identificador do índice vetorial que contém os documentos de schema. Sem ele, a tool não sabe onde procurar contexto.

### 10.4. `sql_dialect`

É o dialeto explicitamente aceito pelo runtime. O código lido suporta `postgresql`, `mysql` e `mssql`.

### 10.5. `top_k`

É a largura máxima da recuperação. Quanto maior, maior cobertura. Quanto menor, menor ruído. O default da tool e do serviço é `5`.

### 10.6. `SchemaContextResult`

É o objeto interno que consolida contexto montado, contagem de documentos, truncamento e mensagem de erro operacional.

### 10.7. `SqlSchemaRagRunResult`

É o resultado estruturado interno da geração. Ele diferencia `sql_text`, `raw_response` e resumo do contexto usado.

### 10.8. `SqlReadOnlyGuardrail`

É o guardrail central que valida se a SQL é comprovadamente somente leitura. Importante: ele é aplicado explicitamente no serviço HTTP dedicado, não dentro da factory da tool.

### 10.9. `correlation_id`

É o identificador de rastreio da execução. A tool exige esse valor em `user_session` quando usada diretamente. O endpoint dedicado resolve e injeta esse valor para o chamador.

## 11. Como a feature funciona por dentro

O fluxo real começa quando alguém cria a tool ou chama o serviço dedicado.

Se a entrada for a tool direta, a factory exige um YAML com `user_session.user_email`, `user_session.correlation_id`, `schema_metadata.vectorstore_id` e `schema_metadata.sql_dialect`.

Se a entrada for o endpoint dedicado, o router aceita várias formas de resolver o YAML, gera ou reaproveita `correlation_id`, valida o payload e chama `Nl2SqlService`.

Depois disso, o serviço faz uma preparação importante: ele força `schema_metadata.sql_dialect` com o dialeto pedido na requisição e injeta `user_session` mesmo que o chamador não tenha montado essa parte manualmente.

Com o YAML pronto, `SqlSchemaRagToolFactory` executa a lógica principal.

Primeiro, tenta construir o vector store pelo contrato canônico `ConfiguredVectorStoreFactory`.

Depois, faz busca por similaridade usando `top_k`. Se o backend não expõe `similarity_search`, ela tenta `search_similar`. Se nenhum dos dois existir, a execução devolve um aviso operacional em vez de contexto útil.

Na sequência, o runtime filtra os documentos retornados. Só seguem adiante os que possuem `document_type=schema_metadata`.

Se nada válido sobrar, a execução não quebra com exceção dura. Ela devolve uma mensagem de aviso estruturada no resultado interno.

Se existirem documentos válidos, o contexto é montado com cabeçalhos por tabela e separado por blocos. Depois disso, o runtime aplica o limite de caracteres antes da chamada ao LLM.

Com o contexto pronto, a factory cria o LLM com temperatura `0.0`, insere dialeto, contexto e pergunta no prompt de geração e chama o modelo com retry externo.

Por fim, a resposta do LLM é normalizada. O runtime tenta extrair SQL de bloco `sql`, de bloco genérico ou, na ausência disso, usa o texto limpo.

No caso do endpoint dedicado, ainda existe uma etapa extra: a SQL candidata passa por `SqlReadOnlyGuardrail`. Só depois desse passo o serviço devolve `success=true` com SQL pronta para revisão.

## 12. Divisão em etapas ou submódulos

### 12.1. Publicação da tool no catálogo

O decorator `@tool_factory` publica a capacidade com `catalog_id=schema_rag_sql`, `tool_name=schema_rag_sql_query_generator`, `factory_returns=single` e tags semânticas associadas.

Valor entregue: a capacidade deixa de ser uma classe isolada e passa a existir como item governável no catálogo interno.

### 12.2. Construção da instância da tool

`create_sql_schema_rag_tool` cria a factory, monta a chave de cache e usa o resource pool para reaproveitar a instância da tool.

Valor entregue: evita reconstrução redundante e mantém isolamento por `vectorstore_id`, dialeto e `correlation_id` via cache key.

### 12.3. Recuperação do contexto de schema

`_search_schema_metadata_result` e `_execute_schema_search` fazem a busca semântica com retry.

Valor entregue: a SQL não nasce do vazio; nasce de contexto técnico recuperado.

### 12.4. Normalização do contexto

`_format_schema_context_result` reorganiza documentos e aplica truncamento.

Valor entregue: reduz ruído e impõe limite operacional antes do LLM.

### 12.5. Geração da proposta de SQL

`_generate_sql_via_llm` cria o prompt, chama o modelo e extrai a SQL da resposta.

Valor entregue: converte pergunta mais contexto em query candidata no dialeto certo.

### 12.6. Governação HTTP da proposta

`Nl2SqlService` envolve a engine para validar dialeto, preparar YAML, produzir diagnósticos, exigir revisão e aplicar guardrail somente leitura.

Valor entregue: transforma uma tool de runtime em resposta administrativa auditável.

## 13. Fluxo principal ponta a ponta

![13. Fluxo principal ponta a ponta](assets/diagrams/docs-readme-sql-schema-rag-tool-diagrama-01.svg)

O ponto mais importante do diagrama é que a validação de somente leitura não pertence ao núcleo da factory. Ela pertence ao serviço HTTP dedicado que consome a factory.

## 14. Contratos, entradas e saídas

Existem dois contratos confirmados no código.

### 14.1. Contrato da tool direta

Entrada de construção:

- `yaml_config`
- `tool_name`
- `top_k`

Campos obrigatórios no `yaml_config`:

- `user_session.user_email`
- `user_session.correlation_id`
- `schema_metadata.vectorstore_id`
- `schema_metadata.sql_dialect`

Contrato de execução da tool:

- argumento único `user_question`
- retorno textual simples

Saída da tool:

- SQL como string, quando houver sucesso;
- texto de aviso ou erro operacional, quando não houver SQL utilizável.

### 14.2. Contrato do endpoint dedicado

Entrada HTTP confirmada:

- `prompt`
- `user_email`
- `dialect`
- `yaml_config`
- `yaml_config_path`
- `yaml_inline_content`
- `encrypted_data`
- `top_k`
- `correlation_id`

Saída HTTP confirmada:

- `success`
- `correlation_id`
- `sql`
- `raw_response`
- `warnings`
- `diagnostics`
- `review_required`
- `execution_context`

Diferença crítica: a tool direta devolve apenas texto; o endpoint dedicado devolve resposta estruturada e auditável.

## 15. O que acontece em caso de sucesso

No caminho feliz da tool direta:

- o vector store é criado com sucesso;
- a busca retorna documentos relevantes;
- pelo menos parte deles é `schema_metadata`;
- o contexto é montado;
- o LLM devolve resposta que pode ser limpa como SQL.

No caminho feliz do endpoint dedicado, além disso:

- `schema_metadata.sql_dialect` é forçado corretamente;
- o guardrail aceita a SQL como somente leitura;
- a resposta volta com `success=true`;
- `review_required=true` continua explícito;
- o `execution_context` devolve resumo seguro do contexto usado.

## 16. O que acontece em caso de erro

Os cenários abaixo foram confirmados no código lido.

### 16.1. YAML inválido na factory

Se `yaml_config` não for mapeamento, a factory falha cedo com erro explícito.

### 16.2. `user_session` ausente ou malformado

Na tool direta, `user_email` e `correlation_id` são obrigatórios. Se faltarem, a construção falha.

### 16.3. `schema_metadata` ausente ou inválido

Se `schema_metadata` não for dicionário ou não trouxer `vectorstore_id`, a construção falha.

### 16.4. Dialeto inválido

Se `sql_dialect` ou `dialect` não estiver entre `postgresql`, `mysql` e `mssql`, a execução é bloqueada antes da geração.

### 16.5. Vector store sem método suportado

Se o backend não expuser `similarity_search` nem `search_similar`, a factory devolve aviso operacional em vez de contexto útil.

### 16.6. Nenhum documento recuperado

Se a busca vier vazia, o runtime devolve mensagem de ausência de metadata relevante.

### 16.7. Documentos recuperados sem `schema_metadata`

Se o vector store trouxer documentos de outro tipo, a execução devolve mensagem informando que não encontrou documentos válidos de schema.

### 16.8. Contexto truncado

Quando o volume recuperado passa do limite, o contexto é truncado. Isso não derruba a execução, mas reduz o material enviado ao LLM.

### 16.9. Falha de geração no LLM

Se o LLM falhar, a factory transforma isso em texto de erro operacional e o serviço dedicado converte o caso em `success=false` com diagnóstico.

### 16.10. Bloqueio pelo guardrail

No endpoint dedicado, se a SQL for mutável, vazia, múltipla ou não reconhecida como somente leitura, a resposta final não expõe a query como SQL válida.

## 17. Observabilidade e diagnóstico

O desenho observado no código privilegia rastreabilidade.

Pontos confirmados de observabilidade:

- `correlation_id` em toda a execução;
- logs na factory e no serviço;
- `source_hint` quando o YAML vem do resolvedor compartilhado;
- `vectorstore_id`, `top_k` e dialeto no contexto de execução do endpoint;
- contagem de documentos totais e de documentos válidos de schema;
- informação de truncamento do contexto;
- diagnósticos estruturados no endpoint dedicado.

Diagnósticos confirmados no serviço:

- `NL2SQL_VECTORSTORE_SELECTED`
- `NL2SQL_DIALECT_SELECTED`
- `NL2SQL_REVIEW_REQUIRED`
- `NL2SQL_SOURCE_HINT`
- `NL2SQL_SCHEMA_CONTEXT_TRUNCATED`
- `NL2SQL_GENERATION_FAILED`
- `NL2SQL_SQL_GUARDRAIL_BLOCKED`
- `NL2SQL_SQL_GUARDRAIL_ALLOWED`
- `NL2SQL_SQL_READY_FOR_REVIEW`

## 18. Superfícies de uso e governança

Esta feature não deve ser tratada como uma única interface homogênea.

### 18.1. Tool de runtime

É a superfície útil para agentes e fluxos internos que precisam de uma string de SQL ou de um aviso textual simples.

### 18.2. Endpoint administrativo

É a superfície útil para backoffice, UI administrativa ou integrações que precisam de contrato HTTP estável, diagnósticos e guardrail de somente leitura.

Essa separação é saudável porque impede que a tool básica precise carregar toda a responsabilidade de governança da resposta HTTP.

## 19. Diferença entre a tool e o endpoint dedicado

Essa é a distinção mais importante do manual.

### 19.1. O que a tool garante

A tool garante busca de contexto, geração de SQL e retorno textual.

### 19.2. O que a tool não garante sozinha

A tool não garante, pelo código lido nesta rodada, que a SQL retornada já passou pelo guardrail central de somente leitura.

### 19.3. O que o endpoint acrescenta

O endpoint acrescenta:

- resolução do YAML de várias origens;
- normalização explícita do dialeto;
- injeção de `user_session`;
- resposta estruturada;
- diagnósticos estáveis;
- aplicação do `SqlReadOnlyGuardrail`.

Na prática, isso significa que a frase SQL Schema RAG Tool gera SQL segura seria imprecisa se dita sem contexto. O desenho correto é: a tool gera uma proposta; o endpoint dedicado é quem a transforma em proposta revisável validada como somente leitura.

## 20. Vantagens práticas

- reduz o volume de schema enviado ao modelo;
- obriga o uso de um `vectorstore_id` específico para metadata de schema;
- filtra documentos de schema e descarta ruído de outros tipos de documento;
- usa retry central em acesso a vector store e LLM;
- mantém `top_k` pequeno por padrão;
- expõe uma superfície simples para agentes e outra estruturada para backoffice;
- permite bloquear SQL insegura no consumidor HTTP dedicado.

## 21. Exemplos práticos guiados

### 21.1. Pergunta simples sobre vendas

Cenário: o operador pergunta pelo total de vendas por mês.

O que acontece: a tool recupera os documentos mais parecidos no vector store, monta o contexto e gera uma SQL no dialeto configurado.

Impacto prático: o operador recebe um rascunho técnico sem ter que abrir manualmente o schema inteiro.

### 21.2. Vector store populado com conteúdo misto

Cenário: o índice vetorial contém documentos de vários domínios, inclusive conteúdo que não é schema.

O que acontece: a tool descarta tudo o que não tenha `document_type=schema_metadata`.

Impacto prático: diminui a chance de a SQL nascer de contexto contaminado.

### 21.3. Schema muito grande

Cenário: os documentos recuperados geram um contexto extenso demais.

O que acontece: o contexto é truncado antes da chamada ao LLM.

Impacto prático: a execução continua, mas o operador precisa saber que parte do contexto ficou de fora.

### 21.4. SQL destrutiva ou inválida

Cenário: a resposta do modelo traz comando não somente leitura.

O que acontece: a tool direta continua sendo apenas retorno textual, mas o endpoint dedicado bloqueia a proposta via guardrail.

Impacto prático: a superfície governada impede que uma resposta perigosa seja apresentada como SQL pronta.

## 22. Explicação 101

Imagine que o banco de dados é uma biblioteca enorme, mas com prateleiras mal nomeadas. Em vez de pedir para a IA olhar a biblioteca inteira, a ferramenta primeiro tenta descobrir quais prateleiras parecem mais ligadas à sua pergunta. Depois, entrega só esse pedaço para o modelo montar a consulta.

Se você usar a tool direta, recebe a sugestão em formato simples. Se usar o endpoint dedicado, existe ainda um fiscal técnico que confere se a consulta é apenas leitura antes de devolvê-la como proposta revisável.

## 23. Limites e pegadinhas

- A feature depende da existência prévia de documentos `schema_metadata` no vector store apontado.
- `top_k` pequeno demais pode perder contexto útil.
- `top_k` maior não substitui metadata ruim.
- A factory não aplica sozinha o guardrail central de somente leitura.
- Contexto truncado não é falha fatal, mas pode reduzir a qualidade da SQL.
- Se o vector store não expuser os métodos esperados, a execução não encontra contexto, mesmo com configuração aparentemente correta.
- Este manual foi reancorado no runtime da tool e do endpoint. A pipeline upstream que produz os documentos de schema não foi remapeada em profundidade nesta rodada.

## 24. Troubleshooting

### 24.1. A tool falha ao ser criada

Sintoma: erro logo na construção da tool.

Causa provável: falta de `user_session`, `vectorstore_id` ou `sql_dialect` no YAML.

### 24.2. O retorno é aviso em vez de SQL

Sintoma: a saída traz texto de aviso ou erro, não uma query.

Causa provável: busca vazia, documentos não marcados como `schema_metadata`, erro de vector store ou falha do LLM.

### 24.3. O endpoint responde com `success=false`

Sintoma: existe `raw_response`, mas `sql` veio nula.

Causa provável: falha de geração ou bloqueio pelo guardrail.

### 24.4. A proposta parece genérica demais

Sintoma: a SQL não usa tabelas ou colunas tão específicas quanto esperado.

Causa provável: contexto insuficiente, metadata pobre ou truncamento operacional.

### 24.5. A proposta foi bloqueada pelo guardrail

Sintoma: diagnóstico `NL2SQL_SQL_GUARDRAIL_BLOCKED`.

Causa provável: a SQL parecia mutável, tinha múltiplas sentenças, estava vazia ou não pôde ser validada como somente leitura.

## 25. Impacto técnico

Tecnicamente, a feature reforça separação de responsabilidades. A recuperação de contexto, a geração de SQL e a governança HTTP ficam em camadas diferentes, o que reduz acoplamento e melhora testabilidade futura.

## 26. Impacto executivo

Executivamente, ela reduz o tempo entre a pergunta de negócio e a primeira proposta de consulta, sem confundir velocidade com autorização cega de execução.

## 27. Impacto comercial

Comercialmente, ela ajuda a posicionar a plataforma como camada de entendimento assistido sobre bancos complexos, em vez de apenas mais um gerador genérico de SQL.

## 28. Impacto estratégico

Estrategicamente, a tool cria uma ponte entre catálogo técnico de dados, vector store e workflows agentic, fortalecendo a tese de plataforma orientada a ativos semânticos reutilizáveis.

## Leituras relacionadas

- [README.md](./README.md): índice por intenção para localizar os manuais vizinhos.
- [README-RAG.md](./README-RAG.md): explica o runtime de retrieval que sustenta esta tool.
- [README-DYNAMIC-SQL-TOOLS.md](./README-DYNAMIC-SQL-TOOLS.md): mostra a família mais ampla de tools SQL governadas.
- [README-SERVICE-API.md](./README-SERVICE-API.md): detalha a borda HTTP onde o endpoint dedicado de NL2SQL vive.
- [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md): aprofunda as origens de YAML aceitas pelo serviço dedicado.
- [tutorial-101-configuracao-yaml-execucao-local-e-nl-para-sql.md](./tutorial-101-configuracao-yaml-execucao-local-e-nl-para-sql.md): reconta o mesmo assunto em formato 101.

## 29. Como colocar para funcionar

Pelo código lido, os pré-requisitos mínimos confirmados são estes.

- documentos de schema já disponíveis no vector store alvo;
- `schema_metadata.vectorstore_id` apontando para esse índice;
- `schema_metadata.sql_dialect` coerente com `postgresql`, `mysql` ou `mssql`;
- `user_session.user_email` e `user_session.correlation_id` quando a tool for usada diretamente;
- acesso ao endpoint `POST /config/nl2sql/generate` com permissão `config.generate` quando a superfície escolhida for HTTP;
- backend vetorial que exponha `similarity_search` ou `search_similar`.

Caminho operacional ponta a ponta para construir ou ingerir os documentos `schema_metadata` não foi revalidado nesta rodada.

## 30. Checklist de entendimento

- Entendi que `schema_rag_sql` é a capacidade canônica desta feature.
- Entendi que a tool direta e o endpoint dedicado não oferecem exatamente o mesmo contrato.
- Entendi que `vectorstore_id` é obrigatório.
- Entendi que o runtime só aceita documentos `schema_metadata`.
- Entendi que o contexto pode ser truncado.
- Entendi que a tool direta não é, por si só, a camada de guardrail.
- Entendi que o endpoint dedicado adiciona diagnósticos e validação de somente leitura.
- Entendi que a qualidade final depende do metadata disponível no índice.

## 31. Evidências no código

- `src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py`
  - Motivo da leitura: núcleo real da tool.
  - Símbolos relevantes: `SqlSchemaRagToolFactory`, `create_sql_schema_rag_tool`, `generate_sql_for_question`, `_execute_schema_search`, `_format_schema_context_result`, `_generate_sql_via_llm`.
  - Comportamento confirmado: valida YAML mínimo, resolve vector store, filtra `document_type=schema_metadata`, trunca contexto, chama o LLM com temperatura zero e devolve SQL ou aviso textual.

- `src/api/services/nl2sql_service.py`
  - Motivo da leitura: camada de governança do endpoint dedicado.
  - Símbolos relevantes: `Nl2SqlService.generate_sql`, `_prepare_yaml_config`, `_normalize_dialect`, `_schema_context_summary`.
  - Comportamento confirmado: injeta `user_session`, fixa dialeto, chama a factory, monta diagnósticos e aplica `SqlReadOnlyGuardrail`.

- `src/api/routers/config_nl2sql_router.py`
  - Motivo da leitura: boundary HTTP da feature.
  - Símbolos relevantes: `generate_nl2sql`, `_resolve_yaml_payload`, `_resolve_correlation_id`.
  - Comportamento confirmado: expõe `POST /config/nl2sql/generate`, resolve YAML de várias origens e converte falhas de contexto em HTTP 400.

- `src/api/schemas/nl2sql_models.py`
  - Motivo da leitura: contrato estável do request e da resposta HTTP.
  - Símbolos relevantes: `Nl2SqlRequest`, `Nl2SqlResponse`, `Nl2SqlDiagnostic`.
  - Comportamento confirmado: a API dedicada aceita múltiplas formas de entrada de YAML e devolve SQL, avisos, diagnósticos e contexto de execução.

- `src/integrations/sql_read_only_guardrail.py`
  - Motivo da leitura: mecanismo de proteção da superfície HTTP dedicada.
  - Símbolos relevantes: `SqlReadOnlyGuardrail`, `SqlReadOnlyValidationResult`.
  - Comportamento confirmado: só aceita uma sentença SQL de leitura em `postgresql`, `mysql` ou `mssql` e bloqueia operações mutáveis.

- `src/agentic_layer/tools/tool_factory_decorator.py`
  - Motivo da leitura: publicação oficial da tool no catálogo.
  - Símbolos relevantes: `tool_factory`.
  - Comportamento confirmado: a factory de `schema_rag_sql` é publicada como tool factory oficial com metadados de catálogo.

- `src/agentic_layer/tools/domain_tools/schema_rag_tools/__init__.py`
  - Motivo da leitura: superfície pública do módulo.
  - Símbolos relevantes: `SqlSchemaRagToolFactory`, `create_sql_schema_rag_tool`.
  - Comportamento confirmado: o módulo expõe diretamente a factory e a função de criação da tool para consumo interno.
