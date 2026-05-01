# Manual de Human-in-the-Loop

## 1. O que esta feature faz

Human-in-the-Loop é a capacidade da plataforma de pausar uma execução agentic sensível, pedir uma decisão humana e retomar o mesmo fluxo com o mesmo estado.

No código atual, essa capacidade existe em duas famílias principais:

1. DeepAgent com pausa antes de tool sensível.
2. Workflow com pausa por aprovação humana ou `human_gate`.

O objetivo da feature não é só pedir confirmação. O objetivo é manter a execução consistente, rastreável e retomável sem abrir um fluxo paralelo fora do runtime.

## 2. Que problema ela resolve

Sem HIL, uma automação sensível precisa escolher entre dois extremos ruins:

1. Executar tudo automaticamente, inclusive ações que exigem revisão humana.
2. Empurrar o processo inteiro para fora do runtime, perdendo contexto, thread e rastreabilidade.

O HIL resolve isso mantendo três garantias importantes:

1. A pausa acontece no próprio runtime, não como convenção textual frágil.
2. O estado da execução é preservado via thread e checkpointer.
3. A retomada usa um contrato formal, não um continue improvisado.

## 3. Conceitos que importam

### 3.1 `thread_id`

É o identificador lógico da execução pausada. A continuação precisa usar exatamente o mesmo valor. Se a thread mudar, o sistema deixa de retomar o processo original.

### 3.2 Envelope `hil`

No fluxo HTTP de agentes, a pausa humana é publicada ao cliente por um envelope `hil`.

O contrato público confirmado no router inclui:

1. `pending`.
2. `protocol_version`.
3. `message`.
4. `allowed_decisions`.
5. `action_requests`.
6. `review_configs`.
7. `resume_endpoint`.

O ponto central é este: a UI não deve tentar adivinhar pausa humana por parsing do texto da resposta. O gatilho oficial é `hil.pending=true`.

### 3.3 `interrupt_on`

No DeepAgent, `interrupt_on` define quais tools exigem revisão humana antes da execução real.

### 3.4 `human_gate`

No fluxo de workflow e em cenários mais genéricos, `human_gate` é a ferramenta que aciona `interrupt(...)` do LangGraph, pausa a execução e espera retomada posterior.

## 4. DeepAgent com HIL

### 4.1 Como o contrato é ligado

O código atual exige uma combinação explícita de configuração:

1. `execution.type=deepagent` no supervisor ativo.
2. `middlewares.human_in_the_loop.enabled=true`.
3. `interrupt_on` no supervisor.
4. `memory.checkpointer` ativo na raiz do YAML.

O validator semântico reforça essas regras. Se HIL estiver ligado sem `interrupt_on`, o assembly falha. Se `interrupt_on` existir com HIL desligado, o assembly também falha. Se faltar checkpointer, o contrato também é rejeitado.

### 4.2 O que o `interrupt_on` aceita

O comportamento confirmado no supervisor e no validator é estrito.

Cada entrada de `interrupt_on`:

1. precisa apontar para uma tool não vazia;
2. pode ser `bool` ou objeto;
3. quando for objeto, precisa trazer `allowed_decisions`;
4. só aceita `approve`, `edit` e `reject`;
5. não aceita decisões repetidas.

Isso significa que o contrato atual controla quais tool-calls exigem revisão humana e quais decisões são permitidas. Ele não suporta decisões arbitrárias fora desse trio.

### 4.3 O que o runtime faz

No `DeepAgentSupervisor`, quando o middleware de HIL está ligado, o runtime injeta `HumanInTheLoopMiddleware` com o mapa `interrupt_on` já normalizado.

Na prática, isso significa:

1. a pausa acontece antes da tool sensível rodar;
2. a revisão humana passa a ser parte do runtime governado;
3. a retomada posterior continua a mesma thread, em vez de abrir outra execução paralela.

## 5. Contrato HTTP de agentes

