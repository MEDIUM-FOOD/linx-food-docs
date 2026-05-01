# Plano de implementacao: comentarios Instagram -> resposta publica + DM

## Objetivo

Entregar suporte completo ao fluxo descrito na proposta comercial:
- Responder automaticamente comentarios e mencoes.
- Converter comentario em DM (dentro das regras Meta, janela 24h).
- Manter follow-up automatico e handoff humano via workflows.
- Consolidar métricas e correlacao cross-channel (Instagram/WhatsApp).

### Para quem esta começando (for dummies)
Este plano explica como transformar comentarios em atendimento real. A
ideia e simples: responder o comentario em publico e continuar a conversa
em DM quando for permitido.

### Impacto para o usuario
- Menos comentarios perdidos.
- Conversas iniciadas no Instagram viram atendimento de fato.
- Rastreabilidade total entre comentario e DM.

### Limites e pegadinhas
- A janela de 24h e obrigatoria para DM.
- Sem comment_id e author_id nao ha correlacao confiavel.

## Premissas e cuidados da Meta
- O envio de DM so pode ocorrer em resposta a uma interacao valida
  (comentario, mencao ou mensagem). Fora da janela de 24h nao é
  permitido enviar novas DMs sem nova interacao.
- Resposta publica ao comentario é permitida; convite para DM deve ser
  imediato e associado ao evento recebido.
- Mensagens promocionais precisam respeitar politicas de Conteudo e
  Mensageria da Meta.

## Passos em ordem de execucao

1) Processar comentarios e mencoes no webhook
- Extender InstagramMessageProcessor para aceitar eventos de comments e
  mentions (além de messaging). Extrair: author_id, post_id, comment_id,
  texto e timestamp.
- Normalizar em IncomingMessagePayload com novo tipo de evento
  (ex.: MessageContentType.TEXT + metadata source="comment").
- Garantir correlacao: usar comment_id e author_id para correlation_id e
  logs.

2) Responder comentario e disparar DM
- Adicionar responder/handler que, ao receber payload de comentario,
  publique reply publico (Graph API /comments) e gere mensagem de
  convite para DM.
- Incluir metadado no OutgoingMessage para acionar "instagram_sequence"
  com duas etapas: reply publico + mensagem DM.
- Reusar InstagramMessageClient para a parte DM; criar pequeno client ou
  funcao auxiliar para endpoint de reply em comentarios (graph edge
  /{comment-id}/replies).

3) Workflow de follow-up e qualificacao
- Criar workflow YAML que, ao receber evento de comentario, envia reply
  curto + DM pedindo contexto (produto/interesse) e dispara pipeline de
  qualificacao (faq/catalogo) via RAG/agent.
- Incluir passo de handoff humano condicional (ex.: low confidence ou
  ticket complexo) utilizando canal Instagram ou redirecionando para
  WhatsApp se permitido pelo cliente.

4) Correlacao e métricas cross-channel
- Persistir correlation_id por author_id/comment_id e reusar em DMs e
  eventual migracao para WhatsApp.
- Registrar eventos em logging system com markers (canal, post_id,
  comment_id) para dashboards unificados.
- Opcional: criar agregador de métricas (comentarios atendidos, tempo de
  primeira resposta, conversao para DM e para WhatsApp).

5) Testes automatizados
- Tests unitarios para parsing de comments/mentions no processor.
- Tests de responder: garantir sequencia reply publico + DM.
- Tests de cliente: mock httpx para /messages e /{comment-id}/replies.
- Tests de workflow: caminho feliz e erro (janela expirada, token
  invalido).

## Conceitos-chave resumidos
- Webhook Instagram: events de comments/mentions chegam via object
  instagram/page; precisamos ler changes[].value.comment_id e author.
- Janela de 24h: DMs so dentro da janela aberta pela ultima interacao do
  usuario. Respostas publicas nao reabrem janela; é preciso que o
  usuario interaja (ex.: responda ao DM) para manter a janela ativa.
- Correlation ID: usar author_id/comment_id para rastrear conversa ponta
  a ponta, inclusive se o lead migrar para WhatsApp.

## Riscos e mitigacoes
- Janela expirada: responder publicamente explicando que precisa de
  nova interacao para DM; logar motivo.
- Tokens/escopos insuficientes: validar escopos instagram_manage_comments
  e instagram_manage_messages no provisionamento.
- Volume alto: implementar backoff/logica de fila para replies se a API
  retornar limits; registrar para reprocessar.

## Entregaveis por etapa
- Etapa 1: Parser de comments/mentions + testes.
- Etapa 2: Responder com reply publico + DM sequence + cliente auxiliar.
- Etapa 3: Workflow YAML de follow-up/qualificacao + hook de handoff.
- Etapa 4: Logging/métricas cross-channel documentados.
- Etapa 5: Suite de testes passando (unit + smoke Instagram).

