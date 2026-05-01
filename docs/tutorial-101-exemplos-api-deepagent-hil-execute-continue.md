# Tutorial 101 com Exemplos Completos de Uso da API: DeepAgent HIL com /agent/execute e /agent/continue

Objetivo deste documento:

1. mostrar o payload HTTP real de pausa do DeepAgent no produto atual,
2. mostrar o payload HTTP real aceito por `/agent/continue`,
3. explicar em linguagem simples como reaproveitar `correlation_id` e consumir o `thread_id` publicado pelo backend,
4. evitar que você monte uma interface de aprovação em cima de lógica escondida no frontend.

## 1. O que este tutorial cobre de verdade

Ele cobre o fluxo HTTP atual do produto, com base em:

1. `src/api/routers/agent_router.py`,
2. `tests/integration/hil/test_hil_deepagent_flow.py`,
3. o contrato atual do `DeepAgentSupervisor`.

Em linguagem simples:

1. `/agent/execute` cria a execução inicial,
2. `/agent/continue` retoma a mesma execução pausada,
3. a pausa deve ser detectada pelo bloco `hil.pending=true`,
4. `metrics.status="paused"` e `metrics.requires_human=true` continuam existindo como compatibilidade observável,
5. o payload oficial de retomada já é tipado e aceito pelo endpoint.

## 2. O que este tutorial promete e o que ele não promete

Ele promete o contrato HTTP único do produto para HIL no DeepAgent.

Leitura prática:

1. o response model público de `/agent/execute` expõe `thread_id` como campo formal,
2. o response model público de `/agent/execute` expõe um bloco `hil` cliente-neutro,
3. esse bloco pode ser usado por uma tela simples com apenas aprovar e rejeitar,
4. o mesmo bloco também carrega `action_requests`, `review_configs`, `interrupt_id` e `interrupt_ids` quando o runtime publica dados ricos,
5. `/agent/continue` aceita `approve`, `edit` e `reject` no mesmo formato de retomada.

Ele não promete compatibilidade direta com uma UI oficial externa sem adaptação.

Isso importa porque:

1. o contrato do produto é estável e pode alimentar qualquer frontend,
2. uma tela simples não precisa conhecer todos os detalhes avançados,
3. uma tela rica pode usar os campos extras sem criar outro fluxo paralelo,
4. o cliente não deve recomputar `thread_id`, nem inventar decisões fora de `hil.allowed_decisions`.

## 3. Pré-requisitos antes da primeira chamada

Você precisa de quatro coisas:

1. um YAML DeepAgent com `middlewares.human_in_the_loop.enabled=true`,
2. `interrupt_on` com o nome real da tool que deve parar,
3. `memory.checkpointer.enabled=true` na raiz,
4. `mode="deepagent"` na request de `/agent/execute`.

Se você ainda não montou o YAML mínimo, use primeiro o exemplo de [tutorial-101-deepagents.md](./tutorial-101-deepagents.md).

## 4. Request real de /agent/execute

Exemplo mínimo do request:

```json
{
  "task": "Quanto é 241 * 17?",
  "user_email": "analista@empresa.com",
  "execution_mode": "direct_sync",
  "mode": "deepagent",
  "encrypted_data": {
    "cipher": "<payload-cifrado-do-yaml>"
  }
}
```

Leitura simples:

1. `task` é a pergunta ou tarefa do usuário,
2. `execution_mode="direct_sync"` ajuda no primeiro teste porque devolve a pausa na mesma resposta HTTP,
3. `mode="deepagent"` força o supervisor DeepAgent,
4. `encrypted_data` é o caminho padrão do produto para transportar o YAML resolvido.

## 5. Response real de pausa em /agent/execute

Exemplo de resposta pausada no contrato atual:

```json
{
  "task": "Quanto é 241 * 17?",
  "response": "DeepAgent pausado aguardando aprovação humana",
  "steps": ["gate_deepagent"],
  "tools_used": [],
  "metrics": {
    "status": "paused",
    "requires_human": true,
    "mode": "deepagent"
  },
  "processing_time_ms": 87.4,
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "thread_id": "agent_0f4d7c2b8a9e4d31b72fa2c9607b6f21",
  "success": true,
  "error": null,
  "hil": {
    "pending": true,
    "protocol_version": "hil-http-v1",
    "message": "DeepAgent pausado aguardando aprovação humana",
    "allowed_decisions": ["approve", "edit", "reject"],
    "action_requests": [
      {
        "name": "calculator",
        "args": {
          "expression": "241 * 17"
        },
        "description": "Revisar chamada da ferramenta calculator antes da execução."
      }
    ],
    "review_configs": [
      {
        "action_name": "calculator",
        "allowed_decisions": ["approve", "edit", "reject"]
      }
    ],
    "interrupt_id": "interrupt-calculator-1",
    "interrupt_ids": ["interrupt-calculator-1"],
    "resume_endpoint": "/agent/continue"
  },
  "execution_mode": "direct_sync"
}
```

