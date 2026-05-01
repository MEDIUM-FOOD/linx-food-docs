**Produto:** Plataforma de Agentes de IA

# Manual de Provisionamento e Operação Instagram Direct

## Visão geral

Este manual descreve como provisionar contas Instagram Business, configurar
webhooks e operar o canal Instagram Direct na Plataforma de Agentes de IA. O processo é análogo
ao fluxo do WhatsApp: provisionamento via Meta Graph API, registro no
ClientDirectory, seleção automática do YAML do cliente e ingestão de eventos
via router multicanal.

## Objetivo

Disponibilizar atendimento e automações em Instagram Direct com onboarding
self-service, garantindo segurança (assinatura de webhook), rastreabilidade
(correlation_id) e isolamento multi-tenant.

## Como o usuário recebe essa feature

1. O time de operações habilita a permissão `provision.instagram` na
  `X-API-Key` do tenant.
2. O portal de provisionamento envia os dados para
  `POST /api/instagram/provision/start`.
3. O canal é registrado no `ClientDirectory` com `channel_id` e
  `secret_token`.
4. O cliente valida o webhook e testa uma mensagem de prova.

### Para quem esta comecando (for dummies)
Pense no Instagram como um novo canal de atendimento. O provisionamento
liga a conta da sua empresa � Plataforma de Agentes de IA, registra o webhook e garante que
as mensagens cheguem com seguranca e rastreio.

### Impacto para o usuario
- Atendimento e automacoes no Instagram com rastreabilidade.
- Menos trabalho manual para operar o canal.
- Seguranca e isolamento entre clientes.

### Limites e pegadinhas
- Webhook sem assinatura valida e rejeitado.
- Tokens curtos ou invalidos nao passam no provisionamento.
- `channel_id` errado impede resolver o YAML automaticamente.

---

## Arquitetura

- `InstagramProvisionerAsync`
  (src/channel_layer/services/instagram_onboarding.py)
  - Serviço assíncrono que valida credenciais, assina webhook e habilita
    mensagens via Graph API.
  - Usa `create_logger_with_correlation` e trata falhas de rede/HTTP.
- Router de provisionamento
  - Endpoint `/api/instagram/provision/start` (FastAPI) em
    src/api/routers/instagram_provision_router.py.
  - Protegido por `PermissionKeys.PROVISION_INSTAGRAM`.
- Router multicanal
  - Endpoints em `/channels/instagram/{channel_id}/messages` (webhook) e
    `/channels/instagram/{channel_id}/send` (envio manual).
  - Seleção automática do YAML via ClientDirectory quando o webhook não envia
    caminho explícito.
  - Validação de assinatura `X-Hub-Signature-256` com `secret_token` do canal.
- Parser de webhooks
  - `InstagramMessageProcessor` converte eventos Meta em
    `IncomingMessagePayload`.
  - Extração de identificador do remetente para enriquecer o correlation_id.
- Envio de mensagens
  - `InstagramMessageClient` (Graph API /messages) busca credenciais no YAML/
    tenant_security_keys com aliases (instagram_access_token etc.).
  - `InstagramResponder` monta payloads com texto, quick replies e mídias.
- Persistência de canais
  - `ChannelRegistry` carrega definições do YAML do tenant.
  - ClientDirectory registra `channel_id`, `client_code`, secret_token e
    caminhos YAML para resolução automática.

---

## 🧭 Jornada de uso (passo a passo)

1. **Preparar credenciais**
  - Token de página, `app_id` e `app_secret` válidos.
2. **Provisionar**
  - Chamar `POST /api/instagram/provision/start` com payload mínimo.
3. **Registrar canal**
  - Persistir `channel_id`, `secret_token` e caminho do YAML no
    `ClientDirectory`.
4. **Validar webhook**
  - Confirmar desafio de assinatura (GET) e envio de evento real.
5. **Testar fluxo**
  - Enviar comentário/DM e conferir resposta automatizada.
6. **Operar em produção**
  - Monitorar logs por `correlation_id` e métricas de resposta.

---

## 🧩 Quando usar

- **SAC via DM**: atendimento individual com histórico e rastreio.
- **Follow-up de comentários**: resposta pública + DM com contexto.
- **Campanhas segmentadas**: envio manual via `/channels/instagram/{id}/send`.
- **Canal unificado**: quando o fluxo usa workflows de comentários + DM.

