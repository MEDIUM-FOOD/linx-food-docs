# 📂 Ferramentas por Finalidade

> **Para Consultores**: Encontre a ferramenta certa para resolver
> seu problema de negócio.
>
> **Total atual**: 239 ferramentas ativas no catálogo central
> (snapshot canônico do builder sincronizado para `integrations.builtin_tool_registry`).
> **Atualizado**: 03/03/2026.

---

## 🎯 Índice por Problema de Negócio

- 💬 Comunicar com clientes/equipe →
  [Comunicação e Mensagens](#-comunicação-e-mensagens)
- 📊 Analisar dados e relatórios →
  [Análise de Dados](#-análise-de-dados)
- 🔍 Buscar informações → [Busca e Pesquisa](#-busca-e-pesquisa)
- 🏟️ Dados esportivos →
  [Dados Esportivos](#-dados-esportivos-sportsradar-sportmonks)
- 🔌 Integrar sistemas externos →
  [Integração e APIs](#-integração-e-apis)
- 📁 Processar arquivos →
  [Arquivos e Documentos](#-arquivos-e-documentos)
- 🗄️ Acessar bancos de dados →
  [Bancos de Dados](#-bancos-de-dados)
- 🇧🇷 Dados e serviços do Brasil →
  [Brasil Específico](#-brasil-específico)
- 🏢 Sistemas corporativos (ERP/CRM) →
  [Sistemas Corporativos](#-sistemas-corporativos)
- 🤖 Inteligência Artificial →
  [IA e Machine Learning](#-ia-e-machine-learning)
- 🔧 Utilitários diversos →
  [Utilidades Gerais](#-utilidades-gerais)

## Leitura relacionada

- Guia geral de tools: [../GUIA-USUARIO-TOOLS.md](../GUIA-USUARIO-TOOLS.md)
- Catálogo alfabético: [./alfabetica.md](./alfabetica.md)
- APIs dinâmicas governadas: [../README-DYNAMIC-API-TOOLS.md](../README-DYNAMIC-API-TOOLS.md)
- SQL dinâmico e procedures: [../README-DYNAMIC-SQL-TOOLS.md](../README-DYNAMIC-SQL-TOOLS.md)
- Integrações governadas por tenant: [../README-INTEGRACOES-GOVERNADAS.md](../README-INTEGRACOES-GOVERNADAS.md)

---

## 💬 Comunicação e Mensagens

**Objetivo**: Enviar notificações, mensagens e se comunicar automaticamente.

### 📱 WhatsApp

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `whatsapp_send_text_message` | Texto rápido | Aviso de pedido |
| `whatsapp_send_template_message` | Template aprovado | Lembrete de agenda |

**💼 Caso de Uso**: Agente de atendimento que notifica clientes via WhatsApp

```yaml
multi_agents:
  - id: "atendimento_whatsapp"
    tools:
      - "whatsapp_send_text_message"
    # Configuração...
```

**Guia rápido (WhatsApp Cloud):**

- Conceito: envio de texto ou template pela Cloud API do WhatsApp.
- Para leigos: texto = mensagem livre; template = modelo aprovado pelo
  WhatsApp para notificações.
- Parâmetros principais:
  - texto: `to_number`, `body`, `preview_url` (bool opcional).
  - template: `to_number`, `template_name`, `language_code` (padrão
    `pt_BR`), `components_json` opcional.
- Credenciais obrigatórias (YAML):
  - `security_keys.WHATSAPP_CLOUD_API_TOKEN`
  - `global_tools_configuration.whatsapp_cloud.phone_number_id`
  - `global_tools_configuration.whatsapp_cloud.api_version` (opcional,
    padrão `v20.0`).
- Saída: JSON da Cloud API; em erro, mensagem textual explicando a causa.
- Exemplo de erro: `to_number` vazio retorna `ValueError` e a tool falha.

### 📸 Instagram

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `instagram_publish_media` | Publica imagem/vídeo/reel | Lançar campanha |
| `instagram_send_direct_message` | DM com texto/link | Atendimento direto |
| `instagram_fetch_insights` | Insights de conta/mídia | Medir alcance |

**💼 Caso de Uso**: Publicar campanha e enviar DM promocional

```yaml
security_keys:
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"

global_tools_configuration:
  instagram_graph:
    business_account_id: "${IG_BUSINESS_ACCOUNT_ID}"
    api_version: "v21.0"

multi_agents:
  - id: "social_instagram"
    tools:
      - "instagram_publish_media"
      - "instagram_send_direct_message"
      - "instagram_fetch_insights"
    # Prompt: "Publique este vídeo e envie DM para o lead X com o link."
```

**Guia rápido (Instagram Graph):**

- Conceito: usa Instagram Graph API para publicar, enviar DM e ler
  insights.
- Para leigos: publicar = postar; DM = mensagem privada; insights =
  métricas de conta ou mídia.
- Parâmetros principais:
  - publish: `media_type` (IMAGE/VIDEO/REEL), `media_url`, `caption`
    opcional, `publish_now` (bool), `thumb_url` opcional.
  - DM: `recipient_id`, `message_text` ou `attachment_url`, opcional
    `payload_json` para estruturas avançadas.
  - insights: `metrics` (lista ou CSV), `target_type` (ACCOUNT/MEDIA),
    `media_id` quando for mídia, `period/since/until` opcionais.
- Credenciais/config (YAML):
  - `security_keys.INSTAGRAM_GRAPH_ACCESS_TOKEN`.
  - `global_tools_configuration.instagram_graph.business_account_id`
    obrigatório.
  - `global_tools_configuration.instagram_graph.api_version` opcional
    (padrão `v21.0`).
- Saída: JSON retornado pela API; em erro, mensagem textual indicando a
  causa (ex.: métrica faltando ou token inválido).

### 💼 Chatwoot (Inbox)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `chatwoot_listar_conversas` | Lista conversas com filtros | Fila aberta |
| `chatwoot_obter_conversa` | Detalhe de conversa | Ver histórico |
| `chatwoot_enviar_mensagem` | Responde na conversa | Follow-up |
| `chatwoot_atualizar_conversa` | Atualiza status/assignee | Encerrar/atribuir |
| `chatwoot_listar_contatos` | Lista contatos | Ver leads |
| `chatwoot_criar_contato` | Cria contato | Registrar lead |
| `chatwoot_listar_agentes` | Lista agentes | Escalonar |
| `chatwoot_health_check` | Health da integração | Diagnóstico rápido |

**💼 Caso de Uso**: Atendente que puxa a fila, abre a conversa e responde.

```yaml
security_keys:
  chatwoot_base_url: "https://chatwoot.suaempresa.com"
  chatwoot_api_token: "${CHATWOOT_API_TOKEN}"
  chatwoot_account_id: "${CHATWOOT_ACCOUNT_ID}"

global_tools_configuration:
  chatwoot:
    timeout: 20

multi_agents:
  - id: "suporte_chatwoot"
    tools:
      - "chatwoot_listar_conversas"
      - "chatwoot_obter_conversa"
      - "chatwoot_enviar_mensagem"
```

**Notas rápidas (Chatwoot):**

- Conceito: inbox multicanal; as tools chamam a API REST da instância.
- Para leigos: é como abrir a fila do help desk e responder direto dali.
- Parâmetros comuns: IDs numéricos de conversa/contato e payloads JSON
  simples para mensagens ou updates.
- Dependência: biblioteca `requests` precisa estar instalada no runtime.

### 🍽️ iFood (Pedidos)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `ifood_buscar_eventos` | Eventos pendentes (polling) | Ler pedidos |
| `ifood_confirmar_eventos` | Confirmar eventos | Liberar fila |

```yaml
security_keys:
  IFOOD_CLIENT_ID: "${IFOOD_CLIENT_ID}"
  IFOOD_CLIENT_SECRET: "${IFOOD_CLIENT_SECRET}"

global_tools_configuration:
  ifood_sdk:
    base_url: "https://merchant-api.ifood.com.br"

multi_agents:
  - id: "operador_ifood"
    tools:
      - "ifood_buscar_eventos"
      - "ifood_confirmar_eventos"
    # Prompt: "Liste eventos PLACED e confirme os IDs retornados."
```

**Explicação (conceitual):** usa o sdk-ifood oficial. `ifood_buscar_eventos`
ISO, UUID em string) e você filtra por `query` obrigatória. Já

**Explicação for dummies:** pense em uma fila de avisos do iFood. A tool
"buscar" lê os avisos; a tool "confirmar" avisa ao iFood que você já leu,
para não receber de novo.

**Parâmetros chave:**

- `ifood_buscar_eventos`: `query` (texto obrigatório, ex.: "placed",
  "cancellation", "payment").
- `ifood_confirmar_eventos`: `eventos_json` (string JSON com lista de IDs,
  ex.: `"[\"evt1\", \"evt2\"]"`).

**Configuração YAML obrigatória:**

- `security_keys.IFOOD_CLIENT_ID` e `security_keys.IFOOD_CLIENT_SECRET`.
- `global_tools_configuration.ifood_sdk.base_url` opcional (padrão oficial) e
  `grant_type` opcional.
**Saídas e mensagens:**
- Sucesso: lista de dicts com `status="sucesso"`, `evento_id` e detalhes
**Exemplos rápidos:**
- Feliz: `query="placed"` → retorna eventos normalizados filtrados por
  texto.
- Erro: `eventos_json="{}"` → `status="erro"`, mensagem: lista obrigatória.
- Rate limit: erro 429 → mensagem sugere aguardar e tentar novamente.

### 📧 Email

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `email_sender` | Envia email automático | Relatório diário de vendas |
| `gmail_send` | Envia via Gmail | Notificação pessoal |
| `outlook_send` | Envia via Outlook | Email corporativo |

**💼 Caso de Uso**: Enviar relatórios automáticos por email

### 💼 Slack / Teams

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `slack_send_message` | Posta no Slack | Alertas de sistema |
| `slack_send_file` | Envia arquivo Slack | Compartilhar relatório |
| `teams_send_message` | Posta no Teams | Notificação corporativa |
**💼 Caso de Uso**: Alertas de monitoramento no Slack

|------------|-----------|----------------|
| `telegram_send` | Envia no Telegram | Notificações pessoais |
| `sms_sender` | Envia SMS | Alertas críticos |

---

## 📊 Análise de Dados

**Objetivo**: Processar, analisar e gerar insights de dados.

### 📈 Análise Avançada

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `pandas_dataframe_agent` | Pandas + IA | Insights rápidos |
| `excel_analyzer` | Analisa planilhas Excel | Encontrar padrões em dados |
| `csv_analyzer` | Analisa arquivos CSV | Relatórios de logs |

**💼 Caso de Uso Real**: Análise automática de vendas

```yaml
multi_agents:
  - id: "analista_vendas"
    tools:
      - "dyn_sql<vendas_mes>"          # Busca dados
      - "pandas_dataframe_agent"       # Analisa
      - "email_sender"                 # Envia relatório
```

### 📉 Visualização

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `plotly_chart_generator` | Gráficos interativos | Dashboard de KPIs |
| `matplotlib_plotter` | Gráficos estáticos | Relatórios PDF |

### 🧮 Cálculos

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `calculator` | Calculadora básica | Cálculos simples |
| `math_calculator` | Matemática avançada | Fórmulas complexas |
| `statistics_calculator` | Estatísticas | Médias, desvios, etc |

---

## 🔍 Busca e Pesquisa

**Objetivo**: Encontrar informações na internet ou em bases próprias.

### 🌐 Busca na Internet

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `duckduckgo_search` | Busca web genérica | Pesquisas gerais |
| `google_serper_wrapper` | Busca Google (API) | Resultados otimizados |
| `tavily_search_wrapper` | Busca otimizada IA | Respostas inteligentes |
| `twitter_search` | Busca tweets recentes (API v2) | Monitorar menções |
| `twitter_user_timeline` | Timeline recente de usuário | Acompanhar perfil |
| `twitter_mentions_search` | Menções a um usuário | Ouvir clientes |
| `twitter_user_lookup` | Perfil público por username | Ver dados básicos |

**💼 Caso de Uso**: Agente que pesquisa concorrência

```yaml
multi_agents:
  - id: "monitoria_social"
    tools:
      - "twitter_search"
    global_tools_configuration:
      twitter_search:
        TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
    # Exemplo de prompt: "Liste menções do produto X nos últimos 30 min"
```

**Notas rápidas (twitter_search):**

- Conceito: busca tweets recentes pela API v2 (Tweepy), com backoff para
  rate limit e normalização de campos (`id`, `text`, `author_id`,
  `created_at`, métricas públicas).
- Para leigos: a tool é um “radar” de menções; você diz o termo e ela devolve
  tweets filtrados e organizados.
- Parâmetros principais: `query` (termo), `max_results` (10-100),
  `since_minutes` (janela opcional para limitar recência).
- Credencial obrigatória: `TWITTER_BEARER_TOKEN` em `global_tools_configuration` com
  placeholder `${TWITTER_BEARER_TOKEN}`. O segredo real deve estar em
  `tenant_security_keys`/`tenant_secrets` ou `.env`.
- Limites: sujeito a rate limit da API v2; ferramenta aplica retentativas
  exponenciais para 429/5xx.
- Erro comum: token ausente gera falha na criação da tool. Solução: cadastre
  o token e reexecute a carga do YAML.

**Outras tools Twitter incluídas:**

- `twitter_user_timeline`: usa `from:<username>` para listar tweets
  recentes de um perfil.
- `twitter_mentions_search`: usa `to:<username>` e `@<username>` para
  encontrar menções.
- `twitter_user_lookup`: retorna dados públicos do perfil (nome,
  bio, métricas).
- Todas usam o mesmo `TWITTER_BEARER_TOKEN` em
  `global_tools_configuration.twitter_search`.

**Guia rápido (timeline/menções/lookup):**

- Conceito: timeline usa `from:<user>`, menções usam `@<user>` ou
  `to:<user>`, lookup consulta o perfil público via API v2.
- Para leigos: timeline = “ver feed do perfil”; menções = “escutar quem
  fala com/da conta”; lookup = “puxar ficha pública do perfil”.
- Parâmetros principais:
  - timeline: `username`, `max_results` (10-100), `since_minutes` opcional.
  - menções: `username`, `max_results` (10-100), `since_minutes` opcional.
  - lookup: apenas `username`.
- Saídas:
  - timeline/menções: `success`, `username`, `count`, `tweets[]`,
    `meta` (`newest_id`, `oldest_id`, `next_token`, `result_count`,
    `start_time`).
  - lookup: `success`, `username`, `user` (id, name, username,
    description, verified, public_metrics, created_at).
- Exemplo feliz (YAML):

```yaml
multi_agents:
  - id: "monitoria_marca"
    tools:
      - "twitter_user_timeline"
      - "twitter_mentions_search"
      - "twitter_user_lookup"
    global_tools_configuration:
      twitter_search:
        TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
    # Prompt: "Liste menções a @minha_conta nos últimos 30 min e traga
    # detalhes do perfil."
```

- Exemplo de erro: se `username` vier vazio, a tool responde
  `success=False` e `error="username obrigatório e deve ser texto."`.

### 📚 Enciclopédias e Conhecimento

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `wikipedia_search` | Busca Wikipedia | Informações gerais |
| `arxiv_search` | Artigos científicos | Pesquisa acadêmica |

### 🎥 Multimídia

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `youtube_search_wrapper` | Busca vídeos YouTube | Tutoriais e reviews |

### 📄 Busca em Documentos Próprios (RAG)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| [**`qa_rag`**](../README-RAG.md) ⭐ | Busca em docs próprios | FAQ automático |
| `qa_rag_with_sources` | RAG com citações | Respostas com fontes |
| `qa_search` | Busca vetorial simples | Pesquisa em base |

**💼 Caso de Uso Real**: Assistente virtual que responde sobre produtos

```yaml
multi_agents:
  - id: "assistente_produtos"
    tools:
      - "qa_rag"              # Busca no catálogo
      - "duckduckgo_search"   # Busca complementar web
```

### 🏟️ Dados Esportivos (Sportsradar/Sportmonks/API-Sports/GoalServe/Opta/Enetpulse)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `sportsradar_resumo_evento` | Resumo de evento por ID | "Resumo do jogo" |
| `sportsradar_partidas_competicao` | Agenda da competição | "Calendário" |
| `sportsradar_times_competicao` | Times inscritos | "Clubes participantes" |
| `sportsradar_perfil_jogador` | Perfil do atleta | "Dados do jogador" |
| `sportsradar_classificacao_competicao` | Tabela da competição | "Tabela" |
| `apisports_partida_por_id` | Partida API-Sports | "Detalhe" |
| `apisports_partidas_por_data` | Jogos API-Sports por data | "Jogos" |
| `apisports_times_por_nome` | Times API-Sports por nome | "Clubes" |
| `apisports_jogadores_por_time` | Jogadores API-Sports | "Elenco" |
| `apisports_classificacao_liga` | Classificação API-Sports | "Tabela" |
| `goalserve_partida_por_id` | Partida GoalServe por ID | "Detalhe" |
| `goalserve_partidas_por_data` | Jogos GoalServe por data | "Jogos" |
| `goalserve_times_por_nome` | Times GoalServe por nome | "Clubes" |
| `goalserve_jogadores_por_time` | Jogadores GoalServe | "Elenco" |
| `goalserve_classificacao_liga` | Classificação GoalServe | "Tabela" |
| `sportmonks_partida_por_id` | Partida por ID | "Detalhe do jogo" |
| `sportmonks_partidas_por_data` | Partidas por data | "Jogos do dia" |
| `sportmonks_time_por_id` | Time por ID | "Dados do clube" |
| `sportmonks_jogador_por_id` | Jogador por ID | "Perfil do atleta" |
| `sportmonks_classificacao_temporada` | Classificação da temporada | "Tabela" |
| `opta_buscar_competicoes` | Ligas e competições | "Campeonatos" |
| `opta_buscar_fixtures` | Partidas/fixtures | "Jogos futuros" |
| `opta_buscar_classificacao` | Standings/classificações | "Tabela" |
| `opta_buscar_equipes` | Equipes/seleções | "Clubes" |
| `opta_buscar_jogadores` | Jogadores | "Elenco" |
| `enetpulse_buscar_eventos` | Eventos/competições | "Calendário" |
| `enetpulse_buscar_fixtures` | Fixtures/partidas | "Jogos" |
| `enetpulse_buscar_odds` | Odds e mercados | "Probabilidades" |
| `enetpulse_buscar_classificacao` | Standings | "Tabela" |
| `enetpulse_buscar_equipes` | Equipes | "Clubes" |

**Config básica:**

- `security_keys`: `SPORTSRADAR_API_KEY`, `SPORTMONKS_API_TOKEN`,
  `APISPORTS_API_KEY`, `GOALSERVE_API_KEY`, `OPTA_API_KEY`,
  `ENETPULSE_API_KEY`
- `global_tools_configuration`: `sportsradar`, `sportmonks`, `apisports`,
  `goalserve`, `opta`, `enetpulse` com endpoints/rotas;
  `base_url` só é configurável em Opta/Enetpulse (self-host)
- Retorno sempre em lista de dicionários; erros 401/403 e 429 têm
  mensagens claras
- Visão agrupada: [esportes.md](esportes.md)
- Detalhes de provedores: [sportsradar.md](sportsradar.md),
  [sportmonks.md](sportmonks.md), [apisports.md](apisports.md),
  [goalserve.md](goalserve.md)

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"
  SPORTMONKS_API_TOKEN: "${SPORTMONKS_API_TOKEN}"

multi_agents:
  - id: "radar_esportivo"
    tools:
      - "sportsradar_resumo_evento"
      - "sportsradar_partidas_competicao"
      - "sportmonks_partidas_por_data"
    local_tools_configuration:
      sportsradar: {}
      sportmonks: {}
```

---

## 🔌 Integração e APIs

**Objetivo**: Conectar com sistemas e APIs externas.

### 🚀 APIs Dinâmicas (Recomendado)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| [**`dyn_api<endpoint_id>`**](api_dinamica.md) ⭐ | HTTP via YAML ou tabela | Qualquer REST publicado |

**💼 Caso de Uso Real**: Integração com múltiplas APIs

Use `dyn_api<endpoint_id>` quando o endpoint estiver declarado no YAML ou publicado como `operation_code` em `integrations.api_operation_registry`. O YAML tem precedencia; a tabela entra quando o endpoint nao esta no YAML.

```yaml
tools:
  - "dyn_api<buscar_clima>"      # OpenWeather
  - "dyn_api<github_issues>"     # GitHub
  - "dyn_api<criar_pedido>"      # ERP próprio

local_tools_configuration:
  api_dynamic:
    endpoints:
      buscar_clima:
        url: "https://api.openweathermap.org/..."
      # ... configurações
```

### 🌍 APIs Específicas

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `http_get` / `http_post` | HTTP básico | Requests simples |
| `openapi_caller` | APIs com spec OpenAPI | Swagger/OpenAPI |
| `graphql_query` | Consultas GraphQL | APIs modernas |
| `ucp_discovery_tool` | Descoberta UCP via manifesto | Diagnóstico de integração |

### 🍽️ Delivery (99Food)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `food99_buscar_pedidos` | Pedidos 99Food por filtro | Novos ou cancelados |

**Explicação (conceitual):** usa a API parceira da 99Food. Você passa um
filtro obrigatório (ex.: `PLACED`, `CANCELLED`, `READY`) e recebe uma
lista de pedidos normalizada em dicionários, pronta para o agente
processar.

**Explicação for dummies:** é como abrir o painel da 99Food e filtrar os
pedidos por status. Você informa o status e a tool devolve a lista já
organizada.

**Parâmetro obrigatório:**

- `query`: texto de filtro (ex.: `PLACED`, `CANCELLED`).

**Configuração YAML obrigatória:**

- `security_keys.FOOD99_CLIENT_ID` e `security_keys.FOOD99_CLIENT_SECRET`.
- Opcional: `security_keys.FOOD99_PARTNER_ID` (se o parceiro exigir).
- Opcional: `global_tools_configuration.food99.*` para customizar `base_url`,
  caminhos e `timeout`.

**Exemplo feliz:**

```yaml
multi_agents:
  - id: "monitor_pedidos_99food"
    tools:
      - "food99_buscar_pedidos"
    global_tools_configuration:
      food99:
        base_url: "https://partner-api.99app.com"
    security_keys:
      FOOD99_CLIENT_ID: "${FOOD99_CLIENT_ID}"
      FOOD99_CLIENT_SECRET: "${FOOD99_CLIENT_SECRET}"
      FOOD99_PARTNER_ID: "${FOOD99_PARTNER_ID}" # opcional

# Prompt: "Liste pedidos PLACED na 99Food".
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória...".

**Mensagens de falha comuns:**

- Autenticação 401/403: orienta revisar client_id/client_secret.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🍽️ Delivery (Goomer)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `goomer_buscar_pedidos` | Pedidos Goomer por filtro | Novos ou cancelados |

**Explicação (conceitual):** consulta a API do Goomer e devolve uma lista
de pedidos já normalizada em dicionários. O filtro é obrigatório e pode
ser um status ou termo aceito pela API do parceiro.

**Explicação for dummies:** é como filtrar a fila de pedidos no painel do
Goomer. Você informa o status (ex.: `PLACED`, `CANCELLED`) e recebe a
lista pronta para o agente analisar ou notificar.

**Parâmetro obrigatório:**

- `query`: texto de filtro (ex.: `PLACED`, `CANCELLED`).

**Configuração YAML obrigatória:**

- `security_keys.GOOMER_CLIENT_ID` e `security_keys.GOOMER_CLIENT_SECRET`.
- `security_keys.GOOMER_ESTABLISHMENT_ID`.
- Opcional: `global_tools_configuration.goomer.*` para customizar `base_url`,
  caminhos e `timeout`/`retry_attempts`/`backoff_base`.

**Exemplo feliz:**

```yaml
multi_agents:
  - id: "monitor_pedidos_goomer"
    tools:
      - "goomer_buscar_pedidos"
    global_tools_configuration:
      goomer:
        base_url: "https://api.goomer.app"
    security_keys:
      GOOMER_CLIENT_ID: "${GOOMER_CLIENT_ID}"
      GOOMER_CLIENT_SECRET: "${GOOMER_CLIENT_SECRET}"
      GOOMER_ESTABLISHMENT_ID: "${GOOMER_ESTABLISHMENT_ID}"

# Prompt: "Liste pedidos PLACED no Goomer".
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🍽️ Delivery (Linx Neemo)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `linx_neemo_buscar_pedidos` | Pedidos Linx Neemo | Novos/cancelados |

**Explicação (conceitual):** integra com a API Linx Neemo via
client_credentials e merchant_id, retornando lista de pedidos
normalizada em dicionários. O filtro é obrigatório e enviado como
status na query string.

**Explicação for dummies:** é como abrir o painel Linx Neemo e aplicar
um filtro de status. Você informa o status (ex.: `PLACED`, `READY` ou
`CANCELLED`) e recebe a lista pronta para análise ou notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `READY`).

**Configuração YAML obrigatória:**

- `security_keys.NEEMO_CLIENT_ID` e `security_keys.NEEMO_CLIENT_SECRET`.
- `security_keys.NEEMO_MERCHANT_ID`.
- `global_tools_configuration.linx_neemo.base_url`.
- `global_tools_configuration.linx_neemo.auth_path` e `orders_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.linx_neemo`.

**Exemplo 1 — monitor básico:**

```yaml
multi_agents:
  - id: "monitor_pedidos_linx_neemo"
    tools:
      - "linx_neemo_buscar_pedidos"
    global_tools_configuration:
      linx_neemo:
        base_url: "https://api.seu-linx-neemo.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
    security_keys:
      NEEMO_CLIENT_ID: "${NEEMO_CLIENT_ID}"
      NEEMO_CLIENT_SECRET: "${NEEMO_CLIENT_SECRET}"
      NEEMO_MERCHANT_ID: "${NEEMO_MERCHANT_ID}"

# Prompt: "Liste pedidos READY no Linx Neemo".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_neemo_whatsapp"
    tools:
      - "linx_neemo_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      linx_neemo:
        base_url: "https://api.seu-linx-neemo.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      NEEMO_CLIENT_ID: "${NEEMO_CLIENT_ID}"
      NEEMO_CLIENT_SECRET: "${NEEMO_CLIENT_SECRET}"
      NEEMO_MERCHANT_ID: "${NEEMO_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos PLACED no Linx Neemo"
# 2) "Envie resumo para +5511999999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_neemo_template"
    tools:
      - "linx_neemo_buscar_pedidos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      linx_neemo:
        base_url: "https://api.seu-linx-neemo.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      NEEMO_CLIENT_ID: "${NEEMO_CLIENT_ID}"
      NEEMO_CLIENT_SECRET: "${NEEMO_CLIENT_SECRET}"
      NEEMO_MERCHANT_ID: "${NEEMO_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos CANCELLED no Linx Neemo"
# 2) "Use template neemo_alert com variaveis {{id}} e {{status}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🍽️ Delivery (Keeta)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `keeta_buscar_pedidos` | Pedidos Keeta por filtro | Novos ou cancelados |

**Explicação (conceitual):** integra com a API Keeta via
client_credentials e merchant_id, retornando uma lista de pedidos em
formato de dicionário. O filtro é obrigatório e enviado como status na
query string.

**Explicação for dummies:** é como abrir o painel do Keeta e aplicar um
filtro de status. Você informa o status (ex.: `PLACED`, `READY` ou
`CANCELLED`) e recebe a lista pronta para análise ou notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `READY`).

**Configuração YAML obrigatória:**

- `security_keys.KEETA_CLIENT_ID` e `security_keys.KEETA_CLIENT_SECRET`.
- `security_keys.KEETA_MERCHANT_ID`.
- `global_tools_configuration.keeta.base_url`.
- `global_tools_configuration.keeta.auth_path` e `orders_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.keeta`.

**Exemplo 1 — monitor Keeta:**

```yaml
multi_agents:
  - id: "monitor_pedidos_keeta"
    tools:
      - "keeta_buscar_pedidos"
    global_tools_configuration:
      keeta:
        base_url: "https://api.seu-keeta.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
    security_keys:
      KEETA_CLIENT_ID: "${KEETA_CLIENT_ID}"
      KEETA_CLIENT_SECRET: "${KEETA_CLIENT_SECRET}"
      KEETA_MERCHANT_ID: "${KEETA_MERCHANT_ID}"

# Prompt: "Liste pedidos PLACED no Keeta".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_keeta_whatsapp"
    tools:
      - "keeta_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      keeta:
        base_url: "https://api.seu-keeta.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      KEETA_CLIENT_ID: "${KEETA_CLIENT_ID}"
      KEETA_CLIENT_SECRET: "${KEETA_CLIENT_SECRET}"
      KEETA_MERCHANT_ID: "${KEETA_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos READY no Keeta"
# 2) "Envie resumo para +5511999999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_keeta_template"
    tools:
      - "keeta_buscar_pedidos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      keeta:
        base_url: "https://api.seu-keeta.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      KEETA_CLIENT_ID: "${KEETA_CLIENT_ID}"
      KEETA_CLIENT_SECRET: "${KEETA_CLIENT_SECRET}"
      KEETA_MERCHANT_ID: "${KEETA_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos CANCELLED no Keeta"
# 2) "Use template keeta_alert com variaveis {{id}} e {{status}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🍽️ Delivery (Linx Delivery Hub)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `linx_delivery_hub_buscar_pedidos` | Pedidos Delivery Hub | Novos/cancelados |

**Explicação (conceitual):** integra com a API Linx Delivery Hub via
client_credentials e merchant_id, retornando lista de pedidos já
normalizada em dicionários. O filtro é obrigatório e enviado como
status na query string.

**Explicação for dummies:** é como abrir o painel do Delivery Hub e
aplicar um filtro de status. Você informa o status (ex.: `PLACED`,
`READY` ou `CANCELLED`) e recebe a lista pronta para análise ou
notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `READY`).

**Configuração YAML obrigatória:**

- `security_keys.DELIVERY_HUB_CLIENT_ID` e
  `security_keys.DELIVERY_HUB_CLIENT_SECRET`.
- `security_keys.DELIVERY_HUB_MERCHANT_ID`.
- `global_tools_configuration.linx_delivery_hub.base_url`.
- `global_tools_configuration.linx_delivery_hub.auth_path` e `orders_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.linx_delivery_hub`.

**Exemplo 1 — monitor Delivery Hub:**

```yaml
multi_agents:
  - id: "monitor_pedidos_delivery_hub"
    tools:
      - "linx_delivery_hub_buscar_pedidos"
    global_tools_configuration:
      linx_delivery_hub:
        base_url: "https://api.delivery-hub-linx.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
    security_keys:
      DELIVERY_HUB_CLIENT_ID: "${DELIVERY_HUB_CLIENT_ID}"
      DELIVERY_HUB_CLIENT_SECRET: "${DELIVERY_HUB_CLIENT_SECRET}"
      DELIVERY_HUB_MERCHANT_ID: "${DELIVERY_HUB_MERCHANT_ID}"

# Prompt: "Liste pedidos PLACED no Linx Delivery Hub".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_delivery_hub_whatsapp"
    tools:
      - "linx_delivery_hub_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      linx_delivery_hub:
        base_url: "https://api.delivery-hub-linx.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      DELIVERY_HUB_CLIENT_ID: "${DELIVERY_HUB_CLIENT_ID}"
      DELIVERY_HUB_CLIENT_SECRET: "${DELIVERY_HUB_CLIENT_SECRET}"
      DELIVERY_HUB_MERCHANT_ID: "${DELIVERY_HUB_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos READY no Delivery Hub"
# 2) "Envie resumo para +5511999999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_delivery_hub_template"
    tools:
      - "linx_delivery_hub_buscar_pedidos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      linx_delivery_hub:
        base_url: "https://api.delivery-hub-linx.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      DELIVERY_HUB_CLIENT_ID: "${DELIVERY_HUB_CLIENT_ID}"
      DELIVERY_HUB_CLIENT_SECRET: "${DELIVERY_HUB_CLIENT_SECRET}"
      DELIVERY_HUB_MERCHANT_ID: "${DELIVERY_HUB_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos CANCELLED no Delivery Hub"
# 2) "Use template delivery_hub_alert com variaveis {{id}} e {{status}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🍽️ Delivery (Delivery Much)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `delivery_much_buscar_pedidos` | Pedidos Delivery Much | Novos/cancelados |

**Explicação (conceitual):** integra com a API Delivery Much via
client_credentials e merchant_id, retornando lista de pedidos já
normalizada em dicionários. O filtro é obrigatório e enviado como
status na query string.

**Explicação for dummies:** é como abrir o painel do Delivery Much e
aplicar um filtro de status. Você informa o status (ex.: `PLACED`,
`READY` ou `CANCELLED`) e recebe a lista pronta para análise ou
notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `READY`).

**Configuração YAML obrigatória:**

- `security_keys.DELIVERY_MUCH_CLIENT_ID` e
  `security_keys.DELIVERY_MUCH_CLIENT_SECRET`.
- `security_keys.DELIVERY_MUCH_MERCHANT_ID`.
- `global_tools_configuration.delivery_much.base_url`.
- `global_tools_configuration.delivery_much.auth_path` e `orders_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.delivery_much`.

**Exemplo 1 — monitor Delivery Much:**

```yaml
multi_agents:
  - id: "monitor_pedidos_delivery_much"
    tools:
      - "delivery_much_buscar_pedidos"
    global_tools_configuration:
      delivery_much:
        base_url: "https://api.delivery-much.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
    security_keys:
      DELIVERY_MUCH_CLIENT_ID: "${DELIVERY_MUCH_CLIENT_ID}"
      DELIVERY_MUCH_CLIENT_SECRET: "${DELIVERY_MUCH_CLIENT_SECRET}"
      DELIVERY_MUCH_MERCHANT_ID: "${DELIVERY_MUCH_MERCHANT_ID}"

# Prompt: "Liste pedidos PLACED no Delivery Much".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_delivery_much_whatsapp"
    tools:
      - "delivery_much_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      delivery_much:
        base_url: "https://api.delivery-much.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      DELIVERY_MUCH_CLIENT_ID: "${DELIVERY_MUCH_CLIENT_ID}"
      DELIVERY_MUCH_CLIENT_SECRET: "${DELIVERY_MUCH_CLIENT_SECRET}"
      DELIVERY_MUCH_MERCHANT_ID: "${DELIVERY_MUCH_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos READY no Delivery Much"
# 2) "Envie resumo para +5511999999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_delivery_much_template"
    tools:
      - "delivery_much_buscar_pedidos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      delivery_much:
        base_url: "https://api.delivery-much.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      DELIVERY_MUCH_CLIENT_ID: "${DELIVERY_MUCH_CLIENT_ID}"
      DELIVERY_MUCH_CLIENT_SECRET: "${DELIVERY_MUCH_CLIENT_SECRET}"
      DELIVERY_MUCH_MERCHANT_ID: "${DELIVERY_MUCH_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos CANCELLED no Delivery Much"
# 2) "Use template delivery_much_alert com variaveis {{id}} e {{status}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🍽️ Delivery (Zé Delivery)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `zedelivery_buscar_pedidos` | Pedidos Zé Delivery | Novos/cancelados |

**Explicação (conceitual):** integra com a API Zé Delivery usando
client_credentials + merchant_id. Sempre exige `query` para filtrar e
retorna lista normalizada de dicionários.

**Explicação for dummies:** é como aplicar um filtro de status no painel
do Zé Delivery. Informe um status (ex.: `PLACED`, `READY`, `CANCELLED`)
e receba a lista já pronta para análise ou notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `READY`).

**Configuração YAML obrigatória:**

- `security_keys.ZEDELIVERY_CLIENT_ID` e
  `security_keys.ZEDELIVERY_CLIENT_SECRET`.
- `security_keys.ZEDELIVERY_MERCHANT_ID`.
- `global_tools_configuration.zedelivery.base_url`.
- `global_tools_configuration.zedelivery.auth_path` e `orders_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.zedelivery`.

**Exemplo 1 — monitor Zé Delivery:**

```yaml
multi_agents:
  - id: "monitor_pedidos_zedelivery"
    tools:
      - "zedelivery_buscar_pedidos"
    global_tools_configuration:
      zedelivery:
        base_url: "https://api.zedelivery.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
    security_keys:
      ZEDELIVERY_CLIENT_ID: "${ZEDELIVERY_CLIENT_ID}"
      ZEDELIVERY_CLIENT_SECRET: "${ZEDELIVERY_CLIENT_SECRET}"
      ZEDELIVERY_MERCHANT_ID: "${ZEDELIVERY_MERCHANT_ID}"

# Prompt: "Liste pedidos PLACED no Zé Delivery".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_zedelivery_whatsapp"
    tools:
      - "zedelivery_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      zedelivery:
        base_url: "https://api.zedelivery.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      ZEDELIVERY_CLIENT_ID: "${ZEDELIVERY_CLIENT_ID}"
      ZEDELIVERY_CLIENT_SECRET: "${ZEDELIVERY_CLIENT_SECRET}"
      ZEDELIVERY_MERCHANT_ID: "${ZEDELIVERY_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos READY no Zé Delivery"
# 2) "Envie resumo para +5511999999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_zedelivery_template"
    tools:
      - "zedelivery_buscar_pedidos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      zedelivery:
        base_url: "https://api.zedelivery.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      ZEDELIVERY_CLIENT_ID: "${ZEDELIVERY_CLIENT_ID}"
      ZEDELIVERY_CLIENT_SECRET: "${ZEDELIVERY_CLIENT_SECRET}"
      ZEDELIVERY_MERCHANT_ID: "${ZEDELIVERY_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos CANCELLED no Zé Delivery"
# 2) "Use template zedelivery_alert com variaveis {{id}} e {{status}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

**Workflow com WhatsApp (exemplo prático):**

```yaml
multi_agents:
  - id: "monitor_pedidos_goomer_whatsapp"
    tools:
      - "goomer_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      goomer:
        base_url: "https://api.goomer.app"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      GOOMER_CLIENT_ID: "${GOOMER_CLIENT_ID}"
      GOOMER_CLIENT_SECRET: "${GOOMER_CLIENT_SECRET}"
      GOOMER_ESTABLISHMENT_ID: "${GOOMER_ESTABLISHMENT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos PLACED no Goomer"
# 2) "Mande um resumo para +5511999999999 pelo WhatsApp"
```

**Workflow com WhatsApp Template (transacional):**

```yaml
multi_agents:
  - id: "alerta_pedidos_goomer_template"
    tools:
      - "goomer_buscar_pedidos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      goomer:
        base_url: "https://api.goomer.app"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      GOOMER_CLIENT_ID: "${GOOMER_CLIENT_ID}"
      GOOMER_CLIENT_SECRET: "${GOOMER_CLIENT_SECRET}"
      GOOMER_ESTABLISHMENT_ID: "${GOOMER_ESTABLISHMENT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos READY no Goomer"
# 2) "Use o template goomer_alert com variáveis: {{id}},
#    {{status}}, {{hora}} e envie para +5511999999999"
#
# Exemplo de components_json (ajuste para o seu template):
# {
#   "body": [{"type": "text", "text": "{{id}}"},
#             {"type": "text", "text": "{{status}}"},
#             {"type": "text", "text": "{{hora}}"}]
# }
```

### 🍽️ Delivery (Cardapio Web)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `cardapio_web_buscar_pedidos` | Pedidos Cardapio Web | Novos/cancelados |

**Explicação (conceitual):** reaproveita a API do Goomer para listar
pedidos do Cardapio Web. Recebe um filtro obrigatório (status ou termo) e
retorna lista de dicionários já normalizados.

**Explicação for dummies:** é como abrir o painel do Cardapio Web e
aplicar um filtro de status. Você informa o status (ex.: `PLACED`,
`CANCELLED`) e recebe a lista pronta para análise ou notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `CANCELLED`).

**Configuração YAML obrigatória:**

- `security_keys.GOOMER_CLIENT_ID`, `security_keys.GOOMER_CLIENT_SECRET`.
- `security_keys.GOOMER_ESTABLISHMENT_ID`.
- Opcional: `global_tools_configuration.goomer.*` para customizar `base_url`,
  caminhos e `timeout`/`retry_attempts`/`backoff_base`.

**Exemplo feliz:**

```yaml
multi_agents:
  - id: "monitor_pedidos_cardapio_web"
    tools:
      - "cardapio_web_buscar_pedidos"
    global_tools_configuration:
      goomer:
        base_url: "https://api.goomer.app"
    security_keys:
      GOOMER_CLIENT_ID: "${GOOMER_CLIENT_ID}"
      GOOMER_CLIENT_SECRET: "${GOOMER_CLIENT_SECRET}"
      GOOMER_ESTABLISHMENT_ID: "${GOOMER_ESTABLISHMENT_ID}"

# Prompt: "Liste pedidos READY no Cardapio Web".
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

**Workflow com WhatsApp (texto):**

```yaml
multi_agents:
  - id: "monitor_cardapio_web_whatsapp"
    tools:
      - "cardapio_web_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      goomer:
        base_url: "https://api.goomer.app"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      GOOMER_CLIENT_ID: "${GOOMER_CLIENT_ID}"
      GOOMER_CLIENT_SECRET: "${GOOMER_CLIENT_SECRET}"
      GOOMER_ESTABLISHMENT_ID: "${GOOMER_ESTABLISHMENT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos PLACED no Cardapio Web"
# 2) "Envie resumo para +5511999999999"
```

**Workflow com WhatsApp Template:**

```yaml
multi_agents:
  - id: "alerta_cardapio_web_template"
    tools:
      - "cardapio_web_buscar_pedidos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      goomer:
        base_url: "https://api.goomer.app"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      GOOMER_CLIENT_ID: "${GOOMER_CLIENT_ID}"
      GOOMER_CLIENT_SECRET: "${GOOMER_CLIENT_SECRET}"
      GOOMER_ESTABLISHMENT_ID: "${GOOMER_ESTABLISHMENT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos READY no Cardapio Web"
# 2) "Use o template cardapio_web_alert com variaveis
#    {{id}}, {{status}}, {{hora}} e envie para +5511999999999"
#
# Exemplo de components_json (ajuste para o seu template):
# {
#   "body": [{"type": "text", "text": "{{id}}"},
#             {"type": "text", "text": "{{status}}"},
#             {"type": "text", "text": "{{hora}}"}]
# }
```

### 🍽️ Delivery (CRM Food)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `crm_food_buscar_pedidos` | Pedidos CRM Food | Novos/cancelados |

**Explicação (conceitual):** integra com a API CRM Food usando client
credentials e retorna lista de pedidos normalizada em dicionários. O
filtro é obrigatório (status ou termo aceito pela API).

**Explicação for dummies:** é como abrir o painel do CRM Food e aplicar
um filtro de status. Você informa o status (ex.: `PLACED`, `CANCELLED`)
e recebe a lista pronta para análise ou notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `CANCELLED`).

**Configuração YAML obrigatória:**

- `security_keys.CRMFOOD_CLIENT_ID`, `security_keys.CRMFOOD_CLIENT_SECRET`.
- `security_keys.CRMFOOD_MERCHANT_ID`.
- `global_tools_configuration.crm_food.base_url`.
- `global_tools_configuration.crm_food.auth_path` e `orders_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.crm_food`.

**Exemplo feliz:**

```yaml
multi_agents:
  - id: "monitor_pedidos_crm_food"
    tools:
      - "crm_food_buscar_pedidos"
    global_tools_configuration:
      crm_food:
        base_url: "https://api.seu-crm-food.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
    security_keys:
      CRMFOOD_CLIENT_ID: "${CRMFOOD_CLIENT_ID}"
      CRMFOOD_CLIENT_SECRET: "${CRMFOOD_CLIENT_SECRET}"
      CRMFOOD_MERCHANT_ID: "${CRMFOOD_MERCHANT_ID}"

# Prompt: "Liste pedidos PLACED no CRM Food".
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🍽️ Delivery (Alloy)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `alloy_buscar_pedidos` | Pedidos Alloy por filtro | Novos ou cancelados |

**Explicação (conceitual):** integra com a API Alloy via client
credentials e merchant_id, retornando uma lista de pedidos em
formato de dicionário. O filtro é obrigatório e enviado como status na
query string.

**Explicação for dummies:** é como abrir o painel do Alloy e aplicar um
filtro de status. Você informa o status (ex.: `PLACED`, `CANCELLED`) e
recebe a lista pronta para análise ou notificação.

**Parâmetro obrigatório:**

- `query`: status ou termo de filtro (ex.: `PLACED`, `CANCELLED`).

**Configuração YAML obrigatória:**

- `security_keys.ALLOY_CLIENT_ID` e `security_keys.ALLOY_CLIENT_SECRET`.
- `security_keys.ALLOY_MERCHANT_ID`.
- `global_tools_configuration.alloy.base_url`.
- `global_tools_configuration.alloy.auth_path` e `orders_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.alloy`.

**Exemplo feliz:**

```yaml
multi_agents:
  - id: "monitor_pedidos_alloy"
    tools:
      - "alloy_buscar_pedidos"
    global_tools_configuration:
      alloy:
        base_url: "https://api.seu-alloy.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
        timeout: 20
        retry_attempts: 3
        backoff_base: 1.0
    security_keys:
      ALLOY_CLIENT_ID: "${ALLOY_CLIENT_ID}"
      ALLOY_CLIENT_SECRET: "${ALLOY_CLIENT_SECRET}"
      ALLOY_MERCHANT_ID: "${ALLOY_MERCHANT_ID}"

# Prompt: "Liste pedidos PLACED no Alloy".
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar credenciais.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

**Workflow com WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_alloy_whatsapp"
    tools:
      - "alloy_buscar_pedidos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      alloy:
        base_url: "https://api.seu-alloy.com"
        auth_path: "/oauth/token"
        orders_path: "/api/v1/orders"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      ALLOY_CLIENT_ID: "${ALLOY_CLIENT_ID}"
      ALLOY_CLIENT_SECRET: "${ALLOY_CLIENT_SECRET}"
      ALLOY_MERCHANT_ID: "${ALLOY_MERCHANT_ID}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Liste pedidos READY no Alloy"
# 2) "Envie resumo para +5511999999999"
```

### 💊 Saúde (Consulta Remédios)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `consulta_remedios_buscar_medicamentos` | Busca medicamentos | Preços |

**Explicação (conceitual):** integra com a API Consulta Remédios via
API Key, exigindo `query` obrigatória. Retorna lista normalizada com
nome, apresentações e preços conforme o provedor disponibiliza.

**Explicação for dummies:** é como usar a busca do site Consulta
Remédios. Você informa o nome do remédio (ex.: `dipirona`) e recebe as
opções prontas para analisar ou avisar um cliente.

**Parâmetro obrigatório:**

- `query`: nome do medicamento ou princípio ativo.

**Configuração YAML obrigatória:**

- `security_keys.CONSULTA_REMEDIOS_API_KEY`.
- `global_tools_configuration.consulta_remedios.base_url`.
- `global_tools_configuration.consulta_remedios.search_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.consulta_remedios`.

**Exemplo 1 — monitor de medicamentos:**

```yaml
multi_agents:
  - id: "monitor_consulta_remedios"
    tools:
      - "consulta_remedios_buscar_medicamentos"
    global_tools_configuration:
      consulta_remedios:
        base_url: "https://api.consultaremedios.com"
        search_path: "/v1/medicamentos"
    security_keys:
      CONSULTA_REMEDIOS_API_KEY: "${CONSULTA_REMEDIOS_API_KEY}"

# Prompt: "Busque dipirona gotas 20ml e retorne preços e apresentações".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_remedios_whatsapp"
    tools:
      - "consulta_remedios_buscar_medicamentos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      consulta_remedios:
        base_url: "https://api.consultaremedios.com"
        search_path: "/v1/medicamentos"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      CONSULTA_REMEDIOS_API_KEY: "${CONSULTA_REMEDIOS_API_KEY}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Busque amoxicilina 500mg"
# 2) "Envie resumo para +5511999999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_remedios_template"
    tools:
      - "consulta_remedios_buscar_medicamentos"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      consulta_remedios:
        base_url: "https://api.consultaremedios.com"
        search_path: "/v1/medicamentos"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      CONSULTA_REMEDIOS_API_KEY: "${CONSULTA_REMEDIOS_API_KEY}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Busque losartana 50mg"
# 2) "Use template remedio_alert com variaveis {{nome}} {{preco}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar o API Key.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### 🚗 Mobilidade (Uber)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `uber_buscar_produtos` | Categorias por coordenada | Opções próximas |

**Explicação (conceitual):** usa a SDK `uber-rides` para consultar a API
da Uber. Você informa uma coordenada (`lat,lng`) e recebe as categorias
disponíveis (ex.: UberX, Comfort) com capacidade e preços base
normalizados em dicionários simples.

**Explicação for dummies:** pense como abrir o app da Uber e ver quais
carros existem perto de um ponto no mapa. Você passa o endereço em
coordenadas e a tool devolve a lista pronta para mostrar para o cliente.

**Parâmetro obrigatório:**

- `query`: texto no formato `lat,lng` (ex.: `-23.5617,-46.6560`).

**Configuração YAML obrigatória:**

- `security_keys.UBER_SERVER_TOKEN`: token de servidor da Uber.
- Opcional: `global_tools_configuration.uber_rides.sandbox_mode: true` para
  ambiente de testes.

**Exemplo feliz:**

```yaml
multi_agents:
  - id: "mobilidade_uber"
    tools:
      - "uber_buscar_produtos"
    global_tools_configuration:
      uber_rides:
        sandbox_mode: true
    security_keys:
      UBER_SERVER_TOKEN: "${UBER_SERVER_TOKEN}"

# Prompt: "Liste opções de corrida para -23.5617,-46.6560".
```

**Exemplo de erro (entrada inválida):**

- `query="avenida paulista"` → retorna `status="erro"` com mensagem
  "Formato inválido. Use 'lat,lng'."

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem orientando revisar o `server_token`.
- Rate limit 429: orienta aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor o token.

### 🔐 Autenticação

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `api_key_manager` | Gerenciar API keys | Rotação de chaves |
| `oauth_handler` | OAuth 2.0 | Login social |

---

## 📁 Arquivos e Documentos

**Objetivo**: Ler, processar e gerar arquivos.

### 📊 Planilhas

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `excel_reader` | Ler Excel (.xlsx) | Importar dados |
| `excel_writer` | Criar Excel | Gerar relatórios |
| `csv_parser` | Processar CSV | Logs e exports |

**💼 Caso de Uso**: Processar relatório de vendas Excel

### 📄 Documentos

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `pdf_reader` | Ler PDF | Extrair texto |
| `pdf_generator` | Criar PDF | Gerar documentos |
| `word_reader` | Ler Word (.docx) | Processar contratos |

### 🗂️ Dados Estruturados

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `json_parse` | Processar JSON | APIs e configs |
| `json_validate` | Validar JSON | Verificar formato |
| `xml_parser` | Processar XML | Notas fiscais |
| `yaml_parser` | Processar YAML | Configurações |

### 📦 Arquivos

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `file_browser` | Navegar em pastas | Listar arquivos |
| `file_reader` | Ler arquivo texto | Logs, configs |
| `zip_file_manager` | Gerenciar ZIPs | Compactar/extrair |

---

## 🗄️ Bancos de Dados

**Objetivo**: Consultar e manipular dados em bancos.

### 💎 SQL Dinâmico (Recomendado)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| [dyn_sql<query_id>](sql_dinamico.md) | SQL via YAML ou tabela | Queries prontas e publicadas |

**💼 Caso de Uso Real**: Relatórios dinâmicos

Use `dyn_sql<query_id>` quando a query estiver declarada no YAML ou publicada como `query_code` em `integrations.sql_query_registry`. O YAML tem precedencia; a tabela entra quando a query nao esta no YAML.

```yaml
tools:
  - "dyn_sql<vendas_por_loja>"
  - "dyn_sql<produtos_top>"

local_tools_configuration:
  sql_dynamic:
    connections:
      erp: { secret_key: "DB_URL" }
    queries:
      vendas_por_loja:
        connection: "erp"
        sql: "SELECT * FROM vendas WHERE loja_id = @p1"
```

### 🗃️ Bancos Específicos

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `sql_executor` | SQL direto | Queries ad-hoc |
| `mongodb_query` | Consultas MongoDB | Dados NoSQL |
| `redis_cache_manager` | Cache Redis | Performance |
| `mcp_database_wrapper` | Wrapper MCP | Protocolo MCP |

---

## 🇧🇷 Brasil Específico

**Objetivo**: Serviços e dados brasileiros.

### 🏛️ BrasilAPI

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `brasilapi_consultar_cep` | Buscar CEP | Validar endereço |
| `brasilapi_consultar_cnpj` | Consultar CNPJ | Verificar empresa |
| `brasilapi_listar_bancos` | Listar bancos | Dados bancários |
| `brasilapi_feriados` | Feriados nacionais | Calendário |

### ⛽ Combustíveis (ANP Revendedores)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `anp_revendedores_buscar` | Revendedores ANP por nome/UF | Achar postos |

**Explicação (conceitual):** integra com a API de Revendedores da ANP
via API Key, exigindo `query` obrigatória (nome, município ou UF). Retorna
lista normalizada de dicionários com os dados do revendedor.

**Explicação for dummies:** é como pesquisar postos de combustível na base
da ANP. Você informa um termo (ex.: `Curitiba`, `Posto Ale`) e recebe a
lista pronta para consulta ou alerta.

**Parâmetro obrigatório:**

- `query`: nome, município ou UF do revendedor.

**Configuração YAML obrigatória:**

- `security_keys.ANP_API_KEY`.
- `global_tools_configuration.anp_revendedores.base_url`.
- `global_tools_configuration.anp_revendedores.search_path`.
- Opcional: `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.anp_revendedores`.

**Exemplo 1 — monitor ANP por UF:**

```yaml
multi_agents:
  - id: "monitor_anp_revendedores"
    tools:
      - "anp_revendedores_buscar"
    global_tools_configuration:
      anp_revendedores:
        base_url: "https://api.anp.gov.br"
        search_path: "/v1/revendedores"
    security_keys:
      ANP_API_KEY: "${ANP_API_KEY}"

# Prompt: "Liste revendedores no PR".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_anp_whatsapp"
    tools:
      - "anp_revendedores_buscar"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      anp_revendedores:
        base_url: "https://api.anp.gov.br"
        search_path: "/v1/revendedores"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      ANP_API_KEY: "${ANP_API_KEY}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Busque postos em Curitiba"
# 2) "Envie resumo para +554199999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_anp_template"
    tools:
      - "anp_revendedores_buscar"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      anp_revendedores:
        base_url: "https://api.anp.gov.br"
        search_path: "/v1/revendedores"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      ANP_API_KEY: "${ANP_API_KEY}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt sugerido:
# 1) "Busque revendedores em Belo Horizonte"
# 2) "Use template alerta_posto com variaveis {{nome}} {{endereco}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar o API Key.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

### ⛽ Meus Postos API

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `meus_postos_buscar` | Postos Meus Postos por termo | Monitorar rede |

**Explicação (conceitual):** integra com a Meus Postos API via API Key.
`query` é obrigatória e aceita nome, município ou UF. Retorna lista
normalizada de dicionários.

**Explicação for dummies:** pesquise postos cadastrados na Meus Postos.
Informe um termo (ex.: `Curitiba`, `Posto Central`) e receba a lista
pronta para alertas ou auditoria.

**Parâmetro obrigatório:**

- `query`: nome, município ou UF do posto.

**Configuração YAML obrigatória:**

- `security_keys.MEUS_POSTOS_API_KEY`.
- `global_tools_configuration.meus_postos.base_url`.
- `global_tools_configuration.meus_postos.search_path`.
- Opcional: `query_param`, `timeout`, `retry_attempts`, `backoff_base` em
  `global_tools_configuration.meus_postos`.

**Exemplo 1 — consulta por cidade:**

```yaml
multi_agents:
  - id: "consulta_meus_postos_cidade"
    tools:
      - "meus_postos_buscar"
    global_tools_configuration:
      meus_postos:
        base_url: "https://api.meuspostos.com"
        search_path: "/v1/postos"
    security_keys:
      MEUS_POSTOS_API_KEY: "${MEUS_POSTOS_API_KEY}"

# Prompt: "Liste postos em Recife".
```

**Exemplo 2 — alerta via WhatsApp (texto):**

```yaml
multi_agents:
  - id: "alerta_meus_postos_whatsapp"
    tools:
      - "meus_postos_buscar"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      meus_postos:
        base_url: "https://api.meuspostos.com"
        search_path: "/v1/postos"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      MEUS_POSTOS_API_KEY: "${MEUS_POSTOS_API_KEY}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompts sugeridos:
# 1) "Busque postos em Belo Horizonte"
# 2) "Envie resumo para +5531999999999"
```

**Exemplo 3 — alerta transacional (template WhatsApp):**

```yaml
multi_agents:
  - id: "alerta_meus_postos_template"
    tools:
      - "meus_postos_buscar"
      - "whatsapp_send_template_message"
    global_tools_configuration:
      meus_postos:
        base_url: "https://api.meuspostos.com"
        search_path: "/v1/postos"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      MEUS_POSTOS_API_KEY: "${MEUS_POSTOS_API_KEY}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompts sugeridos:
# 1) "Busque postos no RJ"
# 2) "Use template alerta_posto com variaveis {{nome}} {{endereco}}"
```

**Exemplo de erro (entrada inválida):**

- `query=""` → retorna `status="erro"` e mensagem "query é obrigatória".

**Mensagens de falha comuns:**

- Autenticação 401/403: mensagem clara para revisar o API Key.
- Rate limit 429: sugere aguardar e tentar novamente.
- Falha de rede: mensagem amigável sem expor tokens.

**💼 Caso de Uso**: Validação de cadastro de cliente

```yaml
multi_agents:
  - id: "validador_cadastro"
    tools:
      - "brasilapi_consultar_cep"
      - "brasilapi_consultar_cnpj"
```

### 💳 Pagamentos Brasil

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `pix_generator` | Gerar QR Code Pix | Cobranças |
| `boleto_generator` | Gerar boleto | Pagamentos |

---

## 🏢 Sistemas Corporativos

**Objetivo**: Integração com ERPs e CRMs.

### 🛒 Linx (Exemplo)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `linx_product_search` | Buscar produtos | Consulta catálogo |
| `linx_sales_report` | Relatório vendas | KPIs diários |
| `linx_inventory_check` | Checar estoque | Disponibilidade |

### 🛍️ Plugg.to (Marketplaces)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `pluggto_buscar_produtos` | Produtos Plugg.to | Catálogo multicanal |

**Explicação (conceitual):** integra com a API Plugg.to via API Key.
`query` é obrigatória (nome, SKU ou categoria) e retorna lista de
produtos normalizados.

**Explicação for dummies:** digite um termo como `tenis nike` ou o SKU
e receba os produtos encontrados na Plugg.to sem expor o bearer token.

**Parâmetro obrigatório:**

- `query`: termo de busca (nome, SKU ou categoria).

**Configuração YAML obrigatória:**

- `security_keys.PLUGGTO_API_KEY`.
- `global_tools_configuration.pluggto.base_url`.
- `global_tools_configuration.pluggto.search_path`.
- Opcionais: `query_param`, `timeout`, `retry_attempts`, `backoff_base`
  em `global_tools_configuration.pluggto`.

**Mensagens de falha comuns:**

- 401/403: autenticação falhou, revise o API Key.
- 429: limite da API, aguarde e tente novamente.
- Falha de rede: mensagem amigável sem expor credenciais.

**Exemplo 1 — catálogo rápido:**

```yaml
multi_agents:
  - id: "catalogo_pluggto"
    tools:
      - "pluggto_buscar_produtos"
    global_tools_configuration:
      pluggto:
        base_url: "https://api.plugg.to"
        search_path: "/products/search"
    security_keys:
      PLUGGTO_API_KEY: "${PLUGGTO_API_KEY}"

# Prompt: "Liste camisetas pretas tamanho M".
```

**Exemplo 2 — SKU específico com query_param customizado:**

```yaml
multi_agents:
  - id: "catalogo_pluggto_sku"
    tools:
      - "pluggto_buscar_produtos"
    global_tools_configuration:
      pluggto:
        base_url: "https://api.plugg.to"
        search_path: "/products/search"
        query_param: "sku"
        timeout: 15
    security_keys:
      PLUGGTO_API_KEY: "${PLUGGTO_API_KEY}"

# Prompt: "Busque SKU 123-ABC e traga atributos completos".
```

**Exemplo 3 — busca resiliente com backoff ajustado:**

```yaml
multi_agents:
  - id: "catalogo_pluggto_backoff"
    tools:
      - "pluggto_buscar_produtos"
    global_tools_configuration:
      pluggto:
        base_url: "https://api.plugg.to"
        search_path: "/products/search"
        retry_attempts: 5
        backoff_base: 2
    security_keys:
      PLUGGTO_API_KEY: "${PLUGGTO_API_KEY}"

# Prompt: "Liste notebooks gamer".
```

### 🛒 Magalu Hub (Marketplace)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `magalu_hub_buscar_produtos` | Produtos Magalu Hub | Catálogo multicanal |

**Explicação (conceitual):** integra com a API Magalu Hub via API Key.
`query` é obrigatória (nome, SKU ou categoria) e retorna lista de
produtos normalizados.

**Explicação for dummies:** digite um termo como `geladeira frost free`
ou um SKU e receba os produtos do Magalu Hub, sem expor o bearer token.

**Parâmetro obrigatório:**

- `query`: termo de busca (nome, SKU ou categoria).

**Configuração YAML obrigatória:**

- `security_keys.MAGALU_HUB_API_KEY`.
- Endpoints são fixos (`https://api.magalu.com` + `/hub/products/search`).
  Não há parametrização YAML para host ou path.
- Opcionais: `query_param`, `timeout`, `retry_attempts`, `backoff_base`
  em `global_tools_configuration.magalu_hub` para ajustar consulta e resiliência.

**Mensagens de falha comuns:**

- 401/403: autenticação falhou, revise o API Key.
- 429: limite da API, aguarde e tente novamente.
- Falha de rede: mensagem amigável sem expor credenciais.

**Exemplo 1 — catálogo rápido:**

```yaml
multi_agents:
  - id: "catalogo_magalu"
    tools:
      - "magalu_hub_buscar_produtos"
    global_tools_configuration:
      magalu_hub: {}
    security_keys:
      MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"

# Prompt: "Liste TVs 55 polegadas 4k".
```

**Exemplo 2 — SKU específico com query_param customizado:**

```yaml
multi_agents:
  - id: "catalogo_magalu_sku"
    tools:
      - "magalu_hub_buscar_produtos"
    global_tools_configuration:
      magalu_hub:
        query_param: "sku"
        timeout: 15
    security_keys:
      MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"

# Prompt: "Busque SKU ABC-123 e traga atributos completos".
```

**Exemplo 3 — busca resiliente com backoff ajustado:**

```yaml
multi_agents:
  - id: "catalogo_magalu_backoff"
    tools:
      - "magalu_hub_buscar_produtos"
    global_tools_configuration:
      magalu_hub:
        retry_attempts: 5
        backoff_base: 2
    security_keys:
      MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"

# Prompt: "Liste notebooks gamer".
```

**Exemplo 4 — alerta de estoque via WhatsApp:**

```yaml
multi_agents:
  - id: "alerta_magalu_whatsapp"
    tools:
      - "magalu_hub_buscar_produtos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      magalu_hub: {}
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt: "Busque 'liquidificador 110v' e envie os primeiros 3 via WhatsApp".
```

**Exemplo 5 — filtro local e resumo curto:**

```yaml
multi_agents:
  - id: "catalogo_magalu_resumo"
    tools:
      - "magalu_hub_buscar_produtos"
    global_tools_configuration:
      magalu_hub: {}
    security_keys:
      MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"

# Prompt: "Busque 'cadeira ergonomica', filtre preço < 800 e resuma em 5 itens".
```

### 🛍️ Amazon SP-API (Marketplace)

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `amazon_sp_api_buscar_produtos` | Produtos Amazon | Catálogo Amazon |

**Explicação (conceitual):** integra com a Amazon Selling Partner API via
token bearer. `query` é obrigatória (nome, SKU ou categoria) e retorna
lista de dicionários normalizados.

**Explicação for dummies:** informe um termo como `kindle` ou um SKU e
receba a lista de produtos da Amazon sem expor o token.

**Parâmetro obrigatório:**

- `query`: termo de busca (nome, SKU ou categoria).

**Configuração YAML obrigatória:**

- `security_keys.AMAZON_SP_API_TOKEN`.
- `global_tools_configuration.amazon_sp_api.base_url`.
- `global_tools_configuration.amazon_sp_api.search_path`.
- Opcionais: `query_param`, `timeout`, `retry_attempts`, `backoff_base`
  em `global_tools_configuration.amazon_sp_api`.

**Mensagens de falha comuns:**

- 401/403: autenticação falhou, revise o token.
- 429: limite da API, aguarde e tente novamente.
- Falha de rede: mensagem amigável sem expor credenciais.

**Exemplo 1 — catálogo básico:**

```yaml
multi_agents:
  - id: "catalogo_amazon"
    tools:
      - "amazon_sp_api_buscar_produtos"
    global_tools_configuration:
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-na.amazon.com"
        search_path: "/products/search"
    security_keys:
      AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

# Prompt: "Liste livros de data science".
```

**Exemplo 2 — SKU específico com query_param customizado:**

```yaml
multi_agents:
  - id: "catalogo_amazon_sku"
    tools:
      - "amazon_sp_api_buscar_produtos"
    global_tools_configuration:
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-eu.amazon.com"
        search_path: "/products/search"
        query_param: "sku"
        timeout: 15
    security_keys:
      AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

# Prompt: "Busque SKU B00TEST123 e traga atributos completos".
```

**Exemplo 3 — busca resiliente com backoff ajustado:**

```yaml
multi_agents:
  - id: "catalogo_amazon_backoff"
    tools:
      - "amazon_sp_api_buscar_produtos"
    global_tools_configuration:
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-fe.amazon.com"
        search_path: "/products/search"
        retry_attempts: 5
        backoff_base: 2
    security_keys:
      AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

# Prompt: "Liste smartwatches com GPS".
```

**Exemplo 4 — alerta de estoque via WhatsApp:**

```yaml
multi_agents:
  - id: "alerta_amazon_whatsapp"
    tools:
      - "amazon_sp_api_buscar_produtos"
      - "whatsapp_send_text_message"
    global_tools_configuration:
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-na.amazon.com"
        search_path: "/products/search"
      whatsapp_cloud:
        phone_number_id: "${WHATSAPP_PHONE_ID}"
    security_keys:
      AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"
      WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

# Prompt: "Busque 'ps5' e envie os 3 mais baratos para +551199999999".
```

**Exemplo 5 — filtro local e resumo curto:**

```yaml
multi_agents:
  - id: "catalogo_amazon_resumo"
    tools:
      - "amazon_sp_api_buscar_produtos"
    global_tools_configuration:
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-na.amazon.com"
        search_path: "/products/search"
    security_keys:
      AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

# Prompt: "Busque 'mouse sem fio', filtre preço < 150 e resuma em 5 itens".
```

### 🏪 Outros ERPs

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `sap_query` | Consultas SAP | Dados ERP |
| `totvs_integration` | Integração TOTVS | Protheus/Datasul |

---

## 🤖 IA e Machine Learning

**Objetivo**: Recursos avançados de inteligência artificial.

### 🧠 Modelos de IA

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `openai_completion` | Completar texto | Geração de conteúdo |
| `claude_chat` | Conversar com Claude | Análises profundas |
| `huggingface_model` | Modelos HuggingFace | ML customizado |

### 🖼️ Visão Computacional

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `image_analyzer` | Analisar imagens | OCR, classificação |
| `face_detection` | Detectar rostos | Biometria |

### 🎤 Áudio e Voz

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `speech_to_text` | Transcrever áudio | Atas de reunião |
| `text_to_speech` | Sintetizar voz | Audiobooks |

---

## 🔧 Utilidades Gerais

**Objetivo**: Ferramentas auxiliares diversas.

### 🆔 Identificadores

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `uuid_generate` | Gerar UUID | IDs únicos |
| `hash_generator` | Gerar hashes | Checksums |

### 📝 Textos

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `string_inverter` | Inverter texto | Testes |
| `text_formatter` | Formatar texto | Padronização |
| `regex_matcher` | Expressões regulares | Validações |

### 📅 Data e Hora

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| `date_calculator` | Cálculos de data | Prazos |
| `timezone_converter` | Converter fusos | Internacional |

### 🔒 Controle e Governança

| Ferramenta | O Que Faz | Exemplo de Uso |
|------------|-----------|----------------|
| [**`human_gate`**](../README-HUMAN-IN-THE-LOOP.md) ⚠️ | Aprovação humana | Ações críticas |
| `audit_logger` | Log de auditoria | Compliance |

**💼 Caso de Uso Crítico**: Transferências financeiras

```yaml
multi_agents:
  - id: "agente_financeiro"
    tools:
      - "human_gate"              # ⚠️ OBRIGATÓRIO
      - "fazer_transferencia"     # Só executa após aprovação
```

---

## 📊 Resumo por Categoria Técnica

| Categoria Técnica | Quantidade | Principais Ferramentas |
|-------------------|------------|------------------------|
| **vendor_tools** | 100 | WhatsApp, Slack, BrasilAPI, Linx |
| **domain_tools** | 63 | SQL dinâmico, API dinâmica, Pandas |
| **external_tools** | 39 | DuckDuckGo, HTTP, YouTube |
| **vector_store_tools** | 4 | RAG, QA Search |
| **custom_tools** | 4 | Calculator, UUID, String |
| **system_tools** | 2 | Human Gate, write_todos |

---

## 💡 Dicas de Seleção

### Escolha Ferramentas Dinâmicas Quando

- Precisa de **flexibilidade** (queries/endpoints variam)
- Quer **configurar no YAML** (sem código Python)
- Tem **múltiplas fontes** (vários bancos/APIs)
- Precisa de **lazy loading** (carregar sob demanda)

### Escolha Ferramentas Específicas Quando

- Integração **já pronta** (WhatsApp, BrasilAPI)
- **Não precisa customizar** (usa como está)
- É um **serviço conhecido** (Slack, Gmail)

---

## 🆘 Precisa de Ajuda?

**Não encontrou o que precisa?**

1. Veja a [lista alfabética completa](alfabetica.md)
2. Confira o [guia central de tools](../GUIA-USUARIO-TOOLS.md)
3. Use ferramentas dinâmicas ([SQL](sql_dinamico.md) ou [API](api_dinamica.md))

---

<div align="center">

**[⬆️ Voltar ao Topo](#-ferramentas-por-finalidade)**
**[🏠 Página Principal](../README.md)**

</div>
