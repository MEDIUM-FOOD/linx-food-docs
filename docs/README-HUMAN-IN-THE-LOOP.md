# 🤝 HUMAN-IN-THE-LOOP - Plataforma de Agentes de IA

**Produto:** Plataforma de Agentes de IA

## 📋 Visão Geral

O sistema Human-in-the-Loop (HIL) da Plataforma de Agentes de IA permite intervenção humana inteligente em execuções agentic sensíveis, combinando automação eficiente com supervisão humana quando necessário. Este documento detalha a implementação técnica do produto atual, cobrindo o HIL padrão de DeepAgent por envelope `hil`, o caminho de workflow com `human_gate` e os endpoints de API usados para pausa, revisão e retomada.

Se o seu objetivo é entender a experiência completa de HIL Generative UI,
com o contrato compartilhado do frontend e o reaproveitamento entre
WebChat v3, Admin WebChat e sidecar AG-UI, leia também
[tutorial-101-human-in-the-loop.md](./tutorial-101-human-in-the-loop.md).

## 📌 Nota de escopo para DeepAgent

Se o seu objetivo é aprovar ou editar chamadas de tool no DeepAgent atual, não comece por `create_human_gate_tool`.

Leitura prática:

- No DeepAgent, o caminho padrão de primeira adoção é ligar `middlewares.human_in_the_loop` e declarar `interrupt_on` no supervisor.
- Nesse caso, o próprio runtime monta o `HumanInTheLoopMiddleware` e pausa a execução antes da tool sensível rodar.
- `human_gate` e `create_human_gate_tool` continuam úteis para workflows gerais e casos customizados, mas não são pré-requisito para o HIL padrão de tool-call no DeepAgent.

Se o seu foco for o fluxo HTTP real do produto, com payloads atuais de `/agent/execute` e `/agent/continue`, leia também [tutorial-101-exemplos-api-deepagent-hil-execute-continue.md](./tutorial-101-exemplos-api-deepagent-hil-execute-continue.md).

## 📌 Nota de contrato HTTP cliente-neutro

No contrato atual do produto para DeepAgent via HTTP, o endpoint `/agent/execute` já publica dois sinais formais quando a execução pausa por revisão humana:

- `thread_id`: identificador formal da thread pausada.
- `hil`: envelope mínimo cliente-neutro com `pending`, `protocol_version`, `message`, `allowed_decisions` e `resume_endpoint`.

Leitura prática:

- O gatilho oficial da interface não deve mais ser parsing de texto em `response`.
- O gatilho oficial deve ser `hil.pending=true`.
- O cliente não deve recomputar `thread_id`; deve reaproveitar exatamente o valor devolvido pelo backend.
- O cliente deve renderizar apenas as decisões publicadas em `hil.allowed_decisions`.
- O endpoint `/agent/continue` continua mais amplo por compatibilidade, mas o contrato público mínimo atual anunciado no execute é `hil-http-v1` com `approve` e `reject`.

## 📌 Como configurar HIL no DeepAgent via YAML atual

Se a dúvida for objetiva, a resposta correta hoje é esta: no DeepAgent,
o HIL padrão é configurado no YAML com `multi_agents[].middlewares.human_in_the_loop.enabled=true`,
com `multi_agents[].interrupt_on` no mesmo supervisor e com
`memory.checkpointer.enabled=true` na raiz. Esse é o caminho aderente ao
contrato atual do produto, à AST agentic, ao validator e ao runtime.

Leitura prática:

- `memory.checkpointer` é o que permite pausar e retomar a mesma thread.
- `selected_supervisor` escolhe o supervisor DeepAgent que vai rodar.
- `execution.type=deepagent` liga o runtime correto.
- `middlewares.human_in_the_loop.enabled=true` liga a pausa humana.
- `interrupt_on` define quais tools param antes da execução real.
- `allowed_decisions` limita o que o humano pode responder.
- a janela genérica de revisão deve nascer do envelope `hil` devolvido pelo backend, não de parsing de texto da LLM.

### O que o contrato atual realmente suporta

O contrato atual do DeepAgent já suporta um conjunto útil de governança,
mas ele tem limites claros que precisam ser respeitados para não criar
YAML enganoso.

O que é suportado hoje:

- escolher quais tools exigem revisão humana;
- limitar decisões a `approve`, `edit` e `reject`, ou a um subconjunto dessas opções;
- customizar uma mensagem geral de revisão com `description_prefix`;
- disparar aprovação assíncrona por canal com `async_approval`;
- renderizar uma UI genérica a partir de `hil.pending`, `action_requests`, `review_configs`, `allowed_decisions`, `thread_id` e `resume_endpoint`.

O que não é suportado hoje:

- condição declarativa no `interrupt_on`, como `valor < 100`;
- decisões arbitrárias como `aprovar_com_ressalva` ou `encaminhar_para_diretoria`;
- mensagem customizada por regra dentro de `interrupt_on`;
- schema visual declarativo em YAML para montar a janela de HIL;
- fallback escondido que transforme regra inválida em comportamento de produção.

Em linguagem simples: o YAML atual sabe dizer qual ferramenta deve parar,
mas não sabe dizer “pare só quando um argumento específico tiver certo
valor”. Ele pausa por nome de tool, não por expressão de negócio.

### Explicação for dummies do YAML atual

Pense no DeepAgent como um operador que usa ferramentas. Algumas
ferramentas são sensíveis, como enviar um e-mail, confirmar uma operação
ou publicar alguma coisa. O HIL serve para colocar um porteiro na frente
dessas ferramentas. Quando o agente tenta passar, o porteiro para a ação
e pede uma decisão humana.

No YAML atual, você não programa a inteligência do porteiro com contas,
comparações e fórmulas. Você só diz quais portas devem ter porteiro. O
backend então cria uma pendência estruturada dizendo o que o agente quer
fazer, quais argumentos quer usar e quais respostas o humano pode dar.

A interface não precisa adivinhar nada. Ela apenas lê o envelope `hil`
publicado pelo backend e monta a revisão com componentes seguros do
produto. Se o envelope permitir só aprovar ou rejeitar, a tela mostra só
esses botões. Se permitir edição, a tela pode abrir uma revisão mais
rica dos argumentos.

### Aprovar ou rejeitar o quê, exatamente?

Essa é a dúvida mais importante de primeira adoção, porque o YAML pode
dar a impressão errada de que a aprovação está ligada a um valor, a uma
faixa ou a uma regra de negócio. No DeepAgent atual, não é isso que
acontece.

Quando o HIL está ligado com `interrupt_on`, o humano não aprova “abaixo
de 100”, “acima de 1000” ou “só se cair nessa condição”. O humano aprova,
edita ou rejeita a chamada proposta da tool protegida.

Em linguagem simples: o agente diz “quero executar esta ferramenta com
estes argumentos”. O HIL transforma isso em uma pendência humana. A
pessoa então decide se aquela chamada específica pode seguir, se precisa
ser ajustada ou se deve ser bloqueada.

O que o backend publica para revisão é, em essência:

- qual tool o agente quer chamar;
- quais argumentos ele quer enviar para essa tool;
- quais decisões humanas são aceitas naquele caso;
- qual thread deve ser retomada depois da resposta.

Então, quando o supervisor protege uma tool como
`registrar_operacao_financeira`, o que está em revisão é a execução
proposta dessa tool, com o conjunto de dados que o agente montou naquele
momento.

#### O que isso significa na prática

Se a tool proposta for registrar uma operação financeira, a revisão pode
mostrar algo como:

- valor proposto;
- descrição da operação;
- cliente, conta, centro de custo ou identificador relacionado;
- qualquer outro argumento que a tool vá receber.

Mas o YAML atual não diz sozinho quando isso deve acontecer por faixa de
valor. Ele apenas diz que, quando essa tool for chamada, a chamada deve
ser revisada por um humano.

Por isso, estas duas frases significam coisas bem diferentes:

- “Sempre que a tool `registrar_operacao_financeira` for chamada, peça revisão humana.”
- “Peça revisão humana apenas quando o valor for menor que 100.”

A primeira frase é suportada hoje pelo contrato atual.

A segunda frase não é suportada nativamente pelo `interrupt_on` atual,
porque ela exige condição de negócio por argumento.

### As 3 respostas reais do contrato atual

Hoje, o contrato oficial do DeepAgent para revisão de tool-call trabalha
com três decisões e apenas três decisões:

1. `approve`
2. `edit`
3. `reject`

Isso é importante por dois motivos:

- não existe no contrato atual uma quarta decisão arbitrária;
- a interface não deve inventar botões fora dessas opções.

#### 1. Aprovar

Quando o humano aprova, ele está dizendo que a tool pode ser executada
com os argumentos exatamente como foram propostos pelo agente.

Exemplo mental simples:

- o agente quer enviar um e-mail para um cliente com determinado assunto;
- a revisão mostra destinatário, assunto e corpo;
- o humano olha e conclui que está tudo certo;
- a decisão é aprovar;
- o runtime retoma e executa a tool com aqueles mesmos dados.

Na prática, aprovar significa “pode executar do jeito que está”.

#### 2. Editar

Quando o humano edita, ele está dizendo que a ação pode acontecer, mas
não exatamente do jeito que o agente propôs. Algum argumento precisa ser
corrigido antes da execução.

Exemplo mental simples:

- o agente quer registrar uma operação com valor 80, mas o centro de
    custo veio errado;
- ou o agente quer enviar um e-mail, mas o destinatário está incorreto;
- ou a descrição da operação precisa ser ajustada para atender uma regra
    interna.

Nesse caso, a pessoa não quer bloquear a ação inteira. Ela quer ajustar
os dados antes de deixar a execução continuar.

Na prática, editar significa “pode executar, mas com argumentos
corrigidos”.