---

##  Exemplos de uso

### Exemplo feliz (provisionamento + teste)

1. `POST /api/instagram/provision/start` com token válido.
2. Registrar `channel_id` no `ClientDirectory`.
3. Enviar comentário de teste e validar reply + DM.

### Exemplo de erro (token inválido)

- Retorna `400` ou `502` no provisionamento.
- Corrigir `access_token` e repetir o `start`.

## Configuração e credenciais

1. Tenant security keys (diretório multi-tenant)
   - Armazene `access_token` de página com `instagram_manage_messages` e
     `pages_messaging`.
   - Opcionalmente configure aliases: `instagram_access_token`,
     `page_access_token`, `page_token`.
2. Channel definition (YAML do cliente)
   - `channels.<channel_id>.security.secret_token`: segredo usado para validar
     `X-Hub-Signature-256`.
   - `channels.<channel_id>.security_keys`: credenciais específicas do canal
     (opcional se já estiverem em `security_keys` global + active_channel).
3. Callback e verify token
   - `callback_url`: endpoint público apontando para
     `/channels/instagram/{channel_id}/messages`.
   - `verify_token`: mesmo valor deve estar acessível ao serviço para responder
     ao desafio de assinatura (via perfil do cliente em ClientDirectory).
4. Versão da API
   - Padrão `v20.0`, configurável via payload de provisionamento e aliases.

---

## Fluxo de provisionamento (backend)

Endpoint: `POST /api/instagram/provision/start`

Payload mínimo:
```json
{
  "instagram_business_account_id": "17841400000000000",
  "page_id": "112233445566778",
  "display_name": "Loja Plataforma de Agentes de IA",
  "email": "contato@prometeu.com.br",
  "access_token": "EAAB...",
  "app_id": "123456789012345",
  "app_secret": "APP_SECRET",
  "callback_url": "https://sua.api/channels/instagram/ig_sac/messages",
  "verify_token": "TOKEN_WEBHOOK",
  "channel_id": "ig_sac",
  "graph_api_version": "v20.0"
}
```

O router:
1) Valida campos (IDs, email, URL, tokens) e permissão do usuário.
2) Usa `InstagramProvisionerAsync` para:
   - `fetch_business_profile(ig_business_account_id)`
   - `subscribe_page_to_messages(page_id)`
   - `configure_webhook(callback_url, verify_token)`
3) Retorna dados do perfil (username) e status.
4) A aplicação deve registrar o canal no ClientDirectory, associando
   `channel_id`, `client_code`, `secret_token` e caminho YAML do tenant.

Resposta típica:
```json
{
  "success": true,
  "instagram_business_account_id": "17841400000000000",
  "ig_username": "lojaprometeu",
  "page_id": "112233445566778",
  "status": "active",
  "message": "Webhook configurado e conta registrada."
}
```

Erros mapeados:
- 400: validação de payload ou token curto.
- 502: erro HTTP da Graph API.
- 503: falha de rede ao acessar a Graph API.

### Validação pós-provisionamento

1) Chame `/api/instagram/provision/start` com token de página válido.
2) Verifique no ClientDirectory se o registro foi criado para o
   `instagram_business_account_id` informado.
3) Confirme no Meta Dashboard que o webhook está ativo para a página.
4) Envie um comentário de teste em um post e valide reply público + DM
   (o log do router multicanal deve conter correlation_id e `ig_sender`).
5) Se usar versão customizada da Graph API, confirme que
   `graph_api_version` apareceu no metadata.

### Campos enviados pelo onboarding HTML

O formulário `ui-admin-gov-provisionamento-instagram.html` publica em
`/api/instagram/provision/start` e grava no metadata do ClientDirectory:

- `instagram_business_account_id`, `page_id`, `page_name`, `display_name` e
  `email`.
- `access_token` (somente hash SHA-256 é persistido), `app_id`, `app_secret`
  (hash combinado) e `verify_token` (hash SHA-256 é persistido).
- `callback_url`, `channel_id`, `graph_api_version` e `registered_by`
  (user_email da chave).
- Nenhum token fica armazenado em claro no metadata; apenas hashes e dados
  públicos (URL, nomes, IDs).

### Ajustes necessários no canal (amanhã)

