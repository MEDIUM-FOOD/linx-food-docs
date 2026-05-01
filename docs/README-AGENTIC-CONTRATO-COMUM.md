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
|---|---|---|
| Precisa de `memory.agent_conversation_history` ou `memory.qa_short_history` como contrato funcional | `execution.type=agent` | Esses contratos continuam vivos no modo clássico e não têm equivalência automática no DeepAgent. |
| Precisa de `interrupt_on`, `permissions`, `middlewares.filesystem`, `deepagent_memory` ou `async_subagents[]` | `execution.type=deepagent` | Esses recursos pertencem ao runtime DeepAgent e são onde ele entrega valor real. |
| Fluxo simples, sem governança local e sem memória persistente por identidade | `execution.type=agent` | Menor complexidade operacional. |
| Recurso novo de profundidade, delegação rica ou memória persistente de longo prazo | `execution.type=deepagent` | A direção evolutiva correta para esses casos é o DeepAgent. |

Regras práticas:
1. Compatibilidades históricas já removidas não são motivo central para escolher `execution.type=agent`.
2. Não trate `deepagent_memory` como sinônimo de histórico conversacional clássico.
3. Não transplante `RunnableWithMessageHistory`, wrapper agente-como-tool nem fallback implícito de filesystem ou shell para o DeepAgent.

## 1. Estrutura raiz canônica

```yaml
memory:
  checkpointer: {}
  agent_conversation_history: {}
  qa_short_history: {}
  qa_long_memory: {}

global_tools_configuration: {}
tools_library: []

selected_workflow: "meu_workflow"
workflows_defaults: {}
workflows: []

multi_agents: []
```

## 2. Bloco `tools_library`

### 2.1 Sintaxe

```yaml
tools_library:
  - strategy: "direct"
    id: "tool_id"
    description: "Descricao para o modelo"
    category: "utility_tools"
    status: "active"
    tags: []
    aliases: []
    config: {}
    metadata: {}
    impl: "package.module.callable"
```

Exemplo `factory`:

```yaml
tools_library:
  - strategy: "factory"
    id: "qa_rag"
    description: "QA com contexto vetorial"
    category: "vector_store_tools"
    status: "active"
    tags: ["qa"]
    aliases: []
    config: {}
    metadata: {}
    factory_impl: "src.agentic_layer.tools.vector_store_tools.vectorstore_toolkit.create_vectorstore_toolkit_tools"
    tool_name: "qa_rag"
    factory_function: "create_vectorstore_toolkit_tools"
    factory_returns: "list"
```

### 2.2 Regras estruturais (obrigatórias)

1. O contrato é estrito: campos fora do schema são rejeitados.
2. Toda tool deve informar `strategy`.
3. `strategy=direct` exige `impl`.
4. `strategy=factory` exige `factory_impl`, `tool_name`, `factory_function` e `factory_returns`.
5. `id` deve respeitar padrão canônico de nome técnico (regex validada em AST).

### 2.3 Campos

| Campo | Tipo | Obrigatorio | Default | Semantica |
|---|---|---|---|---|
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

```yaml
global_tools_configuration:
  dyn_sql:
    timeout_seconds: 20
```

### 3.2 `local_tools_configuration`

Bloco por escopo (workflow, supervisor e agente) para sobrescrever `global_tools_configuration`.

```yaml
workflows:
  - id: "wf_exemplo"
    local_tools_configuration:
      dyn_sql:
        timeout_seconds: 10
```

```yaml
multi_agents:
  - id: "sup_exemplo"
    local_tools_configuration:
      dyn_sql:
        timeout_seconds: 8
    agents:
      - id: "agente_sql"
        description: "Consulta dados"
        tools: ["dyn_sql<consulta_vendas>"]
        local_tools_configuration:
          dyn_sql:
            timeout_seconds: 5
```

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

```yaml
memory:
  checkpointer:
    enabled: true
    backend: "sqlite"
    sqlite:
      path: "./data/checkpointer.db"
```

| Campo | Tipo | Obrigatorio | Default | Regras |
|---|---|---|---|---|
| `enabled` | `bool` | nao | `true` | Se `false`, runtime nao cria checkpointer. |
| `backend` | `str` | nao | `"postgres"` | Aceita `postgres`, `postgresql`, `mysql`, `sqlite`, `redis`, `memory`. |
| `postgres.connection_string` | `str` | condicional | - | Obrigatório quando `backend` é `postgres/postgresql`. |
| `postgresql.connection_string` | `str` | condicional | - | Alias aceito para Postgres. |
| `mysql.connection_string` | `str` | condicional | - | Obrigatório quando `backend` é `mysql`. |
| `redis.url` | `str` | condicional | - | Obrigatório quando `backend` é `redis`. |
| `redis.connection_string` | `str` | condicional | - | Alias aceito para Redis. |
| `sqlite.path` | `str` | condicional | - | Obrigatório quando `backend` é `sqlite`. |

### 4.2 `memory.agent_conversation_history`

```yaml
memory:
  agent_conversation_history:
    enabled: true
    backend: "redis"
    redis:
      url: "${REDIS_URL}"
      key_prefix: "conversation_history:"
      ttl: 604800
```