### 5.1 Entrada inicial com `/agent/execute`

O router de agentes publica o protocolo `hil-http-v1`.

Quando a execução pausa por HIL, a resposta do `POST /agent/execute` continua sendo HTTP bem-sucedido, mas o payload passa a indicar estado pausado. O exemplo do próprio router mostra:

1. `metrics.status=paused`;
2. `metrics.requires_human=true`;
3. `thread_id` presente;
4. envelope `hil` com decisões permitidas e configuração de revisão.

O significado prático é simples: a execução não falhou. Ela apenas entrou em um estado de espera por decisão humana.

### 5.2 Continuação com `/agent/continue`

O endpoint `POST /agent/continue` retoma a execução reutilizando o estado existente.

O fluxo confirmado no código é:

1. registrar o pedido de continuação;
2. resolver a configuração original;
3. selecionar o supervisor correto;
4. executar `Command(resume=...)` com o mesmo `thread_id`;
5. normalizar a saída para o cliente.

Isso é importante porque a retomada não é uma segunda execução do zero. Ela é a continuação formal do mesmo processo pausado.

### 5.3 Decisão assíncrona com `/agent/hil/decisions`

Além do continue direto, o produto também expõe `POST /agent/hil/decisions` para decisão HIL assíncrona.

Esse endpoint recebe token de aprovação, valida status e expiração e retoma a execução de forma atômica. O caso de uso canônico é aprovação humana vinda por canal externo ou fluxo assíncrono de revisão.

## 6. Workflow com HIL

### 6.1 Pausa em nodes e `human_gate`

No lado de workflow, a pausa humana aparece em dois lugares complementares:

1. nodes que montam resposta de bloqueio humano;
2. a tool `human_gate`, que chama `interrupt(...)` do LangGraph.

O `BaseNodeHandler` monta a resposta de bloqueio com:

1. `status=paused` quando a decisão ainda está pendente;
2. `status=failed` quando a aprovação é rejeitada;
3. `metadata.human_approvals` para auditoria;
4. `metadata.requires_human` para sinalizar a necessidade de intervenção.

### 6.2 Continuação com `/workflow/continue`

O endpoint `POST /workflow/continue` retoma uma execução pausada usando `thread_id` e `Command(resume=...)`.

O comportamento prático é parecido com agentes no ponto mais importante: a thread precisa ser a mesma, porque é ela que ancora o estado pausado no checkpointer.

## 7. Fluxo principal ponta a ponta

### 7.1 DeepAgent

O caminho feliz de DeepAgent com HIL fica assim:

1. o cliente chama `/agent/execute`;
2. o backend resolve YAML e autenticação;
3. o supervisor DeepAgent entra com HIL e `interrupt_on` ativos;
4. o runtime pausa antes da tool sensível;
5. o router devolve `thread_id` e envelope `hil`;
6. o humano aprova, edita ou rejeita;
7. o cliente chama `/agent/continue` ou um fluxo assíncrono usa `/agent/hil/decisions`;
8. a execução retoma a mesma thread.

### 7.2 Workflow

O caminho feliz de workflow com HIL fica assim:

1. o workflow executa até encontrar aprovação humana ou `human_gate`;
2. o runtime pausa e preserva o estado;
3. a resposta operacional informa que há dependência humana;
4. o cliente chama `/workflow/continue` com a mesma thread;
5. o workflow retoma do ponto pausado.

## 8. O que acontece em caso de sucesso

Quando tudo dá certo:

1. a pausa é registrada sem perder contexto;
2. a decisão humana fica auditável no metadata ou no contrato HIL;
3. a continuação reutiliza o mesmo `thread_id`;
4. o runtime conclui ou rejeita o fluxo de forma determinística.

## 9. O que acontece em caso de erro

Os erros confirmados no código mais importantes são estes:

### 9.1 Configuração inválida no DeepAgent

