# Manual técnico, executivo, comercial e estratégico: DeepAgent Supervisor completo

## 1. O que é esta feature

O DeepAgent Supervisor é a espinha dorsal agentic usada quando o YAML declara execution.type igual a deepagent no bloco multi_agents. Ele não é apenas uma variante cosmética do supervisor clássico. O código mostra que ele é um runtime governado, com contrato próprio de middlewares, memória persistente, subagentes síncronos e assíncronos, pausa humana declarativa, permissões explícitas de filesystem e integração com o boundary HTTP que suporta execução síncrona, assíncrona e continuação de HIL.

Na prática, esta feature existe para executar agentes compostos com mais autonomia operacional, mas sem abrir mão de controle. O sistema deixa o supervisor usar capacidades como shell, filesystem, memória Redis, subagentes e pausa humana, porém obriga essas capacidades a passarem por um contrato validado e observável.

## 2. Que problema ela resolve

Sem o DeepAgent Supervisor, a plataforma teria duas opções ruins.

A primeira seria usar apenas o supervisor agent tradicional, que resolve muitos cenários de orquestração, mas não foi desenhado para o mesmo conjunto de middlewares governados, memória durável e semântica de subagentes do runtime DeepAgent.

A segunda seria permitir que cada YAML montasse um agente avançado de forma livre, sem AST, sem validação semântica forte e sem checagem de compatibilidade do runtime. Isso criaria um ambiente frágil, difícil de operar e propenso a capacidades escondidas.

O DeepAgent Supervisor resolve esse problema transformando um conjunto sofisticado de recursos agentic em um contrato operacional restrito, validado e observável.

## 3. Visão executiva

Para liderança, esta feature reduz risco em três frentes.

- Reduz risco de autonomia sem governança, porque shell, filesystem, memória e HIL só entram quando o contrato YAML permite e o runtime consegue suportar explicitamente.
- Reduz risco de instabilidade operacional, porque o runtime valida incompatibilidades cedo, usa cache por hash e versão e exige checkpointer quando o fluxo pode pausar por HIL.
- Reduz risco de baixa auditabilidade, porque a execução mantém timeline, telemetria de tools, correlation_id, thread_id e superfícies de continuação controladas.

Em linguagem executiva simples: é a peça que permite usar uma forma mais poderosa de agente sem transformar a plataforma em uma caixa-preta impossível de controlar.

## 4. Visão comercial

Do ponto de vista comercial, o DeepAgent Supervisor é a feature que permite responder a um tipo específico de demanda corporativa: clientes que não querem apenas um chatbot consultivo, mas um agente que consiga trabalhar com múltiplas ferramentas, delegar tarefas, pausar para aprovação humana e manter memória operacional entre interações.

O diferencial vendável não é prometer autonomia irrestrita. O diferencial real, confirmado no código, é oferecer autonomia governada. Isso significa:

- agentes com ferramentas e subagentes especializados;
- shell e filesystem só quando declarados e permitidos;
- pausa humana com continuação formal;
- memória Redis com escopo e política explícitos;
- execução síncrona, assíncrona e background sem mudar o contrato central.

Isso ajuda a vender a plataforma para casos em que a pergunta do cliente é menos “o agente responde?” e mais “o agente consegue operar com segurança dentro do meu processo?”.

## 5. Visão estratégica

Estratégicamente, o DeepAgent Supervisor fortalece a visão de plataforma YAML-first porque separa quatro problemas que costumam ficar misturados em soluções agentic frágeis.

- Definição declarativa do comportamento no YAML.
- Validação estrutural e semântica antes da execução.
- Montagem governada do runtime DeepAgent.
- Operação e continuação seguras por APIs canônicas.

Essa separação prepara a plataforma para evoluir sem depender de hacks. Novos middlewares, novos subagentes, novas políticas de shell ou novos canais de aprovação assíncrona podem entrar preservando o mesmo desenho de AST, parser, validator, resolver e runtime governado.

## 6. Conceitos necessários para entender

### Supervisor governado

É um supervisor cuja autonomia é mediada por um contrato explícito. Ele não recebe liberdade total para usar o que existir na biblioteca. Ele só usa o que a configuração, a validação e o runtime suportam juntos.

### DeepAgent

Neste projeto, DeepAgent é um modo específico de supervisor, não um apelido genérico para qualquer agente sofisticado. O assembly detecta esse alvo, produz AST dedicada, valida regras próprias e seleciona o runtime correspondente no router de agente.

### Middlewares governados

O DeepAgent não nasce com comportamento solto. O código separa middlewares oficiais da stack DeepAgent e middlewares de produto, como auditoria de seleção de tool, telemetria, pós-processamento e captura de erro. Além disso, cada capacidade importante tem um toggle governado no YAML.