- Coluna `execution_mode` na tabela `tenant_channels` deve ficar como
  `workflow` para o channel_id do Instagram.
- `yaml_path` do mesmo registro deve apontar para
  `app/yaml/rag-config-instagram-comment.yaml`.
- O YAML agora possui dois workflows:
  - `workflow_instagram_comment_followup`: focado em comentário → reply
    público + DM.
  - `workflow_instagram_unificado_comment_dm`: trata tanto comentário
    quanto DM com o mesmo fluxo; inclui reply público apenas se houver
    `comment_id` e sempre monta a sequência de DM.
- Escolha qual workflow ativar para o canal (mantenha apenas um como
  enabled=true se quiser um comportamento único). Atualmente os dois
  estão enabled; para uma rota única use o unificado.

---

## Recebimento de mensagens (webhook)

Endpoint: `POST /channels/instagram/{channel_id}/messages`

- Aceita corpo bruto Meta (object `instagram` ou `page`).
- O parser gera `batched_payloads` e extrai `sender_id` para compor
  correlation_id.
- Se o webhook não enviar `yaml_config_path`, o sistema resolve o YAML via
  ClientDirectory usando o `channel_id` e `client_code` do diretório.
- Validação de assinatura: header `X-Hub-Signature-256` comparado ao
  `secret_token` do canal (HMAC SHA-256 do corpo).
- Logs incluem `channel_id`, `client_code`, `ig_sender` e correlation_id.

Desafio de assinatura (GET): `GET /channels/{channel_id}/messages` com
`hub.mode=subscribe`, `hub.challenge`, `hub.verify_token`.
- O verify_token esperado é lido do perfil do cliente no ClientDirectory
  (`meta_webhook_verify_token`).

---

## Envio de mensagens

### Via webhook (automático)
- `ChannelMessageProcessor` seleciona `InstagramResponder` e
  `InstagramMessageClient` conforme `channel_type=instagram`.
- Credenciais resolvidas do YAML e `tenant_security_keys` (aliases suportados).
- Payload final inclui texto, quick replies e mídias (`media_id` ou URL
  reutilizável com `is_reusable=true`).

### Envio manual (útil para suporte)
Endpoint: `POST /channels/instagram/{channel_id}/send`

Exemplo:
```json
{
  "yaml_config_path": "app/yaml/seu-cliente.yaml",
  "recipient_id": "USER_123",
  "text": "Olá!"
}
```

- O router monta mensagens se `messages` não for fornecido.
- Verifica se o `channel_id` é Instagram e se o `client_code` está presente no
  ChannelDefinition.

---

## Boas práticas

- Registre o `secret_token` no YAML do canal e mantenha-o sincronizado com o
  `verify_token` armazenado no ClientDirectory.
- Garanta que o `callback_url` público aponte para o caminho correto do canal
  na API.
- Use cabeçalhos de correlação (`X-Correlation-Id`) na borda; o router preserva
  e compõe novos IDs com o `sender_id` do Instagram.
- Mantenha os aliases de credenciais para evitar quebras quando tokens forem
  rotacionados.
- Monitore logs de `webhook.signature_*` para detectar falhas de assinatura.

---

## Troubleshooting rápido

- 401/403 no webhook: verifique `X-Hub-Signature-256` e `secret_token` do canal.
- 400 no provisionamento: IDs ou URLs inválidos; tokens curtos são rejeitados.
- 502/503 no provisionamento: instabilidade da Graph API; tente novamente.
- Mensagens não saem: confira se `instagram_access_token` está presente em
  `security_keys` ou em `channels.<channel_id>.security_keys`.
- Correlation ausente: certifique-se de enviar `X-Correlation-Id` ou usar
  `channel_id` correto para o router compor o ID com o remetente.

---

## Referências cruzadas

- Router de provisionamento: src/api/routers/instagram_provision_router.py
- Router multicanal: src/api/routers/channel_router.py
- Envio de mensagens: src/channel_layer/clients/instagram_message_client.py
- Parser de webhooks: src/channel_layer/instagram_processor.py
- Responder Instagram: src/channel_layer/responders/instagram_responder.py
- Serviço de onboarding: src/channel_layer/services/instagram_onboarding.py
- Registro multi-tenant: src/security/client_directory.py
