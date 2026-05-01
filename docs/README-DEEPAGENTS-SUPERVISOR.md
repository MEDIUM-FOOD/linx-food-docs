# DeepAgent Supervisor Governado

Este manual explica o modo DeepAgent como sistema em operação. O foco não é listar campos por listar, e sim mostrar por que esse supervisor existe, como ele decide, o que ele protege e onde ele falha fechado.

O ponto mais importante é este: DeepAgent não é um "supervisor clássico com mais botões". Ele é uma forma governada de executar agentes com middlewares explícitos, memória controlada, subagentes, revisão humana e limites operacionais que o produto monta localmente, sem depender de defaults ocultos da biblioteca externa.

## 1. Escopo

O mesmo bloco multi_agents comporta mais de uma família de supervisor. Quando o YAML usa execution.type=agent, ele cai no supervisor clássico. Quando usa execution.type=deepagent, ele entra na montagem governada do DeepAgent. Os dois vivem sob o mesmo contrato macro de selected_supervisor, tools_library, global_tools_configuration, local_tools_configuration e execution.default_mode, mas não executam da mesma forma.

Na prática, o supervisor clássico é útil quando o problema cabe numa coordenação mais direta entre agentes e tools. O DeepAgent entra quando o produto precisa aplicar governança adicional sobre filesystem, shell, deepagent_memory, subagentes, sumarização, proteção de PII, lista de tarefas e human-in-the-loop.

## 2. Ativação do modo deepagent

A chave que muda o comportamento não é um detalhe cosmético. O runtime só entra no caminho governado quando o supervisor ativo declara execution.type=deepagent dentro de multi_agents. Sem isso, o produto permanece no fluxo execution.type=agent.

Esse detalhe importa porque a montagem do DeepAgent exige dependências extras e validações mais duras. O supervisor só segue adiante quando a factory governada, a cadeia de middlewares e o backend compatível estão disponíveis. Se alguma peça obrigatória faltar, o comportamento correto é interromper a inicialização, não degradar silenciosamente.

## 3. Seleção do Supervisor Ativo

A seleção parte de selected_supervisor e do conjunto de multi_agents habilitados. No fluxo de assembly, a decisão também pode nascer da intenção do prompt ou do shape do próprio YAML. Se o documento mistura supervisores clássicos e deepagent, o assembly identifica os dois alvos e aplica merge seletivo por id. Se o request chega em AUTO e encontra execution.type=deepagent, o alvo tende a ser resolvido como supervisor governado.

Em linguagem simples: o produto primeiro decide qual supervisor manda e só depois discute quais ferramentas, memórias e middlewares esse supervisor pode usar. Isso evita duas falhas comuns: editar o supervisor errado e achar que um bloco de configuração de um supervisor vale automaticamente para outro.

## 4. Contrato de `multi_agents[]`

multi_agents[] é o contrato raiz do supervisor. É ali que ficam id, enabled, execution.type, execution.default_mode, limits.timeout_s, limits.max_tool_calls, limits.token_budget, tools_library, global_tools_configuration, local_tools_configuration e o shape dos especialistas subordinados. No caso do DeepAgent, esse mesmo bloco também carrega interrupt_on, permissions, skills, async_subagents e deepagent_memory.

O ganho desse desenho é coerência operacional. O runtime consegue decidir quem é o supervisor ativo, quais limites entram em vigor e quais recursos precisam existir antes da execução. O trade-off é que o documento YAML precisa ser mais explícito. O produto prefere esse custo declarativo a esconder regra crítica em convenção implícita.

## 4. Contrato de `deepagent_memory`

O bloco deepagent_memory existe para uma pergunta específica: a memória durável do agente precisa sobreviver entre execuções? Se a resposta for sim, o produto espera um backend explícito. Pelo código lido, o caminho governado tratado como real é backend: "redis", com redis.url, key_prefix e ttl_seconds definindo onde a memória fica, como o namespace é separado e quanto tempo ela vive.

Esse bloco não é a mesma coisa que o middleware de memória. deepagent_memory trata persistência. O middleware middlewares.memory trata a decisão de expor essa memória ao runtime do agente. Separar essas duas camadas evita a ilusão de que basta apontar um Redis para que o agente automaticamente passe a usar memória de forma segura.

