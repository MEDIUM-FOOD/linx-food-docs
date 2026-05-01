# Tutorial 101: DeepAgent sem mistério

Este tutorial explica DeepAgent do jeito mais prático possível.

Objetivo:

1. Entender quando usar DeepAgent.
2. Saber ligar o modo certo no YAML.
3. Entender o papel de cada middleware em linguagem simples.
4. Evitar os erros de configuração que mais quebram a operação.

## Leitura relacionada

- Aprofundamento técnico completo: [README-DEEPAGENTS-SUPERVISOR.md](./README-DEEPAGENTS-SUPERVISOR.md)
- HIL com payloads reais de pausa e retomada: [tutorial-101-exemplos-api-deepagent-hil-execute-continue.md](./tutorial-101-exemplos-api-deepagent-hil-execute-continue.md)

## 1. O que é DeepAgent

DeepAgent é o modo do produto usado quando a conversa precisa mais do que uma chamada simples.

Em linguagem simples:

1. Ele pode lembrar contexto entre execuções.
2. Ele pode usar especialistas dentro da conversa.
3. Ele pode parar uma ação para revisão humana.
4. Ele pode governar acesso a arquivos, shell e dados sensíveis.

Se o supervisor clássico é um gerente, o DeepAgent é um gerente com controles extras em volta dele.

## 2. Como o DeepAgent é ativado

O modo só ativa quando o supervisor certo junta estas três condições:

1. `selected_supervisor` aponta para ele.
2. O item em `multi_agents[]` está `enabled=true`.
3. `execution.type=deepagent`.

Exemplo base:

Leitura prática do exemplo base:

1. `selected_supervisor` aponta para `sup_deep`.
2. Em `multi_agents`, o item `sup_deep` precisa estar habilitado.
3. `execution.type=deepagent` liga o runtime correto.
4. `execution.default_mode=direct_async` define o modo inicial de execução.

## 3. A nova regra mais importante

O contrato do DeepAgent usa `middlewares`, não `capabilities`.

Isso importa porque:

1. `middlewares` é o interruptor oficial do runtime.
2. O produto não depende mais dos defaults escondidos da lib externa.
3. O mesmo contrato vale para o YAML canônico, para o runtime e para a validação.

## 4. Matriz default oficial

| Middleware | Default | Significado simples |
| --- | --- | --- |
| `filesystem` | `on` | Acesso governado a arquivo. |
| `shell` | `off` | Execução de comando de shell pela tool `execute`. |
| `memory` | `on` | Uso de memória do DeepAgent. |
| `subagents` | `on` | Especialistas dentro da conversa. |
| `human_in_the_loop` | `off` | Revisão humana por tool, com pausa e retomada da mesma thread. |
| `summarization` | `off` | Resumo de contexto longo. |
| `pii` | `on` | Proteção de dados sensíveis. |
| `todo_list` | `on` | Lista de tarefas da execução. |
| `skills` | `off` | Skills declarativas do agente principal. |

## 5. O bloco recomendado no YAML

Leitura prática do bloco recomendado:

1. `filesystem` ligado e `shell` desligado formam o baseline mais seguro para começar.
2. `memory` e `subagents` ligados mantêm memória e especialistas ativos.
3. `human_in_the_loop` e `summarization` podem começar desligados quando o caso ainda não exige pausa humana nem resumo de contexto.
4. `pii` e `todo_list` ligados reforçam proteção de dados e disciplina operacional.
5. `skills` pode permanecer desligado até existir necessidade real e bloco `skills` top-level coerente.

## 6. O que cada middleware faz, sem jargão

### 6.1 Filesystem

Serve para o agente trabalhar com arquivo local.

Mas atenção:

1. Ele não abre o disco inteiro.
2. Ele só funciona com `permissions[]`.
3. Sem policy explícita, o validator bloqueia o YAML.

### 6.2 Shell

Serve para expor a tool `execute`.

Na prática:

1. Shell e filesystem são separados.
2. O comando `execute` fica só no middleware de shell.
3. Você escolhe a policy de execução: `host`, `docker` ou `codex_sandbox`.

### 6.3 Memory

Serve para colocar memória do DeepAgent em uso.

Importante:

1. `middlewares.memory` não é igual a `deepagent_memory`.
2. `deepagent_memory` cuida do store persistente.
3. `middlewares.memory` cuida do middleware que usa essa memória no runtime.

### 6.4 Subagents

Serve para ligar especialistas.

Tem dois tipos:

1. `agents[]`: especialistas síncronos, dentro da conversa.
2. `async_subagents[]`: especialistas remotos de longa duração.

### 6.5 Human in the loop

Serve para pedir aprovação humana.

Você usa quando uma tool não deve continuar sozinha, como:

1. enviar e-mail,
2. executar ação sensível,
3. confirmar mudança importante.

O ponto mais importante para quem nunca usou:

1. No DeepAgent atual, você não precisa criar uma tool HIL separada para começar.
2. O caminho padrão é ligar o middleware `human_in_the_loop` e dizer, em `interrupt_on`, quais tools devem parar.
3. O runtime pausa antes da execução da tool, mostra o pedido de revisão humana e espera a resposta.
4. Depois da decisão, a mesma thread continua do ponto onde parou.

As três decisões possíveis são:

1. `approve`: executa a tool exatamente como o agente propôs.
2. `edit`: muda os argumentos antes de executar.
3. `reject`: não executa a tool.

O contrato HTTP público é único:

1. `/agent/execute` devolve `hil.pending=true` quando há pausa humana.
2. O mesmo bloco `hil` serve para uma tela simples com aprovar/rejeitar e para uma tela rica com `action_requests`, `review_configs` e `edit`.
3. `/agent/continue` valida no servidor se a pausa realmente existe antes de retomar.
4. O cliente usa o `thread_id` publicado pelo backend e não tenta recalcular esse valor.

O que você NÃO precisa para a primeira adoção:

1. Não precisa começar por `human_gate`.
2. Não precisa criar uma tool manual de aprovação.
3. Não precisa modelar um workflow separado só para aprovar uma tool sensível.

Quando a tool HIL customizada entra em cena:

1. Quando você quer pedir aprovação no meio de uma tool própria.
2. Quando o caso é workflow geral do produto e não a revisão padrão de tool do DeepAgent.
3. Quando você quer um formulário humano customizado, diferente do fluxo normal de `interrupt_on`.

### 6.5.1 Por onde começar sem erro

Faça nesta ordem:

1. Ligue `middlewares.human_in_the_loop.enabled=true`.
2. Declare `interrupt_on` com o nome exato da tool que deve parar.
3. Garanta um checkpointer ativo na raiz `memory.checkpointer`.
4. Execute o agente pela API e guarde o `thread_id` publicado pelo backend.
5. Quando houver pausa, retome usando exatamente o mesmo `thread_id`.

### 6.6 Summarization

Serve para resumir contexto antigo.

Use quando:

1. a conversa é longa,
2. o contexto cresce demais,
3. você quer manter custo e clareza sob controle.

### 6.7 PII

Serve para proteger dado sensível.

Exemplos comuns:

1. e-mail,
2. cartão,
3. IP,
4. MAC address,
5. URL.

### 6.8 Todo list

Serve para o agente manter uma lista explícita de passos.

Isso ajuda quando o trabalho é grande, investigativo ou depende de sequência.

### 6.9 Skills

Serve para ligar skills declarativas no agente principal.

Regra prática:

1. `middlewares.skills.enabled=true` exige `skills` top-level.
2. `skills` top-level sem middleware ligado é erro.

## 7. Exemplo mínimo completo de DeepAgent com HIL e checkpointer

Este exemplo foi montado para ser o menor recorte realmente útil do produto atual.

Ele já inclui:

1. checkpointer na raiz do YAML,
2. seleção explícita do supervisor,
3. DeepAgent ligado de verdade,
4. HIL ligado de verdade,
5. uma tool real do catálogo central para você testar a pausa.

Leitura prática do exemplo mínimo com HIL e checkpointer:

1. A raiz ativa `memory.checkpointer` com backend PostgreSQL e connection string via placeholder `${USER_MEMORY_DATABASE_DSN}`.
2. `selected_supervisor` escolhe `sup_deep_hil_minimo` como supervisor ativo.
3. O supervisor fica habilitado com `execution.type=deepagent` e `default_mode=direct_sync`.
4. O prompt do supervisor deixa explícito que a tool `calculator` depende de revisão humana.
5. Em `middlewares`, filesystem e shell ficam desligados, subagents, pii e todo_list ficam ligados, e `human_in_the_loop` também fica ligado.
6. `interrupt_on.calculator` aceita as decisões `approve`, `edit` e `reject`.
7. O agente `especialista_calculo` usa a tool `calculator` e recebe instrução explícita para não inventar resultado.

Leitura simples:

1. O bloco `memory.checkpointer` é o que permite pausar e retomar a mesma thread.
2. O bloco `middlewares.human_in_the_loop` liga a pausa humana no runtime.
3. O bloco `interrupt_on.calculator` diz qual tool deve parar.
4. O bloco `agents[]` declara um especialista real com uma tool real do catálogo.

Por que usei `calculator` neste exemplo:

1. porque ela já existe no catálogo central do produto,
2. porque não depende de credencial externa,
3. porque deixa o primeiro teste de HIL mais barato e mais previsível.

Como adaptar para um caso de produção:

1. troque `calculator` pela tool sensível real, como `email_sender`, `slack_send_message` ou uma `dyn_api<...>` do seu tenant,
2. mantenha `interrupt_on` com o mesmo nome exato da tool,
3. só ligue `filesystem`, `shell`, `skills` ou `deepagent_memory` quando o caso realmente exigir.

Observação operacional importante:

1. o resolvedor aceita `sqlite`, mas o contrato operacional do produto em cloud não deve depender de filesystem local persistente;
2. por isso este exemplo já nasce com `postgresql`, que é o caminho seguro para o checkpointer no ambiente real.

Fluxo do zero, sem mistério:

1. Você chama a execução inicial.
2. O agente tenta usar `calculator`.
3. O runtime pausa antes de executar a tool.
4. O humano aprova, edita ou rejeita.
5. A execução continua na mesma thread.

Se estiver usando a API do produto:

1. A primeira chamada nasce em `/agent/execute`.
2. A retomada acontece em `/agent/continue`.
3. O `thread_id` da retomada precisa ser exatamente o valor publicado na pausa.
4. O tutorial com payloads reais está em [tutorial-101-exemplos-api-deepagent-hil-execute-continue.md](./tutorial-101-exemplos-api-deepagent-hil-execute-continue.md).

## 8. Exemplo de shell ligado

Leitura prática do exemplo com shell ligado:

1. `filesystem` e `shell` ficam ligados ao mesmo tempo.
2. O shell usa `workspace_root=/workspace`, expõe a tool `execute` e pode subir com `startup_commands` simples, como `pwd`.
3. A policy de execução fica em `execution_policy.type=host`.
4. `permissions` precisa liberar explicitamente as operações e caminhos necessários, como leitura em `/memories/**`.
5. Os demais middlewares podem seguir no baseline seguro: memory, subagents, pii e todo_list ligados; HIL, summarization e skills desligados.

## 9. Exemplo de skills no agente principal

Leitura prática do exemplo com skills:

1. `middlewares.skills.enabled=true` liga o middleware de skills no agente principal.
2. A lista top-level `skills` precisa existir e pode apontar para um diretório como `/skills/main/`.
3. O `response_format` pode exigir resposta estruturada, por exemplo um objeto com campo obrigatório `summary`.
4. Os outros middlewares podem continuar no arranjo padrão, com filesystem, memory, subagents, pii e todo_list ativos, e shell, HIL e summarization desligados.

## 10. O que aparece no log quando o agente nasce

Na inicialização, o runtime registra um plano governado de middlewares.

Na prática, esse log mostra:

1. o que ficou `on`,
2. o que ficou `off`,
3. a `shell_policy`,
4. as `memory_sources`,
5. a lista de classes de middleware que realmente entrou no runtime.

Para HIL, confira duas pistas simples:

1. `Plano governado de middlewares do DeepAgent` deve mostrar `human_in_the_loop=on`.
2. `Plano de factory DeepAgent` deve mostrar `checkpointer=on` e `interrupt_on=on`.

Se você quer conferir se o YAML foi respeitado, esse é o log mais útil.

## 11. Erros que mais acontecem

1. Ligar `filesystem` sem `permissions[]`.
2. Declarar `permissions[]` com `filesystem` desligado.
3. Ligar `human_in_the_loop` sem `interrupt_on`.
4. Declarar `interrupt_on` sem ligar `human_in_the_loop`.
5. Ligar `skills` sem preencher `skills` top-level.
6. Ligar `memory` sem `deepagent_memory.enabled=true`.
7. Usar `headers` em `async_subagents[]` sem `url`.
8. Tentar usar `memory` ou `context_schema` no topo do supervisor DeepAgent.
9. Ligar HIL e esquecer o checkpointer operacional na raiz `memory.checkpointer`.
10. Tentar retomar a execução com `thread_id` diferente do valor publicado pelo backend.

## 12. Checklist final

Antes de publicar um YAML DeepAgent, confirme:

1. `selected_supervisor` está correto.
2. `execution.type=deepagent` está correto.
3. O bloco `middlewares` está explícito.
4. `permissions[]` existe se `filesystem` estiver ligado.
5. `interrupt_on` existe se HIL estiver ligado.
6. O checkpointer operacional existe se HIL estiver ligado.
7. A retomada vai usar exatamente o `thread_id` publicado pelo backend na pausa.
8. `skills` existe se `middlewares.skills.enabled=true`.
9. `deepagent_memory` existe e está coerente com `middlewares.memory`.
10. O log do plano de middlewares apareceu na inicialização.

## 13. Resumo em uma frase

DeepAgent é o modo certo quando você precisa memória persistente, especialistas, revisão humana e governança local no mesmo fluxo, e agora tudo isso é controlado explicitamente por `middlewares`.
