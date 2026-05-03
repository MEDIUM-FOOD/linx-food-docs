# Manual tecnico, operacional e de uso: NL2SQL completo

## 1. Slice executavel confirmado no codigo

O slice executavel de NL2SQL deste projeto e um boundary administrativo proprio, com contrato HTTP estavel e servico dedicado. A superficie publica confirmada e POST /config/nl2sql/generate. Esse endpoint recebe um prompt em linguagem natural, resolve o contexto YAML do tenant ou runtime ativo, injeta user_session, chama a engine schema_rag_sql e devolve uma resposta estruturada com SQL proposta, diagnosticos, warnings, correlation_id e metadados seguros da execucao.

O ponto tecnico mais importante e este: NL2SQL nao foi implementado como dyn_sql nem como /agent/execute com prompt livre. Ele e um slice separado, com objetivo especifico de gerar SQL revisavel sob governanca.

## 2. Contrato HTTP de entrada

O contrato tipado esta em src/api/schemas/nl2sql_models.py. Os campos confirmados sao estes.

1. prompt: pergunta em linguagem natural.
2. user_email: operador da execucao.
3. dialect: postgresql, mysql ou mssql.
4. yaml_config, yaml_config_path, yaml_inline_content ou encrypted_data: fonte de contexto.
5. top_k: quantidade maxima de documentos de schema recuperados.
6. correlation_id opcional.

O endpoint aceita varias formas de resolver o YAML, mas nao aceita dialeto livre. Isso e importante porque o guardrail depende do dialect informado.

## 3. Contrato HTTP de saida

O retorno tipado tambem esta em nl2sql_models. Os campos principais sao estes.

1. success.
2. correlation_id.
3. sql proposta ou nula.
4. raw_response do motor.
5. warnings.
6. diagnostics estruturados.
7. review_required.
8. execution_context.

O contrato deixa claro que o fluxo e sempre revisavel. review_required vem como verdadeiro por padrao, inclusive no sucesso.

## 4. Boundary HTTP real

O router fica em src/api/routers/config_nl2sql_router.py. O comportamento confirmado e este.

1. Prefixo /config/nl2sql.
2. POST /generate.
3. Permissao exigida: CONFIG_GENERATE.
4. Rate limit de configuracao.
5. correlation_id resolvido por payload, request.state ou factory central.
6. YAML resolvido por quatro caminhos: yaml_config em memoria, yaml_config_path, yaml_inline_content ou encrypted_data.
7. Erro de contexto devolvido como 400.
8. Erro interno devolvido como 500 com mensagem fechada.

Esse boundary existe para manter NL2SQL desacoplado de outras superficies agenticas e para publicar um contrato administrativo previsivel.

## 5. Servico dedicado

O servico fica em src/api/services/nl2sql_service.py. Ele e a camada que transforma o request administrativo em uma execucao backend organizada.

### 5.1. Validacoes iniciais

Antes de chamar qualquer motor, o servico valida.

1. yaml_config precisa ser mapeamento valido.
2. prompt nao pode estar vazio.
3. user_email nao pode estar vazio.
4. dialect precisa ser postgresql, mysql ou mssql.
5. schema_metadata precisa existir e ser objeto.
6. schema_metadata.vectorstore_id precisa existir.

### 5.2. Enriquecimento do YAML

O servico faz deepcopy do YAML recebido e injeta.

1. schema_metadata.sql_dialect com o dialeto normalizado.
2. user_session.user_email.
3. user_session.correlation_id.

Esse ponto e importante porque a engine schema_rag_sql exige correlation_id e user_email no YAML.

### 5.3. Montagem de diagnosticos

O servico constroi diagnosticos estaveis ao longo do fluxo. Os codigos confirmados incluem.

1. NL2SQL_VECTORSTORE_SELECTED.
2. NL2SQL_DIALECT_SELECTED.
3. NL2SQL_REVIEW_REQUIRED.
4. NL2SQL_SOURCE_HINT.
5. NL2SQL_SCHEMA_CONTEXT_TRUNCATED.
6. NL2SQL_GENERATION_FAILED.
7. NL2SQL_SQL_GUARDRAIL_BLOCKED.
8. NL2SQL_SQL_GUARDRAIL_ALLOWED.
9. NL2SQL_SQL_READY_FOR_REVIEW.

Esses diagnosticos sao a principal trilha estruturada para UI, suporte e auditoria.

## 6. Engine schema_rag_sql

O motor efetivo de geracao fica em src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py. Essa factory faz quatro coisas.

1. Valida o YAML minimo exigido.
2. Abre o vector store configurado para schema metadata.
3. Recupera documentos relevantes por similaridade semantica.
4. Chama o LLM com prompt orientado ao dialeto e ao contexto recuperado.

