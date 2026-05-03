# Manual técnico, executivo, comercial e estratégico: DeepAgent Supervisor completo

## 1. Escopo técnico deste manual

Este manual descreve o caminho técnico real do DeepAgent Supervisor no repositório. O foco aqui é explicar o ciclo YAML para AST, a seleção do supervisor ativo, a montagem governada do runtime, o contrato de middlewares, memória Redis, HIL, subagentes, execução síncrona, execução assíncrona e continuação por thread_id.

## 2. Onde o DeepAgent Supervisor entra no sistema

O DeepAgent Supervisor entra no sistema em quatro pontos principais.

- No assembly agentic, quando o alvo é deepagent_supervisor.
- No resolver de configuração, quando o supervisor ativo tem modo deepagent.
- No router de agente, quando o request informa mode deepagent ou quando o YAML resolve esse modo.
- No runtime de background e na continuação HIL, quando uma execução já iniciada precisa ser retomada ou executada fora do ciclo síncrono.

## 3. Ciclo YAML até runtime

O caminho confirmado no código é este.

### 3.1 Detecção do alvo

O assembly olha para multi_agents e identifica itens cujo execution.type é deepagent. Em modo AUTO, isso faz o alvo deepagent_supervisor aparecer como candidato válido.

### 3.2 AST dedicada

O parser DeepAgent produz DeepAgentSupervisorAST. Essa AST inclui campos específicos que o supervisor clássico não tem como contrato principal.

Principais blocos confirmados:

- middlewares
- skills
- response_format
- interrupt_on
- permissions
- agents especializados
- async_subagents
- deepagent_memory herdada da estrutura de supervisor

### 3.3 Validação semântica

Depois da AST, o validator semântico impõe regras de coerência operacional. Ele não apenas checa tipos. Ele verifica compatibilidade entre recursos.

Regras confirmadas no código:

- middlewares.filesystem.enabled igual a true exige permissions explícitas no mesmo escopo.
- permissions só podem existir quando filesystem está habilitado.
- interrupt_on só pode existir quando HIL está habilitado.
- HIL habilitado exige interrupt_on no supervisor.
- HIL habilitado exige memory.checkpointer.enabled igual a true no YAML.
- middlewares.skills.enabled exige skills top-level.
- skills top-level sem middlewares.skills.enabled são rejeitadas.
- deepagent_memory.enabled exige middlewares.memory.enabled.
- deepagent_memory.backend deve ser redis.
- scope aceita apenas user, agent ou org.
- scope org exige user_session.tenant_id.
- policy aceita apenas read_only ou read_write.
- async_subagents precisam de name, description e graph_id válidos.
- headers de async_subagents exigem URL explícita.

### 3.4 Resolução do supervisor ativo

SupervisorConfigResolver prepara um ActiveSupervisorContext. Ele aplica selected_supervisor, enabled, defaults e merge de camadas relevantes. O DeepAgent Supervisor consome esse contexto em vez de ler o YAML cru diretamente em toda parte.

O que isso entrega na prática:

- id do supervisor ativo
- modo deepagent
- agentes resolvidos
- tools_library efetiva
- deepagent_memory já anexada ao contexto
- skills, response_format, interrupt_on, permissions e middlewares do supervisor

## 4. Contrato declarativo do supervisor DeepAgent

### 4.1 Middlewares governados

O bloco middlewares do supervisor DeepAgent possui estes toggles explícitos na AST.

- filesystem
- shell
- memory
- subagents
- background_execution_subagent
- human_in_the_loop
- summarization
- pii
- todo_list
- skills

Esses toggles são a fonte de verdade do comportamento do runtime. O parser também rejeita tentativas de usar campos removidos do contrato antigo, como planner e capabilities.

### 4.2 Permissions

Permissions são regras declarativas de filesystem com:

- operations contendo apenas read e write
- paths absolutos sem pontos duplos
- mode allow ou deny

O runtime converte essas regras em FilesystemPermission apenas quando a factory suporta o parâmetro permissions e quando filesystem está habilitado.

### 4.3 Interrupt_on

Interrupt_on mapeia tool para bool ou para um objeto com allowed_decisions. As decisões aceitas são approve, edit e reject. O validator cruza essas tools com o catálogo efetivo do escopo para impedir referências a ferramentas inexistentes.

### 4.4 deepagent_memory

deepagent_memory é um bloco separado do middleware toggle. Ele não substitui middlewares.memory. Os dois precisam estar coerentes.