### HIL

HIL significa Human in the Loop, ou pausa humana para aprovação, edição ou rejeição de uma ação. No DeepAgent, HIL não é um detalhe opcional sem contrato. Quando habilitado, ele exige interrupt_on, exige checkpointer e pode acionar aprovação assíncrona por canais governados.

### deepagent_memory

É a configuração de memória persistente do DeepAgent. No código lido, ela só é aceita com backend Redis, com escopos user, agent ou org e políticas read_only ou read_write.

### Subagente síncrono e assíncrono

O supervisor pode delegar para dois tipos de subagente. O síncrono entra como parte do runtime local governado. O assíncrono entra como especificação externa com graph_id e, opcionalmente, URL e headers para transporte HTTP.

## 7. Como a feature funciona por dentro

O ciclo de vida real do DeepAgent Supervisor começa antes do invoke. O assembly lê o YAML, detecta execution.type igual a deepagent, converte esse fragmento para uma AST dedicada, executa validação semântica e produz um artefato canônico que o runtime aceita.

Na execução, o router de agente recebe o pedido, resolve o YAML ativo, escolhe o modo deepagent, prepara user_session e correlation_id e instancia DeepAgentSupervisor. Esse supervisor carrega o contexto ativo via SupervisorConfigResolver, inicializa ToolsFactory e MemoryFactory compartilhadas, resolve a factory governada do runtime DeepAgent, compõe a pilha extra de middlewares do produto, monta store e backend Redis quando deepagent_memory está habilitada, resolve checkpointer e então cria o agente com create_agent sob um conjunto de restrições explícitas.

Depois disso, o run pode seguir por três trilhas principais:

- execução síncrona via /agent/execute;
- execução assíncrona via /agent/execute com envelope de task;
- retomada de pausa humana via /agent/continue ou por decisão HIL externa.

## 8. Divisão em etapas ou submódulos

### 8.1 Contrato declarativo de DeepAgent

Esta etapa existe para definir o que o YAML pode expressar. A AST especializada de DeepAgent Supervisor introduz middlewares governados, skills, response_format, interrupt_on, permissions, async_subagents e agentes especializados com campos próprios.

O valor dessa etapa é impedir que o produto aceite sintaxe informal ou incompatível com o runtime real.

### 8.2 Parser e validação semântica

O parser filtra apenas supervisores cujo execution.type é deepagent, injeta defaults mínimos e converte para DeepAgentSupervisorAST. Depois, o validator semântico aplica as regras que o runtime realmente exige.

Essa etapa existe porque AST válida não basta. É preciso garantir coerência entre HIL, interrupt_on, checkpointer, deepagent_memory, filesystem, permissions, skills e subagentes.

### 8.3 Resolução do supervisor ativo

O SupervisorConfigResolver unifica selected_supervisor, enabled, defaults e modo de execução. Ele transforma o YAML em ActiveSupervisorContext, que o runtime consome.

Esse submódulo existe para separar sintaxe declarada de contexto executável.

### 8.4 Runtime governado do DeepAgent

Esta é a parte que realmente monta o agente. O supervisor resolve LLM via WarmResourcePool, agrega tools, deriva subagentes, injeta middlewares governados, aplica permissões de filesystem, configura shell, memória, HIL, skills e summarization quando suportados e só então cria o agente.

### 8.5 Operação e continuação

Depois de montado, o DeepAgent Supervisor não termina no invoke. O projeto inclui contratos explícitos para execução assíncrona, pausa HIL, continuação tipada por thread_id e decisão externa segura por POST.

## 9. Pipeline ou fluxo principal

O fluxo principal confirmado no código pode ser explicado assim.

1. O YAML chega ao assembly ou ao boundary HTTP.
2. O sistema detecta que o alvo é deepagent_supervisor porque execution.type é deepagent.
3. O parser DeepAgent converte o fragmento para AST dedicada.
4. O validator semântico bloqueia combinações inválidas, como HIL sem interrupt_on ou filesystem sem permissions.
5. O router resolve o modo deepagent e prepara o contexto de execução.
6. DeepAgentSupervisor inicializa factories, runtime cache, middlewares extras, store, backend e checkpointer.
7. O agente é criado com create_agent sob contrato governado.
8. O invoke produz resposta final, pausa HIL ou erro.
9. Se houver pausa HIL, a continuação usa Command resume reaproveitando o thread_id anterior.

## 10. Decisões técnicas e trade-offs

### Usar contrato governado em vez de repassar todo o poder da biblioteca

Ganho: reduz deriva arquitetural e evita comportamento não suportado pelo produto.

Custo: torna o sistema mais opinado e menos permissivo.

Impacto: mais segurança operacional e menos surpresa em produção.