Essa e a principal diferenca tecnica entre NL2SQL serio e abordagem ingenua. O motor nao trabalha com o schema inteiro. Ele trabalha com recuperacao semanticamente dirigida.

## 7. Por que o desenho escala para bases grandes

O comportamento confirmado da factory tem tres mecanismos que explicam a boa adequacao a bases gigantes.

### 7.1. Busca top_k em vez de dump total

O vector store e consultado com similarity_search ou search_similar, sempre limitado por top_k. O valor padrao e 5 e o endpoint aceita de 1 a 20. Isso impede a explosao de contexto comum em bases grandes.

### 7.2. Filtragem explicita para schema_metadata

Mesmo que o vector store devolva outros documentos, o motor so considera document_type=schema_metadata como contexto valido para NL2SQL. Isso reduz ruido semantico.

### 7.3. Truncamento operacional do contexto

Depois de formatar o contexto recuperado, a factory aplica um limite de caracteres. Se ultrapassar DEFAULT_SCHEMA_CONTEXT_MAX_CHARS, o texto e truncado e o slice registra esse fato em diagnostico.

Em termos práticos, isso protege custo, previsibilidade e qualidade do prompt em bases gigantes.

## 8. Estrutura do prompt de geracao

O prompt de geracao explicita.

1. Dialeto SQL.
2. Contexto de schema recuperado.
3. Pergunta do usuario.
4. Regras para usar apenas tabelas e colunas existentes no contexto.
5. Proibicao de comandos mutaveis e multiplas sentencas.

Esse prompt nao confia no LLM para decidir tudo sozinho. Ele enquadra o problema como geracao controlada sobre um contexto restrito.

## 9. Formacao do contexto de schema

O contexto e montado por tabela, com cabecalho schema.tabela quando disponivel e corpo contendo o documento recuperado. A exportacao de schema metadata por tabela, confirmada em SchemaMetadataJsonExporter, inclui.

1. database_code.
2. database_name.
3. schema_name.
4. table_name.
5. business_name quando existir.
6. table_description.
7. colunas.
8. chave primaria.
9. chaves estrangeiras.
10. sample_rows apenas com opt-in explicito.

Isso explica por que o slice ajuda em bases com nomes ruins: a unidade de recuperacao e um documento de contexto de tabela, nao apenas o identificador cru.

## 10. Pipeline de schema metadata

O preparo de schema metadata aparece em duas camadas relevantes.

### 10.1. ETL novo de metadados

TableSchemaMetadataETLProcessor recebe source_database e target_database, testa conexoes, filtra tabelas, extrai colunas, PK, FK e grava tudo em dbschemas. Ele e generico no contrato e nao hardcodeia dominio de negocio.

### 10.2. Exportacao para ingestao semantica

SchemaMetadataJsonExporter le dbschemas, monta um documento JSON por tabela e devolve em memoria ou arquivo. Esse documento e o material consumido pelo vector store de schema metadata.

O ponto tecnico importante e este: NL2SQL depende de um acervo preparado antes, nao de introspecao improvisada no meio do request.

## 11. Suporte confirmado de bases e dialetos

Aqui e necessario ser preciso.

### 11.1. Dialetos aceitos pelo slice NL2SQL

O endpoint e o guardrail dedicados validam explicitamente apenas.

1. postgresql.
2. mysql.
3. mssql.

### 11.2. Readers confirmados no preparo de schema metadata

A factory de readers lida nesta tarefa confirma suporte para.

1. PostgreSQL.
2. SQL Server.
3. Oracle.

Essa diferenca precisa aparecer no manual tecnico porque afeta expectativa de implantacao. O desenho e generico para bases de ERP, mas o suporte executavel confirmado hoje nao e identico em todas as camadas.

## 12. Guardrail de somente leitura

O guardrail fica em src/integrations/sql_read_only_guardrail.py e usa sqlglot. O comportamento confirmado e este.

1. Normaliza o dialect para o parser correspondente.
2. Bloqueia SQL vazia.
3. Bloqueia parse invalido.
4. Bloqueia multiplas sentencas.
5. Bloqueia nos mutaveis como Insert, Update, Delete, Drop, Create, Alter, Merge, Truncate e Execute.
6. Permite apenas tipos raiz de leitura como Select, Union, Intersect, Except, Show e Describe.

Isso e o que impede uma proposta destrutiva de sair do slice como SQL pronta para uso.

## 13. Fluxo principal de ponta a ponta

O fluxo principal confirmado no codigo e este.