#### 3. Rejeitar

Quando o humano rejeita, ele está dizendo que aquela chamada da tool não
deve acontecer.

Exemplo mental simples:

- o agente tentou disparar uma operação que não deveria existir;
- ou tentou enviar uma mensagem para o destinatário errado;
- ou a ação não faz sentido para aquele contexto;
- ou a pessoa responsável prefere interromper a automação naquele ponto.

Na prática, rejeitar significa “essa chamada específica da tool não pode
ser executada”.

### Casos comuns de revisão humana

Para deixar mais concreto, estes são alguns casos muito comuns em que a
dúvida aparece:

#### Caso comum 1: envio de e-mail

O que o humano revisa:

- destinatário;
- assunto;
- corpo;
- anexos ou contexto adicional, quando existirem.

O que ele pode fazer:

- aprovar se tudo estiver certo;
- editar se precisar corrigir destinatário ou texto;
- rejeitar se o envio não deve acontecer.

#### Caso comum 2: operação financeira

O que o humano revisa:

- valor;
- descrição;
- conta, cliente ou centro de custo;
- qualquer campo sensível que vá para a tool.

O que ele pode fazer:

- aprovar se a operação estiver correta do jeito proposto;
- editar se algum dado estiver incorreto, mas a operação ainda fizer sentido;
- rejeitar se a operação não deve ser registrada.

Ponto crítico:

- o valor pode aparecer para revisão humana;
- isso não significa que o YAML já sabe pausar “só abaixo de 100” ou “só acima de 1000”.

#### Caso comum 3: publicação ou ação externa

O que o humano revisa:

- o destino da publicação;
- o conteúdo enviado;
- o identificador do recurso externo;
- o impacto da ação.

O que ele pode fazer:

- aprovar quando a publicação estiver pronta;
- editar quando algum campo precisar ajuste;
- rejeitar quando a ação externa não deve acontecer.

### Onde entra a ideia de faixa de valor

Quando alguém pergunta “aprovar ou rejeitar acima ou abaixo de qual
valor?”, normalmente essa pessoa está descrevendo uma regra de negócio,
não apenas um ponto de revisão humana.

Exemplos de regra de negócio:

- pedir aprovação apenas para desconto acima de 10 por cento;
- pedir aprovação apenas para valor abaixo de 100;
- bloquear automaticamente transferência acima de 5000 sem dupla checagem;
- exigir edição quando um campo vier vazio.

Essas regras dependem de condição determinística. O `interrupt_on` atual
do DeepAgent não foi modelado para isso. Ele foi modelado para decidir
quais tools entram em revisão humana, não para avaliar expressões de
negócio sobre os argumentos.

Por isso, se a necessidade real for faixa de valor, o caminho correto é
um destes:

- aceitar a revisão da tool inteira, se isso for operacionalmente aceitável;
- modelar a condição em Workflow, quando o processo for previsível;
- colocar a regra na tool ou em uma policy de domínio;
- evoluir formalmente o contrato do DeepAgent para suportar condição declarativa.

### Como explicar isso para um usuário final

Uma forma simples de explicar sem jargão é esta:

- o HIL atual aprova uma ação proposta pelo agente;
- ele não aprova sozinho uma regra de valor;
- o valor pode aparecer para a pessoa revisar;
- mas a decisão de pausar por faixa de valor ainda precisa de uma regra de negócio fora do `interrupt_on` atual.

### Exemplos guiados de configuração

Os exemplos abaixo não são “pseudoideias”. Eles mostram como pensar a
configuração de forma aderente ao contrato atual do produto.

#### Exemplo guiado 1: HIL mínimo síncrono

Use esse modelo mental quando você quer o menor recorte funcional
possível.

Configuração prática:

- `memory.checkpointer.enabled=true` na raiz;
- um supervisor ativo com `execution.type=deepagent`;
- `middlewares.human_in_the_loop.enabled=true`;
- `interrupt_on.calculator.allowed_decisions=[approve, edit, reject]`;
- um subagente ou supervisor que realmente tenha acesso à tool `calculator`.

Fluxo para o usuário:

1. o agente tenta chamar `calculator`;
2. a execução pausa antes da tool rodar;
3. o backend devolve `thread_id` e `hil`;
4. a interface mostra a revisão;
5. o humano aprova, edita ou rejeita;
6. a execução retoma com o mesmo `thread_id`.

Impacto prático:

- é a forma mais barata de começar;
- protege uma tool real sem criar tool HIL manual;
- já funciona com a janela genérica baseada no contrato `hil`.

#### Exemplo guiado 2: mensagem geral customizada

Use esse modelo quando a revisão precisa deixar mais claro para o
operador o tipo de cuidado esperado.

Configuração prática:

- manter `middlewares.human_in_the_loop.enabled=true`;
- adicionar `middlewares.human_in_the_loop.description_prefix` com uma mensagem geral, como “Revise esta ação antes da execução”;
- proteger a tool sensível em `interrupt_on`, por exemplo `enviar_email`;
- permitir `approve`, `edit` e `reject` quando a pessoa puder corrigir destinatário, assunto ou corpo.

O que isso muda para o usuário:

- a pendência chega com uma mensagem de revisão mais clara;
- a pessoa entende melhor por que a ação foi parada;
- a UI continua sendo a mesma, porque a renderização vem do envelope `hil`.

Pegadinha importante:

- o contrato atual não possui `interrupt_on.enviar_email.message` nem `interrupt_on.enviar_email.ui_schema`;
- a personalização por regra ainda não existe;
- a mensagem rica da janela continua vindo do payload estruturado do backend.

#### Exemplo guiado 3: aprovação assíncrona por WhatsApp e e-mail

Use esse modelo quando a execução for em background e não houver uma tela
aberta esperando a resposta humana.

Configuração prática:

- `execution.default_mode=direct_async`;
- `middlewares.human_in_the_loop.enabled=true`;
- `middlewares.human_in_the_loop.async_approval.enabled=true`;
- `ttl_seconds` para controlar validade do pedido;
- `expiration_policy=expire` ou `fail_run`;
- `channels[]` com pelo menos um canal habilitado e com `template_id`;
- `approvers[]` com identidade explícita;
- `interrupt_on` apontando para a tool sensível, por exemplo `registrar_operacao_financeira`.

Exemplo de leitura operacional:

- WhatsApp exige `channel_user_ids.whatsapp` em pelo menos um aprovador;
- e-mail exige `user_email` em pelo menos um aprovador;
- canal habilitado sem `template_id` deve falhar em vez de cair em mensagem padrão escondida;
- sem aprovador válido, a configuração deve falhar fechada.

Impacto prático:

- o agente pausa do mesmo jeito;
- a plataforma registra um pedido auditável;
- a aprovação pode chegar por canal externo sem tela aberta;
- a continuação continua presa ao mesmo `thread_id` e ao mesmo fluxo de HIL.

#### Exemplo guiado 4: HIL concentrado em um subagente

Use esse modelo quando o supervisor coordena vários especialistas, mas só
um deles executa a ação sensível.

Configuração prática:

- manter `middlewares.human_in_the_loop.enabled=true` no supervisor;
- declarar `interrupt_on` no supervisor, porque o validator exige esse bloco quando HIL está ligado;
- se necessário, adicionar `agents[].interrupt_on` no subagente que realmente chama a tool protegida;
- garantir que a tool exista no escopo efetivo desse subagente.

Impacto prático:

- a governança continua centralizada;
- o especialista sensível pode ter override explícito de revisão;
- a documentação e os testes do runtime já tratam esse cenário como contrato vivo.

#### Exemplo guiado 5: regra “pedir aprovação se valor for menor que 100”

Esse é o ponto mais importante para evitar erro de arquitetura.

Hoje, isso não cabe no `interrupt_on` do DeepAgent atual. O contrato não
tem `condition`, `when`, `rules`, `threshold`, `lt` nem expressão por
valor dentro dessa chave. Se você escrever algo assim no YAML, estará
criando uma configuração sem garantia de comportamento real.

O que acontece hoje se você tentar resolver isso só com DeepAgent YAML:

- se proteger a tool inteira com `interrupt_on`, toda chamada dessa tool pode pausar;
- se tentar inventar um campo condicional, o contrato não promete usar esse campo;
- se depender apenas do prompt da LLM para decidir quando pedir aprovação, a governança fica frágil.

Soluções corretas para esse tipo de regra:

- aceitar a sobreaprovação e proteger a tool inteira;
- modelar a condição em Workflow, quando o fluxo for determinístico;
- colocar a regra de negócio em uma tool ou policy de domínio;
- evoluir formalmente o contrato do DeepAgent para suportar condições, com AST, validator, runtime, testes e documentação sincronizados.

### Checklist rápido de configuração correta

Antes de publicar um YAML DeepAgent com HIL, confirme:

1. `selected_supervisor` aponta para o supervisor certo.
2. `execution.type=deepagent` está no supervisor ativo.
3. `memory.checkpointer.enabled=true` está na raiz quando HIL está ligado.
4. `middlewares.human_in_the_loop.enabled=true` está no mesmo supervisor que usa `interrupt_on`.
5. `interrupt_on` está em `multi_agents[].interrupt_on`, e não dentro de `middlewares.human_in_the_loop`.
6. cada tool em `interrupt_on` existe no escopo real do supervisor ou do subagente.
7. `allowed_decisions` usa apenas `approve`, `edit` e `reject`.
8. a interface usa `hil.pending=true` como gatilho oficial.
9. a retomada reaproveita exatamente o `thread_id` devolvido pelo backend.
10. se houver `async_approval`, existem canal habilitado, `template_id` e aprovadores compatíveis.

## 📌 Nota de aprovação assíncrona em background

