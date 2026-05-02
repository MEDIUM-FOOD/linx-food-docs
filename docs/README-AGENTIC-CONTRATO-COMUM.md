# Manual Tecnico Canônico: Contrato YAML Agentic

Este documento consolida o contrato YAML compartilhado entre:

1. `workflow`
2. `supervisor`
3. `deepagent`

Se você está começando agora, leia primeiro:

1. `docs/README-AGENTIC-INICIANTES.md`

Escopo:

1. Sintaxe comum e blocos reutilizáveis.
2. Regras de precedência de configuração.
3. Restrições reais aplicadas no runtime atual.

Fora de escopo:

1. Parâmetros internos de cada ferramenta do catálogo.
2. Detalhes de domínios específicos de tools (`dyn_sql`, `dyn_api`, etc.).

## 0.1 Matriz Oficial de Escolha do Runtime

| Situação | Escolha correta | Por quê |
| --- | --- | --- |
| Precisa de `memory.agent_conversation_history` ou `memory.qa_short_history` como contrato funcional | `execution.type=agent` | Esses contratos continuam vivos no modo clássico e não têm equivalência automática no DeepAgent. |
| Precisa de `interrupt_on`, `permissions`, `middlewares.filesystem`, `deepagent_memory` ou `async_subagents[]` | `execution.type=deepagent` | Esses recursos pertencem ao runtime DeepAgent e são onde ele entrega valor real. |
| Fluxo simples, sem governança local e sem memória persistente por identidade | `execution.type=agent` | Menor complexidade operacional. |
| Recurso novo de profundidade, delegação rica ou memória persistente de longo prazo | `execution.type=deepagent` | A direção evolutiva correta para esses casos é o DeepAgent. |

Regras práticas:

1. Compatibilidades históricas já removidas não são motivo central para escolher `execution.type=agent`.
2. Não trate `deepagent_memory` como sinônimo de histórico conversacional clássico.
3. Não transplante `RunnableWithMessageHistory`, wrapper agente-como-tool nem fallback implícito de filesystem ou shell para o DeepAgent.

## 1. Estrutura raiz canônica

Leitura prática da raiz canônica:

1. O bloco `memory` agrupa `checkpointer`, `agent_conversation_history`, `qa_short_history` e `qa_long_memory`.
2. `global_tools_configuration` guarda o menor nível de precedência para configuração compartilhada de tools.
3. `tools_library` é a lista de definição das tools disponíveis para assembly e runtime.
4. `selected_workflow` aponta qual workflow será materializado quando o documento estiver no modo de workflow.
5. `workflows_defaults` contém defaults reaproveitáveis entre workflows.
6. `workflows` é a coleção de workflows declarados.
7. `multi_agents` é a coleção canônica de supervisores e agentes especialistas.

## 2. Bloco `tools_library`

### 2.1 Sintaxe

Estrutura mínima da estratégia `direct`:

1. A tool entra em `tools_library` como item de lista.
2. O item precisa trazer `strategy=direct`, `id`, `description` e `category`.
3. Campos opcionais de catálogo incluem `status`, `tags`, `aliases`, `config` e `metadata`.
4. Como a resolução é direta, o campo crítico é `impl`, que aponta para o callable real.

Exemplo `factory`:

Leitura prática da estratégia `factory`:

1. O item continua em `tools_library`, mas troca `impl` por metadados de fábrica.
2. `factory_impl` aponta para a implementação que constrói a tool ou o conjunto de tools.
3. `tool_name` identifica a tool lógica exposta ao assembly.
4. `factory_function` declara qual função da fábrica será chamada.
5. `factory_returns` informa se o retorno esperado é lista, tool única ou callable.

### 2.2 Regras estruturais (obrigatórias)

1. O contrato é estrito no fluxo oficial: campos fora do contrato
   governado devem ser tratados como inválidos pelos modelos e
   validadores oficiais, sem depender de tolerância silenciosa.
