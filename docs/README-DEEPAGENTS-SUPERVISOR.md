# Manual Técnico: DeepAgent Supervisor

Este manual descreve o contrato real do modo `deepagent` no produto atual.

Regra principal:

1. O contrato público do DeepAgent agora usa `multi_agents[].middlewares` como fonte de verdade para ligar ou desligar middlewares.
2. `capabilities` saiu do contrato governado. Se ainda aparecer em YAML ou documentação antiga, trate como legado inválido.
3. O runtime do produto não depende mais dos defaults escondidos da lib `deepagents`. A montagem é feita por uma factory governada localmente no próprio projeto.

## 1. Escopo

Pense no supervisor clássico como um coordenador que conversa com tools e agentes.

Pense no DeepAgent como esse mesmo coordenador com mais governança:

1. Ele pode manter memória persistente entre execuções.
2. Ele pode ligar especialistas síncronos e assíncronos.
3. Ele pode pausar uma tool para revisão humana.
4. Ele pode limitar acesso a arquivo e shell de forma declarativa.
5. Ele pode aplicar proteção de PII, sumarização, skills e lista de tarefas pelo stack de middlewares.

Na prática, isso significa menos comportamento escondido e mais controle declarativo no YAML.

## Leitura relacionada

- Supervisor clássico: [README-AGENTE-SUPERVISOR.md](./README-AGENTE-SUPERVISOR.md)
- Workflow determinístico: [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md)
- Contrato comum do YAML agentic: [README-AGENTIC-CONTRATO-COMUM.md](./README-AGENTIC-CONTRATO-COMUM.md)
- Configuração YAML da plataforma: [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md)
- HIL no produto atual: [README-HUMAN-IN-THE-LOOP.md](./README-HUMAN-IN-THE-LOOP.md)
- Versão didática 101 deste assunto: [tutorial-101-deepagents.md](./tutorial-101-deepagents.md)

## 2. Ativação do modo deepagent

O bloco que governa o runtime é este:

```yaml
multi_agents:
  - id: "sup_deep"
    enabled: true
    execution:
      type: "deepagent"
      default_mode: "direct_async"
    middlewares:
      filesystem:
        enabled: true
      shell:
        enabled: false
      memory:
        enabled: true
        sources: []
      subagents:
        enabled: true
      human_in_the_loop:
        enabled: false
      summarization:
        enabled: false
      pii:
        enabled: true
        rules: []
      todo_list:
        enabled: true
      skills:
        enabled: false
```

Leitura prática:

1. Esse bloco é o interruptor oficial do DeepAgent.
2. O produto registra em log o estado `on` ou `off` de cada middleware na criação do agente.
3. O runtime aplica a mesma governança também aos subagentes síncronos.
4. O YAML canônico e os YAMLs reais do repositório já foram migrados para esse formato.

## 3. Matriz default oficial

Esta é a matriz default governada do produto hoje.

| Middleware | Default oficial | O que faz na prática | Pré-requisito importante |
| --- | --- | --- | --- |
| `filesystem` | `on` | Expõe tools oficiais de arquivo com policy declarativa. | Exige `permissions[]` explícitas no mesmo supervisor. |
| `shell` | `off` | Expõe a tool `execute` com policy de execução governada. | Exige backend DeepAgent e `execution_policy` válido quando ligado. |
| `memory` | `on` | Injeta memória do DeepAgent no runtime quando há fontes declaradas. | Exige `deepagent_memory.enabled=true`. |
| `subagents` | `on` | Liga subagentes síncronos e assíncronos dentro da conversa. | Exige backend DeepAgent. |
| `human_in_the_loop` | `off` | Pausa tools para aprovação, edição ou rejeição humana. | Exige `interrupt_on` e uso com checkpointer ativo. |
| `summarization` | `off` | Resume contexto para reduzir volume e manter a conversa útil. | Exige backend DeepAgent. |
| `pii` | `on` | Protege dados sensíveis na entrada, saída e resultados de tools. | Pode usar regras default ou regras declaradas. |
| `todo_list` | `on` | Mantém plano de tarefas do agente dentro do runtime. | Não exige campos extras. |
| `skills` | `off` | Liga skills declarativas no agente principal. | Exige `skills` top-level com pelo menos um caminho. |