Para agentes DeepAgent executados em background, sem uma tela aberta, o
contrato governado agora usa
`middlewares.human_in_the_loop.async_approval`. Esse bloco informa quais
canais podem receber o pedido, quais aprovadores podem decidir, por
quanto tempo o pedido é válido e o que fazer quando ele expira.

Leitura prática:

- `interrupt_on` continua sendo a regra que define quais tools pausam
    para revisão humana.
- `async_approval` não inventa uma segunda forma de HIL; ele configura
    como a mesma pausa pode pedir decisão por WhatsApp ou e-mail.
- Canal ativo precisa de `template_id`, para evitar mensagem padrão
    escondida.
- Aprovador precisa de identidade explícita, como `user_email`,
    `user_code` ou identificador de canal.
- Se `async_approval.enabled=true`, também é obrigatório manter
    `middlewares.human_in_the_loop.enabled=true`, `interrupt_on` e
    checkpointer ativo.

Em linguagem simples: o agente para no mesmo ponto de HIL de sempre. A
diferença é que a plataforma também sabe para quem avisar, por qual
canal avisar e até quando aquela autorização vale.

### Como o HIL assíncrono funciona de ponta a ponta

O fluxo implementado hoje atende o DeepAgent em background. Background
significa que a execução pode continuar no servidor sem uma tela aberta
esperando a resposta humana.

O processo real é este:

1. O DeepAgent chega a uma tool protegida por `interrupt_on`.
2. O runtime pausa antes da tool sensível executar.
3. A plataforma registra a pausa em `public.agent_hil_approval_requests`.
4. O registro recebe `correlation_id`, `thread_id`, `task_id`, prazo de
    expiração, decisões permitidas, aprovadores e metadata auditável.
5. A plataforma gera um token seguro para a resposta humana. O banco
    guarda apenas o hash do token, não o token bruto.
6. O serviço de notificação envia o pedido por WhatsApp ou e-mail,
    conforme o YAML.
7. A decisão recebida pelo canal é validada contra token, status,
    expiração, canal e aprovador permitido.
8. A primeira decisão válida resolve o pedido no banco de forma atômica.
9. A execução original é retomada com `Command(resume=...)`, usando o
    mesmo `thread_id`.
10. Decisões duplicadas, expiradas ou vindas de pessoa errada são
    recusadas sem retomar o agente.

Em linguagem simples: a aprovação vira um chamado seguro. O agente fica
parado, a pessoa certa recebe o pedido, e a plataforma só libera o
processo se a resposta for válida e ainda estiver dentro do prazo.

### Como ativar no YAML

A ativação fica dentro do supervisor DeepAgent, no caminho
`multi_agents[].middlewares.human_in_the_loop.async_approval`.

Configuração mínima que precisa existir no mesmo supervisor:

1. `middlewares.human_in_the_loop.enabled=true`.
2. `multi_agents[].interrupt_on` com as tools sensíveis.
3. `middlewares.human_in_the_loop.async_approval.enabled=true`.
4. `middlewares.human_in_the_loop.async_approval.ttl_seconds` com o
    tempo de validade do pedido, entre 60 segundos e 7 dias.
5. `middlewares.human_in_the_loop.async_approval.expiration_policy`, com
    `expire` ou `fail_run`.
6. `middlewares.human_in_the_loop.async_approval.channels[]` com os
    canais habilitados.
7. `middlewares.human_in_the_loop.async_approval.approvers[]` com as
    pessoas autorizadas a decidir.

Para WhatsApp:

1. Use `channels[].type=whatsapp`.
2. Informe `channels[].channel_id` apontando para um canal WhatsApp
    cadastrado na camada de canais.
3. Informe `channels[].template_id`.
4. Em cada aprovador que receber WhatsApp, informe
    `approvers[].channel_user_ids.whatsapp`.

Para e-mail:

1. Use `channels[].type=email`.
2. Informe `channels[].template_id`.
3. Cada aprovador por e-mail precisa ter `approvers[].user_email`.
4. O YAML também precisa conter a configuração SMTP já usada pela
    plataforma: `tool_config.smtp_email.smtp_host` e credenciais em
    `security_keys.SMTP_USERNAME` e `security_keys.SMTP_PASSWORD`.

### Como configurar o canal de WhatsApp de forma correta

No HIL assíncrono, o bloco `async_approval.channels[]` não cria sozinho
um canal novo. Ele referencia um canal que já precisa existir e estar
operacional no tenant.

Em linguagem simples: `channel_id` não é um apelido livre. Ele é o nome
ou identificador do canal já provisionado e reconhecido pela camada de
canais da plataforma. Se esse canal não existir de verdade no tenant, o
HIL não terá para onde enviar a aprovação.

O que precisa estar alinhado para WhatsApp funcionar bem:

1. o número WhatsApp já precisa estar provisionado no tenant;
2. o webhook do canal já precisa apontar para o fluxo multicanal da plataforma;
3. o `channel_id` usado em `async_approval.channels[]` precisa apontar para esse canal real;
4. o `template_id` precisa existir no provedor e ser compatível com o tipo de envio que o canal suporta;
5. pelo menos um aprovador precisa ter `channel_user_ids.whatsapp` válido;
6. o worker oficial precisa estar com o plano de controle multicanal pronto.

Leitura prática do `channel_id`:

- use o identificador real do canal provisionado para o tenant;
- mantenha o mesmo nome operacional usado no onboarding e nos fluxos ativos;
- não crie um `channel_id` genérico no YAML esperando que a plataforma descubra o canal por inferência;
- se houver mais de um número WhatsApp no tenant, escolha explicitamente qual deles deve receber o pedido de aprovação.

Leitura prática do `template_id`:

- ele representa o template de mensagem aprovado no provedor;
- o HIL usa template para evitar mensagem solta e comportamento escondido;
- se o template não existir ou não estiver aprovado, o envio pode falhar mesmo com o resto da configuração correto;
- o ideal é que o template deixe claro que se trata de uma aprovação pendente, com prazo e contexto resumido.

Documentação para configuração do canal WhatsApp:

- visão funcional consolidada: [docs/README-WHATSAPP-PROVISIONING.md](./README-WHATSAPP-PROVISIONING.md)
- tutorial passo a passo de provisionamento: [docs/tutorial-101-provisionamento-whatsapp.md](./tutorial-101-provisionamento-whatsapp.md)
- operação do worker e plano multicanal: [docs/GUIA-DIDATICO-EXECUCAO-CANAIS.md](./GUIA-DIDATICO-EXECUCAO-CANAIS.md)
- detalhes do ecossistema e onboarding Meta: [docs/tools/whatsapp_business.md](./tools/whatsapp_business.md)

### Como configurar o canal de e-mail de forma correta

No caso de e-mail, o raciocínio é um pouco diferente do WhatsApp. O
HIL assíncrono não depende de um `channel_id` físico como um número
WhatsApp provisionado na Meta. Ele depende principalmente de uma rota de
envio SMTP funcional e de aprovadores com endereço de e-mail válido.

O que precisa estar alinhado para e-mail funcionar bem:

1. o canal `email` precisa estar habilitado em `async_approval.channels[]`;
2. o `template_id` do e-mail precisa estar definido na configuração de HIL;
3. pelo menos um aprovador precisa ter `approvers[].user_email`;
4. a configuração SMTP da plataforma precisa estar presente e válida;
5. as credenciais precisam estar resolvidas no tenant, sem depender de fallback implícito.

Leitura prática da configuração SMTP:

- `tool_config.smtp_email.smtp_host` define o servidor de envio;
- `security_keys.SMTP_USERNAME` e `security_keys.SMTP_PASSWORD` carregam as credenciais;
- se essas chaves estiverem ausentes ou inválidas, o problema não é do HIL em si, mas do canal de entrega;
- o erro operacional esperado deve aparecer como falha de notificação, não como aprovação silenciosa.

Leitura prática do `template_id` no e-mail:

- ele representa o modelo lógico de mensagem que o serviço de notificação usa;
- a intenção é manter o mesmo padrão de governança do WhatsApp, sem texto improvisado por fluxo;
- o template deve deixar claro quem precisa aprovar, o que está pendente, qual o prazo e qual ação a aprovação destrava.

Estado atual da documentação de e-mail:

- hoje não encontrei no repositório um manual dedicado só para provisionamento ou operação de canal SMTP equivalente ao material de WhatsApp;
- por isso, a referência mais próxima e canônica para e-mail neste momento continua sendo este manual de HIL junto com o catálogo funcional de tools.

Referência complementar para tools de e-mail:

- visão funcional de `email_sender`, `gmail_send` e `outlook_send`: [docs/tools/por_finalidade.md](./tools/por_finalidade.md)

### Como escolher entre WhatsApp e e-mail no HIL assíncrono

Use WhatsApp quando:

- a aprovação precisa chegar rápido;
- o aprovador opera majoritariamente no celular;
- o tenant já tem canal WhatsApp provisionado e webhook funcional;
- o template da notificação já está aprovado no provedor.

Use e-mail quando:

- o aprovador trabalha em contexto mais corporativo e depende de caixa de entrada;
- a decisão pode carregar contexto textual maior;
- o tenant já possui SMTP corporativo estável;
- o fluxo não depende de interação móvel imediata.

Use os dois canais quando:

- a aprovação é crítica e você quer redundância operacional;
- existe risco real de o aprovador perder uma única notificação;
- o processo precisa combinar velocidade no celular com trilha formal por e-mail.

### Erros de configuração mais comuns em canais assíncronos

1. configurar `channel_id` no YAML sem o canal real existir no tenant;
2. usar `template_id` inexistente ou não aprovado;
3. cadastrar aprovador WhatsApp sem `channel_user_ids.whatsapp`;
4. cadastrar aprovador de e-mail sem `user_email`;
5. assumir que `async_approval` cria sozinho o canal ou o template;
6. esquecer que o worker oficial precisa subir o plano multicanal para o envio operar de ponta a ponta;
7. tratar problema de SMTP ou Meta como se fosse erro do contrato HIL.

