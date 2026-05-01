# 📣 Manual de Tools de Social e Mensageria

## Visão geral

Agrupa as ferramentas de redes sociais e mensageria já disponíveis no
catálogo (Instagram Graph, Twitter/X v2, WhatsApp Cloud API e Chatwoot) em
um único manual prático.

## Por que existe

- Evitar caça às ferramentas em listas alfabéticas.
- Deixar claro quais credenciais e blocos YAML cada canal exige.
- Trazer exemplos combinando publicação, captura de buzz e atendimento.

## Explicação conceitual

- Todas as tools usam `user_session.correlation_id` para log e rastreio.
- Autenticação vem de `security_keys`; parâmetros variáveis ficam em
  `global_tools_configuration` no YAML (quando aplicável).
- Saídas são JSON ou listas de dicionários já normalizados para o agente.
- Tratamento de erros prioriza mensagens explícitas sem vazar tokens.

## Explicação for dummies

Peça: "publique no Instagram e mande DM para os 3 primeiros comentários" ou
"busque o buzz no Twitter e envie um resumo no WhatsApp". As tools chamam
as APIs oficiais, cuidam de autenticação e já devolvem dados prontinhos
para você só usar no prompt.

## Catálogo por provedor

### Instagram Graph
- `instagram_publish_media`: posta imagem/vídeo/reel.
- `instagram_send_direct_message`: envia DM com texto/link ou payload
  customizado.
- `instagram_fetch_insights`: lê métricas de conta ou mídia.

**YAML (obrigatório)**

```yaml
security_keys:
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"

global_tools_configuration:
  instagram_graph:
    business_account_id: "1789..." # obrigatório
    api_version: "v21.0"            # opcional (default v21.0)
    timeout: 15                      # opcional (s)
```

### Twitter / X (API v2)
- `twitter_search`: busca recente por termo/hashtag/menção.
- `twitter_user_timeline`: timeline pública de um usuário.
- `twitter_mentions_search`: menções a um usuário.
- `twitter_user_lookup`: dados básicos de perfil por username.

**YAML (obrigatório)**

```yaml
security_keys:
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
```

**Parâmetros principais**
- `max_results` (10-100, default 10) e `since_minutes` opcional em
  `twitter_search`.
- `limit` (10-100) nas demais tools.

### WhatsApp Cloud API
- `whatsapp_send_text_message`: envia texto livre (opcional preview de
  link).
- `whatsapp_send_template_message`: envia template aprovado.

**YAML (obrigatório)**

```yaml
security_keys:
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

global_tools_configuration:
  whatsapp_cloud:
    phone_number_id: "5511999999999" # obrigatório
    api_version: "v20.0"             # opcional (default v20.0)
    timeout: 15                       # opcional (s)
```

### Chatwoot
- `chatwoot_listar_conversas`
- `chatwoot_obter_conversa`
- `chatwoot_enviar_mensagem`
- `chatwoot_atualizar_conversa`
- `chatwoot_listar_contatos`
- `chatwoot_criar_contato`
- `chatwoot_listar_agentes`
- `chatwoot_health_check`

**YAML (obrigatório)**

```yaml
security_keys:
  chatwoot_base_url: "https://chatwoot.suaempresa.com"
  chatwoot_api_token: "${CHATWOOT_API_TOKEN}"
  chatwoot_account_id: "${CHATWOOT_ACCOUNT_ID}"

global_tools_configuration:
  chatwoot:
    timeout: 30 # opcional (s)
```

## Como o usuário recebe essa feature

1. Preencha `security_keys` com os tokens acima.
2. Configure `global_tools_configuration` apenas onde o provedor exige
  (Instagram,
   WhatsApp e Chatwoot). Twitter depende só do bearer.
3. Inclua as tools em `multi_agents` ou `workflows` e gere o catálogo via
   builder padrão.

## Exemplos práticos

### 1) Buzz pré-jogo (Twitter + WhatsApp)

```yaml
security_keys:
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

global_tools_configuration:
  whatsapp_cloud:
    phone_number_id: "5511999999999"

multi_agents:
  - id: "buzz_pre_jogo"
    tools:
      - "twitter_search"
      - "whatsapp_send_text_message"
```

Prompt: "Busque tweets sobre Palmeiras hoje e envie resumo + horário para
+55119999...".

### 2) Lançamento com card (Instagram)

```yaml
security_keys:
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"

global_tools_configuration:
  instagram_graph:
    business_account_id: "1789..."

multi_agents:
  - id: "post_lancamento"
    tools:
      - "instagram_publish_media"
      - "instagram_send_direct_message"
```

Prompt: "Publique este vídeo com legenda X e mande DM com cupom para o
perfil 123".

### 3) Atendimento unificado (Chatwoot)

```yaml
security_keys:
  chatwoot_base_url: "https://chatwoot.suaempresa.com"
  chatwoot_api_token: "${CHATWOOT_API_TOKEN}"
  chatwoot_account_id: "${CHATWOOT_ACCOUNT_ID}"

global_tools_configuration:
  chatwoot:
    timeout: 20

multi_agents:
  - id: "cs_chatwoot"
    tools:
      - "chatwoot_listar_conversas"
      - "chatwoot_obter_conversa"
      - "chatwoot_enviar_mensagem"
```

Prompt: "Liste conversas abertas, abra a 123 e responda com status do
pedido".

## Impacto para o usuário

- Menos fricção para publicar, monitorar buzz e responder clientes num só
  pacote.
- Logs com correlação facilitam auditoria e troubleshooting.
- Exemplos prontos aceleram onboarding e reduzem erros de parâmetro.

## Limites e pegadinhas

- Tokens expiram: valide `INSTAGRAM_GRAPH_ACCESS_TOKEN` e bearer do
  Twitter antes de campanhas.
- WhatsApp Cloud cobra por janela de conversa; use template apenas quando
  necessário.
- Chatwoot exige `requests` no ambiente; sem isso as tools não sobem.
- Rate limit: Twitter e Instagram podem retornar 429; aguarde e repita.

## Troubleshooting

- 401/403 em qualquer provedor: confira token em `security_keys` e ID
  obrigatório (IG user, phone_number_id, account_id do Chatwoot).
- WhatsApp: mensagem não chega? Verifique `phone_number_id`, formato do
  número de destino e se o template está aprovado.
- Instagram: erro ao publicar vídeo? Confira `media_type` (IMAGE/VIDEO/
  REEL) e `media_url` público.
- Twitter: `query` vazia devolve erro imediato; normalize o termo.
- Chatwoot: `LOGGER_FACTORY_ERROR` indica client não inicializado; revise
  `base_url`, `api_token`, `account_id` e dependência `requests`.

## Observação sobre canais não suportados

Slack/Teams/Telegram não estão no catálogo atual. Caso precise, avalie
Chatwoot para inbox multicanal ou abra uma issue solicitando nova
integração.