Importante:

1. Embora existam defaults internos, o padrão recomendado do produto é declarar o bloco `middlewares` completo no YAML.
2. Isso evita ambiguidade operacional e evita depender de leitura implícita.

## 4. Contrato de `deepagent_memory`

`deepagent_memory` governa a persistencia de memoria do DeepAgent. Em linguagem simples: esse bloco diz onde a memoria duravel fica guardada, por quanto tempo e com qual escopo logico.

Quando ligado, o backend suportado é `backend: "redis"`. O caminho pratico da URL é `redis.url`, com `key_prefix` para separar namespaces e `ttl_seconds` para definir a vida util dos registros.

Os campos mais importantes do mesmo item de `multi_agents[]` são:

| Campo | Obrigatório | Papel prático |
| --- | --- | --- |
| `id` | sim | Identifica o supervisor DeepAgent. |
| `enabled` | sim | Permite seleção real do supervisor. |
| `execution.type=deepagent` | sim | Ativa o runtime DeepAgent. |
| `execution.default_mode` | não | Define o modo padrão do endpoint quando o request não informa `execution_mode`. |
| `middlewares` | sim por governança recomendada | Liga e desliga os middlewares reais do runtime. |
| `permissions[]` | condicional | Policy de filesystem. Só faz sentido quando `middlewares.filesystem.enabled=true`. |
| `deepagent_memory` | condicional | Configura store persistente Redis e namespace lógico da memória. |
| `interrupt_on` | condicional | Define pausa humana por tool e a política de decisão de cada uma. |
| `skills` | condicional | Skills top-level do agente principal, quando `middlewares.skills.enabled=true`. |
| `response_format` | não | Structured output do agente principal. |
| `agents[]` | sim | Subagentes síncronos do DeepAgent. |
| `async_subagents[]` | não | Especialistas remotos de longa duração. |

## 5. Contrato de `agents[]` (subagentes)

`agents[]` declara os especialistas sincronizados dentro do mesmo supervisor DeepAgent. Cada item precisa ter identidade clara, descrição operacional e lista de tools quando o agente puder chamar ferramentas.

Na prática, o subagente herda a governança do supervisor e ainda pode declarar ajustes próprios, como `model`, `skills`, `response_format`, `interrupt_on` e `permissions`, quando esses campos forem necessários para aquele especialista.

## 6. Middleware Chain

A cadeia governada de middlewares é montada pelo runtime local do projeto. Os nomes relevantes que aparecem no código e nos logs são: `ToolCallLimitMiddleware`, `ModelCallLimitMiddleware`, `LLMToolSelectorMiddleware`, `ToolRetryMiddleware`, `ModelRetryMiddleware`, `ContextEditingMiddleware`, `ClearToolUsesEdit`, `FilesystemMiddleware`, `ShellToolMiddleware`, `ToolSelectionAuditMiddleware`, `ToolExecutionMiddleware`, `ResponsePostProcessingMiddleware` e `ErrorHandlingMiddleware`.

Em termos simples: essa cadeia é a fila de proteções aplicada antes, durante e depois da execução do agente. Ela controla limites, retry, seleção/execução de tools, edição de contexto, filesystem, shell, auditoria, pós-processamento e tratamento de erro.

## 6.1 Explicação 101 de cada middleware

### 5.1 Filesystem

Em linguagem simples:

1. É o middleware que deixa o agente ler ou escrever arquivos locais de forma controlada.
2. Ele não libera acesso total ao disco.
3. Ele só trabalha junto com `permissions[]`.

Exemplo:

```yaml
middlewares:
  filesystem:
    enabled: true
permissions:
  - operations: ["read"]
    paths: ["/memories/**"]
    mode: "allow"
  - operations: ["write"]
    paths: ["/memories/**"]
    mode: "deny"
```

O que validar:

1. `permissions[]` precisa existir no mesmo supervisor.
2. Cada regra precisa declarar `operations`, `paths` e `mode`.
3. Caminhos precisam ser absolutos e sem `..`.

### 5.2 Shell

Em linguagem simples:

1. É o middleware que expõe a tool `execute` para comandos de shell.
2. Ele é separado do filesystem. Arquivo e shell não são a mesma coisa.
3. O runtime exclui `execute` do filesystem e deixa esse comando só no middleware de shell.

Exemplo:

```yaml
middlewares:
  shell:
    enabled: true
    workspace_root: "/workspace"
    tool_name: "execute"
    startup_commands: ["pwd"]
    shutdown_commands: []
    env:
      APP_ENV: "dev"
    execution_policy:
      type: "host"
```

Tipos aceitos em `execution_policy.type`:

1. `host`
2. `docker`
3. `codex_sandbox`

O que validar:

1. Se ligar shell, o runtime precisa de backend DeepAgent válido.
2. A tool criada continua sendo `execute`, salvo se `tool_name` for alterado.
3. O shell policy aparece no log de inicialização do agente.

### 5.3 Memory

Em linguagem simples:

1. Esse middleware controla como a memória entra no runtime do agente.
2. Ele não é a mesma coisa que o store Redis.
3. `deepagent_memory` fala de persistência. `middlewares.memory` fala do middleware que usa essa memória.

Exemplo:

```yaml
middlewares:
  memory:
    enabled: true
    sources:
      - "/memories/shared.md"
deepagent_memory:
  enabled: true
  backend: "redis"
  scope: "user"
  policy: "read_write"
  redis:
    url: "{$DEEPAGENT_REDIS_URL}"
```

O que validar:

1. `middlewares.memory.enabled=true` exige `deepagent_memory.enabled=true`.
2. Se você desligar `middlewares.memory`, desligue `deepagent_memory` no mesmo YAML.
3. `sources` é opcional. Sem fontes, o middleware fica ligado mas não recebe lista adicional.

### 5.4 Subagents

Em linguagem simples:

1. Esse middleware liga especialistas dentro da conversa.
2. Ele vale tanto para subagentes síncronos quanto para `async_subagents[]`.
3. O produto também aplica o contrato governado aos subagentes síncronos, não só ao supervisor principal.

Exemplo:

```yaml
middlewares:
  subagents:
    enabled: true
agents:
  - id: "analista"
    description: "Especialista em análise"
    tools: ["json_parse"]
async_subagents:
  - name: "pesquisador_remoto"
    description: "Pesquisa longa"
    graph_id: "graph-research"
    url: "https://agent.example.com"
```

O que validar:

1. `async_subagents[].headers` sem `url` é erro.
2. Subagente síncrono pode ter `model`, `skills`, `response_format`, `interrupt_on` e `permissions` próprios.

### 5.5 Human in the loop

Em linguagem simples:

1. Esse é o jeito padrão de pedir revisão humana no DeepAgent atual.
2. Você não precisa criar uma tool HIL separada para começar.
3. No fluxo normal, basta ligar `middlewares.human_in_the_loop`, declarar `interrupt_on` e executar o agente com checkpointer ativo.
4. O middleware intercepta a chamada da tool antes da execução real e devolve um pedido de revisão humana.
5. Depois da decisão, a mesma thread é retomada e o fluxo continua do ponto onde parou.

Pergunta prática mais importante:

1. Para primeira adoção de DeepAgent, não use `human_gate`, `create_human_gate_tool` nem uma tool manual de aprovação.
2. Esses recursos continuam existindo no produto para workflows e cenários customizados.
3. Para aprovar tools do DeepAgent, o caminho oficial é o middleware `HumanInTheLoopMiddleware`, montado pelo próprio supervisor.