O que não deve ser feito:

1. Não preencher `tools_library` manualmente para essa feature.
2. Não criar canal sem `template_id` esperando mensagem padrão.
3. Não configurar aprovador sem identidade clara.
4. Não usar `async_approval` sem `human_in_the_loop.enabled=true`.
5. Não assumir suporte a Workflow ou AgentSupervisor clássico neste
    primeiro corte de HIL assíncrono por canais.

### Configuração operacional do banco e da expiração

A fonte de verdade do pedido HIL assíncrono é a tabela
`public.agent_hil_approval_requests`. A conexão deve ser configurada por
ambiente usando `AGENT_HIL_APPROVAL_DSN`. Sem essa variável, o repositório
falha de forma explícita; não existe fallback para Redis.

Variáveis opcionais:

1. `AGENT_HIL_APPROVAL_SCHEMA`: schema da tabela. Se ausente, usa
    `public`.
2. `AGENT_HIL_APPROVAL_TABLE`: nome da tabela. Se ausente, usa
    `agent_hil_approval_requests`.
3. `AGENT_HIL_APPROVAL_MAINTENANCE_ENABLED`: ativa o job periódico que
    fecha pedidos expirados.
4. `AGENT_HIL_APPROVAL_MAINTENANCE_INTERVAL_SECONDS`: intervalo entre
    rodadas do job.
5. `AGENT_HIL_APPROVAL_MAINTENANCE_LIMIT_PER_RUN`: limite de pedidos
    fechados por rodada.

Mesmo com o job desligado, uma decisão recebida depois do prazo é
recusada no momento da validação. O job serve para limpar estados
pendentes vencidos e deixar a operação mais observável.

### Política de expiração

`expiration_policy=expire` fecha o pedido como expirado quando o prazo
vence. Esse é o comportamento seguro padrão: a ação sensível não é
executada e ninguém aprova automaticamente.

`expiration_policy=fail_run` fecha o pedido como falho quando o prazo
vence. Esse modo indica que, para aquele processo, a ausência de resposta
humana deve ser tratada como falha operacional. Ele também não executa a
tool sensível e não fabrica uma decisão de rejeição.

O que não existe hoje:

1. Não existe auto-approve.
2. Não existe auto-reject por canal.
3. Não existe lembrete ou escalation sem novos campos YAML governados.
4. Não existe retomada automática com `reject` apenas por expiração.

### Segurança e auditoria

O HIL assíncrono foi desenhado para impedir aprovações acidentais ou
indevidas.

Regras aplicadas:

1. O token de aprovação é gerado no servidor.
2. O banco guarda apenas o hash do token.
3. O token bruto não é gravado em log estruturado.
4. A decisão precisa vir de canal e aprovador compatíveis com o pedido.
5. A primeira decisão válida vence.
6. Decisão duplicada não altera um pedido já resolvido.
7. Pedido expirado não retoma o agente.
8. O mesmo `correlation_id` e o mesmo `thread_id` acompanham o fluxo.

Logs estruturados relevantes:

1. `hil.pause.created`: pedido durável criado.
2. `hil.notification.dispatch.started`: envio de notificação iniciado.
3. `hil.notification.dispatch.finished`: envio concluído com contagens.
4. `hil.notification.channel.failed`: falha de envio em um canal.
5. `hil.decision.received`: decisão recebida para validação.
6. `hil.decision.accepted`: decisão aceita e persistida.
7. `hil.decision.rejected`: decisão recusada por segurança, expiração ou
    conflito.
8. `hil.continuation.started`: retomada do agente iniciada.
9. `hil.continuation.finished`: retomada finalizada.
10. `hil.pause.expired`: pedido fechado por expiração.

Esses eventos permitem reconstruir a história: quem recebeu o pedido,
por qual canal, qual decisão chegou, se ela foi aceita e se o agente
retomou.

### Como responder uma aprovação

No WhatsApp, a decisão chega por botão interativo. A bridge de canal
intercepta esse botão antes do fluxo normal de mensagem, valida o payload
HIL e aciona o caso de uso interno de decisão. Isso evita que uma
aprovação vire uma mensagem comum do agente.

No e-mail, o fluxo seguro usa o endpoint de decisão HIL por método POST:
`/agent/hil/decisions`. O e-mail pode transportar o código seguro, mas
não deve aprovar por link GET automático. A decisão precisa passar pelo
endpoint que valida token, aprovador, canal e expiração.

As decisões aceitas pelo contrato continuam sendo `approve`, `reject` e
`edit`. Porém, os botões de canal gerados pelo serviço de notificação
usam apenas `approve` e `reject`, porque edição exige payload tipado e
interface própria.

### Troubleshooting

1. Sintoma: o pedido não foi criado.
  Verifique se `async_approval.enabled=true`, se o supervisor é DeepAgent
  e se `AGENT_HIL_APPROVAL_DSN` está configurado.

2. Sintoma: nenhuma mensagem saiu por WhatsApp.
  Verifique `channels[].channel_id`, `channels[].template_id`, cadastro
  do canal WhatsApp e credenciais do provider. O log deve mostrar
  `hil.notification.channel.failed` quando o envio falhar.

3. Sintoma: e-mail não foi enviado.
  Verifique `tool_config.smtp_email.smtp_host`, `security_keys.SMTP_USERNAME`
  e `security_keys.SMTP_PASSWORD`. O erro deve aparecer como falha de
  notificação, não como aprovação silenciosa.

4. Sintoma: resposta recusada como expirada.
  O prazo em `ttl_seconds` venceu. O agente não será retomado por essa
  decisão. Crie uma nova execução ou ajuste o TTL para o processo real.

5. Sintoma: resposta recusada como proibida.
  O canal ou aprovador não corresponde ao que foi configurado em
  `async_approval.approvers`. Verifique `user_email`, `user_code` e
  `channel_user_ids.whatsapp`.

6. Sintoma: segunda resposta não muda o resultado.
  Isso é esperado. A primeira decisão válida resolve o pedido e respostas
  posteriores são tratadas como conflito ou já resolvidas.

### Limites atuais

O HIL assíncrono por canais está implementado para DeepAgent em
background. Workflow e AgentSupervisor clássico têm fluxos próprios de
HIL e não foram incluídos neste primeiro corte.

Lembrete e escalation ainda não rodam porque não existe campo YAML
governado para isso. Quando esse recurso for necessário, ele precisa
nascer no contrato AST/YAML, no validator, no runtime e nos testes antes
de qualquer execução real.

## 🧩 Generative UI do HIL

Generative UI do HIL é a capacidade de montar a interface de revisão
humana a partir do envelope `hil` publicado pelo backend. A tela não
precisa ter uma regra fixa para cada tool sensível, nem precisa descobrir
por texto livre se existe aprovação pendente. Ela lê `hil.pending`,
`allowed_decisions`, `action_requests`, `review_configs`, `thread_id` e
`resume_endpoint`, e transforma esses campos em controles de revisão.

Essa abordagem é importante porque separa a decisão operacional da
aparência da tela. O backend continua sendo a autoridade sobre o que está
pausado, quais decisões são válidas e qual thread deve ser retomada. O
frontend apenas materializa essa pendência de forma adequada ao público:
uma tela simples pode mostrar dois botões, enquanto uma tela rica pode
mostrar cada ação, seus argumentos e uma edição controlada antes da
retomada.

### Explicação conceitual

Em uma UI tradicional, cada aprovação costuma nascer como uma tela
específica: uma tela para aprovar e-mail, outra para desconto, outra para
pedido e assim por diante. No HIL atual, o contrato permite um caminho
mais reutilizável. O runtime publica uma lista estruturada de ações
pendentes e a interface decide como desenhar a experiência sem alterar o
fluxo de backend.

O termo “generative” aqui não significa executar código visual produzido
pelo modelo, nem renderizar HTML, JavaScript ou CSS retornado pelo agente.
Significa que a UI é dirigida por dados estruturados do contrato HIL. A
interface gera seus próprios controles seguros, com componentes conhecidos
do produto, usando apenas as decisões e ações autorizadas pelo servidor.

### Explicação for dummies

Imagine que o agente chegou em uma etapa sensível e precisa de permissão
antes de seguir. Em vez de mandar uma frase solta como “preciso de
aprovação”, ele entrega um formulário organizado para a tela: qual ação
quer fazer, quais dados pretende usar, quais botões podem aparecer e para
onde a resposta deve ser enviada.

A tela então monta a revisão como se estivesse preenchendo uma ficha. Se
a ficha só permite aprovar ou rejeitar, aparecem só esses botões. Se a
ficha permite editar, a tela pode abrir campos para corrigir os argumentos
antes da execução. O usuário decide, a interface envia a decisão e o
backend retoma a mesma execução no ponto exato em que parou.

### Componente compartilhado do frontend

No frontend atual do produto, a responsabilidade foi dividida em duas
peças reutilizáveis.

A primeira peça é `HilContract`. Ela recebe a resposta do
backend, normaliza nomes de campos e monta `resume.decisions` para a
retomada. Em termos simples, ela traduz o contrato HTTP do produto para
um formato estável que qualquer tela pode reaproveitar.

A segunda peça é `HilReviewPanel`. Ela recebe o contrato HIL já
normalizado e materializa a revisão visual com mensagem, ações pendentes,
argumentos e botões permitidos. Essa peça não chama API, não cria
`correlation_id`, não recalcula `thread_id` e não conhece regra de
negócio da tela. Ela só renderiza a revisão e devolve a decisão humana
por callback.