2. Toda tool deve informar `strategy`.
3. `strategy=direct` exige `impl`.
4. `strategy=factory` exige `factory_impl`, `tool_name`, `factory_function` e `factory_returns`.
5. `id` deve respeitar padrão canônico de nome técnico (regex validada em AST).

### 2.3 Campos

| Campo | Tipo | Obrigatorio | Default | Semantica |
| --- | --- | --- | --- | --- |
| `strategy` | `direct \| factory` | sim | - | Estratégia de resolução da tool no runtime. |
| `id` | `str` | sim | - | Identificador único da ferramenta. |
| `description` | `str` | sim | - | Texto usado no contexto do modelo. |
| `category` | `str` | sim | - | Categoria funcional da tool. |
| `status` | `active \| disabled \| deprecated` | nao | `active` | Estado de publicação da tool. |
| `tags` | `list[str]` | nao | `[]` | Marcadores livres para busca e organização. |
| `aliases` | `list[str]` | nao | `[]` | Nomes alternativos da mesma tool. |
| `config` | `dict` | nao | `{}` | Configuração base da ferramenta. |
| `metadata` | `dict` | nao | `{}` | Metadados extras de documentação/controle. |
| `impl` | `str` | condicional | - | Obrigatório quando `strategy=direct`. |
| `factory_impl` | `str` | condicional | - | Obrigatório quando `strategy=factory`. |
| `tool_name` | `str` | condicional | - | Obrigatório quando `strategy=factory`. |
| `factory_function` | `str` | condicional | - | Obrigatório quando `strategy=factory`. |
| `factory_returns` | `list \| tool \| callable` | condicional | `list` | Obrigatório quando `strategy=factory`. |

### 2.4 Regras semânticas relevantes

As validações semânticas de `tools_library` geram diagnósticos específicos:

1. `TOOLS_LIBRARY_ID_DUPLICADO` para IDs repetidos na mesma lista.
2. `TOOLS_LIBRARY_TAG_DUPLICADA` para repetição de tags na mesma tool.
3. `TOOLS_LIBRARY_ALIAS_DUPLICADO` para aliases repetidos na mesma tool.
4. `TOOLS_LIBRARY_ALIAS_CONFLITO_ID` quando alias repete o próprio `id`.

## 3. Blocos de configuração de ferramentas

### 3.1 `global_tools_configuration`

Bloco raiz com menor precedência para overrides de tools.

Exemplo conceitual de uso:

1. Dentro de `global_tools_configuration`, uma chave como `dyn_sql` pode declarar `timeout_seconds=20`.
2. Esse valor vale como baseline até que um escopo mais interno o sobrescreva.

### 3.2 `local_tools_configuration`

Bloco por escopo (workflow, supervisor e agente) para sobrescrever `global_tools_configuration`.

Exemplo conceitual no nível de workflow:

1. Um workflow pode declarar `local_tools_configuration`.
2. Dentro desse bloco, a mesma tool `dyn_sql` pode reduzir `timeout_seconds` para `10`.
3. Isso cria um override local sem alterar o baseline global.

Exemplo conceitual no nível de supervisor e agente:

1. Um supervisor pode definir `local_tools_configuration` com `dyn_sql.timeout_seconds=8`.
2. Um agente filho desse supervisor pode usar a tool `dyn_sql<consulta_vendas>`.
3. O mesmo agente pode declarar novo override local, por exemplo `dyn_sql.timeout_seconds=5`.
4. O efeito prático é a precedência crescente do escopo mais externo para o mais interno.

### 3.3 `local_mcp_configuration`

Bloco local usado para sobrepor `global_mcp_configuration` no escopo de workflow, supervisor ou agente.

Campos mínimos garantidos pelo contrato governado:

1. `enabled`
2. `tool_name_prefix`

Comportamento real do runtime:

1. `MCPConfigResolver` compõe o bloco global com o bloco local do escopo ativo.
2. Em workflow, a composição usa `global_mcp_configuration` + `workflows[].local_mcp_configuration`.
3. Em agente, a composição usa `global_mcp_configuration` + `multi_agents[].local_mcp_configuration` + `multi_agents[].agents[].local_mcp_configuration`.
4. Para o assembly, o parser NL e o catálogo efetivo, um bloco MCP conta como presente quando `enabled` não é `false` e o escopo final traz `servers` ou `tools`.
5. Para resolver conexões MCP reais em runtime, o bloco final ainda precisa de `servers`; `tools` sozinhas servem como sinal de catálogo e guardrail, mas não abrem conexão por conta própria.

### 3.4 Precedência de tools

Ordem de merge (da menor para maior precedência):

1. `global_tools_configuration`
2. `local_tools_configuration` do escopo pai (`workflow` ou `supervisor`)
3. `local_tools_configuration` do escopo filho (`agent`)

Regras:

1. Merge profundo por chave.
2. Valores escalares do escopo mais interno sobrescrevem os anteriores.
3. Tool inexistente no catálogo falha na resolução.

## 4. Bloco `memory`

### 4.1 `memory.checkpointer`

Exemplo conceitual:

1. `memory.checkpointer.enabled=true` mantém o checkpointer ativo.
2. `memory.checkpointer.backend=sqlite` escolhe o backend local.
3. `memory.checkpointer.sqlite.path=./data/checkpointer.db` informa onde o estado será persistido.

| Campo | Tipo | Obrigatorio | Default | Regras |
| --- | --- | --- | --- | --- |
| `enabled` | `bool` | nao | `true` | Se `false`, runtime nao cria checkpointer. |
| `backend` | `str` | nao | `"postgres"` | Aceita `postgres`, `postgresql`, `mysql`, `sqlite`, `redis`, `memory`. |
| `postgres.connection_string` | `str` | condicional | - | Obrigatório quando `backend` é `postgres/postgresql`. |
| `postgresql.connection_string` | `str` | condicional | - | Alias aceito para Postgres. |
| `mysql.connection_string` | `str` | condicional | - | Obrigatório quando `backend` é `mysql`. |
| `redis.url` | `str` | condicional | - | Obrigatório quando `backend` é `redis`. |
| `redis.connection_string` | `str` | condicional | - | Alias aceito para Redis. |
| `sqlite.path` | `str` | condicional | - | Obrigatório quando `backend` é `sqlite`. |

### 4.2 `memory.agent_conversation_history`

Exemplo conceitual:

1. `memory.agent_conversation_history.enabled=true` mantém o histórico clássico ativo.
2. `backend=redis` escolhe Redis como armazenamento.
3. `redis.url=${REDIS_URL}` usa placeholder de segredo para o endpoint.
4. `redis.key_prefix=conversation_history:` define o prefixo das chaves.
5. `redis.ttl=604800` define o tempo de retenção em segundos.

| Campo | Tipo | Obrigatorio | Default |
| --- | --- | --- | --- |
| `enabled` | `bool` | nao | `true` |
| `backend` | `str` | nao | `"redis"` |
| `redis.url` | `str` | condicional | - |
| `redis.key_prefix` | `str` | nao | `"conversation_history:"` |
| `redis.ttl` | `int` | nao | `604800` |
| `sqlite.path` | `str` | condicional | - |
| `sqlite.table_name` | `str` | nao | `"conversation_history"` |

### 4.3 `memory.qa_short_history`

Exemplo conceitual:

1. `enabled=true` e `backend=redis` mantêm o histórico curto ativo em Redis.
2. `session_strategy=user_email` informa como a sessão será segregada.
3. `max_messages=10` limita a janela curta.
4. `input_messages_key=question` e `history_messages_key=chat_history` alinham o contrato com o slice de QA.
5. `redis.url=${REDIS_URL}`, `redis.key_prefix=qa_history_casa_moderna:` e `redis.ttl=604800` completam a persistência.

### 4.4 Inventário atual do histórico conversacional clássico