Pré-requisitos operacionais:

1. `middlewares.human_in_the_loop.enabled=true` exige `interrupt_on`.
2. Siga a recomendação do fabricante e mantenha checkpointer ativo; no produto isso vem da seção raiz `memory.checkpointer`.
3. A retomada precisa reutilizar o mesmo `thread_id` da pausa.
4. As tools listadas em `interrupt_on` precisam existir no escopo real do supervisor ou do subagente.

Fluxo real, sem jargão:

1. O agente roda normalmente até propor uma chamada de tool marcada em `interrupt_on`.
2. O middleware monta uma lista `action_requests` com o nome da tool, os argumentos e a descrição de revisão.
3. A interface ou a API mostra isso para um humano decidir.
4. O humano responde com uma decisão por item, na mesma ordem recebida.
5. O runtime retoma a mesma thread e executa a tool aprovada, a tool editada ou rejeita a tool sem executá-la.

Decisões disponíveis:

1. `approve`: executa a tool exatamente com os argumentos propostos.
2. `edit`: altera os argumentos antes da execução.
3. `reject`: não executa a tool e registra a rejeição no fluxo.

Contrato HTTP público:

1. `/agent/execute` publica `thread_id` e `hil` quando a execução pausa.
2. `thread_id` é opaco para o cliente; o cliente guarda e reenvia o valor, sem recomputar.
3. O mesmo bloco `hil` atende uma tela simples com aprovar/rejeitar e uma tela rica com `action_requests`, `review_configs`, `edit` e múltiplas decisões.
4. `/agent/continue` valida a pausa no servidor antes de retomar, cruzando `correlation_id`, `thread_id`, usuário, supervisor e quantidade de ações pendentes.
5. Pausa HIL em execução assíncrona é estado `paused`, não conclusão da task.

Esse bloco `hil` também é a base da Generative UI de revisão humana. A
interface pode gerar a tela de aprovação a partir do contrato estruturado,
sem parsing de texto e sem regra fixa por tool. O backend continua
decidindo quais ações estão pendentes e quais decisões são aceitas; a UI
apenas materializa essa pendência com componentes seguros do produto.

Exemplo:

```yaml
middlewares:
  human_in_the_loop:
    enabled: true
    description_prefix: "A ferramenta precisa de aprovação"
interrupt_on:
  enviar_email:
    allowed_decisions: ["approve", "edit", "reject"]
```

Esse recorte já basta para começar no DeepAgent.

O que ele ainda precisa fora desse trecho:

1. Um checkpointer ativo na raiz do YAML.
2. Um `thread_id` publicado pelo backend e reaproveitado sem alteração na retomada.
3. Uma superfície de revisão humana que leia `action_requests` e envie as decisões de volta.

O que validar:

1. `middlewares.human_in_the_loop.enabled=true` exige `interrupt_on`.
2. `interrupt_on` sem HIL ligado é erro.
3. As tools listadas em `interrupt_on` precisam existir no escopo real do agente.
4. Para operação estável, trate checkpointer como obrigatório sempre que houver pausa e retomada humana.
5. A retomada precisa usar o mesmo `thread_id` da pausa.

### 5.5.1 Aprovação assíncrona por canais

O bloco `middlewares.human_in_the_loop.async_approval` define o contrato
de aprovação humana para execuções em background, quando não existe uma
tela aberta esperando a resposta. Ele não substitui `interrupt_on`: o
`interrupt_on` continua dizendo qual tool deve parar, e o
`async_approval` diz como a plataforma pode pedir a decisão por canais
como WhatsApp ou e-mail.

Campos principais:

1. `enabled`: liga ou desliga a aprovação assíncrona.
2. `ttl_seconds`: tempo de validade do pedido, entre 60 segundos e 7 dias.
3. `expiration_policy`: comportamento quando o prazo vence; valores
  aceitos são `expire` e `fail_run`.
4. `require_approver_match`: exige que a resposta venha de aprovador
  conhecido.