Na prática, isso significa que WebChat v3, Admin WebChat e sidecar AG-UI
passaram a compartilhar a mesma base visual de revisão. Cada tela mantém
apenas o seu wiring fino: onde mostrar o painel, como coletar edição
quando existir e para qual fluxo local enviar a decisão.

O ponto mais importante dessa divisão é este: `HilReviewPanel` não faz
rede por conta própria e não escolhe o endpoint de retomada. WebChat v3
e Admin WebChat usam `HilContract` para montar `resume.decisions` e
chamam o continue formal do agente. No AG-UI, o sidecar reaproveita o
mesmo renderer visual, mas a interface dona do fluxo ainda precisa ligar
o callback da decisão ao continue formal correspondente. Isso preserva a
fronteira do backend e evita um segundo contrato escondido no browser.

### Fluxo funcional

```mermaid
sequenceDiagram
        participant Usuario as Usuário
        participant UI as Interface HIL
        participant API as /agent/execute
        participant Runtime as DeepAgent
        participant Continue as /agent/continue

        Usuario->>UI: Solicita uma tarefa agentic
        UI->>API: Envia a execução
        API->>Runtime: Executa com HIL configurado
        Runtime-->>API: Pausa antes da tool sensível
        API-->>UI: Publica thread_id e envelope hil
        UI->>Usuario: Renderiza revisão a partir do contrato
        Usuario->>UI: Aprova, edita ou rejeita
        UI->>Continue: Envia resume.decisions com a mesma thread
        Continue->>Runtime: Valida a pausa e retoma o fluxo
```

### Regras práticas para a interface

- Use `hil.pending=true` como gatilho oficial de pausa humana.
- Reutilize exatamente o `thread_id` publicado pelo backend.
- Preserve o mesmo `correlation_id` da execução original.
- Renderize somente decisões presentes em `hil.allowed_decisions` ou nas
    regras específicas de `review_configs`.
- Mostre cada item de `hil.action_requests` quando a tela precisar de
    revisão rica.
- Envie uma decisão por ação pendente, mantendo a mesma ordem recebida.
- Use `edited_action` apenas quando a decisão for `edit`.
- Não faça parsing de `response` para descobrir se existe HIL pendente.
- Não invente botão, decisão ou endpoint fora do contrato publicado.

Na prática, isso permite duas experiências usando o mesmo backend: uma UI
simples de aprovação, adequada para primeira adoção, e uma UI rica de
revisão, adequada para processos com argumentos sensíveis, múltiplas ações
ou necessidade de correção humana antes da execução real.

### 🎯 Características Principais

- **🔄 Interrupts do workflow**: Pausas programáticas em workflows de
    agentes
- **🚪 Human Gate Tool**: Ferramenta especializada para intervenção
    humana
- **🌐 API Endpoints**: `/agent/execute` e `/agent/continue` para controle
- **☁️ Stateless Cloud**: Arquitetura stateless com checkpointing
    externo
- **🔐 Controle de Acesso**: Validação granular de permissões
- **📊 Auditoria Completa**: Log detalhado de todas as intervenções

## 🏗️ Arquitetura Human-in-the-Loop

### 🎭 Sistema de Interrupts

```python
from workflow_engine.graph import StateGraph, START, END
from workflow_engine.checkpoint import PostgresSaver

class HumanInTheLoopGraph:
    """
    Grafo de execução com pontos de intervenção humana.

    CARACTERÍSTICAS:
    • Interrupts programáticos em pontos críticos
    • Estado persistente via PostgreSQL checkpointer
    • Retomada seamless após intervenção
    • Timeout automático para intervenções
    """

    def __init__(self, yaml_config: dict):
        self.yaml_config = yaml_config
        self.checkpointer = self._create_checkpointer()
        self.graph = self._build_graph()

    def _build_graph(self) -> StateGraph:
        """
        Constrói grafo com pontos de intervenção humana.
        """

        workflow = StateGraph(AgentState)

        # Nodes principais
        workflow.add_node("agent_processing", self.agent_processing_node)
        workflow.add_node("human_review", self.human_review_node)
        workflow.add_node("final_processing", self.final_processing_node)

        # Edges com condições
        workflow.add_edge(START, "agent_processing")
        workflow.add_conditional_edges(
            "agent_processing",
            self.should_interrupt_for_human,
            {
                "human_review": "human_review",
                "continue": "final_processing"
            }
        )
        workflow.add_edge("human_review", "final_processing")
        workflow.add_edge("final_processing", END)

        # Configurar interrupts
        workflow.add_interrupt("human_review")

        return workflow.compile(checkpointer=self.checkpointer)
```

### 🚪 Human Gate Tool Implementation

```python
# src/agentic_layer/tools/system_tools/human_input.py

from typing import Any, Dict, Optional
from datetime import datetime, timedelta
import asyncio
import uuid

from ..base_prometeu_tool import BasePrometeuTool

class HumanGateTool(BasePrometeuTool):
    """
    Tool para solicitar intervenção humana em workflows de agentes.

    FUNCIONALIDADES:
    • Pausar execução aguardando input humano
    • Timeout configurável para respostas
    • Múltiplos tipos de intervenção (approval, input, selection)
    • Integração com sistema de notificações
    • Auditoria completa de intervenções
    """

    name: str = "human_gate"
    description: str = (
        "Solicita intervenção humana em pontos críticos do workflow. "
        "Pausa execução até receber input humano ou timeout."
    )

    def __init__(self, yaml_config: dict, timeout_minutes: int = 30):
        """
        Inicializa Human Gate Tool.

        Args:
            yaml_config: Configuração do sistema
            timeout_minutes: Timeout para resposta humana
        """
        super().__init__(yaml_config=yaml_config)
        self.timeout_minutes = timeout_minutes
        self.pending_requests = {}

    def _run(
        self,
        request_type: str,
        message: str,
        context: Dict[str, Any] = None,
        options: list = None,
        required_approval: bool = False,
        timeout_minutes: int = None
    ) -> Dict[str, Any]:
        """
        Execução síncrona - wrapper para versão async.
        """
        import asyncio

        try:
            return asyncio.run(
                self._arun(request_type, message, context, options, required_approval, timeout_minutes)
            )
        except Exception as e:
            self.logger.exception(
                "Erro no Human Gate | correlation_id: %s",
                self.correlation_id
            )
            return {
                "status": "error",
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            }

    async def _arun(
        self,
        request_type: str,
        message: str,
        context: Dict[str, Any] = None,
        options: list = None,
        required_approval: bool = False,
        timeout_minutes: int = None
    ) -> Dict[str, Any]:
        """
        Solicita intervenção humana.

        Args:
            request_type: Tipo de intervenção (approval, input, selection, review)
            message: Mensagem para o humano
            context: Contexto adicional da situação
            options: Opções para seleção (se type=selection)
            required_approval: Se aprovação é obrigatória
            timeout_minutes: Timeout específico para esta solicitação

        Returns:
            Dict[str, Any]: Resposta da intervenção humana
        """

        try:
            # Gerar ID único para a solicitação
            request_id = str(uuid.uuid4())
            effective_timeout = timeout_minutes or self.timeout_minutes

            # Criar solicitação
            request_data = {
                "request_id": request_id,
                "correlation_id": self.correlation_id,
                "user_email": self.user_email,
                "request_type": request_type,
                "message": message,
                "context": context or {},
                "options": options,
                "required_approval": required_approval,
                "timeout_minutes": effective_timeout,
                "created_at": datetime.now(),
                "expires_at": datetime.now() + timedelta(minutes=effective_timeout),
                "status": "pending"
            }

            # Armazenar solicitação pendente
            self.pending_requests[request_id] = request_data

            self.logger.info(
                "🚨 Human Gate ativado | type: %s | request_id: %s | "
                "timeout: %d min | correlation_id: %s",
                request_type, request_id, effective_timeout, self.correlation_id
            )

            # Enviar notificação (integração com sistema de notificações)
            await self._send_notification(request_data)

            # Salvar estado para recuperação
            await self._save_request_state(request_data)

            # CRITICAL: Ponto crítico de interrupt do workflow
            # Este é o ponto onde o workflow pausa esperando intervenção
            raise self._create_human_interrupt(request_data)

        except Exception as e:
            self.logger.exception(
                "Erro ao criar Human Gate | correlation_id: %s",
                self.correlation_id
            )
            raise

    def _create_human_interrupt(self, request_data: Dict[str, Any]) -> Exception:
        """
        Cria interrupt para o motor de workflow.

        Este interrupt pausa o workflow até que uma resposta humana
        seja fornecida via API endpoint /agent/continue.
        """

        # O motor de workflow interpreta como sinal para pausar
        from workflow_engine.errors import WorkflowInterrupt

        interrupt_data = {
            "type": "human_input_required",
            "request_id": request_data["request_id"],
            "message": request_data["message"],
            "request_type": request_data["request_type"],
            "options": request_data["options"],
            "context": request_data["context"],
            "expires_at": request_data["expires_at"].isoformat(),
            "api_endpoint": "/agent/continue"
        }

        return WorkflowInterrupt(interrupt_data)

    async def _send_notification(self, request_data: Dict[str, Any]) -> None:
        """
        Envia notificação sobre solicitação de intervenção.
        """

        try:
            notification_service = NotificationService(self.yaml_config)

            await notification_service.send_human_intervention_request(
                user_email=self.user_email,
                request_id=request_data["request_id"],
                message=request_data["message"],
                urgency=self._determine_urgency(request_data),
                expires_at=request_data["expires_at"]
            )

            self.logger.debug(
                "Notificação enviada | request_id: %s | correlation_id: %s",
                request_data["request_id"], self.correlation_id
            )

        except Exception as e:
            self.logger.warning(
                "Falha ao enviar notificação: %s | correlation_id: %s",
                str(e), self.correlation_id
            )

    async def _save_request_state(self, request_data: Dict[str, Any]) -> None:
        """
        Salva estado da solicitação para recuperação.
        """

        try:
            state_manager = StateManager(self.yaml_config)

            await state_manager.save_human_request(
                request_id=request_data["request_id"],
                correlation_id=self.correlation_id,
                data=request_data
            )

        except Exception as e:
            self.logger.warning(
                "Falha ao salvar estado: %s | correlation_id: %s",
                str(e), self.correlation_id
            )

    def _determine_urgency(self, request_data: Dict[str, Any]) -> str:
        """
        Determina urgência da solicitação baseado no contexto.
        """

        if request_data["required_approval"]:
            return "high"
        elif request_data["timeout_minutes"] <= 5:
            return "urgent"
        elif request_data["timeout_minutes"] <= 30:
            return "medium"
        else:
            return "low"

    async def process_human_response(
        self,
        request_id: str,
        response: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Processa resposta humana e retoma workflow.

        Args:
            request_id: ID da solicitação
            response: Resposta fornecida pelo humano

        Returns:
            Dict[str, Any]: Resultado processado
        """

        try:
            # Validar solicitação
            if request_id not in self.pending_requests:
                raise ValueError(f"Solicitação {request_id} não encontrada")

            request_data = self.pending_requests[request_id]

            # Verificar se não expirou
            if datetime.now() > request_data["expires_at"]:
                raise ValueError("Solicitação expirada")

            # Processar resposta baseado no tipo
            processed_response = await self._process_response_by_type(
                request_data, response
            )

            # Atualizar estado
            request_data["status"] = "completed"
            request_data["response"] = processed_response
            request_data["completed_at"] = datetime.now()

            # Log de auditoria
            self.logger.info(
                " Human Gate completado | request_id: %s | "
                "response_type: %s | correlation_id: %s",
                request_id, response.get("type", "unknown"), self.correlation_id
            )

            # Remover da lista pendente
            del self.pending_requests[request_id]

            return processed_response

        except Exception as e:
            self.logger.exception(
                "Erro ao processar resposta humana | request_id: %s | correlation_id: %s",
                request_id, self.correlation_id
            )
            raise

    async def _process_response_by_type(
        self,
        request_data: Dict[str, Any],
        response: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Processa resposta baseado no tipo de solicitação.
        """

        request_type = request_data["request_type"]

        if request_type == "approval":
            return self._process_approval_response(response)
        elif request_type == "input":
            return self._process_input_response(response)
        elif request_type == "selection":
            return self._process_selection_response(request_data, response)
        elif request_type == "review":
            return self._process_review_response(response)
        else:
            raise ValueError(f"Tipo de solicitação não suportado: {request_type}")

    def _process_approval_response(self, response: Dict[str, Any]) -> Dict[str, Any]:
        """Processa resposta de aprovação."""

        approved = response.get("approved", False)
        reason = response.get("reason", "")

        return {
            "type": "approval",
            "approved": approved,
            "reason": reason,
            "action": "proceed" if approved else "halt"
        }

    def _process_input_response(self, response: Dict[str, Any]) -> Dict[str, Any]:
        """Processa resposta de input."""

        user_input = response.get("input", "")

        return {
            "type": "input",
            "user_input": user_input,
            "action": "proceed_with_input"
        }

    def _process_selection_response(
        self,
        request_data: Dict[str, Any],
        response: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Processa resposta de seleção."""

        selected_option = response.get("selected_option")
        available_options = request_data["options"]

        if selected_option not in available_options:
            raise ValueError("Opção selecionada não é válida")

        return {
            "type": "selection",
            "selected_option": selected_option,
            "action": "proceed_with_selection"
        }

    def _process_review_response(self, response: Dict[str, Any]) -> Dict[str, Any]:
        """Processa resposta de revisão."""

        approved = response.get("approved", False)
        modifications = response.get("modifications", [])
        comments = response.get("comments", "")

        return {
            "type": "review",
            "approved": approved,
            "modifications": modifications,
            "comments": comments,
            "action": "proceed_with_modifications" if modifications else ("proceed" if approved else "halt")
        }


# Factory function para o catálogo builtin persistido
def create_human_gate_tool(yaml_config: dict, **kwargs) -> HumanGateTool:
    """
    Factory function para criar Human Gate Tool.

    Args:
        yaml_config: Configuração do sistema
        **kwargs: Argumentos específicos da tool

    Returns:
        HumanGateTool: Instância configurada
    """
    return HumanGateTool(yaml_config=yaml_config, **kwargs)
```