No runtime lido, ele aceita:

- enabled
- backend igual a redis
- scope igual a user, agent ou org
- policy igual a read_only ou read_write
- redis.url obrigatório
- redis.key_prefix opcional, com default deepagent_memory
- redis.ttl_seconds opcional e positivo

## 5. Bootstrap do DeepAgentSupervisor

O método initialize do DeepAgentSupervisor executa esta sequência.

1. Carrega o contexto ativo e AppConfig.
2. Inicializa ToolsFactory e MemoryFactory compartilhadas.
3. Resolve a factory governada do DeepAgent.
4. Compõe a pilha extra de middlewares do produto.
5. Constrói backend e store Redis se deepagent_memory estiver habilitada.
6. Obtém checkpointer.
7. Cria o agente DeepAgent sob contrato governado.

Se qualquer etapa falhar, o supervisor registra evento de lifecycle, stack trace e erro de inicialização antes de propagar a falha.

## 6. Cache e reutilização do DeepAgent

O supervisor usa WarmResourcePool com chave baseada em:

- shared_tenant_key derivado do hash canônico do YAML
- supervisor_id
- hash do YAML
- versão de cache do supervisor

Se houver hit com hash e versão compatíveis, o artefato DeepAgent é reutilizado. Se houver incompatibilidade, a entrada é invalidada e o agente é recriado.

Isso é importante porque a montagem do runtime DeepAgent é cara. O cache evita reconstrução desnecessária do mesmo supervisor dentro do mesmo contexto de tenant e configuração.

## 7. Resolução da factory e dependências obrigatórias

O método _resolve_deep_agent_factory não aceita qualquer runtime parcialmente instalado. Ele verifica a presença de referências consideradas obrigatórias.

Dependências exigidas no runtime lido:

- FilesystemMiddleware
- SubAgentMiddleware
- AsyncSubAgentMiddleware
- PatchToolCallsMiddleware
- _PermissionMiddleware
- _ToolExclusionMiddleware

Se alguma delas estiver ausente, o supervisor falha com ImportError explícito. O objetivo é impedir que um YAML válido aparente funcionar sobre uma instalação incompleta do runtime.

## 8. Montagem governada do agente

O coração do comportamento está em _create_governed_deep_agent. Esse método monta a pilha final obedecendo ao contrato governado do produto.

### 8.1 Pré-condições

Antes de montar, o método valida condições como:

- filesystem habilitado exige backend
- subagents habilitado exige backend
- skills habilitado exige backend
- memory habilitado exige backend
- summarization habilitado exige backend
- permissions exige backend

### 8.2 Ordem lógica da pilha

O método acrescenta middlewares em uma ordem controlada, combinando recursos do produto com recursos da biblioteca.

Componentes relevantes confirmados:

- TodoListMiddleware
- PIIMiddleware
- SkillsMiddleware
- FilesystemMiddleware
- ShellToolMiddleware
- SubAgentMiddleware
- AsyncSubAgentMiddleware
- SummarizationMiddleware
- PatchToolCallsMiddleware
- middlewares extras do produto já compostos antes
- middlewares extras resolvidos pelo perfil do runtime, quando presentes
- prompt caching para modelos Anthropic, quando a referência existe
- middleware de exclusão de tools
- middleware de permissões

### 8.3 Middlewares extras do produto

Além dos middlewares do runtime DeepAgent, o supervisor adiciona uma pilha própria.

Ela inclui:

- ToolCallLimitMiddleware
- ModelCallLimitMiddleware
- LLMToolSelectorMiddleware
- ToolRetryMiddleware
- ModelRetryMiddleware
- ContextEditingMiddleware
- ClearToolUsesEdit
- ToolSelectionAuditMiddleware
- ToolExecutionMiddleware
- ResponsePostProcessingMiddleware
- ErrorHandlingMiddleware

O supervisor valida a ordem esperada dos middlewares principais para evitar drift silencioso.

## 9. Ferramentas e subagentes

### 9.1 Coleta de tools do supervisor

O supervisor percorre os agentes configurados e resolve tools únicas via ToolsFactory e resolve_agent_tools_with_context. Isso impede duplicação desnecessária de ferramentas no conjunto agregado.

### 9.2 Subagentes síncronos

Cada subagente síncrono pode receber:

- name ou id
- description
- model opcional
- skills opcionais
- response_format opcional
- interrupt_on opcional
- permissions opcionais
- tools

