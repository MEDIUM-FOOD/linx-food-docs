# Plano de consolidacao: Instagram comentarios -> resposta publica + DM

## 1. O que este plano e

Este documento nao descreve uma implementacao do zero.

O codigo lido mostra que a base tecnica para o fluxo de comentario do
Instagram ja existe em partes importantes: o parser do webhook aceita
comentarios e mencoes, o responder sabe montar reply publico e sequencia
de mensagens, o cliente HTTP sabe enviar DM e responder comentario
publicamente, e o onboarding ja registra webhook com campos de comments,
mentions e messages.

O problema real agora nao e "criar suporte Instagram". O problema real e
consolidar essa base em uma capacidade operacional de produto, com regra
de negocio clara, governanca, telemetria, workflow dedicado e criterios
de go-live que evitem prometer uma automacao ponta a ponta sem cobertura
completa.

Leitura 101: a fundacao do predio ja foi erguida. O que falta e fechar a
obra, ligar a instalacao eletrica, testar tudo e provar que o fluxo fica
seguro em producao.

## 2. Que problema este plano resolve

Sem esse plano de consolidacao, a plataforma fica em uma zona perigosa:

- o repositorio da a impressao de que comentarios do Instagram ja podem
  virar atendimento completo;
- o comercial pode vender uma esteira ponta a ponta antes de existirem
  controles reais de elegibilidade, janela e observabilidade;
- a operacao pode ativar um tenant e descobrir tarde demais que faltava
  workflow, politica de handoff ou medicao de conversao.

Na pratica, este plano existe para transformar capacidade tecnica parcial
em capacidade operacional governada.

## 3. Visao executiva

Para a lideranca, este plano reduz um risco classico de plataforma:
confundir bloco tecnico isolado com feature pronta para cliente.

O valor executivo e direto:

- reduz risco de go-live incompleto;
- organiza entregas por fases auditaveis;
- separa o que ja esta implementado do que ainda depende de produto e
  operacao;
- cria base para demonstrar maturidade em automacao de canal social sem
  inflar promessas.

## 4. Visao comercial

Comercialmente, o fluxo comentario -> reply publico -> DM e valioso
porque transforma engajamento publico em conversa privada e, depois,
atendimento qualificado.

Mas a promessa vendavel precisa ser honesta.

Hoje, com base no codigo lido, a mensagem correta para cliente e esta:

- a plataforma ja possui fundacoes para receber eventos do Instagram,
  responder comentarios e enviar DM;
- a operacionalizacao completa por tenant ainda depende de workflow,
  politicas de negocio e observabilidade especificas;
- a entrega segura deve ser tratada como consolidacao assistida, e nao
  como funcionalidade irrestrita pronta para qualquer tenant.

## 5. Visao estrategica

Este plano fortalece a estrategia da plataforma em quatro frentes:

- reaproveita o core de canais ja existente, em vez de criar um fluxo
  paralelo exclusivo para Instagram;
- reforca a direcao YAML-first, porque a parte que falta esta mais em
  orquestracao e configuracao do que em adaptadores HTTP;
- melhora governanca de rollout, separando capacidade interna de oferta
  comercial pronta;
- prepara terreno para correlacao de atendimento entre canais, mas sem
  afirmar isso como realidade antes de evidencias adicionais.

## 6. O que o codigo ja confirma hoje

### 6.1 Entrada de eventos

O parser do Instagram aceita objetos de webhook do tipo instagram e page.
Ele converte eventos de messaging, postback, status e tambem eventos de
comments e mentions em IncomingMessagePayload.

Para comentario, o parser ja extrai e preserva:

- sender_id;
- comment_id;
- post_id ou media_id;
- field do evento;
- verb;
- source_event bruto para auditoria.

Isso e importante porque o sistema ja nao trata comentario como texto
anonimo. Ele guarda identificadores que permitem acao posterior e
rastreabilidade basica.

### 6.2 Saida para reply publico e DM

O responder do Instagram ja suporta dois mecanismos relevantes:

- instagram_comment_reply: instrucao para resposta publica ao comentario;
- instagram_sequence: sequencia customizada de mensagens para o canal.

Na pratica, isso significa que o core de resposta ja consegue montar uma
saida hibrida: primeiro reply publico, depois DM ou passos adicionais.

### 6.3 Entrega na Graph API

O cliente do Instagram ja implementa dois caminhos de entrega:

- envio de mensagens via endpoint de messages da conta de negocio;
- resposta publica via endpoint de replies do comment_id.