---

## 🌐 API Endpoints para HIL

### 🚀 Endpoint: /agent/execute

```python
# API - Agent Execute Endpoint

@router.post("/execute")
async def execute_agent(
    request: AgentInvokeRequest,
    correlation_id: str = Header(None, alias="X-Correlation-ID")
) -> AgentInvokeResponse:
    """
    Inicia execução de agente com suporte a Human-in-the-Loop.

    FUNCIONALIDADES:
    • Inicia workflow de agente
    • Detecta necessidade de intervenção humana
    • Retorna status com instruções de continuação
    • Mantém estado via checkpointer
    """

    try:
        # Gerar correlation_id se não fornecido
        if not correlation_id:
            correlation_id = str(uuid.uuid4())

        logger = create_logger_with_correlation(correlation_id)

        logger.info(
            "📨 Agent execute iniciado | user: %s | yaml_config: %s",
            request.user_email, request.yaml_config_name
        )

        # Carregar configuração
        yaml_config = load_yaml_config(request.yaml_config_name)
        yaml_config['user_session'] = {
            'correlation_id': correlation_id,
            'user_email': request.user_email
        }

        # Criar orchestrator
        orchestrator = MainOrchestrator(yaml_config)

        # Executar com suporte a interrupts
        execution_result = await orchestrator.execute_with_interrupts(
            query=request.query,
            thread_id=request.thread_id or correlation_id
        )

        # Verificar se houve interrupt para Human-in-the-Loop
        if execution_result.get("status") == "interrupted":
            interrupt_data = execution_result.get("interrupt_data")

            logger.info(
                "🚨 Workflow interrompido para intervenção humana | "
                "request_id: %s | correlation_id: %s",
                interrupt_data.get("request_id"), correlation_id
            )

            return AgentInvokeResponse(
                status="interrupted",
                correlation_id=correlation_id,
                thread_id=execution_result["thread_id"],
                human_intervention=HumanInterventionData(
                    request_id=interrupt_data["request_id"],
                    request_type=interrupt_data["request_type"],
                    message=interrupt_data["message"],
                    options=interrupt_data.get("options"),
                    context=interrupt_data.get("context"),
                    expires_at=interrupt_data["expires_at"],
                    continue_endpoint="/agent/continue"
                ),
                partial_result=execution_result.get("partial_result")
            )
        else:
            # Execução completa sem interrupção
            logger.info(
                " Workflow concluído sem intervenção | correlation_id: %s",
                correlation_id
            )

            return AgentInvokeResponse(
                status="completed",
                correlation_id=correlation_id,
                thread_id=execution_result["thread_id"],
                result=execution_result["result"]
            )

    except Exception as e:
        logger.exception(
            "  Erro em agent execute | correlation_id: %s",
            correlation_id
        )

        return AgentInvokeResponse(
            status="error",
            correlation_id=correlation_id,
            error=str(e)
        )
```

### 🔄 Endpoint: /agent/continue

```python
# API - Agent Continue Endpoint

@router.post("/continue")
async def agent_continue(
    request: AgentContinueRequest,
    correlation_id: str = Header(None, alias="X-Correlation-ID")
) -> AgentContinueResponse:
    """
    Continua execução de agente após intervenção humana.

    FUNCIONALIDADES:
    • Recebe resposta da intervenção humana
    • Retoma workflow do ponto de interrupção
    • Mantém contexto e estado completos
    • Suporta múltiplas rodadas de intervenção
    """

    try:
        logger = create_logger_with_correlation(correlation_id)

        logger.info(
            "🔄 Agent continue iniciado | "
            "user: %s | correlation_id: %s",
            request.user_email, correlation_id
        )

        # Validar solicitação de intervenção
        intervention_manager = InterventionManager()

        request_id = request.request_id

        if not await intervention_manager.validate_request(request_id, request.user_email):
            raise HTTPException(
                status_code=403,
                detail=f"Acesso negado à solicitação {request_id}"
            )

        # Processar resposta humana
        human_response = await intervention_manager.process_human_response(
            request_id=request_id,
            response=request.human_response,
            user_email=request.user_email
        )

        logger.info(
            " Resposta humana processada | request_id: %s | "
            "action: %s | correlation_id: %s",
            request_id, human_response.get("action"), correlation_id
        )

        # Retomar workflow
        orchestrator = MainOrchestrator.from_checkpoint(
            thread_id=request.thread_id,
            correlation_id=correlation_id
        )

        # Continuar execução com resposta humana
        execution_result = await orchestrator.resume_with_human_input(
            human_response=human_response,
            thread_id=request.thread_id
        )

        # Verificar se há nova interrupção
        if execution_result.get("status") == "interrupted":
            interrupt_data = execution_result.get("interrupt_data")

            logger.info(
                "🚨 Nova interrupção detectada | new_request_id: %s | correlation_id: %s",
                interrupt_data.get("request_id"), correlation_id
            )

            return AgentContinueResponse(
                status="interrupted",
                correlation_id=correlation_id,
                thread_id=execution_result["thread_id"],
                human_intervention=HumanInterventionData(
                    request_id=interrupt_data["request_id"],
                    request_type=interrupt_data["request_type"],
                    message=interrupt_data["message"],
                    options=interrupt_data.get("options"),
                    context=interrupt_data.get("context"),
                    expires_at=interrupt_data["expires_at"],
                    continue_endpoint="/agent/continue"
                ),
                partial_result=execution_result.get("partial_result")
            )
        else:
            # Execução final completa
            logger.info(
                "🎯 Workflow completamente concluído | correlation_id: %s",
                correlation_id
            )

            return AgentContinueResponse(
                status="completed",
                correlation_id=correlation_id,
                thread_id=execution_result["thread_id"],
                result=execution_result["result"]
            )

    except HTTPException:
        raise
    except Exception as e:
        logger.exception(
            "  Erro em agent continue | request_id: %s | correlation_id: %s",
            request_id, correlation_id
        )

        return AgentContinueResponse(
            status="error",
            correlation_id=correlation_id,
            error=str(e)
        )
```

