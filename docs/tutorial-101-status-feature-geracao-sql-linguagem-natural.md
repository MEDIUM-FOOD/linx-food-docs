# Tutorial 101: geração de SQL por linguagem natural

## 1. O que esta feature é

Esta feature permite transformar uma pergunta em linguagem natural em uma proposta de SQL. No código atual do projeto, isso aparece de duas formas diferentes:

1. SQL governado por consulta já cadastrada.
2. SQL gerada com apoio de metadados de schema e LLM.

Entender essa diferença é o ponto mais importante do tutorial, porque as duas abordagens resolvem problemas parecidos, mas com riscos, controles e maturidade operacional diferentes.

## 2. O problema que a feature resolve

Quem trabalha com operação, atendimento técnico ou produto frequentemente sabe a pergunta de negócio, mas não sabe escrever a consulta SQL correta.

A feature existe para encurtar essa distância. Em vez de exigir que toda pessoa conheça o banco e escreva SQL manualmente, o sistema oferece caminhos assistidos para chegar a uma consulta revisável.

O ganho prático é velocidade com governança. O risco que a feature tenta reduzir é gerar SQL livre sem contexto, sem limite de leitura e sem revisão humana.

## 3. Os dois modos de uso no projeto

### 3.1 Modo A: SQL governado

Neste modo, o sistema não inventa uma query nova. Ele executa uma query que já foi preparada e governada antes.

A sintaxe parametrizada `dyn_sql<...>` existe justamente para isso. O `ToolLoader` reconhece esse formato e transforma o parâmetro em `query_id`. Depois, a factory de SQL dinâmico monta a tool e executa a consulta configurada.

Este modo é o mais previsível para produção porque trabalha em cima de consultas já conhecidas.

### 3.2 Modo B: SQL assistida por schema

Neste modo, o sistema usa metadados de schema indexados, recupera contexto relevante e pede para um LLM propor uma SQL.

No projeto, esse caminho está concentrado no motor `schema_rag_sql`. A factory valida `correlation_id`, `user_email`, `schema_metadata.vectorstore_id` e `schema_metadata.sql_dialect` antes de seguir.

Este modo é mais flexível, mas depende de schema metadata real e revisão humana obrigatória.

## 4. Conceitos que você precisa entender antes de usar

### 4.1 `schema_metadata.vectorstore_id`

É o identificador do acervo vetorial que guarda os metadados do schema do banco. Sem ele, o motor generativo não sabe de onde puxar contexto estrutural.

### 4.2 `sql_dialect`

É o dialeto SQL esperado pelo banco alvo. O serviço dedicado aceita apenas `postgresql`, `mysql` e `mssql`.

### 4.3 Guardrail de somente leitura

Mesmo quando o LLM consegue gerar uma SQL, ela ainda passa pelo `SqlReadOnlyGuardrail`. Isso existe para bloquear propostas mutáveis ou perigosas antes que elas virem algo utilizável.

### 4.4 Revisão humana

A saída do fluxo dedicado não deve ser tratada como execução automática. O próprio serviço marca a resposta como proposta assistida e exige revisão.

## 5. Como o caminho dedicado funciona

O endpoint dedicado é `POST /config/nl2sql/generate`.

O fluxo real confirmado no código é este:

1. o router resolve `correlation_id`;
2. resolve o YAML por payload inline ou por resolução compartilhada;
3. entrega o contexto ao `Nl2SqlService`;
4. o serviço valida prompt, email e dialeto;
5. prepara o YAML de trabalho com `user_session` e `schema_metadata`;
6. chama `SqlSchemaRagToolFactory.generate_sql_for_question`;
7. recebe a proposta de SQL;
8. aplica o guardrail de somente leitura;
9. devolve resposta com diagnóstico e revisão obrigatória.

O significado prático é simples: esse endpoint existe para gerar proposta revisável, não para executar SQL automaticamente em nome do usuário.

## 6. Como o motor generativo funciona por dentro

A `SqlSchemaRagToolFactory` concentra o coração do caminho generativo.

Ela faz quatro coisas importantes:

1. valida se existe contexto mínimo para trabalhar;
2. busca metadados de schema no vector store configurado;
3. limita o contexto antes de enviar ao LLM;
4. pede uma única consulta SQL no dialeto correto.

Se o vector store falhar, se o contexto vier vazio ou se o LLM devolver algo inválido, a factory devolve resultado sem SQL utilizável. Isso é uma proteção importante contra falsa sensação de sucesso.

## 7. Como o caminho governado funciona por dentro

O caminho governado é diferente. Ele não começa no schema metadata. Ele começa em uma query ou procedure previamente cadastrada.

O `ToolLoader` identifica ferramentas parametrizadas e resolve o nome do parâmetro correto para cada família. Para `dyn_sql`, o parâmetro é `query_id`. Depois disso, a `DynamicSqlToolFactory` cria a tool com nome obrigatório, conexão, parâmetros e modo de fetch.