5. `channels`: lista de canais habilitados. Cada canal precisa declarar
  `type` (`whatsapp` ou `email`) e `template_id` quando estiver ativo.
6. `approvers`: lista de pessoas autorizadas. Cada aprovador precisa ter
  `user_email` ou `user_code`. Para WhatsApp, informe também
  `channel_user_ids.whatsapp`.

Regras obrigatórias:

1. `async_approval.enabled=true` exige
  `middlewares.human_in_the_loop.enabled=true`.
2. Se `async_approval.enabled=true`, deve existir ao menos um canal
  habilitado e ao menos um aprovador.
3. Canal `email` exige ao menos um aprovador com `user_email`.
4. Canal `whatsapp` exige ao menos um aprovador com
  `channel_user_ids.whatsapp`.
5. Canais não podem ser acionados por inferência. Sem `template_id`, o
  contrato falha em vez de cair em mensagem padrão escondida.

Em linguagem simples: esse bloco transforma a pausa HIL em um pedido
auditável que pode sair por canal assíncrono. O agente continua parado;
a plataforma registra quem pode decidir, por quanto tempo o pedido vale e
por qual canal a pessoa será avisada.

Como o runtime usa essa configuração:

1. O DeepAgent pausa quando uma tool listada em `interrupt_on` é chamada.
2. O runtime cria um pedido em `public.agent_hil_approval_requests`.
3. A plataforma envia a notificação pelos canais habilitados.
4. O pedido fica pendente até uma decisão válida chegar ou até vencer o
  prazo em `ttl_seconds`.
5. Se a decisão for válida, a mesma thread é retomada com
  `Command(resume=...)`.
6. Se o pedido vencer, a tool sensível não é executada.

Configuração operacional necessária:

1. `AGENT_HIL_APPROVAL_DSN` precisa apontar para o PostgreSQL que contém
  a tabela de aprovações.
2. `AGENT_HIL_APPROVAL_SCHEMA` e `AGENT_HIL_APPROVAL_TABLE` podem ser
  usados quando a tabela não estiver no caminho padrão
  `public.agent_hil_approval_requests`.
3. `AGENT_HIL_APPROVAL_MAINTENANCE_ENABLED` ativa o job que fecha pedidos
  expirados.
4. `AGENT_HIL_APPROVAL_MAINTENANCE_INTERVAL_SECONDS` controla o intervalo
  do job.
5. `AGENT_HIL_APPROVAL_MAINTENANCE_LIMIT_PER_RUN` limita quantos pedidos
  vencidos são fechados por rodada.

Regras de segurança:

1. O token bruto só aparece no canal de aprovação. O banco guarda o hash.
2. O log estruturado não deve registrar o token bruto.
3. `require_approver_match=true` exige que a resposta venha de uma pessoa
  configurada em `approvers`.
4. Uma decisão aceita resolve o pedido de forma atômica. Outra resposta
  posterior não retoma a execução de novo.
5. Pedidos expirados são recusados mesmo que o job periódico ainda não
  tenha fechado a linha no banco.

Eventos de log esperados:

1. `hil.pause.created`, quando o pedido durável nasce.
2. `hil.notification.dispatch.started`, quando o envio começa.
3. `hil.notification.dispatch.finished`, quando o envio termina.
4. `hil.decision.received`, quando uma decisão chega.
5. `hil.decision.accepted`, quando uma decisão é persistida.
6. `hil.decision.rejected`, quando a decisão é recusada.
7. `hil.continuation.started` e `hil.continuation.finished`, quando a
  execução original é retomada.
8. `hil.pause.expired`, quando o job fecha um pedido vencido.

Limites atuais:

1. O primeiro corte executável é DeepAgent em background.
2. `async_approval` não adiciona suporte automático ao Workflow nem ao
  AgentSupervisor clássico.
3. Reminder e escalation não são executados hoje porque ainda não existem
  campos YAML governados para isso.