1. Cliente administrativo envia prompt, user_email, dialect e contexto YAML.
2. Router resolve correlation_id e YAML.
3. Servico valida entrada e enriquece o YAML.
4. Servico instancia SqlSchemaRagToolFactory.
5. Factory abre o vector store configurado por vectorstore_id.
6. Factory executa busca semantica top_k no acervo de schema metadata.
7. Factory filtra e formata o contexto recuperado.
8. Factory corta o contexto se passar do limite operacional.
9. Factory chama o LLM com prompt orientado ao dialeto.
10. Servico remove code fences se necessario.
11. Servico passa a SQL candidata pelo SqlReadOnlyGuardrail.
12. Servico devolve sucesso ou falha estruturada com diagnostics e execution_context.

## 14. Como a UI administrativa usa o slice

Os testes de frontend confirmam que a pagina ui-admin-plataforma-sql-natural.html usa JS e CSS externos dedicados e chama this.api.generateNl2Sql. O contrato testado deixa explicito que a pagina usa o endpoint dedicado de NL2SQL e nao reenfileira geracao via /agent/execute.

Isso importa porque mostra que a feature nao e apenas API interna. Existe uma superficie administrativa pronta para operacao e demonstracao.

## 15. Como usar em outras bases de dados de ERP

O reuso correto do slice segue esta logica.

1. Preparar schema metadata da base de origem.
2. Exportar o acervo por tabela e indexa-lo no vector store configurado.
3. Criar ou apontar um YAML runtime com schema_metadata.vectorstore_id e sql_dialect.
4. Chamar o endpoint dedicado com prompt, user_email, dialect e o contexto YAML.

O ganho e que o servico nao depende de nomes PDV especificos no contrato. O exemplo PDV e apenas um runtime concreto ja pronto no repositório.

## 16. O que o slice nao faz

1. Nao executa automaticamente a SQL gerada.
2. Nao substitui dyn_sql para queries previamente aprovadas.
3. Nao garante suporte universal a qualquer dialeto so porque a arquitetura e generica.
4. Nao elimina a necessidade de qualidade minima do acervo de schema metadata.
5. Nao compensa sozinho um catalogo pobre, sem descricoes e sem relacoes bem materializadas.

## 17. O que acontece em caso de sucesso

No caminho feliz, o retorno contem success=true, sql com a proposta final, warnings com lembrete de revisao, diagnostics informativos e execution_context com vectorstore_id, top_k, dialect, tool_catalog_id schema_rag_sql, resumo do schema_context e resultado do guardrail.

## 18. O que acontece em caso de erro

### 18.1. Erro de configuracao

Se schema_metadata faltar ou vectorstore_id vier vazio, o servico lança ValueError com mensagem clara. O router converte isso em 400.

### 18.2. Falha de geracao

Se o motor nao conseguir produzir SQL utilizavel, o retorno vem com success=false, sql nula e diagnostico NL2SQL_GENERATION_FAILED.

### 18.3. Bloqueio do guardrail

Se a SQL for mutavel ou invalida para o fluxo somente leitura, o retorno vem com success=false, sql nula e diagnostico NL2SQL_SQL_GUARDRAIL_BLOCKED.

### 18.4. Truncamento de contexto

Se o contexto de schema exceder o limite, o slice nao falha necessariamente, mas registra NL2SQL_SCHEMA_CONTEXT_TRUNCATED como warning.

## 19. Observabilidade e diagnostico

O diagnostico desse slice deve partir destes pontos.

1. correlation_id da execucao.
2. diagnostics retornados ao cliente.
3. execution_context.vectorstore_id.
4. execution_context.dialect.
5. execution_context.schema_context.document_count e schema_document_count.
6. execution_context.schema_context.truncated.
7. execution_context.sql_guardrail.allowed e reason_code.

Em termos praticos, isso permite diferenciar.

1. problema de contexto YAML.
2. problema de recuperacao de schema.
3. problema de geracao pelo LLM.
4. problema de seguranca no guardrail.

## 20. Testes que comprovam o comportamento

### 20.1. Testes unitarios do servico

test_admin_nl2sql_api confirma.

1. injecao de user_session.
2. limpeza de code fence.
3. erro claro sem vectorstore.
4. falha controlada quando o motor erra.
5. bloqueio de SQL destrutiva.

### 20.2. Testes de integracao do endpoint

test_admin_nl2sql_api de integracao confirma.

1. permissao CONFIG_GENERATE.
2. retorno tipado com correlation_id.
3. diagnosticos esperados.
4. bloqueio do guardrail no boundary HTTP.
5. uso do YAML PDV real com vectorstore dedicado.

### 20.3. Testes de contrato do YAML

test_nl2sql_pdv_yaml_contract confirma.

1. tools_library vazio.
2. schema_metadata.enabled=true.
3. vectorstore_id explicito.
4. sql_dialect explicito.
5. alinhamento do runtime com o vector store de schema metadata.