## 5. Contrato de `agents[]`

No nível do supervisor, agents[] representa os especialistas sincronizados subordinados ao coordenador principal. Eles compartilham governança macro, mas ainda podem receber model, tools, interrupt_on, permissions, response_format e skills próprios. O valor dessa estrutura não está em "ter vários agentes". Está em tornar explícito quais tarefas merecem especialização e quais ferramentas cada especialista pode tocar.

Quando essa fronteira é respeitada, o supervisor deixa de ser um monólito com prompt gigante e passa a orquestrar responsabilidades menores. Quando ela não é respeitada, o YAML continua válido, mas a operação perde clareza: fica difícil saber se a falha veio da coordenação, do especialista ou da tool.

## 5. Contrato de `agents[]` (subagentes)

No DeepAgent, o papel dos subagentes vai além de delegar tarefas. O runtime recompõe esses especialistas com a mesma pilha governada do produto. Isso significa que um subagente não recebe automaticamente tudo o que o pai tem. O código recompõe model, tools, exclusões, permissões, skills, shell, filesystem, summarization e PII conforme o contrato ativo.

Em termos práticos, isso protege o sistema contra um erro comum em arquiteturas multiagente: imaginar que qualquer subagente pode herdar tudo do coordenador principal sem risco. Aqui a herança é filtrada, recomposta e registrada em log.

## 6. Middleware Chain

A cadeia de middlewares é o coração operacional do DeepAgent. Ela existe porque a plataforma não quer depender do comportamento padrão da lib como fonte de verdade do produto. Os nomes mais importantes que aparecem no runtime governado são ToolCallLimitMiddleware, ModelCallLimitMiddleware, LLMToolSelectorMiddleware, ToolRetryMiddleware, ModelRetryMiddleware, ContextEditingMiddleware, ClearToolUsesEdit, FilesystemMiddleware, ShellToolMiddleware, ToolSelectionAuditMiddleware, ToolExecutionMiddleware, ResponsePostProcessingMiddleware e ErrorHandlingMiddleware.

Esses nomes não devem ser lidos como inventário decorativo. Cada um marca um tipo de proteção. Há middlewares para limitar chamadas, outros para escolher tool, outros para retry, outros para editar contexto, outros para guardar filesystem e shell, outros para auditar seleção, executar ferramenta, pós-processar resposta e tratar erro. Em conjunto, eles contam a política operacional do supervisor.

O runtime também acrescenta middlewares opcionais controlados pelo YAML, como memória, PII, skills, todo_list, human_in_the_loop e sumarização. O efeito prático é importante: a plataforma consegue dizer no log o que está on e o que está off para aquela execução, em vez de deixar o operador deduzir isso a partir de efeitos laterais.

## 7. Precedência de Configuração de Tools

As tools não entram no DeepAgent por improviso. O ponto de partida continua sendo tools_library, depois global_tools_configuration, depois local_tools_configuration e, por fim, ajustes específicos do agente ou subagente. Em paralelo, o runtime ainda pode excluir ferramentas por perfil do modelo ou pelo próprio middleware, como acontece quando FilesystemMiddleware está ligado e o shell precisa reservar execute para ShellToolMiddleware.

O ganho dessa precedência é previsibilidade. O operador consegue explicar por que uma tool apareceu, foi sobrescrita ou foi bloqueada. O risco evitado é clássico: ter o mesmo id de ferramenta configurado em vários lugares e descobrir o resultado final só em produção.

## 7. Limites e Retries

O bloco limits.timeout_s, limits.max_tool_calls e limits.token_budget existe para impedir consumo sem freio. Esses limites são a camada declarativa do problema. Na execução real, o supervisor complementa isso com middlewares como ToolCallLimitMiddleware, ModelCallLimitMiddleware, ToolRetryMiddleware e ModelRetryMiddleware.

Essa combinação é importante porque limite e retry resolvem problemas diferentes. Limite controla quanto o agente pode gastar. Retry controla como o sistema reage a falha transitória. Misturar os dois leva a duas ilusões perigosas: achar que retry substitui limite ou achar que limite basta para lidar com integração instável.

## 8. Memória no Supervisor Clássico