4. Canais sem `template_id` falham; não existe mensagem padrão escondida.
5. Edição por canal assíncrono exige payload estruturado e interface
  própria. Os botões enviados por canal usam aprovação ou rejeição.

### 5.6 Summarization

Em linguagem simples:

1. Esse middleware resume partes antigas da conversa para o contexto não crescer sem controle.
2. Ele ajuda a manter custo e volume sob governança.
3. Ele não é ligado por padrão porque nem todo fluxo precisa resumir.

Exemplo:

```yaml
middlewares:
  summarization:
    enabled: true
    trigger:
      messages_since_last_summary: 8
    keep:
      recent_messages: 6
    summary_prompt: "Resuma decisões, fatos e pendências em português claro."
```

O que validar:

1. Esse middleware exige backend DeepAgent.
2. Use quando a conversa for longa ou iterativa.
3. Não ligue só por hábito; ligue quando houver ganho operacional real.

### 5.7 PII

Em linguagem simples:

1. PII significa dado sensível, como e-mail, cartão, IP, MAC address e URL.
2. Esse middleware protege esses dados antes que eles saiam circulando livremente.
3. Ele já tem regras default quando você não declara `rules`.

Exemplo com regras próprias:

```yaml
middlewares:
  pii:
    enabled: true
    rules:
      - pii_type: "email"
        strategy: "redact"
        apply_to_input: true
        apply_to_output: true
      - pii_type: "credit_card"
        strategy: "mask"
        apply_to_tool_results: true
```

Estratégias aceitas:

1. `block`
2. `redact`
3. `mask`
4. `hash`

### 5.8 Todo list

Em linguagem simples:

1. Esse middleware ajuda o agente a manter uma lista explícita de tarefas durante a execução.
2. Ele é útil em investigação longa, decomposição e acompanhamento de progresso.
3. Ele já nasce ligado por padrão porque é um recurso operacional útil e de baixo risco.

Exemplo:

```yaml
middlewares:
  todo_list:
    enabled: true
    system_prompt: "Sempre mantenha o plano atualizado antes de concluir."
    tool_description: "Atualiza a lista de tarefas do DeepAgent."
```

### 5.9 Skills

Em linguagem simples:

1. Esse middleware liga skills declarativas no agente principal do DeepAgent.
2. Ele não olha automaticamente para `skills` só porque o campo existe.
3. O interruptor de `middlewares.skills.enabled` precisa estar ligado.

Exemplo:

```yaml
middlewares:
  skills:
    enabled: true
skills:
  - "/skills/main/"
  - "/skills/qa/"
```

O que validar:

1. `middlewares.skills.enabled=true` exige lista top-level `skills` com ao menos um caminho.
2. `skills` top-level sem esse middleware ligado é erro.

## 7. Limites e Retries

Limites evitam execuções sem freio. Retries repetem uma tentativa quando a falha parece transitória, como timeout ou instabilidade temporária de uma chamada externa.

No runtime atual, os limites usam `ToolCallLimitMiddleware` e `ModelCallLimitMiddleware`. Os retries usam `ToolRetryMiddleware` e `ModelRetryMiddleware`. A seleção de tool por LLM passa por `LLMToolSelectorMiddleware`, e a limpeza controlada de contexto pode usar `ContextEditingMiddleware` com `ClearToolUsesEdit`.

## 7.1 Exemplos prontos de uso

### 6.1 Exemplo base com a matriz default oficial