O runtime reaplica a governança nesses subagentes, inclusive filesystem, shell, summarization, skills, prompt caching e restrição de tools.

### 9.3 Async subagents

Async subagents seguem um contrato próprio com:

- name obrigatório
- description obrigatória
- graph_id obrigatório
- url opcional
- headers opcionais, mas só quando url existe

O objetivo é permitir delegação para grafos ou runtimes externos sem misturar esse transporte com o contrato dos subagentes locais.

### 9.4 Subagente automático de background execution

Quando middlewares.background_execution_subagent.enabled está on e middlewares.subagents também está on, o supervisor injeta automaticamente um subagente chamado background_execution com tools específicas de agendamento, cancelamento, reprogramação e consulta de solicitações agentic em background.

## 10. Memória persistente DeepAgent

### 10.1 Resolução do bloco

O runtime lê deepagent_memory a partir do ActiveSupervisorContext. Se o bloco estiver ausente ou desabilitado, store e backend não são criados.

### 10.2 Backend Redis obrigatório

Quando a memória está habilitada, o runtime exige backend redis e um objeto redis com URL válida. Placeholders do tipo ${VAR} ou {$VAR} podem ser resolvidos via security_keys.

### 10.3 Escopos

O namespace do StoreBackend é derivado do escopo.

- user usa user_email
- agent usa supervisor_id
- org usa tenant_id

Se o escopo for org, o runtime exige user_session.tenant_id explícito.

### 10.4 Política de escrita

O store respeita policy read_only ou read_write. Em read_only, tentativas de PutOp falham com PermissionError.

### 10.5 Contrato do DeepAgentRedisStore

DeepAgentRedisStore implementa BaseStore sobre Redis e usa retry externo central em todas as operações Redis relevantes. Ele também exige que deepagent_memory.redis.url coincida com REDIS_PROMETEU_GENERIC_RAG_URL enquanto o DeepAgent usa o Redis global do projeto.

O store mantém:

- chaves namespaced por prefixo e namespace lógico
- set de namespaces conhecidos
- TTL configurável
- refresh de TTL quando o item é lido com refresh_ttl

## 11. Checkpointer

O checkpointer é resolvido via MemoryFactory reutilizando o mesmo mecanismo de memória do ecossistema de supervisor. No contrato lido, ele é obrigatório para HIL funcionar corretamente. O validator semântico trata sua ausência como erro quando HIL está habilitado.

## 12. Execução do DeepAgent

### 12.1 Input normal

Quando a mensagem não é um Command resume, o supervisor constrói payload com messages contendo HumanMessage.

### 12.2 Continuação

Quando a mensagem é um Command com resume, o supervisor reutiliza esse payload diretamente.

### 12.3 Invoke

O invoke usa config configurável construída pelo próprio supervisor, versão v2 e timeout resolvido por InvokeTimeoutGuard.

### 12.4 Normalização da saída

Depois do invoke, o supervisor normaliza o resultado para um dicionário padrão com:

- final_response
- metadata
- metrics
- thread_id
- mode deepagent
- success
- execution_timeline
- tools_usage
- diagnostics quando houver
- hil quando houver interrupção humana

Se houver HIL e não houver final_response explícita, o supervisor preenche a resposta com a mensagem de pausa aguardando decisão humana.

## 13. Contrato HTTP público

### 13.1 POST /agent/execute

É o endpoint público para iniciar execução agentic. Quando o request informa mode deepagent, o router direciona para DeepAgentSupervisor. O endpoint pode devolver:

- resposta final síncrona
- pausa HIL normalizada com thread_id e hil
- envelope assíncrono com task_id e URLs de acompanhamento

### 13.2 POST /agent/continue

É o endpoint público para retomar uma execução pausada. Ele exige o mesmo thread_id da pausa anterior, reaproveita o YAML original e executa Command resume no supervisor correto, agent ou deepagent.

### 13.3 POST /agent/hil/decisions

É o endpoint para decisão HIL externa segura. Ele recebe token de aprovação, valida status e expiração, resolve a decisão de forma atômica e dispara a continuação sem depender de um GET inseguro.

## 14. Execução assíncrona e background

O router possui um caminho dedicado para execução assíncrona do DeepAgentSupervisor. Nesse fluxo:

1. O callback de progresso é criado.
2. O YAML é preparado com correlation_id e user_email.
3. O supervisor é inicializado.
4. O run ocorre em executor.
5. O resultado é normalizado.
6. Se houver HIL, a pausa pode disparar notificação de aprovação assíncrona.