O cliente tambem trata erros HTTP, mascara token em log e devolve falha
explicita quando a chamada quebra.

### 6.4 Provisionamento e onboarding

O onboarding ja configura webhook do aplicativo com os campos comments,
mentions e messages. Isso prova que o caminho de assinatura do webhook ja
foi pensado para esse escopo, e nao apenas para DM tradicional.

### 6.5 Superficie operacional

O router ja oferece pontos de apoio para operacao e diagnostico:

- endpoint de submissao de mensagens Instagram;
- endpoint manual de envio;
- endpoint de historico;
- endpoint de teste do webhook para validar parse sem depender de evento
  real da Meta.

### 6.6 Cobertura automatizada existente

Os testes unitarios lidos ja cobrem pontos criticos da base:

- normalizacao de comentario no parser;
- bloco de comment_reply no responder;
- roteamento do cliente para _send_comment_reply;
- configuracao do onboarding com comments, mentions e messages.

Conclusao pratica: a fundacao tecnica minima existe e ja possui alguma
protecao automatizada.

## 7. O que ainda nao foi confirmado no codigo lido

Alguns pontos do documento antigo eram apresentados como prontos, mas nao
foram confirmados na leitura forense desta rodada.

### 7.1 Janela de 24 horas

Nao foi confirmada, no codigo lido para este tema, uma regra explicita de
elegibilidade que bloqueie DM fora da janela de 24 horas da Meta.

Isso nao significa que a regra nao exista em outro ponto do repositorio.
Significa apenas que este comportamento nao deve ser documentado como
garantia operacional sem leitura adicional que prove a checagem.

### 7.2 Workflow dedicado de follow-up

Nao foi encontrado, nesta rodada, um workflow canonicamente nomeado para
comentario Instagram com qualificacao e handoff humano.

Isso indica que a orquestracao de negocio ainda precisa ser tratada como
lacuna de produto ou, no minimo, como item a confirmar antes de vender o
fluxo como pronto.

### 7.3 Correlacao cross-channel com WhatsApp

Nao foi confirmada implementacao explicita de migracao ou correlacao
Instagram -> WhatsApp usando comment_id como elo operacional oficial.

Existe suporte geral a canais e telemetria, mas nao ha evidencia lida de
um fluxo cross-channel dedicado para esse caso.

### 7.4 Metricas de conversao comentario -> DM

Nao foi encontrada, nesta rodada, uma camada dedicada que consolide
metricas especificas dessa jornada, como taxa de reply publico, taxa de
DM enviada, taxa de resposta do usuario e conversao para atendimento.

## 8. Arquitetura da consolidacao

Este plano deve ser entendido em quatro blocos logicos.

### 8.1 Bloco de captura

Responsabilidade: receber evento bruto da Meta e traduzi-lo para o
contrato interno do sistema.

O que ja existe:

- parser de webhook para comments, mentions, mensagens, postbacks e
  status;
- preservacao de metadados essenciais para comentario.

O que ainda precisa amadurecer:

- regra explicita para distinguir comentario apto a reply + DM de evento
  que deve apenas ser registrado;
- contratos mais fortes para comentarista, post de origem e elegibilidade.

### 8.2 Bloco de orquestracao

Responsabilidade: decidir se a interacao gera reply publico, DM, handoff
humano, silencio controlado ou apenas telemetria.

O que ja existe:

- responder e cliente capazes de executar reply publico e DM.

O que ainda precisa amadurecer:

- workflow YAML dedicado;
- regra de negocio para janela, template de convite, qualificacao e
  fallback;
- politicas por tenant e por campanha.

### 8.3 Bloco de entrega

Responsabilidade: executar a chamada para Graph API e retornar falha ou
sucesso observavel.

O que ja existe:

- cliente HTTP com envio de DM;
- helper dedicado para comment replies;
- tratamento de erro e retorno explicito.

O que ainda precisa amadurecer:

- retry governado por politica de negocio desse caso de uso;
- criterios de reprocessamento operacional;
- diferencas entre falha definitiva, falha transitoria e falha de regra
  da Meta.

### 8.4 Bloco de observabilidade

Responsabilidade: provar o que aconteceu e permitir diagnostico de ponta
a ponta.

O que ja existe:

- logs e telemetria geral do canal;
- endpoint de teste do webhook;
- historico operacional local do canal.

O que ainda precisa amadurecer:

- dashboard e indicadores especificos do funil comentario -> DM;
- trilha clara de decisao para motivos de nao envio;
- playbook operacional para suporte e sucesso do cliente.