### 20.4. Testes de frontend

Os testes de frontend confirmam que a pagina administrativa usa o endpoint dedicado, exige selecao de dialeto e nao reusa o fluxo generico de agente para gerar SQL.

## 21. Troubleshooting

### 21.1. Prompt retorna falha sem SQL

Causa provavel: contexto de schema insuficiente ou acervo de schema metadata ruim.

Como confirmar: procurar NL2SQL_GENERATION_FAILED e revisar raw_response parcial nos diagnostics.

### 21.2. SQL foi bloqueada

Causa provavel: o LLM respondeu com mutacao ou tipo de statement nao permitido.

Como confirmar: olhar execution_context.sql_guardrail.reason_code e statement_type.

### 21.3. Resultado ruim em base enorme

Causa provavel: top_k baixo demais, acervo mal indexado ou metadata pouco descritiva.

Como confirmar: revisar schema_context.document_count, schema_document_count, truncation e qualidade dos documentos exportados.

### 21.4. YAML parece correto mas o endpoint recusa

Causa provavel: schema_metadata ausente, vectorstore_id vazio ou dialect fora do conjunto suportado.

Como confirmar: revisar mensagens 400 e diagnostics iniciais.

## 22. Explicacao 101

Tecnicamente, o slice funciona como um funil. Primeiro ele encontra, dentro de um catalogo enorme de estrutura, so as partes do schema que parecem relevantes para a pergunta. Depois pede ao modelo uma SQL baseada apenas nesse recorte. No fim, um validador confere se a resposta nao faz nada perigoso. E isso que torna a feature mais robusta para bases grandes do que abordagens que tentam mandar tudo para o modelo de uma vez.

## 23. Checklist de entendimento

- Entendi o endpoint dedicado de NL2SQL.
- Entendi como o servico enriquece e valida o YAML.
- Entendi o papel de schema_rag_sql.
- Entendi por que top_k e truncamento ajudam em bases grandes.
- Entendi o papel do guardrail de somente leitura.
- Entendi como o slice pode ser reutilizado em outras bases de ERP.
- Entendi a assimetria atual entre readers de metadata e dialetos do endpoint.

## 24. Evidencias no codigo

- src/api/routers/config_nl2sql_router.py
  - Motivo da leitura: confirmar o boundary HTTP e a resolucao de YAML.
  - Comportamento confirmado: /config/nl2sql/generate e a rota dedicada do slice.

- src/api/services/nl2sql_service.py
  - Motivo da leitura: confirmar validacoes, diagnosticos e montagem da resposta.
  - Comportamento confirmado: o servico injeta user_session, chama schema_rag_sql e aplica guardrail.

- src/api/schemas/nl2sql_models.py
  - Motivo da leitura: confirmar contrato de request e response.
  - Comportamento confirmado: top_k, dialect, review_required e diagnostics sao campos oficiais do slice.

- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Motivo da leitura: confirmar recuperacao top_k, filtragem e truncamento.
  - Comportamento confirmado: o contexto e semanticamente recuperado e limitado antes do LLM.

- src/ingestion_layer/schema_metadata/schema_metadata_exporter.py
  - Motivo da leitura: confirmar exportacao do acervo por tabela.
  - Comportamento confirmado: o documento de schema inclui estrutura rica para a busca semantica.

- src/etl_layer/providers/data/table_schema_metadata_processor.py
  - Motivo da leitura: confirmar pipeline generico de preparo de metadata.
  - Comportamento confirmado: origem e destino sao configurados por contrato, sem hardcode do PDV.

- src/schema_metadata/reader_factory.py
  - Motivo da leitura: confirmar supporte de readers no preparo de schema metadata.
  - Comportamento confirmado: factory detecta PostgreSQL, SQL Server e Oracle por DSN.

- src/integrations/sql_read_only_guardrail.py
  - Motivo da leitura: confirmar politica de somente leitura.
  - Comportamento confirmado: mutacao e multiplas sentencas sao bloqueadas.

- app/yaml/rag-config-pdv-nl2sql.yaml
  - Motivo da leitura: confirmar runtime real de exemplo.
  - Comportamento confirmado: runtime aponta para vectorstore de schema metadata e dialeto explicito.

- tests/unit/test_admin_nl2sql_api.py
  - Motivo da leitura: confirmar comportamentos protegidos por unidade.
  - Comportamento confirmado: code fence, guardrail e erros claros estao cobertos.

- tests/integration/test_admin_nl2sql_api.py
  - Motivo da leitura: confirmar o endpoint em uso real.
  - Comportamento confirmado: permissao, correlation_id, diagnosticos e YAML PDV real estao cobertos.