O runtime de background execution também consegue instanciar DeepAgentSupervisor diretamente para executar runs agendados em background.

## 15. Continuação HIL

AgentHilContinuationService decide qual supervisor retomar com base no supervisor_mode. Quando o modo é deepagent, ele usa a factory de DeepAgentSupervisor, monta Command resume e executa o run no thread_id original.

Isso garante que a continuação não seja um atalho textual improvisado. Ela é uma retomada formal do estado pausado.

## 16. Caminho feliz

Um cenário feliz típico é este.

1. O YAML declara um supervisor deepagent habilitado.
2. O assembly valida a configuração.
3. O request chega em /agent/execute com mode deepagent.
4. O supervisor resolve contexto, tools, middlewares, store e checkpointer.
5. O agente é criado ou reaproveitado do cache.
6. O invoke retorna resposta final ou pausa HIL coerente.
7. Se houver pausa, /agent/continue ou /agent/hil/decisions retomam a execução com o mesmo thread_id.

## 17. Cenários de erro confirmados

### Campo removido no YAML

Sintoma: parser ou validator falha.

Causa confirmada: uso de planner, capabilities, memory top-level ou context_schema no supervisor DeepAgent.

### Filesystem sem permissions

Sintoma: validator rejeita o supervisor.

Causa confirmada: middlewares.filesystem.enabled igual a true sem permissions explícitas.

### HIL sem checkpointer

Sintoma: validator semântico rejeita a configuração.

Causa confirmada: middlewares.human_in_the_loop.enabled igual a true sem memory.checkpointer.enabled.

### deepagent_memory incompatível

Sintoma: validator ou runtime falha cedo.

Causa confirmada: backend diferente de redis, URL ausente, escopo inválido, policy inválida ou tenant_id ausente em escopo org.

### Runtime DeepAgent incompatível

Sintoma: ImportError na inicialização.

Causa confirmada: ausência de middlewares ou classes obrigatórias no ambiente instalado.

## 18. Observabilidade e diagnóstico

Para investigar DeepAgent Supervisor, os pontos principais são estes.

### 18.1 Logs de bootstrap

Eles mostram seleção da factory, middlewares registrados, plano de runtime, configuração do store Redis e cache hit ou miss do supervisor.

### 18.2 Timeline de execução

O payload final inclui execution_timeline combinando lifecycle trace e DeepAgentRuntimeTelemetry.

### 18.3 tools_usage

O supervisor devolve resumo de uso de tools, o que ajuda a separar falha de ferramenta de falha de orquestração.

### 18.4 correlation_id e thread_id

Correlation_id reconstrói a execução ponta a ponta. Thread_id identifica a conversa ou a pausa HIL específica que precisa ser retomada.

## 19. Como colocar para funcionar

Com base no código lido, o caminho mínimo é este.

1. Declarar um supervisor em multi_agents com execution.type deepagent.
2. Garantir selected_supervisor coerente quando houver mais de um supervisor habilitado.
3. Configurar middlewares explicitamente conforme a capacidade desejada.
4. Se HIL estiver habilitado, ligar memory.checkpointer.enabled e declarar interrupt_on.
5. Se deepagent_memory estiver habilitada, configurar Redis global e deepagent_memory.redis.url coerente com REDIS_PROMETEU_GENERIC_RAG_URL.
6. Chamar /agent/execute com mode deepagent ou deixar o router resolver esse modo pelo YAML ativo.

## 20. Exemplos práticos guiados

### Exemplo 1: DeepAgent síncrono com HIL

Cenário: o cliente envia /agent/execute com mode deepagent e o supervisor possui human_in_the_loop habilitado.

Processamento: o invoke chega até uma tool mapeada em interrupt_on, a execução pausa e o payload final inclui hil e thread_id.

Saída: o cliente usa /agent/continue ou /agent/hil/decisions para retomar.

### Exemplo 2: DeepAgent com memória persistente por usuário

Cenário: deepagent_memory.enabled está on com scope user.

Processamento: o runtime monta StoreBackend sobre DeepAgentRedisStore usando namespace user e user_email.

Saída: o agente reaproveita memória dentro desse escopo, respeitando a política de escrita.

### Exemplo 3: execução assíncrona com pausa humana

Cenário: /agent/execute decide ou recebe execução assíncrona.

Processamento: o callback de progresso registra running, depois paused quando surge HIL. O sistema pode disparar aprovação assíncrona por canal governado.

Saída: a operação acompanha o estado pela task e pela correlação do run.