## Checklist detalhado de ativacao (Instagram)

### 0. Insumos e credenciais
- Meta App com scopes: instagram_manage_comments, instagram_manage_messages,
  pages_show_list, pages_read_engagement, pages_manage_metadata, pages_messaging.
- Facebook Page conectada a uma conta Instagram Business/Creator.
- Page Access Token com os scopes acima (ideal: long-lived, ou renovar
  automaticamente).
- Page ID e Instagram Business ID (servem para external_id do canal).
- Verify Token para o webhook (mantido no cofre/tenant, nao hardcode).

### 1. Provisionar tenant e canal
- Criar registro em tenants com meta_app_id, meta_access_token (page token),
  meta_graph_api_version e verify token, se este for o ponto centralizado de
  secrets (ou usar tenant_security_keys conforme padrao do projeto).
- Criar registro em tenant_channels com:
  - channel_type: instagram
  - external_id: page_id ou instagram business id
  - yaml_path: rag-config-instagram-comment.yaml
  - status: active
  - descricao: Follow-up comentario -> reply + DM

### 2. Publicar o YAML
- Garantir que o arquivo esteja acessivel: app/yaml/rag-config-instagram-comment.yaml.
- Conferir access_key e correlation_id coerentes com o tenant criado.
- Ajustar tags, prefixes Redis e vector_store/llm se o tenant usar outros ids.

### 3. Configurar Webhook na Meta
- No App Dashboard, aba Webhooks (ou Messenger/Instagram):
  - Callback URL: endpoint publico do sistema (ex.: https://api.domínio/webhook/meta).
  - Verify Token: o mesmo salvo no tenant/keystore.
- Assinar campos para o objeto instagram:
  - comments
  - messages
  - mentions (se disponivel)
- Assinar campos para o objeto page (se o router usar page events):
  - feed, mention, messages
- Testar a subscription pelo próprio painel (Send Test Webhook) e validar 200.

### Cobertura pelo onboarding HTML
- Automatizado pela página ui-admin-gov-provisionamento-instagram.html
  (envio para /api/instagram/provision/start):
  - Coleta X-API-Key e cifra localmente.
  - Envia business_id, page_id, page_name (opcional), display_name,
    email comercial, page access token, app_id, app_secret,
    callback_url, verify_token, channel_id (opcional) e graph_version.
  - Esperado: backend valida token/escopos, registra webhook com
    callback/verify_token informados e cria/atualiza canal se
    implementado no endpoint.
- Ainda requer ação manual/verificação:
  - Confirmar via debug_token os escopos e expiração do token.
  - Verificar no painel Meta se o webhook ficou ativo (200 no desafio e
    assinaturas marcadas).
  - Garantir roteamento interno (ChannelRegistry, workflow) e YAML do
    tenant corretos.
  - Executar teste E2E (comentário → reply público + DM) e revisar logs.

### 4. Roteamento interno
- Garantir que o ChannelRegistry aponte para o channel_id recém-criado quando a
  source for instagram.
- Garantir que o router de entrada encaminhe eventos de comment/mention para o
  workflow workflow_instagram_comment_followup.
- Confirmar que o processor popula metadata.comment_id e metadata.post_id.

### 5. Checagem de escopos e tokens
- Validar via Graph API debug_token que o Page Token tem todos os scopes e que
  nao esta expirado.
- Se usar long-lived token, registrar renovacao automatica ou alarme de expirar.

### 6. Teste E2E guiado
- Passo 1: publicar post no Instagram da pagina de teste.
- Passo 2: de uma conta externa, comentar no post.
- Passo 3: observar logs e verificar reply publico gerado.
- Passo 4: verificar DM enviada ao mesmo autor (janela de 24h respeitada).
- Passo 5: responder ao DM e garantir que a janela permanece ativa.
- Passo 6: checar registros de log com correlation_id baseado em comment_id.

### 7. Validacoes de erro
- Comentario com janela expirada: esperar log de aviso e ausência de DM.
- Token inválido/escopo faltando: esperar logger.exception e status de falha
  na chamada HTTP; corrigir token e repetir teste.
- Rate limit: verificar backoff/retry do cliente; se excedido, reprocessar fila.

### 8. Checklist de go-live
- Tokens válidos e guardados no keystore/tenant.
- Webhook verificado (200 no desafio) e subscricoes ativas para comments/messages.
- Canal instagram em status active na tenant_channels.
- YAML ativo e apontado no canal.
- Teste E2E executado com reply publico + DM bem-sucedidos.