## 9. Plano por fases

### Fase 1. Fechar contrato operacional minimo

Objetivo: tirar a feature da zona cinzenta.

Escopo:

- definir quais eventos de comments e mentions sao elegiveis para reply
  publico e para tentativa de DM;
- confirmar ou implementar a regra de janela de 24 horas;
- definir invariantes obrigatorias de metadata, como comment_id, sender_id
  e post_id;
- fechar a politica de erro: quando loga, quando reprocessa e quando
  falha fechado.

Saida esperada:

- contrato funcional claro e testavel;
- documento de regra de negocio sem ambiguidade;
- proibicao explicita de prometer automacao irrestrita antes desse gate.

### Fase 2. Criar orquestracao de negocio por YAML

Objetivo: transformar a base tecnica em fluxo configuravel de produto.

Escopo:

- criar workflow ou supervisor especifico para follow-up de comentario;
- padronizar mensagem publica inicial e convite para DM;
- incluir classificacao de casos para FAQ, vendas, suporte ou handoff;
- garantir que o tenant consiga ativar ou desativar a jornada por
  configuracao.

Saida esperada:

- fluxo agentic reproduzivel;
- comportamento por tenant sem hardcode em router ou cliente;
- ponto unico de governanca para o canal.

### Fase 3. Observabilidade e metricas

Objetivo: tornar a jornada auditavel e gerenciavel.

Escopo:

- medir comentario recebido, reply publicado, DM enviada, DM entregue,
  resposta do usuario e handoff;
- registrar motivos de nao envio, como regra de elegibilidade ou erro do
  provider;
- criar indicadores operacionais para suporte e lideranca.

Saida esperada:

- capacidade real de diagnostico;
- base para SLA operacional;
- evidencias para comercial e customer success.

### Fase 4. Go-live controlado

Objetivo: ativar a feature sem ampliar risco de reputacao ou de
compliance.

Escopo:

- selecionar tenants piloto;
- validar credenciais, webhook e fluxo real;
- acompanhar taxa de erro e conversao;
- revisar linguagem das mensagens e politicas da Meta.

Saida esperada:

- rollout progressivo;
- aprendizado real de campo;
- base para oferta comercial mais ampla.

## 10. Criterios de aceite reais

O fluxo comentario -> reply publico + DM so deve ser tratado como pronto
quando todos os itens abaixo estiverem fechados:

- existe regra explicita e testada para elegibilidade do envio;
- existe workflow canonicamente associado ao caso de uso;
- o sistema registra por que enviou, por que nao enviou e o que falhou;
- o tenant consegue ser provisionado sem passos informais escondidos;
- existe validacao automatizada do caminho feliz e dos principais erros;
- operacao consegue diagnosticar falha sem ler codigo-fonte.

## 11. O que acontece em caso de sucesso

No estado desejado de consolidacao, o fluxo de sucesso deve ser este:

- o webhook recebe comentario ou mencao;
- o parser produz payload interno com identificadores confiaveis;
- a camada de orquestracao decide se ha reply publico, DM ou ambos;
- o responder monta a sequencia certa;
- o cliente entrega cada etapa na Graph API;
- telemetria e historico permitem provar o que ocorreu.

Hoje, a maior parte da parte tecnica desse caminho ja existe. O que ainda
nao esta comprovado de ponta a ponta e a governanca de negocio desse
fluxo.

## 12. O que acontece em caso de erro

Os erros hoje confirmados no codigo lido estao mais concentrados na
entrega e na configuracao:

- webhook invalido dispara ValueError no parser;
- mensagem sem recipient ou sem bloco message gera erro no cliente;
- comment_reply sem comment_id ou sem texto falha explicitamente;
- token placeholder ou credencial ausente tambem falham de forma
  explicita;
- erros HTTP da Graph API sao convertidos em ChannelDeliveryError.

O que ainda precisa amadurecer e o tratamento de erro de negocio, por
exemplo: comentario inelegivel, janela vencida, tenant sem workflow ativo
ou politica comercial que proiba DM naquele contexto.

## 13. Troubleshooting

### Sintoma: comentario chegou, mas nao virou DM

Causas provaveis:

- nao havia workflow de negocio configurado para esse tenant;
- a resposta gerada nao montou instagram_sequence;
- a elegibilidade para DM ainda nao foi formalizada;
- houve falha de entrega na Graph API.

Como investigar:

- validar o parse no endpoint de teste do webhook;
- inspecionar historico do canal e logs da entrega;
- confirmar se o payload final contem comment_reply e mensagens de DM;
- revisar credenciais e status HTTP retornado pelo provider.

### Sintoma: reply publico nao sai

Causas provaveis:

- comment_id ausente no metadata;
- instagram_comment_reply nao foi montado;
- erro HTTP no endpoint de replies.

Como investigar:

- confirmar se o parser preservou comment_id;
- revisar a montagem do outgoing_message;
- checar erro retornado pelo cliente do Instagram.

### Sintoma: comercial quer vender fluxo completo

Causa provavel:

- confusao entre base tecnica existente e operacao completa pronta.

Acao recomendada:

- usar este documento como criterio de maturidade;
- prometer apenas o que estiver fechado nas fases 1 a 4;
- nao transformar suporte basal em feature enterprise pronta sem gate de
  rollout.

## 14. Explicacao 101

Pense neste fluxo como uma equipe de atendimento em quatro pessoas.

- a primeira pessoa recebe o comentario e anota quem falou e em qual
  post;
- a segunda decide se vale responder em publico, chamar para DM ou escalar;
- a terceira envia as mensagens na API da Meta;
- a quarta acompanha se deu certo e registra o historico.

O repositorio ja contratou a primeira e a terceira pessoa. A segunda e a
quarta ainda precisam de treinamento e processo formal. E por isso que o
assunto ainda deve ser tratado como consolidacao, nao como capacidade
plena pronta para qualquer cliente.

## 15. Checklist de entendimento

- Entendi que o parser de comments e mentions ja existe.
- Entendi que o responder ja sabe montar reply publico e DM.
- Entendi que o cliente HTTP ja envia comentario e DM por caminhos
  separados.
- Entendi que o onboarding ja assina comments, mentions e messages.
- Entendi que o gargalo principal nao e mais o adapter HTTP.
- Entendi que a lacuna maior esta em regra de negocio, workflow e
  observabilidade.
- Entendi que a janela de 24 horas nao foi confirmada no codigo lido desta
  rodada.
- Entendi que a oferta comercial precisa ser menor que a ambicao tecnica
  ate o rollout controlado.

## 16. Evidencias no codigo

- src/channel_layer/instagram_processor.py
  - Motivo da leitura: confirmar se comments e mentions ja entram no
    contrato interno.
  - Simbolo relevante: InstagramMessageProcessor._build_comment_payload.
  - Comportamento confirmado: parser extrai comment_id, post_id e sender.

- src/channel_layer/responders/instagram_responder.py
  - Motivo da leitura: verificar se reply publico e sequencia de DM ja
    existem.
  - Simbolo relevante: metodos build comment reply e build sequence.
  - Comportamento confirmado: outgoing_message pode gerar reply publico e
    passos customizados.

- src/channel_layer/clients/instagram_message_client.py
  - Motivo da leitura: confirmar entrega real para DM e replies.
  - Simbolo relevante: metodos send batch e send comment reply.
  - Comportamento confirmado: cliente envia mensagens e responde
    comentarios via endpoint dedicado.

- src/channel_layer/services/instagram_onboarding.py
  - Motivo da leitura: confirmar campos de webhook provisionados.
  - Simbolo relevante: configure_webhook.
  - Comportamento confirmado: webhook usa comments, mentions e messages.

- src/api/routers/channel_router.py
  - Motivo da leitura: localizar superficie operacional e endpoint de
    teste.
  - Simbolo relevante: submit_instagram_message e test_instagram_webhook.
  - Comportamento confirmado: existe rota de entrada normalizada e rota de
    teste seguro do parser.

- tests/unit/test_instagram_message_processor.py
  - Motivo da leitura: verificar cobertura do parse de comentario.
  - Comportamento confirmado: comentario vira payload com comment_id e
    post_id.

- tests/unit/test_instagram_message_client.py
  - Motivo da leitura: verificar cobertura do caminho comment_reply.
  - Comportamento confirmado: send_batch roteia comment_reply para helper
    dedicado.

- tests/unit/test_instagram_responder.py
  - Motivo da leitura: verificar cobertura da composicao reply publico +
    DM.
  - Comportamento confirmado: payload final pode incluir comentario e DM
    na mesma resposta.

- tests/unit/test_instagram_onboarding.py
  - Motivo da leitura: verificar onboarding e campos assinados.
  - Comportamento confirmado: onboarding cobre subscribed_fields e
    configure webhook.