### Exigir checkpointer quando HIL está habilitado

Ganho: garante retomada coerente do estado pausado.

Custo: obriga infraestrutura e configuração explícitas.

Impacto: é uma escolha correta, porque pausa sem persistência de estado seria uma falsa segurança.

### Aceitar deepagent_memory só com Redis

Ganho: reduz ambiguidade e força um backend compartilhado e observável.

Custo: limita experimentação com backends alternativos.

Impacto: coerência arquitetural e menor risco de memória não reprodutível.

### Separar middlewares oficiais da biblioteca e middlewares do produto

Ganho: o produto consegue auditar e enriquecer o runtime sem depender de comportamento oculto da lib.

Custo: mais trabalho de montagem.

Impacto: melhora telemetria, troubleshooting e previsibilidade.

## 11. Configurações que mudam o comportamento

As configurações mais relevantes confirmadas no código são estas.

- execution.type: define se o supervisor entra no alvo deepagent.
- middlewares.filesystem.enabled: ativa a camada de filesystem e passa a exigir permissions.
- middlewares.shell.enabled: ativa shell persistente governada.
- middlewares.memory.enabled: ativa uso de deepagent_memory.
- middlewares.subagents.enabled: permite subagentes síncronos e assíncronos.
- middlewares.background_execution_subagent.enabled: injeta o subagente automático de background, mas só se subagents também estiver on.
- middlewares.human_in_the_loop.enabled: ativa revisão humana e passa a exigir interrupt_on e checkpointer.
- middlewares.human_in_the_loop.async_approval: configura pedido assíncrono de aprovação.
- middlewares.skills.enabled: permite usar skills top-level e exige backend.
- interrupt_on: define quais tools podem gerar pausa humana e quais decisões são aceitas.
- permissions: define acesso de filesystem por operações read e write e por caminhos absolutos.
- deepagent_memory: define backend Redis, escopo, política, URL, prefixo e TTL.

## 12. O que acontece em caso de sucesso

No caminho feliz, o DeepAgent Supervisor monta o agente, reutiliza cache quando o hash e a versão permitem, resolve o modelo, aplica middlewares na ordem governada e devolve um payload normalizado com final_response, metrics, tools_usage, execution_timeline, mode deepagent e, quando necessário, bloco hil.

Quando o fluxo é assíncrono, o callback de progresso recebe estados intermediários e pode terminar em completed ou paused. Quando o fluxo entra em HIL, a resposta já sai estruturada para continuação posterior.

## 13. O que acontece em caso de erro

O código confirma alguns comportamentos importantes.

- Se a factory DeepAgent ou seus middlewares obrigatórios não estiverem disponíveis, a inicialização falha cedo com erro explícito.
- Se um campo top-level ainda não suportado, como planner, capabilities, memory top-level ou context_schema, aparecer no YAML, parser e validator tratam isso como erro.
- Se HIL estiver habilitado sem checkpointer, o validator bloqueia o contrato.
- Se filesystem estiver habilitado sem permissions, o validator bloqueia o contrato.
- Se deepagent_memory estiver habilitada sem Redis válido, o runtime falha explicitamente.
- Se o runtime não aceitar um parâmetro opcional suportado pelo YAML, o supervisor falha com ImportError em vez de mascarar o problema.

## 14. Observabilidade e diagnóstico

O DeepAgent Supervisor foi desenhado para contar a história da execução. Isso aparece em três níveis.

- Lifecycle trace do supervisor, com eventos de inicialização, execução, sucesso e erro.
- DeepAgentRuntimeTelemetry, que registra uso de tools, interrupções, resposta final e timeline.
- Logs estruturados no boundary HTTP e no runtime de background, com correlation_id, thread_id, status e markers.

Na prática, isso ajuda a distinguir se o problema veio de contrato YAML inválido, falha de runtime DeepAgent, falta de compatibilidade da factory, erro de tool, erro de continuação HIL ou política de middleware inconsistente.

## 15. Impacto técnico

Tecnicamente, esta feature empurra a plataforma para um desenho mais maduro de agentes. Ela reforça AST como fonte de verdade, validação semântica forte, separação entre contrato declarativo e montagem de runtime, uso de cache de artefato compilado e integração limpa com background execution e HIL.

## 16. Impacto executivo

Executivamente, o DeepAgent Supervisor reduz o gap entre “agente demonstrável” e “agente operável”. Ele transforma um conjunto de capacidades agentic avançadas em algo que pode ser governado, auditado e retomado sem improviso.

## 17. Impacto comercial

Comercialmente, isso permite posicionar a plataforma para casos que exigem coordenação entre tools, memória, revisão humana e automação parcial, sem vender uma autonomia descontrolada que o cliente depois não consegue aprovar internamente.