Na prática, esse modo é melhor quando o negócio já sabe exatamente quais consultas quer oferecer com segurança operacional.

## 8. O que já está pronto hoje

Com base no código lido, o que já está claramente implementado é:

1. um endpoint dedicado de NL2SQL com contrato estável;
2. validação explícita de prompt, email e dialeto;
3. uso do motor `schema_rag_sql` no backend dedicado;
4. guardrail central de somente leitura;
5. caminho governado por `dyn_sql<...>` no runtime de tools.

## 9. O que ainda depende de contexto real do tenant

O que não nasce pronto só por existir código é o contexto de dados do tenant.

Para o modo generativo funcionar de verdade, o tenant precisa ter:

1. metadados de schema indexados;
2. `schema_metadata.vectorstore_id` correto;
3. `schema_metadata.sql_dialect` correto;
4. configuração YAML coerente com esse backend.

Sem isso, o fluxo dedicado pode existir, mas não terá base suficiente para produzir SQL útil.

## 10. O que acontece em caso de sucesso

Quando tudo está correto:

1. o endpoint aceita a pergunta;
2. o contexto de schema é recuperado;
3. a SQL é gerada;
4. o guardrail aprova como somente leitura;
5. a resposta volta com `success=true`, diagnósticos e aviso de revisão humana.

## 11. O que acontece em caso de erro

Os erros mais importantes confirmados no código são estes:

### 11.1 Falta de contexto mínimo

Se `yaml_config` não for válido, se `prompt` vier vazio, se `user_email` vier vazio ou se o dialeto não estiver entre os aceitos, o serviço falha cedo.

### 11.2 Falta de `vectorstore_id`

Se `schema_metadata.vectorstore_id` não estiver configurado, a factory não consegue trabalhar e bloqueia a geração.

### 11.3 SQL gerada, mas bloqueada

Mesmo quando o LLM devolve uma resposta, o guardrail pode bloquear a SQL se ela não for validada como somente leitura.

### 11.4 Falha na busca de schema

Se o backend vetorial falhar ou se o contexto recuperado for insuficiente, o resultado volta sem SQL utilizável.

## 12. Quando usar cada abordagem

Use SQL governado quando:

1. a pergunta do negócio já cabe em consultas conhecidas;
2. você quer máxima previsibilidade;
3. a operação precisa de menor risco.

Use SQL assistida por schema quando:

1. a pergunta é aberta demais para virar uma query fixa;
2. existe schema metadata confiável;
3. a jornada inclui revisão humana antes de uso.

## 13. Como pensar a feature sem jargão

Uma boa analogia é esta:

1. `dyn_sql<...>` é pedir um prato que já existe no cardápio.
2. `schema_rag_sql` é pedir ao cozinheiro uma receita nova olhando os ingredientes disponíveis.

O cardápio é mais previsível. A receita nova é mais flexível. O projeto implementa os dois caminhos porque eles resolvem momentos diferentes do produto.

## 14. Checklist de entendimento

Se você terminou o tutorial e entendeu estes pontos, o básico está correto:

1. entendi que existem dois caminhos de SQL por linguagem natural no projeto;
2. entendi que o endpoint dedicado gera proposta e não execução automática;
3. entendi que o guardrail de somente leitura é obrigatório;
4. entendi que o modo generativo depende de schema metadata por tenant;
5. entendi que `dyn_sql<...>` é o caminho governado e mais previsível.

## 15. Exercícios guiados

### Exercício 1

Objetivo: distinguir modo governado de modo generativo.

Passos:

1. leia a descrição de `ToolLoader.parse_parameterized_tool` e `resolve_param_key`;
2. depois leia a inicialização de `SqlSchemaRagToolFactory`.

O que observar:

1. o primeiro fluxo resolve `query_id`;
2. o segundo exige `schema_metadata.vectorstore_id` e `sql_dialect`.

Resposta esperada: um caminho parte de query cadastrada; o outro parte de contexto de schema.

### Exercício 2

Objetivo: entender por que o endpoint dedicado não executa SQL automaticamente.

Passos:

1. leia o fluxo do router dedicado;
2. depois leia o `Nl2SqlService`.

O que observar:

1. a resposta sempre carrega revisão humana;
2. o guardrail pode bloquear a proposta mesmo quando houve geração.

Resposta esperada: o contrato existe para proposta revisável, não para disparo cego de SQL.

## 16. Evidências no código

1. `src/api/routers/config_nl2sql_router.py`: endpoint dedicado e resolução de contexto.
2. `src/api/services/nl2sql_service.py`: validação de entrada, uso do motor generativo e guardrail.
3. `src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py`: geração assistida via schema metadata e LLM.
4. `src/agentic_layer/supervisor/tool_loader.py`: reconhecimento de `dyn_sql<...>` e resolução de `query_id`.
5. `src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py`: caminho governado de SQL dinâmica.