```yaml
selected_supervisor: "sup_deep"
multi_agents:
  - id: "sup_deep"
    enabled: true
    execution:
      type: "deepagent"
      default_mode: "direct_async"
    middlewares:
      filesystem:
        enabled: true
      shell:
        enabled: false
      memory:
        enabled: true
        sources: []
      subagents:
        enabled: true
      human_in_the_loop:
        enabled: false
      summarization:
        enabled: false
      pii:
        enabled: true
        rules: []
      todo_list:
        enabled: true
      skills:
        enabled: false
    permissions:
      - operations: ["read"]
        paths: ["/memories/**"]
        mode: "allow"
      - operations: ["write"]
        paths: ["/memories/**"]
        mode: "deny"
    deepagent_memory:
      enabled: true
      backend: "redis"
      scope: "user"
      policy: "read_write"
      redis:
        url: "{$DEEPAGENT_REDIS_URL}"
        key_prefix: "deepagent_memory"
        ttl_seconds: 604800
    prompt: ""
    agents:
      - id: "analista"
        description: "Especialista em análise"
        tools: ["json_parse"]
```

### 6.2 Exemplo mínimo com revisão humana por middleware

```yaml
middlewares:
  filesystem:
    enabled: true
  shell:
    enabled: false
  memory:
    enabled: true
    sources: []
  subagents:
    enabled: true
  human_in_the_loop:
    enabled: true
  summarization:
    enabled: false
  pii:
    enabled: true
    rules: []
  todo_list:
    enabled: true
  skills:
    enabled: false
interrupt_on:
  enviar_email:
    allowed_decisions: ["approve", "edit", "reject"]
```

Leitura prática desse exemplo:

1. Isso já é o suficiente para a primeira adoção do HIL em DeepAgent.
2. Não é preciso cadastrar uma tool HIL extra só para revisar `enviar_email`.
3. O runtime do supervisor monta o `HumanInTheLoopMiddleware` automaticamente quando esse bloco está ligado.
4. Se você quiser um gate humano dentro de uma tool customizada, isso já é um caso avançado e pertence a outro padrão do produto.

### 6.3 Exemplo com shell ligado de forma governada

```yaml
middlewares:
  filesystem:
    enabled: true
  shell:
    enabled: true
    workspace_root: "/workspace"
    tool_name: "execute"
    startup_commands: ["pwd"]
    shutdown_commands: []
    env:
      APP_ENV: "dev"
    execution_policy:
      type: "docker"
      extra_run_args: ["--network=none"]
  memory:
    enabled: true
    sources: []
  subagents:
    enabled: true
  human_in_the_loop:
    enabled: false
  summarization:
    enabled: false
  pii:
    enabled: true
    rules: []
  todo_list:
    enabled: true
  skills:
    enabled: false
permissions:
  - operations: ["read"]
    paths: ["/memories/**"]
    mode: "allow"
```

## 7. Logging e observabilidade

Na criação do agente, o runtime registra um log com este formato lógico:

1. `filesystem=on|off`
2. `shell=on|off`
3. `shell_policy={...}`
4. `memory=on|off`
5. `memory_sources=[...]`
6. `subagents=on|off`
7. `human_in_the_loop=on|off`
8. `summarization=on|off`
9. `pii=on|off`
10. `todo_list=on|off`
11. `skills=on|off`
12. `middleware_runtime=[lista das classes aplicadas]`

Nome da mensagem registrada:
`Plano governado de middlewares do DeepAgent`

Leitura prática:

1. Esse log existe para provar o que realmente entrou no runtime.
2. Ele é a fonte mais rápida para saber se o YAML foi respeitado.
3. Ele também mostra detalhes importantes como `shell_policy` e `memory_sources`.
4. Para HIL, esse log mostra se `human_in_the_loop=on` no runtime.

Para revisão humana, confira também outro log importante:

1. `Plano de factory DeepAgent`
2. Nele você consegue ver se `checkpointer=on` e `interrupt_on=on` antes da execução.
3. Se um desses dois estiver `off`, o DeepAgent não está pronto para um fluxo humano de pausa e retomada confiável.

## 8. O que o runtime faz hoje

O runtime atual do projeto:

1. Resolve o contrato `middlewares`.
2. Monta o agente pela factory governada local `_create_governed_deep_agent`.
3. Usa `langchain.agents.create_agent` por baixo.
4. Injeta middlewares oficiais e middlewares extras do produto na ordem correta.
5. Aplica a mesma governança também aos subagentes síncronos.
6. Separa filesystem e shell: arquivo não vira comando, comando não vira arquivo.