Se `interrupt_on` estiver mal formado, usar tool inexistente ou listar decisão inválida, o assembly rejeita o YAML.

### 9.2 HIL sem checkpointer

O contrato do DeepAgent exige checkpointer. Sem isso, a pausa humana não é tratada como configuração válida.

### 9.3 Continuação com thread errada

Se a thread de retomada não for a mesma da pausa, o runtime deixa de continuar o processo correto.

### 9.4 Decisão assíncrona para workflow

O serviço de decisão HIL assíncrona ainda não suporta retomada de workflow. O próprio código devolve erro explícito quando a tentativa é feita fora dos modos `agent` e `deepagent`.

## 10. Observabilidade e diagnóstico

Para diagnosticar HIL, faça estas perguntas na ordem certa:

1. A pausa é realmente HIL ou apenas erro de execução?
2. O payload trouxe `thread_id` e envelope `hil`?
3. O YAML tem checkpointer e contrato HIL coerentes?
4. A continuação usou exatamente a mesma thread?
5. A decisão enviada respeitou as decisões permitidas publicadas pelo backend?

No lado de agentes, o `agent_router` é a fonte principal para entender a publicação do envelope HIL. No lado de workflow, a combinação entre `BaseNodeHandler`, `human_gate` e `workflow_router` conta a história real da pausa e da retomada.

## 11. Limites e pegadinhas

Os limites reais do produto hoje são:

1. o trio de decisões permitido em DeepAgent é fixo: `approve`, `edit` e `reject`;
2. workflow e agentes não compartilham exatamente o mesmo contrato HTTP de continuação;
3. o endpoint de decisão assíncrona ainda não cobre workflow;
4. a UI não deve inventar decisões nem recalcular `thread_id`;
5. `human_gate` é uma ferramenta de pausa do runtime, não um input manual via terminal.

## 12. Troubleshooting rápido

### 12.1 O cliente não detectou a pausa humana

Causa provável: a interface está olhando texto da resposta em vez de `hil.pending`.

### 12.2 O DeepAgent não aceita o YAML com HIL

Causa provável: falta de `interrupt_on`, HIL ligado sem checkpointer ou decisões inválidas fora de `approve`, `edit` e `reject`.

### 12.3 A continuação não retoma o ponto certo

Causa provável: `thread_id` diferente da execução pausada.

### 12.4 A decisão assíncrona falhou em workflow

Causa provável: o suporte assíncrono atual está restrito a `agent` e `deepagent`.

## 13. Explicação 101

Pense no HIL como um freio oficial dentro do motor da execução.

Ele não joga a tarefa para fora e não manda o operador se virar depois. Ele pausa a esteira no ponto certo, guarda onde ela parou e espera a decisão humana. Quando a decisão chega, a esteira volta a andar do mesmo ponto.

É isso que diferencia HIL real de um fluxo improvisado por mensagem ou por texto solto de LLM.

## 14. Evidências no código

1. `src/api/routers/agent_router.py`: publica `hil-http-v1`, `thread_id`, `/agent/continue` e `/agent/hil/decisions`.
2. `src/api/services/agent_hil_continuation_service.py`: caso de uso interno de continuação HIL de agentes.
3. `src/api/services/hil_approval_decision_service.py`: decisão assíncrona e limite atual de suporte a workflow.
4. `src/agentic_layer/supervisor/deep_agent_supervisor.py`: normalização de `interrupt_on` e injeção de `HumanInTheLoopMiddleware`.
5. `src/config/agentic_assembly/validators/deepagent_semantic_validator.py`: validações canônicas de HIL, `interrupt_on` e checkpointer.
6. `src/agentic_layer/tools/system_tools/human_input.py`: implementação de `human_gate` via `interrupt(...)`.
7. `src/agentic_layer/workflow/nodes/base.py`: resposta padronizada de bloqueio humano em workflow.
8. `src/api/routers/workflow_router.py`: retomada formal de workflow pausado.
