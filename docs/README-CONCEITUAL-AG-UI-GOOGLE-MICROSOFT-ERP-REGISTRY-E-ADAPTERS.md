# Manual detalhado da etapa: Registry e adapters do AG-UI

## 1. O que esta etapa faz

Esta etapa conecta o protocolo comum aos runtimes e dominios concretos. Ela existe para permitir que AG-UI suporte agent, deepagent, workflow e dominios governados sem if hardcoded no core do lifecycle.

Em linguagem simples: e a tomada de adaptadores que diz quem executa cada executionKind.

## 2. Onde ela entra no fluxo

No codigo lido, o router resolve o AgUiAdapterRegistry padrao, transforma esse registro em mapping e entrega o mapping ao orchestrator. Quando um run chega, o orchestrator escolhe o adapter pelo execution_kind informado no contexto.

## 3. O que entra e o que sai

Entradas confirmadas:

- execution_kind textual vindo do request
- adapters registrados explicitamente
- services auxiliares para discovery e suporte de resume/HIL

Saidas confirmadas:

- mapping fechado de execution_kind para adapter
- catalogo publico de capabilities por execution_kind
- traducao do resultado de runtimes internos para eventos AG-UI

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. AgUiAdapterRegistry.default registra agent, deepagent, workflow e retail_demo;
2. register rejeita execution_kind vazio e registro duplicado;
3. as_mapping devolve copia por convencao imutavel do catalogo;
4. AgUiCapabilitiesService.default publica um menu publico por execution_kind;
5. agent e deepagent usam adapters que traduzem seus runtimes internos para AgUiRuntimeExecutionResult;
6. esses adapters reaproveitam suporte comum para extrair mensagem, resolver YAML, retomar HIL e registrar pausas;
7. workflow tambem adapta o resultado do WorkflowOrchestrator, mas hoje rejeita resume AG-UI;
8. retail_demo pluga um dominio governado proprio sem depender dos runtimes agentic.

O detalhe mais importante e que extensibilidade aqui nao significa improviso. Existe catalogo explicito, contrato unico e falha fechada quando um execution_kind nao foi registrado.

## 5. Decisoes tecnicas importantes

### 5.1. Registry explicito evita fallback escondido

O catalogo default registra apenas quatro execution kinds. Se um quinto tipo for necessario, ele precisa ser plugado conscientemente.

### 5.2. Agent e deepagent compartilham suporte de resume/HIL

Os adapters agentic reaproveitam execute_agentic_resume e register_agentic_hil_pause. Isso centraliza a ponte com o caso de uso canônico de continuacao humana.

### 5.3. Workflow ainda tem limite claro de resume

O adapter de workflow executa e traduz o resultado, mas quando context.resume chega ele falha explicitamente com AG_UI_WORKFLOW_RESUME_UNSUPPORTED. Isso e importante porque evita fingir suporte a uma continuacao que ainda nao esta fechada nesse contrato.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- execution_kind nao registrado quebra antes de qualquer runtime rodar;
- registrar duas vezes o mesmo execution_kind gera erro imediato;
- esquecer de expor capability publica para um novo adapter cria assimetria entre discovery e execucao;
- workflow ainda nao oferece resume AG-UI equivalente ao caminho agentic;
- adapter novo que nao normalize direito seu resultado compromete o contrato do stream.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- erro AG_UI_ADAPTER_NOT_FOUND ao tentar um executionKind novo;
- lista de execution kinds conhecidos pelo registry;
- discovery de capabilities sem o execution_kind esperado;
- erro AG_UI_WORKFLOW_RESUME_UNSUPPORTED quando a UI tenta retomar workflow pelo mesmo contrato de agent;
- pausa HIL sem retomada funcional quando o adapter nao usa o suporte comum correto.

Em linguagem simples: se AG-UI parece pouco generico, esta camada mostra exatamente onde a generalizacao esta ou nao esta de fato implementada.

## 8. Exemplo pratico guiado

Cenario: o produto quer adicionar um execution_kind novo para um modulo financeiro.

1. cria-se um adapter que traduza o runtime financeiro para eventos AG-UI;
2. registra-se esse adapter no registry;
3. publica-se a capability correspondente no discovery;
4. o orchestrator passa a resolvelo sem mudar sua propria logica de lifecycle.

O valor desta etapa e permitir crescimento do slice sem reescrever o core a cada novo dominio.

## 9. Evidencias no codigo

- src/api/services/ag_ui_adapter_registry.py
  - Simbolo relevante: AgUiAdapterRegistry.default
  - Comportamento confirmado: registro explicito de agent, deepagent, workflow e retail_demo.
- src/api/services/ag_ui_adapter_registry.py
  - Simbolo relevante: register
  - Comportamento confirmado: proibicao de execution_kind vazio ou duplicado.
- src/api/services/ag_ui_capabilities_service.py
  - Simbolo relevante: AgUiCapabilitiesService.default
  - Comportamento confirmado: discovery publico por execution_kind, incluindo catalogo retail_demo.
- src/api/services/ag_ui_agent_adapter.py
  - Simbolo relevante: execute com execute_agentic_resume e emit_runtime_result_events
  - Comportamento confirmado: adapter agent reutiliza suporte comum de resume e HIL.
- src/api/services/ag_ui_deepagent_adapter.py
  - Simbolo relevante: AgUiDeepAgentAdapter
  - Comportamento confirmado: deepagent segue o mesmo padrao de traducao e resume do caminho agentic.
- src/api/services/ag_ui_workflow_adapter.py
  - Simbolo relevante: AgUiWorkflowAdapter.execute
  - Comportamento confirmado: workflow executa e traduz, mas ainda rejeita resume AG-UI.