### 📋 Data Models para API

```python
# models/api_models.py

from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, List
from datetime import datetime

class AgentInvokeRequest(BaseModel):
    """Request para /agent/execute."""

    query: str = Field(..., description="Query para processar")
    user_email: str = Field(..., description="Email do usuário")
    yaml_config_name: str = Field(..., description="Nome da configuração YAML")
    thread_id: Optional[str] = Field(None, description="ID do thread para continuidade")
    timeout_minutes: Optional[int] = Field(30, description="Timeout para intervenções")

class HumanInterventionData(BaseModel):
    """Dados da intervenção humana necessária."""

    request_id: str = Field(..., description="ID único da solicitação")
    request_type: str = Field(..., description="Tipo: approval, input, selection, review")
    message: str = Field(..., description="Mensagem para o humano")
    options: Optional[List[str]] = Field(None, description="Opções para seleção")
    context: Optional[Dict[str, Any]] = Field(None, description="Contexto adicional")
    expires_at: str = Field(..., description="Data/hora de expiração")
    continue_endpoint: str = Field(..., description="Endpoint para continuar")

class AgentInvokeResponse(BaseModel):
    """Response de /agent/execute."""

    status: str = Field(..., description="Status: completed, interrupted, error")
    correlation_id: str = Field(..., description="ID de correlação")
    thread_id: Optional[str] = Field(None, description="ID do thread")
    result: Optional[Dict[str, Any]] = Field(None, description="Resultado se completo")
    partial_result: Optional[Dict[str, Any]] = Field(None, description="Resultado parcial")
    human_intervention: Optional[HumanInterventionData] = Field(None, description="Dados da intervenção")
    error: Optional[str] = Field(None, description="Mensagem de erro")

class HumanResponseData(BaseModel):
    """Resposta humana para intervenção."""

    type: str = Field(..., description="Tipo da resposta")
    approved: Optional[bool] = Field(None, description="Aprovação (para approval/review)")
    input: Optional[str] = Field(None, description="Input textual (para input)")
    selected_option: Optional[str] = Field(None, description="Opção selecionada (para selection)")
    reason: Optional[str] = Field(None, description="Razão da decisão")
    modifications: Optional[List[Dict[str, Any]]] = Field(None, description="Modificações (para review)")
    comments: Optional[str] = Field(None, description="Comentários adicionais")

class AgentContinueRequest(BaseModel):
    """Request para /agent/continue."""

    request_id: str = Field(..., description="ID da intervenção pendente")
    user_email: str = Field(..., description="Email do usuário")
    thread_id: str = Field(..., description="ID do thread")
    human_response: HumanResponseData = Field(..., description="Resposta da intervenção humana")

class AgentContinueResponse(BaseModel):
    """Response de /agent/continue."""

    status: str = Field(..., description="Status: completed, interrupted, error")
    correlation_id: str = Field(..., description="ID de correlação")
    thread_id: str = Field(..., description="ID do thread")
    result: Optional[Dict[str, Any]] = Field(None, description="Resultado se completo")
    partial_result: Optional[Dict[str, Any]] = Field(None, description="Resultado parcial")
    human_intervention: Optional[HumanInterventionData] = Field(None, description="Nova intervenção se necessária")
    error: Optional[str] = Field(None, description="Mensagem de erro")
```

---

## ☁️ Arquitetura Stateless Cloud

### 🔧 PostgreSQL Checkpointer

```python
# src/agentic_layer/checkpoint/cloud_checkpointer.py

from workflow_engine.checkpoint import PostgresSaver
from typing import Optional, Dict, Any
import os

class CloudCheckpointer:
    """
    Checkpointer otimizado para ambientes cloud stateless.

    CARACTERÍSTICAS:
    • PostgreSQL como backend persistente
    • Otimizado para alta concorrência
    • Cleanup automático de estados antigos
    • Compressão de estados grandes
    • Sharding por tenant/usuário
    """

    def __init__(self, yaml_config: dict):
        self.yaml_config = yaml_config
        self.connection_string = self._get_connection_string()
        self.checkpointer = self._create_postgres_saver()

    def _get_connection_string(self) -> str:
        """
        Obtém connection string do PostgreSQL.

        FONTES (em ordem de prioridade):
        1. Configuração YAML
        2. Variável de ambiente
        3. Configuração padrão para desenvolvimento
        """

        # 1. Configuração YAML
        memory_config = self.yaml_config.get('memory', {})
        if memory_config.get('connection_string'):
            return memory_config['connection_string']

        # 2. Variável de ambiente
        env_connection = os.getenv('POSTGRES_CHECKPOINT_URL')
        if env_connection:
            return env_connection

        # 3. Fallback para desenvolvimento
        return "postgresql://user:password@localhost:5432/prometeu_checkpoints"

    def _create_postgres_saver(self) -> PostgresSaver:
        """
        Cria PostgreSQL checkpointer com configurações otimizadas.
        """

        # Context manager approach (padrão oficial)
        context_manager = PostgresSaver.from_conn_string(self.connection_string)
        saver = context_manager.__enter__()

        # Setup das tabelas na primeira vez
        saver.setup()

        return saver

    async def cleanup_old_checkpoints(self, days_to_keep: int = 30) -> int:
        """
        Remove checkpoints antigos para otimizar performance.

        Args:
            days_to_keep: Dias para manter checkpoints

        Returns:
            int: Número de checkpoints removidos
        """

        # Implementar lógica de cleanup baseada em timestamp
        # Isso ajuda a manter a base de dados performática
        pass

    def get_thread_state(self, thread_id: str) -> Optional[Dict[str, Any]]:
        """
        Obtém estado atual de um thread específico.
        """

        try:
            # Usar checkpointer para obter último estado
            state = self.checkpointer.get_tuple(
                {"configurable": {"thread_id": thread_id}}
            )

            return state.checkpoint if state else None

        except Exception as e:
            logger.exception("Erro ao obter estado do thread %s: %s", thread_id, e)
            return None

    def list_user_threads(self, user_email: str) -> List[str]:
        """
        Lista threads associados a um usuário.

        Útil para implementar histórico de conversações
        e continuidade de sessões.
        """

        # Implementar query para listar threads por usuário
        # Baseado em metadados do checkpoint
        pass
```

### 🔄 State Management

```python
# src/agentic_layer/state/state_manager.py

class StateManager:
    """
    Gerenciador de estado para workflows HIL.

    RESPONSABILIDADES:
    • Persistir estado de intervenções pendentes
    • Recuperar contexto após interrupções
    • Manter histórico de decisões humanas
    • Cleanup automático de estados expirados
    """

    def __init__(self, yaml_config: dict):
        self.yaml_config = yaml_config
        self.checkpointer = CloudCheckpointer(yaml_config)

    async def save_human_request(
        self,
        request_id: str,
        correlation_id: str,
        data: Dict[str, Any]
    ) -> None:
        """
        Salva solicitação de intervenção humana.
        """

        state_key = f"human_request_{request_id}"

        await self.checkpointer.put(
            key=state_key,
            value={
                "request_data": data,
                "correlation_id": correlation_id,
                "created_at": datetime.now().isoformat(),
                "status": "pending"
            }
        )

    async def get_human_request(self, request_id: str) -> Optional[Dict[str, Any]]:
        """
        Recupera solicitação de intervenção humana.
        """

        state_key = f"human_request_{request_id}"

        return await self.checkpointer.get(state_key)

    async def complete_human_request(
        self,
        request_id: str,
        response: Dict[str, Any]
    ) -> None:
        """
        Marca solicitação como completa.
        """

        state_key = f"human_request_{request_id}"

        # Atualizar estado
        existing_state = await self.get_human_request(request_id)
        if existing_state:
            existing_state["status"] = "completed"
            existing_state["response"] = response
            existing_state["completed_at"] = datetime.now().isoformat()

            await self.checkpointer.put(state_key, existing_state)

    async def cleanup_expired_requests(self) -> int:
        """
        Remove solicitações expiradas.
        """

        # Implementar cleanup baseado em expires_at
        # Retornar número de solicitações removidas
        pass
```

---

## 🔐 Segurança e Auditoria

### 🛡️ Validação de Acesso

```python
class HILSecurityValidator:
    """
    Validador de segurança para Human-in-the-Loop.
    """

    def __init__(self, yaml_config: dict):
        self.yaml_config = yaml_config
        self.rbac = RoleBasedAccessControl(yaml_config)

    async def validate_intervention_access(
        self,
        user_email: str,
        request_id: str,
        intervention_type: str
    ) -> bool:
        """
        Valida se usuário pode responder à intervenção.
        """

        try:
            # 1. Verificar se usuário existe
            user_roles = await self.rbac.get_user_roles(user_email)
            if not user_roles:
                return False

            # 2. Verificar permissões para tipo de intervenção
            required_permission = f"hil.{intervention_type}"
            if not await self.rbac.user_has_permission(user_email, required_permission):
                return False

            # 3. Verificar se solicitação ainda é válida
            request_data = await self.get_request_data(request_id)
            if not request_data:
                return False

            # 4. Verificar se não expirou
            expires_at = datetime.fromisoformat(request_data["expires_at"])
            if datetime.now() > expires_at:
                return False

            return True

        except Exception as e:
            logger.exception(
                "Erro na validação de acesso HIL | user: %s | request_id: %s",
                user_email, request_id
            )
            return False
```