| Contrato | Classificação | Evidência de uso real | Decisão atual |
| --- | --- | --- | --- |
| `memory.agent_conversation_history` | uso real no modo clássico | `src/agentic_layer/supervisor/agent_builder.py` ainda lê esse bloco, monta `_history_config` e encapsula agentes com `RunnableWithMessageHistory`; há cobertura em `tests/unit/supervisor/test_agent_builder_legacy_history.py` e teste de guarda adicional no runtime clássico. | manter suportado no modo `agent`; não migrar automaticamente para DeepAgent. |
| `memory.qa_short_history` | uso real de configuração compartilhada | `src/agentic_layer/supervisor/config_resolver.py` e `src/agentic_layer/workflow/config_resolver.py` ainda resolvem esse bloco na raiz `memory`, e o slice de QA continua consumindo a configuração curta. | manter como contrato vivo; não tratar `deepagent_memory` como equivalente. |

Leitura prática:

1. Esses dois blocos não são “legado morto” hoje.
2. O fato de o DeepAgent ter `deepagent_memory` não prova equivalência funcional com histórico conversacional clássico.
3. A decisão atual do produto é preservar o suporte clássico enquanto esses contratos tiverem uso real comprovado.

### 4.5 `memory.qa_long_memory`

Exemplo conceitual:

1. `enabled=true` mantém a memória longa ativa.
2. `storage.backend=postgres` e `storage.postgres.connection_string=${USER_MEMORY_DATABASE_DSN}` definem a persistência principal.
3. A camada `conversation` pode trabalhar com `window_size=10` e `session_ttl_minutes=60`.
4. A camada `session_history` pode limitar `max_sessions=50` e `session_ttl_days=30`.
5. A camada `contextual` pode usar `max_context_items=10`, `context_window_hours=72` e `similarity_threshold=0.65`.

## 5. Blocos reutilizáveis de execução

### 5.1 `retry_policy`

Exemplo conceitual:

1. `max_attempts=3` define o teto de tentativas.
2. `backoff_seconds=0.5` aplica espera incremental entre tentativas.
3. `breaker_threshold=3` pode encerrar cedo quando o limite de falhas for atingido.

| Campo | Tipo | Default | Regras |
| --- | --- | --- | --- |
| `max_attempts` | `int` | `1` | `>= 1` |
| `backoff_seconds` | `float` | `0.0` | `>= 0` |
| `breaker_threshold` | `int` | `max_attempts` | `>= 1`; encerra tentativas cedo quando alcançado |

### 5.2 `human_approval`

Exemplo conceitual:

1. `enabled=true` ativa o gate humano.
2. `decision_path=metadata.approval.decision` diz onde o runtime procura a decisão.
3. `reason=Aprovacao obrigatoria` explica por que a pausa existe.
4. `approved_values` pode aceitar valores como `aprovar`, `ok`, `sim` e `approve`.
5. `rejected_values` pode aceitar `rejeitar`, `nao`, `não`, `reject` e `stop`.
6. `payload_paths` pode expor `variables.pedido` e `last_output` como contexto para quem aprova.

Regras:

1. Se `enabled=false`, execução segue sem gate humano.
2. Se `decision_path` não contém valor aprovado/rejeitado, node retorna `status=paused` com `metadata.requires_human=true`.
3. Valores de aprovação/rejeição são comparados em lowercase.

### 5.3 `failure_policy`

Exemplo conceitual:

1. `mode=request_human` desvia a falha para decisão humana.
2. `human_message=Falha tecnica. Favor decidir proximo passo.` define a mensagem operacional exibida no gate.

| Campo | Tipo | Default | Valores aceitos |
| --- | --- | --- | --- |
| `mode` | `str` | `"abort"` | `abort`, `continue`, `request_human` |
| `human_message` | `str` | `""` | Texto opcional para modo `request_human` |

## 6. DSL de expressões seguras

A DSL de expressões é usada em `if`, `rule_router.when`, `set.assign.expr`, `function.expression` e `edges[].when`.

Escopo disponível:

1. `vars` / `variables`
2. `metadata`
3. `input_text`
4. `last_output`
5. `context`

Funções permitidas (`SAFE_FUNCTIONS`):

1. `abs`, `all`, `any`, `bool`, `dict`, `float`, `int`, `len`, `list`, `max`, `min`, `round`, `set`, `sorted`, `str`, `sum`, `tuple`, `json_loads`

Operadores suportados:

1. Aritméticos: `+`, `-`, `*`, `/`, `//`, `%`, `**`
2. Comparação: `==`, `!=`, `>`, `>=`, `<`, `<=`, `in`, `not in`
3. Booleanos: `and`, `or`, `not`
4. Acesso por atributo e índice: `vars.pedido.total`, `vars.itens[0]`

Restrições:

1. Chamadas de função fora da whitelist são bloqueadas.
2. Slicing (`[1:3]`) não é permitido.
3. Expressões booleanas devem retornar `bool` explícito.

## 7. Caminhos de dados (`reads` e `writes`)

Formato de path aceito:

1. `variables.campo`
2. `vars.campo.subcampo`
3. `metadata.chave`
4. `last_output`
5. `variables.lista[0].nome`

Leitura prática do exemplo:

1. `reads` pode incluir caminhos como `variables.pedido.total` e `metadata.plan_cursor`.
2. `writes` pode registrar saídas em caminhos como `variables.resultados.validacao`.

## 8. Templates de prompt e valores

Suporte a placeholders em `prompt.system` e valores de `set/tool/function`:

1. Placeholders permitidos no prompt: `{input_text}`, `{last_output}`, `{context}`, `{metadata}`, `{variables}`, `{vars}`
2. Em `set.assign` e payloads de tool:
   - `{"expr": "..."}` para expressão segura
   - `{"from": "vars.caminho"}` para copiar caminho

Leitura prática do exemplo:

1. `params.assign.resumo` pode interpolar o identificador do pedido com placeholder textual.
2. `params.assign.total` pode usar `expr` para somar `variables.pedido.subtotal` e `variables.pedido.frete`.
3. `params.assign.cliente` pode usar `from` para copiar `variables.pedido.cliente`.

## 9. Placeholders de segredo (`${VAR}`)

Campos críticos aceitam placeholders `${NOME}` com resolução via `security_keys` e variáveis de ambiente.

Leitura prática do exemplo:

1. `memory.checkpointer.backend=postgres` escolhe Postgres para persistência.
2. `memory.checkpointer.postgres.connection_string=${CHECKPOINTER_DSN}` delega a credencial para placeholder de segredo.

## 10. Checklist rápido de validade

1. Use `multi_agents` como lista canônica de supervisores.
2. Sempre mantenha IDs únicos (`workflow`, `supervisor`, `agent`, `node`, `tool`).
3. Em blocos de retry/failure/human approval, use apenas campos suportados.
4. Não referencie tools fora de `tools_library` efetivo.
5. Mantenha exemplos YAML parseáveis e com tipos corretos.

## 11. Evidência no código

1. `src/config/agentic_assembly/ast/document.py`, `workflow.py`, `supervisor.py` e `deepagent.py`, que definem os blocos canônicos do contrato agentic.
2. `src/config/agentic_assembly/parsers/`, que transforma YAML em AST tipada.
3. `src/config/agentic_assembly/validators/`, que aplica validações estruturais e semânticas.
4. `src/config/agentic_assembly/drift_detector.py`, que prova o papel de `selected_workflow`, `selected_supervisor` e dos hashes governados.
5. `src/agentic_layer/workflow/config_resolver.py` e `src/agentic_layer/supervisor/config_resolver.py`, que materializam o contrato no runtime.

## 12. Lacunas no código

- Não encontrado no código: um gerador único de inventário humano de todos os campos do contrato agentic consumindo diretamente AST e resolvers em tempo real.
  Onde deveria estar: `src/config/agentic_assembly/` ou pipeline de docs sincronizadas.
