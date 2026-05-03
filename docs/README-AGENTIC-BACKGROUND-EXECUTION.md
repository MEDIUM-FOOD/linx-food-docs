# Execução Agentic em Background

Este documento explica a capacidade de executar agentes, deepagents e workflows em background. O foco é deixar claro o modelo mental, o que é persistido, como acompanhar uma execução e quais limites operacionais existem hoje.

Se a dúvida específica for como essa capacidade se conecta ao processo
de scheduler, ao pedido preservado em linguagem natural e à comunicação
HIL assíncrona, complemente esta leitura com
[README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md)
e
[README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](./README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md).

## 1. O que esta capacidade resolve

Nem todo pedido feito a um agente precisa responder diretamente na tela do chat.

Alguns pedidos são melhores como trabalho em segundo plano. Exemplos práticos:

1. gerar um relatório diário;
2. rodar uma análise demorada;
3. executar uma rotina agentic em horário definido;
4. acompanhar uma operação sem prender a interface do usuário;
5. deixar o worker executar o runtime real com rastreabilidade.

Em linguagem simples: o usuário continua pedindo algo em formato de prompt, mas a resposta não precisa voltar imediatamente para o chat. A plataforma transforma esse pedido em uma solicitação de execução background.

## 2. Conceito principal

O item agendado não é apenas um agente ou workflow estático.

O item agendado é uma solicitação de execução. Essa solicitação guarda o comando que normalmente seria enviado pelo usuário no prompt, mais o alvo que deve executar esse comando.

Isso importa porque preserva a intenção original do usuário. A plataforma não agenda apenas “rode o agente X”. Ela agenda “rode este pedido do usuário no agente, deepagent ou workflow indicado”.

## 3. Peças do fluxo

### 3.1. Solicitação

A solicitação representa o pedido original do usuário.

Ela guarda informações como comando solicitado, comando normalizado, payload de entrada, tenant, usuário, alvo e snapshot seguro da configuração YAML.

### 3.2. Agenda

A agenda define quando a solicitação deve virar execução real.

Ela pode representar execução única, intervalo recorrente ou cron. Cron é a forma textual comum de dizer “execute nesse calendário”, por exemplo todo dia em certo horário.

### 3.3. Run

Run é a execução concreta.

Uma mesma solicitação pode gerar vários runs ao longo do tempo quando a agenda é recorrente. Cada run tem status próprio, `correlation_id`, telemetria, resultado final ou erro.

### 3.4. Worker

O worker é o processo que executa o trabalho pesado fora da API.

No fluxo background, a API e o scheduler apenas preparam e despacham o trabalho. A execução agentic real acontece pelo worker oficial, sem criar fila paralela ou caminho alternativo.

## 4. Status importantes

Os status mais úteis para operação são:

1. `queued`: o run foi criado e aguarda despacho;
2. `dispatching`: o run está sendo entregue ao worker;
3. `running`: o runtime real está executando;
4. `waiting_hil`: o agente pausou aguardando decisão humana;
5. `completed`: a execução terminou com resultado;
6. `failed`: a execução terminou com erro registrado;
7. `cancelled`: a execução ou agenda foi cancelada;
8. `expired` ou `skipped`: a janela não foi executada conforme política da agenda.

## 5. Human-in-the-loop em background

Human-in-the-loop, ou HIL, significa pausa para revisão humana.

Quando um agente ou deepagent em background pede aprovação, a plataforma registra uma solicitação HIL durável ligada ao `run_id`. Isso permite rastrear qual execução está aguardando uma decisão humana.

Quando a decisão humana chega por canal externo, a plataforma retoma o agente ou deepagent e sincroniza o run associado. Se a retomada terminar bem, o run sai de `waiting_hil` para `completed`. Se a retomada falhar, o run vai para `failed` com erro estruturado. Se a retomada gerar uma nova pausa HIL sem emissão durável vinculada ao run, o run também falha fechado para evitar espera invisível.

O comportamento atual é intencionalmente conservador para workflows. Se um workflow em background retornar HIL, o runtime falha fechado. Isso acontece porque a retomada assíncrona de workflow ainda não tem caminho durável equivalente ao de agentes e deepagents. Falhar fechado é melhor do que deixar uma execução presa sem continuação confiável.

## 6. Segurança do snapshot YAML

O snapshot YAML é uma cópia da configuração necessária para o worker reconstruir o runtime da execução.

Esse snapshot não deve persistir segredos abertos. Por isso, a plataforma remove `security_keys`, converte segredos reidratáveis de volta para placeholders e bloqueia segredo literal em campo sensível fora do contrato seguro.

Na prática, isso evita que uma chave de API, token ou senha fique gravada no ledger de background. O worker reidrata os segredos no momento da execução usando o diretório seguro do tenant.