O que é garantido nesse payload hoje:

1. `thread_id` já vem pronto como campo público formal,
2. `hil.pending=true` é o gatilho oficial para qualquer cliente detectar a pausa humana,
3. `hil.allowed_decisions` publica o que a UI pode renderizar como decisões,
4. `hil.resume_endpoint` publica o endpoint oficial de retomada,
5. `correlation_id` continua sendo o identificador lógico da execução,
6. `hil.action_requests` publica quais ações precisam de revisão,
7. `hil.review_configs` publica quais decisões são permitidas por ação,
8. `metrics.status="paused"` e `metrics.requires_human=true` continuam disponíveis para observabilidade e compatibilidade.

Como usar isso em uma tela simples:

1. se `hil.allowed_decisions` vier apenas com `approve` e `reject`, renderize só dois botões,
2. se vier também `edit`, renderize uma opção de edição,
3. se a tela não souber editar, ela deve esconder `edit` e não enviar essa decisão,
4. a tela simples continua usando o mesmo `hil`, sem endpoint alternativo.

Como usar isso em uma tela rica:

1. mostre cada item de `hil.action_requests` para o humano,
2. mostre os argumentos em `args` de forma legível,
3. respeite `review_configs` para saber quais decisões cabem em cada ação,
4. envie uma decisão por ação, na mesma ordem recebida.

Como isso vira Generative UI do HIL:

1. a tela não precisa conhecer previamente cada tool do produto,
2. ela lê o envelope `hil` e monta a revisão a partir dos campos recebidos,
3. `action_requests` vira a lista de ações a revisar,
4. `review_configs` e `allowed_decisions` viram os controles permitidos,
5. `resume_endpoint`, `thread_id` e `correlation_id` orientam a retomada,
6. `edit` só deve aparecer quando a tela souber montar `edited_action` de
  forma correta.

Neste contexto, “Generative UI” não quer dizer que o agente envia HTML ou
JavaScript para o browser executar. Quer dizer que a tela gera a
experiência de revisão usando componentes seguros do próprio produto, com
base no contrato estruturado publicado pelo backend.

### Como o frontend compartilhado usa esse contrato

No frontend atual, o fluxo foi separado em duas peças para evitar que cada
tela reinvente HIL.

1. `HilContract` normaliza o envelope publicado pelo backend.
2. Essa mesma peça monta `resume.decisions` na ordem correta das ações.
3. `HilReviewPanel` recebe o contrato já normalizado e renderiza a
  revisão visual.
4. A tela dona do fluxo continua responsável apenas por ligar o callback
  de decisão ao seu request local de retomada.

Em linguagem simples: o backend decide o que precisa de aprovação,
`HilContract` organiza a ficha e `HilReviewPanel` mostra a
ficha para o operador. A tela não precisa criar um painel novo toda vez
que surgir uma tool sensível diferente.

## 6. Como obter o thread_id agora

Leitura honesta do contrato atual:

1. o backend publica o `thread_id` no response de pausa,
2. o valor é opaco, ou seja, o cliente não deve tentar entender nem reconstruir,
3. ele não é mais o prefixo simples do `correlation_id`,
4. a obrigação do cliente é guardar e reenviar exatamente o mesmo valor no `/agent/continue`.

## 7. Request real de /agent/continue com approve

Exemplo mínimo de retomada com aprovação:

```json
{
  "resume": {
    "decisions": [
      {
        "type": "approve"
      }
    ]
  },
  "user_email": "analista@empresa.com",
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "thread_id": "agent_0f4d7c2b8a9e4d31b72fa2c9607b6f21",
  "mode": "deepagent",
  "yaml_config": "app/yaml/hil-deepagent-minimo.yaml"
}
```

O que cada campo faz na prática:

1. `resume.decisions` é a lista oficial de decisões humanas,
2. `correlation_id` precisa ser o mesmo da chamada inicial,
3. `thread_id` precisa apontar para a mesma thread pausada,
4. `mode="deepagent"` garante que o router retome o supervisor correto,
5. `yaml_config` ou `encrypted_data` servem para o endpoint recuperar a configuração do agente.

## 8. Response real de /agent/continue com approve

Exemplo de resposta concluída:

```json
{
  "resume": {
    "decisions": [
      {
        "type": "approve"
      }
    ]
  },
  "response": "DeepAgent retomado e finalizado",
  "steps": ["continue_deepagent"],
  "tools_used": [],
  "metrics": {
    "status": "completed",
    "mode": "deepagent",
    "stage": "continue"
  },
  "processing_time_ms": 94.1,
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "thread_id": "agent_0f4d7c2b8a9e4d31b72fa2c9607b6f21",
  "success": true,
  "error": null
}
```

Leitura simples:

1. o endpoint ecoa `resume`,
2. a resposta final já volta normalizada para cliente HTTP,
3. `thread_id` agora aparece na resposta do continue,
4. `metrics.status="completed"` mostra que a retomada terminou.

## 9. Request real de /agent/continue com edit

O endpoint `/agent/continue` aceita `edit` no mesmo contrato usado por `approve` e `reject`.

Leitura prática:

1. `hil.allowed_decisions` é a fonte de verdade do que um cliente pode renderizar,
2. se `edit` aparecer em `hil.allowed_decisions`, o cliente pode enviar `type="edit"`,
3. `edited_action` informa a tool revisada e os argumentos corrigidos,
4. isso não cria outro fluxo; é a mesma lista `resume.decisions`.

Exemplo de payload com edição:

```json
{
  "resume": {
    "decisions": [
      {
        "type": "edit",
        "edited_action": {
          "name": "calculator",
          "args": {
            "expression": "241 * 17"
          }
        }
      }
    ]
  },
  "user_email": "analista@empresa.com",
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "thread_id": "agent_0f4d7c2b8a9e4d31b72fa2c9607b6f21",
  "mode": "deepagent",
  "yaml_config": "app/yaml/hil-deepagent-minimo.yaml"
}
```

Regra prática:

1. `edited_action` só pode existir quando `type="edit"`,
2. `name` deve ser o nome da tool que estava interrompida,
3. `args` precisa respeitar o contrato real da tool escolhida.

## 10. Request real de /agent/continue com reject

Quando a decisão humana for rejeitar a ação, o shape oficial é este:

```json
{
  "resume": {
    "decisions": [
      {
        "type": "reject"
      }
    ]
  },
  "user_email": "analista@empresa.com",
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "thread_id": "agent_0f4d7c2b8a9e4d31b72fa2c9607b6f21",
  "mode": "deepagent",
  "yaml_config": "app/yaml/hil-deepagent-minimo.yaml"
}
```

Leitura prática:

1. o endpoint aceita `reject`,
2. o efeito final depende do supervisor e do fluxo da tool interrompida,
3. a resposta final costuma sinalizar rejeição em `metrics.status`, em vez de parecer uma conclusão normal.

## 11. A ordem das decisions importa

O contrato do continue assume que as decisões humanas chegam na mesma ordem das ações pendentes.

Em linguagem simples:

1. `hil.action_requests` é a lista de ações pendentes,
2. `resume.decisions` precisa ter a mesma quantidade de itens,
3. a primeira decisão responde à primeira action request,
4. a segunda decisão responde à segunda action request,
5. não reorganize a lista por conta própria no frontend.

## 12. O que um cliente simples precisa fazer

Fluxo mínimo de integração:

1. chamar `/agent/execute`,
2. detectar `hil.pending=true`,
3. guardar `correlation_id`,
4. guardar `thread_id` recebido do backend,
5. guardar `hil.resume_endpoint`,
6. renderizar apenas as decisões que a sua tela suporta dentro de `hil.allowed_decisions`,
7. montar `resume.decisions` com uma decisão por `hil.action_requests`,
8. enviar `/agent/continue` usando o `resume_endpoint` publicado.

## 13. Erros mais comuns no onboarding

1. esquecer `mode="deepagent"` na request inicial,
2. tentar retomar com outro `correlation_id`,
3. tentar retomar com outro `thread_id`,
4. ignorar `hil.allowed_decisions` e inventar botões que o contrato não publicou,
5. enviar `edited_action` fora de `type="edit"`,
6. usar um nome de tool diferente do que o agente realmente interrompeu,
7. enviar quantidade de decisões diferente da quantidade de `hil.action_requests`,
8. ligar HIL no YAML sem checkpointer ativo na raiz.

## 14. Resumo direto

Se você nunca usou esse fluxo, memorize só isso:

1. execute chama `/agent/execute`,
2. pausa aparece por `hil.pending=true`,
3. continue chama `/agent/continue`,
4. o `thread_id` já vem pronto no execute e deve ser reaproveitado sem recomputar,
5. o `resume` é sempre uma lista `decisions`, uma por `action_requests`,
6. tela simples e tela rica usam o mesmo contrato `hil`,
7. `correlation_id` e `thread_id` precisam apontar para a mesma execução.