O repositório ainda precisa explicar a diferença entre a memória do supervisor clássico e a do DeepAgent porque o contrato comum compartilha termos parecidos. No supervisor clássico, memory.checkpointer e memory.agent_conversation_history representam formas de continuidade conversacional e checkpoint dentro do fluxo execution.type=agent. No DeepAgent, deepagent_memory e middlewares.memory tornam esse problema mais governado e mais explícito.

Na prática, isso evita que alguém leia memory.checkpointer em um documento e presuma que está configurando a mesma coisa que deepagent_memory. Os dois tratam memória, mas em camadas e runtimes diferentes.

## 9. Execução e Payload de Retorno

Do ponto de vista do consumidor, o supervisor precisa devolver resposta útil e rastreável. Por isso a documentação do contrato comum continua relevante aqui: execution_timeline e tools_usage ajudam a entender o que o agente tentou fazer, em que ordem e com quais ferramentas. Eles não substituem logs, mas reduzem o caminho entre o sintoma e a hipótese.

O caminho feliz de execução é este: o YAML seleciona o supervisor certo, o runtime valida middlewares e backend, monta o agente governado, executa a tarefa e devolve saída com rastreabilidade. O caminho de erro também é deliberado: se filesystem, skills, subagents, summarization, memória ou permissions exigirem backend e ele não existir, a inicialização falha fechada antes da execução seguir.

## 10. Execução e Retorno

No DeepAgent, execução significa mais do que chamar um modelo. Significa montar o agente com a ordem correta de middlewares, separar subagentes síncronos e assíncronos, materializar policy de shell, configurar FilesystemMiddleware, ligar ou desligar HumanInTheLoopMiddleware, expor memória quando middlewares.memory estiver enabled e só então chamar a factory governada.

O retorno útil não é só a resposta em si. É a combinação de resposta, logs, execution_timeline, tools_usage e marcadores de inicialização que mostram quais capacidades estavam ligadas. Em ambiente corporativo, isso reduz o custo de suporte porque transforma uma pergunta vaga, como "o agente ignorou a tool", em uma pergunta verificável, como "a tool estava excluída, sem permissão, fora do catálogo ou bloqueada por middleware?".

## 11. Sucesso, erro e troubleshooting

Quando tudo funciona, o log deixa claro qual factory foi selecionada, quais middlewares estavam ativos, se havia checkpointer, store, backend e runtime cache, e qual supervisor estava respondendo. Quando algo falha, o padrão observado no código é falhar cedo com mensagem explícita: execution_policy inválida, permissions sem filesystem, skills sem backend, memory sem backend, interrupt_on ausente quando human_in_the_loop está ligado, ou dependência obrigatória do runtime DeepAgent não encontrada.

A leitura operacional correta segue esta ordem.

1. Confirmar selected_supervisor e execution.type.
2. Confirmar se deepagent_memory e middlewares.memory estão coerentes.
3. Confirmar se permissions existem quando filesystem está enabled.
4. Confirmar se ShellToolMiddleware tem policy válida.
5. Confirmar se o backend DeepAgent realmente foi inicializado.
6. Ler os marcadores de plano governado de middlewares para ver o estado on ou off de cada capacidade.

## 12. Explicação 101

Imagine um coordenador de trabalho que, antes de deixar alguém agir, checa limite, ferramental, autorização, memória, revisão humana e proteção de dados. Isso é o DeepAgent neste projeto. Ele existe para que o agente não pareça poderoso apenas porque responde bem. Ele precisa responder com controle, rastreabilidade e fronteiras claras.

## 13. Evidências no código

- [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py): resolução do target, parse do documento e validação semântica do slice agentic.
- [src/config/agentic_assembly/parsers/deepagent_parser.py](../src/config/agentic_assembly/parsers/deepagent_parser.py): filtragem e parse leniente do supervisor DeepAgent.
- [src/config/agentic_assembly/validators/deepagent_semantic_validator.py](../src/config/agentic_assembly/validators/deepagent_semantic_validator.py): validação forte do contrato de supervisor e memória.
- [tests/smoke/test_deepagent_runtime_smoke.py](../tests/smoke/test_deepagent_runtime_smoke.py): evidência do caminho feliz do runtime real.