## 21. Explicação 101

O DeepAgent Supervisor é como um coordenador de equipe com acesso a ferramentas, memória e especialistas, mas trabalhando dentro de um manual interno obrigatório. Ele não pode simplesmente fazer qualquer coisa porque a biblioteca permite. Ele só faz o que o contrato do produto diz que é permitido, o que a validação aprova e o que a infraestrutura realmente suporta.

## 22. Limites e pegadinhas

- selected_supervisor mal configurado pode impedir a seleção correta do modo deepagent.
- A AST aceitar uma forma estrutural não significa que a combinação semântica está correta.
- deepagent_memory não substitui checkpointer; são responsabilidades diferentes.
- Thread_id é obrigatório para continue; não pode ser reconstruído por aproximação.
- O cache do supervisor economiza custo, mas não substitui validação de contrato.

## 23. Checklist de entendimento

- Entendi o caminho YAML até AST.
- Entendi as regras do validator semântico.
- Entendi como o SupervisorConfigResolver prepara o contexto ativo.
- Entendi a pilha governada de middlewares do runtime.
- Entendi o papel de deepagent_memory e do DeepAgentRedisStore.
- Entendi o papel do checkpointer.
- Entendi os endpoints de execute, continue e hil decisions.
- Entendi como o background execution também aciona o DeepAgent Supervisor.

## 24. Evidências no código

- src/config/agentic_assembly/ast/deepagent.py
  - Motivo da leitura: confirmar o contrato AST do DeepAgent Supervisor.
  - Símbolos relevantes: DeepAgentSupervisorAST, DeepAgentMiddlewaresAST, DeepAgentAgentAST.
  - Comportamento confirmado: estrutura tipada dos campos deepagent.

- src/config/agentic_assembly/parsers/deepagent_parser.py
  - Motivo da leitura: confirmar filtragem por execution.type e bloqueio de campos removidos.
  - Símbolos relevantes: parse, collect_supervisor_contract_diagnostics.
  - Comportamento confirmado: parser dedicado ao alvo deepagent.

- src/config/agentic_assembly/validators/deepagent_semantic_validator.py
  - Motivo da leitura: confirmar coerência obrigatória do contrato.
  - Símbolos relevantes: DeepAgentSemanticValidator, \_validate_permissions_list, \_validate_hil_checkpointer.
  - Comportamento confirmado: regras de HIL, permissions, skills, memory e async_subagents.

- src/agentic_layer/supervisor/config_resolver.py
  - Motivo da leitura: confirmar seleção do supervisor ativo e contexto executável.
  - Símbolos relevantes: SupervisorConfigResolver, ActiveSupervisorContext.
  - Comportamento confirmado: selected_supervisor, enabled e modo deepagent resolvidos antes do runtime.

- src/agentic_layer/supervisor/deep_agent_supervisor.py
  - Motivo da leitura: confirmar bootstrap, cache, invoke e montagem governada do agente.
  - Símbolos relevantes: initialize, run, _create_governed_deep_agent, _build_deepagent_store_backend, _import_deepagents.
  - Comportamento confirmado: runtime DeepAgent governado com cache, memória, HIL e middleware stack controlada.

- src/agentic_layer/supervisor/deepagent_redis_store.py
  - Motivo da leitura: confirmar store Redis da memória persistente.
  - Símbolos relevantes: DeepAgentRedisStore, \_assert_write_allowed, \_refresh_ttl.
  - Comportamento confirmado: BaseStore com TTL, retry, namespace e política de escrita.

- src/api/routers/agent_router.py
  - Motivo da leitura: confirmar fronteira pública do DeepAgent.
  - Símbolos relevantes: /agent/execute, /agent/continue, /agent/hil/decisions, _execute_agent_with_deepagent_supervisor.
  - Comportamento confirmado: execução, pausa, retomada e uso assíncrono do DeepAgent Supervisor.

- src/api/services/agent_hil_continuation_service.py
  - Motivo da leitura: confirmar retomada formal por supervisor_mode.
  - Símbolos relevantes: _build_supervisor, _run_supervisor.
  - Comportamento confirmado: DeepAgentSupervisor é recriado e retomado por Command resume.

- src/agentic_layer/background_execution/runtime.py
  - Motivo da leitura: confirmar integração com execução agentic em background.
  - Símbolos relevantes: _execute_deepagent.
  - Comportamento confirmado: background execution também instancia DeepAgentSupervisor e chama run.