### 📊 Sistema de Auditoria

```python
class HILAuditLogger:
    """
    Logger de auditoria para intervenções humanas.
    """

    def __init__(self, yaml_config: dict):
        self.yaml_config = yaml_config
        self.audit_store = AuditStore(yaml_config)

    async def log_intervention_request(
        self,
        request_id: str,
        correlation_id: str,
        user_email: str,
        request_type: str,
        context: Dict[str, Any]
    ) -> None:
        """
        Log de solicitação de intervenção.
        """

        audit_record = {
            "event_type": "hil_request_created",
            "request_id": request_id,
            "correlation_id": correlation_id,
            "user_email": user_email,
            "request_type": request_type,
            "context": context,
            "timestamp": datetime.now().isoformat(),
            "source": "human_gate_tool"
        }

        await self.audit_store.log(audit_record)

    async def log_intervention_response(
        self,
        request_id: str,
        correlation_id: str,
        user_email: str,
        response: Dict[str, Any],
        processing_time_seconds: float
    ) -> None:
        """
        Log de resposta à intervenção.
        """

        audit_record = {
            "event_type": "hil_response_received",
            "request_id": request_id,
            "correlation_id": correlation_id,
            "user_email": user_email,
            "response": response,
            "processing_time_seconds": processing_time_seconds,
            "timestamp": datetime.now().isoformat(),
            "source": "agent_continue_endpoint"
        }

        await self.audit_store.log(audit_record)

    async def generate_hil_report(
        self,
        start_date: datetime,
        end_date: datetime,
        user_email: Optional[str] = None
    ) -> Dict[str, Any]:
        """
        Gera relatório de atividades HIL.
        """

        records = await self.audit_store.query_records(
            start_date=start_date,
            end_date=end_date,
            user_email=user_email,
            event_types=["hil_request_created", "hil_response_received"]
        )

        return {
            "period": {
                "start_date": start_date.isoformat(),
                "end_date": end_date.isoformat()
            },
            "total_requests": len([r for r in records if r["event_type"] == "hil_request_created"]),
            "total_responses": len([r for r in records if r["event_type"] == "hil_response_received"]),
            "response_rate": self._calculate_response_rate(records),
            "avg_response_time": self._calculate_avg_response_time(records),
            "by_request_type": self._group_by_request_type(records),
            "by_user": self._group_by_user(records) if not user_email else None
        }
```

---

## 📱 Exemplos de Uso

### 🎯 Cenário 1: Aprovação de Documento Crítico

```python
# Exemplo de workflow com aprovação obrigatória

async def document_approval_workflow():
    """
    Workflow que solicita aprovação humana para documento crítico.
    """

    # Análise inicial do documento
    document_analysis = await analyze_document(document_path)

    # Verificar se precisa aprovação humana
    if document_analysis["risk_score"] > 0.7:

        # Solicitar intervenção humana
        human_gate = HumanGateTool(yaml_config)

        approval_request = await human_gate._arun(
            request_type="approval",
            message=f"""
            Documento de alto risco detectado!

            Arquivo: {document_analysis['filename']}
            Risk Score: {document_analysis['risk_score']}

            Principais riscos identificados:
            {format_risks(document_analysis['risks'])}

            Aprovação necessária para prosseguir com o processamento.
            """,
            context={
                "document_path": document_path,
                "risk_score": document_analysis["risk_score"],
                "risks": document_analysis["risks"]
            },
            required_approval=True,
            timeout_minutes=60
        )

        # Workflow pausa aqui até resposta humana
        # Retomada acontece via /agent/continue endpoint

    # Continuar processamento se aprovado
    return await process_approved_document(document_path)
```

### 🎯 Cenário 2: Seleção de Estratégia

```python
# Exemplo de workflow com seleção de opções

async def strategy_selection_workflow():
    """
    Workflow que permite seleção humana de estratégia.
    """

    # Analisar situação
    situation_analysis = await analyze_current_situation()

    # Gerar opções estratégicas
    strategy_options = await generate_strategy_options(situation_analysis)

    # Solicitar seleção humana
    human_gate = HumanGateTool(yaml_config)

    selection_request = await human_gate._arun(
        request_type="selection",
        message="Múltiplas estratégias viáveis identificadas. Selecione a abordagem preferida:",
        context={
            "situation": situation_analysis,
            "analysis_timestamp": datetime.now().isoformat()
        },
        options=[
            "Estratégia Conservadora - Baixo risco, retorno moderado",
            "Estratégia Equilibrada - Risco médio, retorno balanceado",
            "Estratégia Agressiva - Alto risco, potencial alto retorno"
        ],
        timeout_minutes=30
    )

    # Workflow pausa para seleção humana
    # Continua com estratégia selecionada
```

### 🎯 Cenário 3: Input Customizado

```python
# Exemplo de workflow com input textual

async def custom_input_workflow():
    """
    Workflow que solicita input textual específico.
    """

    # Detectar necessidade de input customizado
    if requires_custom_input(context):

        human_gate = HumanGateTool(yaml_config)

        input_request = await human_gate._arun(
            request_type="input",
            message="""
            Situação especial detectada que requer input customizado.

            Por favor, forneça instruções específicas sobre como proceder:

            Contexto:
            - Processo: Análise de contrato
            - Cliente: Cliente Premium
            - Cláusula problemática identificada

            Instruções necessárias:
            """,
            context={
                "process_type": "contract_analysis",
                "client_tier": "premium",
                "issue_type": "problematic_clause"
            },
            timeout_minutes=45
        )

    # Workflow pausa para input humano
    # Continua com instruções fornecidas
```

---

## 📊 Monitoramento e Métricas

### 📈 Métricas de HIL

```python
class HILMetrics:
    """
    Coleta e análise de métricas HIL.
    """

    def collect_metrics(self) -> Dict[str, Any]:
        """
        Coleta métricas completas do sistema HIL.
        """

        return {
            "requests": {
                "total_requests": self.count_total_requests(),
                "pending_requests": self.count_pending_requests(),
                "completed_requests": self.count_completed_requests(),
                "expired_requests": self.count_expired_requests(),
                "by_type": self.group_requests_by_type()
            },
            "response_times": {
                "avg_response_time_minutes": self.calculate_avg_response_time(),
                "median_response_time_minutes": self.calculate_median_response_time(),
                "p95_response_time_minutes": self.calculate_p95_response_time()
            },
            "success_rates": {
                "overall_response_rate": self.calculate_response_rate(),
                "approval_rate": self.calculate_approval_rate(),
                "timeout_rate": self.calculate_timeout_rate()
            },
            "user_activity": {
                "active_users": self.count_active_users(),
                "most_active_users": self.get_most_active_users(),
                "user_response_rates": self.calculate_user_response_rates()
            }
        }
```

---

## 🚀 Roadmap e Melhorias

### 📅 Próximas Implementações

#### 🔄 Fase 1: Melhorias Core

- [ ] Suporte a intervenções em lote
- [ ] Delegação de intervenções
- [ ] Templates de mensagens customizáveis
- [ ] Notificações push em tempo real

#### 📱 Fase 2: Interface de Usuário

- [ ] Dashboard web para intervenções
- [ ] App mobile para aprovações
- [ ] Integração com Slack/Teams
- [ ] Sistema de escalação automática

#### 🤖 Fase 3: IA Assistida

- [ ] Predição de necessidade de intervenção
- [ ] Sugestões automáticas de resposta
- [ ] Análise de padrões de decisão
- [ ] Otimização automática de workflows

---

## 📚 Referências e Recursos

### 📖 Documentação Relacionada

- [Supervisor clássico](./README-AGENTE-SUPERVISOR.md)
- [DeepAgent governado](./README-DEEPAGENTS-SUPERVISOR.md)
- [Workflow determinístico](./README-AGENTE-WORKFLOW.md)
- [Guia de tools do produto](./GUIA-USUARIO-TOOLS.md)
- [AG-UI e renderização da pausa humana](./README-AG-UI.md)

### 🔗 Links Úteis

- [Tutorial de DeepAgent com HIL via API](./tutorial-101-exemplos-api-deepagent-hil-execute-continue.md)
- [Tutorial 101 de HIL Generative UI](./tutorial-101-human-in-the-loop.md)
- [Tutorial de DeepAgents](./tutorial-101-deepagents.md)
- [Configuração YAML da plataforma](./README-CONFIGURACAO-YAML.md)
- [AST agentic e fluxo validate/confirm](./README-AST-AGENTIC-DESIGNER.md)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Models](https://docs.pydantic.dev/)

### 💡 Exemplos de Integração

```bash
# Cliente HTTP para testar endpoints
curl -X POST "http://localhost:8000/agent/execute" \\
  -H "Content-Type: application/json" \\
  -H "X-Correlation-ID: test-123" \\
  -d '{
    "query": "Analise este documento crítico",
    "user_email": "admin@company.com",
    "yaml_config_name": "rag-config-hil-demo.yaml"
  }'

# Resposta de intervenção
curl -X POST "http://localhost:8000/agent/continue" \\
  -H "Content-Type: application/json" \\
  -H "X-Correlation-ID: test-123" \\
  -d '{
        "request_id": "req-456",
    "user_email": "admin@company.com",
    "thread_id": "thread-789",
    "human_response": {
      "type": "approval",
      "approved": true,
      "reason": "Documento revisado e aprovado"
    }
  }'
```

---

**🤝 Human-in-the-Loop** - Plataforma de Agentes de IA
*Intervenção humana inteligente em workflows autônomos*