| Campo | Tipo | Obrigatorio | Default |
|---|---|---|---|
| `enabled` | `bool` | nao | `true` |
| `backend` | `str` | nao | `"redis"` |
| `redis.url` | `str` | condicional | - |
| `redis.key_prefix` | `str` | nao | `"conversation_history:"` |
| `redis.ttl` | `int` | nao | `604800` |
| `sqlite.path` | `str` | condicional | - |
| `sqlite.table_name` | `str` | nao | `"conversation_history"` |

### 4.3 `memory.qa_short_history`

```yaml
memory:
  qa_short_history:
    enabled: true
    backend: "redis"
    session_strategy: "user_email"
    max_messages: 10
    input_messages_key: "question"
    history_messages_key: "chat_history"
    redis:
      url: "${REDIS_URL}"
      key_prefix: "qa_history_casa_moderna:"
      ttl: 604800
```

    ### 4.4 Inventário atual do histórico conversacional clássico

    | Contrato | Classificação | Evidência de uso real | Decisão atual |
    |---|---|---|---|
    | `memory.agent_conversation_history` | uso real no modo clássico | `src/agentic_layer/supervisor/agent_builder.py` ainda lê esse bloco, monta `_history_config` e encapsula agentes com `RunnableWithMessageHistory`; há cobertura em `tests/unit/supervisor/test_agent_builder_legacy_history.py` e teste de guarda adicional no runtime clássico. | manter suportado no modo `agent`; não migrar automaticamente para DeepAgent. |
    | `memory.qa_short_history` | uso real de configuração compartilhada | `src/agentic_layer/supervisor/config_resolver.py` e `src/agentic_layer/workflow/config_resolver.py` ainda resolvem esse bloco na raiz `memory`, e o slice de QA continua consumindo a configuração curta. | manter como contrato vivo; não tratar `deepagent_memory` como equivalente. |

    Leitura prática:
    1. Esses dois blocos não são “legado morto” hoje.
    2. O fato de o DeepAgent ter `deepagent_memory` não prova equivalência funcional com histórico conversacional clássico.
    3. A decisão atual do produto é preservar o suporte clássico enquanto esses contratos tiverem uso real comprovado.

### 4.4 `memory.qa_long_memory`

```yaml
memory:
  qa_long_memory:
    enabled: true
    storage:
      backend: "postgres"
      postgres:
        connection_string: "${USER_MEMORY_DATABASE_DSN}"
    layers:
      conversation:
        window_size: 10
        session_ttl_minutes: 60
      session_history:
        max_sessions: 50
        session_ttl_days: 30
      contextual:
        max_context_items: 10
        context_window_hours: 72
        similarity_threshold: 0.65
```

## 5. Blocos reutilizáveis de execução

### 5.1 `retry_policy`

```yaml
retry_policy:
  max_attempts: 3
  backoff_seconds: 0.5
  breaker_threshold: 3
```

| Campo | Tipo | Default | Regras |
|---|---|---|---|
| `max_attempts` | `int` | `1` | `>= 1` |
| `backoff_seconds` | `float` | `0.0` | `>= 0` |
| `breaker_threshold` | `int` | `max_attempts` | `>= 1`; encerra tentativas cedo quando alcançado |

### 5.2 `human_approval`

```yaml
human_approval:
  enabled: true
  decision_path: "metadata.approval.decision"
  reason: "Aprovacao obrigatoria"
  approved_values: ["aprovar", "ok", "sim", "approve"]
  rejected_values: ["rejeitar", "nao", "não", "reject", "stop"]
  payload_paths:
    - "variables.pedido"
    - "last_output"
```

Regras:
1. Se `enabled=false`, execução segue sem gate humano.
2. Se `decision_path` não contém valor aprovado/rejeitado, node retorna `status=paused` com `metadata.requires_human=true`.
3. Valores de aprovação/rejeição são comparados em lowercase.

### 5.3 `failure_policy`

```yaml
failure_policy:
  mode: "request_human"
  human_message: "Falha tecnica. Favor decidir proximo passo."
```

| Campo | Tipo | Default | Valores aceitos |
|---|---|---|---|
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

Exemplo:

```yaml
reads:
  - "variables.pedido.total"
  - "metadata.plan_cursor"
writes:
  - "variables.resultados.validacao"
```

## 8. Templates de prompt e valores

Suporte a placeholders em `prompt.system` e valores de `set/tool/function`:
1. Placeholders permitidos no prompt: `{input_text}`, `{last_output}`, `{context}`, `{metadata}`, `{variables}`, `{vars}`
2. Em `set.assign` e payloads de tool:
   - `{"expr": "..."}` para expressão segura
   - `{"from": "vars.caminho"}` para copiar caminho

```yaml
params:
  assign:
    resumo: "Pedido {variables.pedido.id}"
    total:
      expr: "variables.pedido.subtotal + variables.pedido.frete"
    cliente:
      from: "variables.pedido.cliente"
```

## 9. Placeholders de segredo (`${VAR}`)

Campos críticos aceitam placeholders `${NOME}` com resolução via `security_keys` e variáveis de ambiente.

Exemplo:

```yaml
memory:
  checkpointer:
    backend: "postgres"
    postgres:
      connection_string: "${CHECKPOINTER_DSN}"
```

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