## 9. Erros de configuração mais comuns

1. `middlewares.filesystem.enabled=true` sem `permissions[]`.
2. `permissions[]` declarado com `middlewares.filesystem.enabled=false`.
3. `middlewares.human_in_the_loop.enabled=true` sem `interrupt_on`.
4. `interrupt_on` declarado sem ligar `human_in_the_loop`.
5. `middlewares.skills.enabled=true` sem `skills` top-level.
6. `deepagent_memory.enabled=true` com `middlewares.memory.enabled=false`.
7. `async_subagents[].headers` sem `async_subagents[].url`.
8. Uso de `memory` ou `context_schema` no topo do supervisor DeepAgent.
9. Ligar HIL e esquecer o checkpointer operacional na raiz `memory.checkpointer`.
10. Retomar a execução com `thread_id` diferente daquele usado na pausa.

## 10. Execução e Retorno

O DeepAgent executa pelo contrato público do endpoint de agentes. Em execução comum, ele devolve resposta final, métricas e linha do tempo operacional. Em execução com pausa humana, ele devolve `thread_id` e bloco `hil`, para que o cliente aprove, edite ou rejeite as ações pendentes e depois retome pelo mesmo `thread_id`.

Na prática, sucesso não significa apenas ausência de exceção. O retorno precisa mostrar estado coerente, `correlation_id` preservado, timeline rastreável e, quando houver tools, `tools_usage` compatível com o que foi executado.

## 11. Checklist rápido antes de publicar um YAML DeepAgent

1. O supervisor ativo está em `selected_supervisor`.
2. O item em `multi_agents[]` está com `enabled=true`.
3. `execution.type` é `deepagent`.
4. O bloco `middlewares` está explícito e completo.
5. Se `filesystem` estiver `on`, `permissions[]` existe.
6. Se `shell` estiver `on`, `execution_policy` foi revisado.
7. Se `human_in_the_loop` estiver `on`, `interrupt_on` existe.
8. Se `human_in_the_loop` estiver `on`, o checkpointer operacional também está ativo.
9. Se houver pausa humana, a retomada vai reutilizar o mesmo `thread_id`.
10. Se `skills` estiver `on`, a lista top-level `skills` existe.
11. Se `memory` estiver `on`, `deepagent_memory.enabled=true` também existe.
12. O log esperado de middlewares apareceu na inicialização.

## 12. Resumo executivo

O DeepAgent do produto hoje é governado por `middlewares`, não por `capabilities`.

O efeito prático disso é simples:

1. O YAML decide explicitamente o que entra no runtime.
2. O runtime registra em log o que entrou de verdade.
3. O validator bloqueia contratos inconsistentes antes da execução.
4. O canônico, os YAMLs reais, a AST, o resolver e o runtime agora falam a mesma língua.

## 14. Mapeamento AST

Este manual corresponde ao alvo de supervisor deepagent do assembly agentic.

Mapeamento principal:

1. `DeepAgentSupervisorAST` representa o item de `multi_agents[]` quando `execution.type=deepagent`.
2. `DeepAgentSupervisorAST.middlewares` compila para `multi_agents[].middlewares`.
3. `DeepAgentSupervisorAST.deepagent_memory` compila para `multi_agents[].deepagent_memory`.
4. `DeepAgentSupervisorAST.permissions` compila para `multi_agents[].permissions`.
5. `DeepAgentSupervisorAST.interrupt_on` compila para `multi_agents[].interrupt_on`.
6. `DeepAgentSupervisorAST.skills` compila para `multi_agents[].skills`.
7. `DeepAgentSupervisorAST.response_format` compila para `multi_agents[].response_format`.
8. `DeepAgentSupervisorAST.agents[]` compila para `multi_agents[].agents[]`.
9. `DeepAgentSupervisorAST.async_subagents[]` compila para `multi_agents[].async_subagents[]`.