## 7. APIs administrativas

As APIs administrativas servem para acompanhar a operação depois que a solicitação foi criada.

Rotas disponíveis:

1. `GET /admin/background-executions/requests`: lista solicitações criadas a partir de prompts;
2. `GET /admin/background-executions/schedules`: lista agendas background do tenant informado por `access_key`;
3. `GET /admin/background-executions/runs/recent`: lista runs recentes;
4. `GET /admin/background-executions/runs/active`: lista runs ainda ativos, incluindo os que aguardam HIL;
5. `GET /admin/background-executions/events`: lista eventos persistidos do ledger por `run_id`, `correlation_id` ou ambos;
6. `GET /admin/background-executions/hil`: lista pedidos HIL duráveis sem expor token nem hash;
7. `GET /admin/background-executions/runs/last`: consulta o último resultado por `request_id` ou `schedule_id`;
8. `GET /admin/background-executions/runs/{run_id}`: consulta um run específico;
9. `DELETE /admin/background-executions/schedules/{schedule_id}`: cancela uma agenda preservando o histórico.

Essas rotas exigem permissão administrativa própria:

1. `admin.background_execution.read` para consulta;
2. `admin.background_execution.write` para cancelamento de agenda.

## 8. O que a API administrativa não faz

A API administrativa não substitui a criação da solicitação background pelo fluxo agentic.

Ela existe para observar e controlar o que já foi registrado: agendas, runs, resultados, erros e estados ativos. Isso mantém a responsabilidade separada. A criação nasce do pedido do usuário e das tools internas; a administração acompanha o histórico e controla a operação.

## 9. Como investigar uma execução

Para investigar um problema, comece pelo `run_id` e pelo `correlation_id`.

O `run_id` identifica a execução no ledger background. O `correlation_id` identifica a história de ponta a ponta nos logs. O mesmo `correlation_id` deve ser propagado sem troca pela API, scheduler, worker e runtime.

Leitura recomendada:

1. consultar a solicitação para confirmar qual comando foi persistido;
2. consultar a agenda para confirmar quando o run deveria nascer;
3. consultar o run específico na API administrativa;
4. verificar `status`, `error_type`, `error_message` e `telemetry`;
5. ler os eventos persistidos do run para reconstruir a sequência operacional;
6. se estiver `waiting_hil`, consultar a fila de aprovações HIL do tenant;
7. usar o `correlation_id` para abrir somente os logs cujo nome começa com esse identificador;
8. confirmar se houve início, execução, conclusão, fallback, retry ou erro.

## 10. Regras operacionais importantes

1. Fallback só existe se houver requisito explícito.
2. `correlation_id` nasce uma vez e deve ser propagado sem alteração.
3. A execução real deve passar pelo worker oficial.
4. O scheduler cria runs vencidos, mas não deve executar runtime agentic diretamente.
5. Segredos não devem ser persistidos no snapshot YAML.
6. Workflow HIL em background deve falhar fechado até existir retomada durável.
7. Eventos de run devem ser persistidos no ledger, não apenas escritos em log.
8. APIs administrativas devem reutilizar o serviço de domínio, sem duplicar lógica de banco ou criar fluxo paralelo.

## 11. Evidências no código

- [src/agentic_layer/background_execution/models.py](../src/agentic_layer/background_execution/models.py): modelos de solicitação, agenda, run, resultado e escopo.
- [src/agentic_layer/background_execution/services.py](../src/agentic_layer/background_execution/services.py): casos de uso de agendamento, consulta, cancelamento e listagem.
- [src/agentic_layer/background_execution/runtime.py](../src/agentic_layer/background_execution/runtime.py): runtime real de Agent, DeepAgent e Workflow em background.
- [src/agentic_layer/background_execution/postgres_repository.py](../src/agentic_layer/background_execution/postgres_repository.py): persistência PostgreSQL no schema de background.
- [src/agentic_layer/tools/system_tools/background_execution.py](../src/agentic_layer/tools/system_tools/background_execution.py): tools internas que criam e consultam execuções background.
- [src/api/routers/admin/background_execution_router.py](../src/api/routers/admin/background_execution_router.py): APIs administrativas de requests, schedules, runs, eventos e HIL.
- [src/api/services/admin/background_execution_service.py](../src/api/services/admin/background_execution_service.py): camada administrativa que resolve tenant por `access_key` e delega ao serviço de domínio.
- [src/api/services/background_execution_hil_run_finalizer.py](../src/api/services/background_execution_hil_run_finalizer.py): sincronização do run após decisão HIL externa.
- [scripts/sql/20260502_create_agent_background_schema.sql](../scripts/sql/20260502_create_agent_background_schema.sql): schema SQL da capacidade background.