## 18. Impacto estratégico

Estrategicamente, o DeepAgent Supervisor aproxima a plataforma de uma camada agentic de alto valor, mas preservando a disciplina arquitetural do projeto. Ele funciona como ponte entre designer YAML, runtime governado e operação corporativa.

## 19. Explicação 101

Uma forma simples de entender o DeepAgent Supervisor é pensar nele como um gerente de operação assistido. Ele consegue falar com especialistas, usar ferramentas, guardar memória e pedir aprovação humana quando precisa. Mas ele só recebe esse poder depois que o sistema confere se o processo foi configurado corretamente, se o ambiente suporta o que foi pedido e se existe uma forma segura de pausar e continuar a execução.

Ou seja: ele não é só um agente mais forte. Ele é um agente mais forte com freios, contrato e trilha de auditoria.

## 20. Limites e pegadinhas

- DeepAgent não é sinônimo de qualquer supervisor agentic avançado. Ele é um alvo específico do assembly.
- HIL sem checkpointer não é aceito.
- Filesystem sem permissions explícitas não é aceito.
- Skills top-level sem middleware de skills habilitado não são aceitas.
- deepagent_memory não aceita backend arbitrário no runtime lido; o contrato exige Redis.
- O README antigo de DeepAgents não foi encontrado no acervo lido, então a nova documentação canônica precisou nascer diretamente do código real.

## 21. Troubleshooting

### Sintoma: YAML parece válido, mas o DeepAgent não inicializa

Causa provável: campo suportado pela AST, mas incompatível com a factory instalada, ou middleware obrigatório ausente no runtime.

Como confirmar: verifique logs de seleção da factory e mensagens de ImportError do bootstrap.

### Sintoma: HIL nunca pausa, ou pausa mas não continua

Causa provável: interrupt_on ausente, checkpointer ausente, thread_id inconsistente ou continuação fora do contrato.

Como confirmar: verifique o validator semântico, o payload hil normalizado e o uso de /agent/continue.

### Sintoma: memória DeepAgent não persiste

Causa provável: deepagent_memory desabilitada, scope inválido, policy incompatível ou URL Redis divergente do contrato global.

Como confirmar: verifique o bloco deepagent_memory, a URL Redis e os logs de configuração do store Redis.

## 22. Checklist de entendimento

- Entendi o que diferencia o DeepAgent Supervisor do supervisor clássico.
- Entendi o papel da AST dedicada.
- Entendi por que parser e validator têm regras próprias.
- Entendi a relação entre middlewares governados, HIL, permissions e memory.
- Entendi como o boundary HTTP escolhe mode deepagent.
- Entendi como a continuação HIL funciona.
- Entendi os limites e riscos operacionais reais.

## 23. Evidências no código

- src/agentic_layer/supervisor/deep_agent_supervisor.py
  - Motivo da leitura: confirmar bootstrap, cache, middlewares, memória, HIL e invoke.
  - Símbolos relevantes: DeepAgentSupervisor, _create_governed_deep_agent, _build_deepagent_store_backend, _compose_deepagents_extra_middleware.
  - Comportamento confirmado: runtime governado, cache por hash e versão, store Redis opcional e continuação por Command resume.

- src/config/agentic_assembly/ast/deepagent.py
  - Motivo da leitura: confirmar o contrato declarativo do DeepAgent.
  - Símbolos relevantes: DeepAgentSupervisorAST, DeepAgentMiddlewaresAST, DeepAgentAsyncApprovalAST.
  - Comportamento confirmado: sintaxe tipada para middlewares, permissions, interrupt_on, skills e async_subagents.

- src/config/agentic_assembly/parsers/deepagent_parser.py
  - Motivo da leitura: confirmar filtragem e bloqueio de campos removidos.
  - Símbolos relevantes: DeepAgentParser, collect_supervisor_contract_diagnostics.
  - Comportamento confirmado: parser só aceita execution.type deepagent e rejeita planner, capabilities, memory top-level e context_schema.

- src/config/agentic_assembly/validators/deepagent_semantic_validator.py
  - Motivo da leitura: confirmar regras semânticas do contrato.
  - Símbolos relevantes: DeepAgentSemanticValidator, \_validate_permissions_list, \_validate_interrupt_on_mapping.
  - Comportamento confirmado: HIL exige interrupt_on e checkpointer; filesystem exige permissions; deepagent_memory exige Redis.

- src/api/routers/agent_router.py
  - Motivo da leitura: confirmar contratos HTTP públicos.
  - Símbolos relevantes: /agent/execute, /agent/continue, /agent/hil/decisions.
  - Comportamento confirmado: execução deepagent síncrona, assíncrona e continuação HIL cliente-neutra.
