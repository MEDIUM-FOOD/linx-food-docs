# 📣 Guia de Tools de Esportes

Visão única das ferramentas esportivas para consultas de jogos, ligas,
times, atletas e odds. Todas seguem o padrão `@tool_factory`, exigem
`user_session.correlation_id` e registram logs sem expor bearer.

## 🗺️ Mapa rápido (por provedor)

- **Sportsradar**: eventos, agenda de competição, times, perfil jogador,
  tabela da competição.
- **Sportmonks**: partidas por ID/data, time por ID, jogador por ID,
  classificação por temporada.
- **API-Sports**: partida por ID, partidas por data, time por nome,
  jogadores por time, classificação de liga.
- **GoalServe**: partida por ID, partidas por data, time por nome,
  jogadores por time, classificação de liga.
- **Opta**: competições, fixtures, standings, equipes, jogadores.
- **Enetpulse**: eventos, fixtures, odds, standings, equipes.

## 🔐 Credenciais e blocos YAML

| Provedor | Chave em `security_keys` | Bloco `global_tools_configuration` |
|----------|--------------------------|-----------------------------|
| Sportsradar | `SPORTSRADAR_API_KEY` | `sportsradar.endpoints.*` |
| Sportmonks | `SPORTMONKS_API_TOKEN` | `sportmonks.endpoints.*` |
| API-Sports | `APISPORTS_API_KEY` | `apisports.endpoints.*` |
| GoalServe | `GOALSERVE_API_KEY` | `goalserve.auth_param`, `goalserve.endpoints.*` |
| Opta | `OPTA_API_KEY` | `opta.base_url`, `opta.query_param`, `opta.routes.*` (obrigatórios: competitions, fixtures, standings, teams, players) |
| Enetpulse | `ENETPULSE_API_KEY` | `enetpulse.base_url`, `enetpulse.query_param`, `enetpulse.routes.*` (obrigatórios: events, fixtures, odds, standings, teams) |

> Dica: mantenha as chaves somente em `security_keys`; rotas ficam no
> bloco de `global_tools_configuration` do cliente. `base_url` só é
> configurável para Opta e Enetpulse (self-host ou URLs dedicadas).

## 🧰 Catálogo por provedor

### Sportsradar
- `sportsradar_resumo_evento`
- `sportsradar_partidas_competicao`
- `sportsradar_times_competicao`
- `sportsradar_perfil_jogador`
- `sportsradar_classificacao_competicao`

**Parâmetros de query**
- `resumo_evento`: ID `sr:sport_event:<uuid>`
- `partidas_competicao`: `sr:competition:<id>`
- `times_competicao`: `sr:competition:<id>`
- `perfil_jogador`: `sr:player:<id>`
- `classificacao_competicao`: `sr:competition:<id>`

### Sportmonks
- `sportmonks_partida_por_id`
- `sportmonks_partidas_por_data`
- `sportmonks_time_por_id`
- `sportmonks_jogador_por_id`
- `sportmonks_classificacao_temporada`

**Parâmetros de query**
- Partida por ID: `123456`
- Partidas por data: `YYYY-MM-DD`
- Time por ID: numérico
- Jogador por ID: numérico
- Classificação: ID de temporada (ano ou código)

### API-Sports
- `apisports_partida_por_id`
- `apisports_partidas_por_data`
- `apisports_times_por_nome`
- `apisports_jogadores_por_time`
- `apisports_classificacao_liga`

**Parâmetros de query**
- Partida por ID: `123456`
- Partidas por data: `YYYY-MM-DD`
- Times por nome: texto
- Jogadores por time: `team=<id>&season=<ano>`
- Classificação: `league=<id>&season=<ano>`

### GoalServe
- `goalserve_partida_por_id`
- `goalserve_partidas_por_data`
- `goalserve_times_por_nome`
- `goalserve_jogadores_por_time`
- `goalserve_classificacao_liga`

**Parâmetros de query**
- Partida por ID: `123456`
- Partidas por data: `YYYY-MM-DD`
- Times por nome: texto
- Jogadores por time: `team=<id>&season=<ano>`
- Classificação: `league=<id>&season=<ano>`

> Config: base_url fixo `https://www.goalserve.com`;
> `goalserve.auth_param` (padrão `key`) e `goalserve.endpoints.*`
> aceitam override.

### Opta
- `opta_buscar_competicoes`
- `opta_buscar_fixtures`
- `opta_buscar_classificacao`
- `opta_buscar_equipes`
- `opta_buscar_jogadores`

**Parâmetros de query**
- Todos exigem `query` (texto). Query param default: `q`.
- Rotas obrigatórias em `opta.routes`: `competitions`, `fixtures`,
  `standings`, `teams`, `players`.

### Enetpulse
- `enetpulse_buscar_eventos`
- `enetpulse_buscar_fixtures`
- `enetpulse_buscar_odds`
- `enetpulse_buscar_classificacao`
- `enetpulse_buscar_equipes`

**Parâmetros de query**
- Todos exigem `query` (texto). Query param default: `q`.
- Rotas obrigatórias em `enetpulse.routes`: `events`, `fixtures`,
  `odds`, `standings`, `teams`.


## ⚙️ Formatos de resposta e erros

- Resposta sempre em **lista de dicionários** para padronizar o fluxo.
- Erros 401/403: mensagem clara de autenticação falhou (sem bearer).
- Erro 429: mensagem de rate limit e sugestão para tentar depois.
- Erros 4xx/5xx: mensagem genérica mantendo lista de dicts.

## ⚡ Exemplos rápidos (yaml)

1) Radar completo (agenda + standings)

```yaml
security_keys:
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"
  SPORTMONKS_API_TOKEN: "${SPORTMONKS_API_TOKEN}"

multi_agents:
  - id: "radar_futebol"
    tools:
      - "apisports_partidas_por_data"
      - "sportmonks_classificacao_temporada"
    local_tools_configuration:
      sportmonks:
        endpoints:
          standings_by_season: "/v3/football/standings/seasons/{query}"
```

2) Briefing da competição (agenda + tabela)

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"

multi_agents:
  - id: "briefing_competicao"
    tools:
      - "sportsradar_partidas_competicao"
      - "sportsradar_classificacao_competicao"
```

3) Perfil jogador (Sportsradar) + elenco (API-Sports)

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"

multi_agents:
  - id: "perfil_elenco"
    tools:
      - "sportsradar_perfil_jogador"
      - "apisports_jogadores_por_time"
```

4) Agenda do dia (Sportmonks)

```yaml
security_keys:
  SPORTMONKS_API_TOKEN: "${SPORTMONKS_API_TOKEN}"

multi_agents:
  - id: "agenda_dia"
    tools:
      - "sportmonks_partidas_por_data"
```

5) Times + jogadores (API-Sports)

```yaml
security_keys:
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"

multi_agents:
  - id: "times_jogadores"
    tools:
      - "apisports_times_por_nome"
      - "apisports_jogadores_por_time"
```
6) Odds + eventos (Enetpulse)

```yaml
security_keys:
  ENETPULSE_API_KEY: "${ENETPULSE_API_KEY}"

multi_agents:
  - id: "odds_eventos_enetpulse"
    tools:
      - "enetpulse_buscar_eventos"
      - "enetpulse_buscar_odds"
    local_tools_configuration:
      enetpulse:
        base_url: "https://api.enetpulse.example"
        routes:
          events: "/v1/events/search"
          odds: "/v1/odds/search"
          fixtures: "/v1/fixtures/search"
          standings: "/v1/standings/search"
          teams: "/v1/teams/search"
```

7) Fixtures + standings (Opta)

```yaml
security_keys:
  OPTA_API_KEY: "${OPTA_API_KEY}"

multi_agents:
  - id: "fixtures_standings_opta"
    tools:
      - "opta_buscar_fixtures"
      - "opta_buscar_classificacao"
    local_tools_configuration:
      opta:
        base_url: "https://api.opta.example"
        routes:
          competitions: "/v1/competitions/search"
          fixtures: "/v1/fixtures/search"
          standings: "/v1/standings/search"
          teams: "/v1/teams/search"
          players: "/v1/players/search"
```

8) Elenco + tabela (GoalServe)

```yaml
security_keys:
  GOALSERVE_API_KEY: "${GOALSERVE_API_KEY}"

multi_agents:
  - id: "elenco_tabela_goalserve"
    tools:
      - "goalserve_jogadores_por_time"
      - "goalserve_classificacao_liga"
    local_tools_configuration:
      goalserve:
        auth_param: "key"
        endpoints:
          players_by_team: "/getfeed/footballplayers"
          standings_by_league: "/getfeed/footballstandings"
```

## 🧪 Casos de uso avançados

1) **Scout pré-jogo** (fixtures + elenco)
- Tools: `apisports_partidas_por_data`, `apisports_jogadores_por_time`,
  `sportsradar_resumo_evento`.
- Prompt: "Liste jogos de {data}, principais jogadores e um resumo do
  confronto X vs Y".

2) **Mercado de transferências**
- Tools: `apisports_times_por_nome`, `apisports_jogadores_por_time`,
  `sportsradar_perfil_jogador`.
- Prompt: "Ache laterais brasileiros na Europa e traga clube atual e
  perfil".

3) **Monitor de ligas menores com odds**
- Tools: `sportmonks_partidas_por_data`, `enetpulse_buscar_odds`.
- Prompt: "Quais jogos hoje nas divisões B/C e quais odds disponíveis?".

4) **Cobertura de competição (Opta)**
- Tools: `opta_buscar_competicoes`, `opta_buscar_fixtures`,
  `opta_buscar_classificacao`.
- Prompt: "Monte briefing da competição X com tabela e próximos jogos".

5) **Análise de performance por time (GoalServe)**
- Tools: `goalserve_partidas_por_data`, `goalserve_jogadores_por_time`,
  `goalserve_classificacao_liga`.
- Prompt: "Forme visão da performance do time Y na temporada Z".

6) **Odds vs forma recente**
- Tools: `enetpulse_buscar_odds`, `apisports_partidas_por_data`,
  `sportmonks_partida_por_id`.
- Prompt: "Compare odds do jogo X com histórico recente dos clubes".

## 📣 Exemplos multicanal (sports + redes sociais)

1) **Matchday buzz + push** (pré-jogo em todos os canais)

```yaml
security_keys:
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

multi_agents:
  - id: "matchday_multicanal"
    tools:
      - "apisports_partidas_por_data"
      - "twitter_search"
      - "instagram_publish_media"
      - "whatsapp_send_template_message"
    local_tools_configuration:
      apisports:
        endpoints:
          fixtures_by_date: "/fixtures"
      instagram_graph:
        business_account_id: "1789..."
        api_version: "v21.0"
      whatsapp_cloud:
        phone_number_id: "5511999999999"
        api_version: "v20.0"
```

- Fluxo sugerido: pegar fixtures do dia, buscar tweets sobre o
  clássico, publicar card no Instagram (imagem em `media_url`) e
  disparar template no WhatsApp para a base.

2) **Resumo pós-jogo** (recortes + resposta em DM e WhatsApp)

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

multi_agents:
  - id: "resumo_pos_jogo"
    tools:
      - "sportsradar_resumo_evento"
      - "twitter_mentions_search"
      - "instagram_send_direct_message"
      - "whatsapp_send_text_message"
    local_tools_configuration:
      instagram_graph:
        business_account_id: "1789..."
      whatsapp_cloud:
        phone_number_id: "5511999999999"
```

- Fluxo sugerido: consolidar estatísticas do jogo via Sportsradar,
  capturar menções no Twitter, responder torcedores VIP no Instagram
  Direct e enviar um texto curto de pós-jogo no WhatsApp.

## 🛍️ Exemplos sports + Twitter + Instagram + Amazon

1) **Merch pós-jogo** (recorte + social + vitrine)

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"
  AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

multi_agents:
  - id: "merch_pos_jogo"
    tools:
      - "sportsradar_resumo_evento"
      - "twitter_mentions_search"
      - "instagram_publish_media"
      - "amazon_sp_api_buscar_produtos"
    local_tools_configuration:
      instagram_graph:
        business_account_id: "1789..."
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-na.amazon.com"
        search_path: "/products/2020-08-26/search"
        query_param: "keywords"
```

- Fluxo sugerido: puxar resumo do jogo, capturar menções do craque,
  publicar card do destaque no Instagram e buscar camisetas ou itens do
  clube na Amazon para recomendar.

2) **Pré-venda da final** (hype + catálogo)

```yaml
security_keys:
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"
  AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

multi_agents:
  - id: "prevenda_final"
    tools:
      - "apisports_partidas_por_data"
      - "twitter_search"
      - "instagram_send_direct_message"
      - "amazon_sp_api_buscar_produtos"
    local_tools_configuration:
      apisports:
        endpoints:
          fixtures_by_date: "/fixtures"
      instagram_graph:
        business_account_id: "1789..."
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-na.amazon.com"
        search_path: "/products/2020-08-26/search"
        query_param: "keywords"
```

- Fluxo sugerido: achar fixture da final, buscar buzz no Twitter com a
  hashtag oficial, enviar DM no Instagram para lista VIP e retornar
  produtos oficiais (camiseta, cachecol) via Amazon para ofertar.

## 🛒 Exemplos sports + Instagram + varejo

1) **Camisa oficial pós-jogo** (destaque + vitrine)

```yaml
security_keys:
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"
  MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"

multi_agents:
  - id: "camisa_pos_jogo"
    tools:
      - "apisports_partidas_por_data"
      - "instagram_publish_media"
      - "magalu_hub_buscar_produtos"
    local_tools_configuration:
      apisports:
        endpoints:
          fixtures_by_date: "/fixtures"
      instagram_graph:
        business_account_id: "1789..."
      magalu_hub: {}
```

- Fluxo sugerido: pegar fixture que acabou, publicar arte do campeão no
  Instagram e buscar camisas oficiais no Magalu para recomendação.

2) **Kit torcedor para a final** (recomendação multiloja)

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"
  INSTAGRAM_GRAPH_ACCESS_TOKEN: "${INSTAGRAM_GRAPH_ACCESS_TOKEN}"
  AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"
  PLUGGTO_API_KEY: "${PLUGGTO_API_KEY}"

multi_agents:
  - id: "kit_torcedor_final"
    tools:
      - "sportsradar_resumo_evento"
      - "instagram_send_direct_message"
      - "amazon_sp_api_buscar_produtos"
      - "pluggto_buscar_produtos"
    local_tools_configuration:
      instagram_graph:
        business_account_id: "1789..."
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-na.amazon.com"
        search_path: "/products/2020-08-26/search"
        query_param: "keywords"
      pluggto:
        base_url: "https://api.plugg.to"
        search_path: "/products/search"
```

- Fluxo sugerido: resumir o jogo, enviar DM para segmentados com link de
  produtos e comparar kits (camisa, cachecol, bandeira) entre Amazon e
  Plugg.to para ofertar o melhor preço.

## 📲 Exemplos sports + WhatsApp

1) **Alerta de início de jogo** (push rápido)

```yaml
security_keys:
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

multi_agents:
  - id: "alerta_jogo"
    tools:
      - "apisports_partidas_por_data"
      - "whatsapp_send_template_message"
    local_tools_configuration:
      apisports:
        endpoints:
          fixtures_by_date: "/fixtures"
      whatsapp_cloud:
        phone_number_id: "5511999999999"
        api_version: "v20.0"

```
```

- Fluxo sugerido: localizar partidas do dia e disparar template com hora
  e adversário para a lista de contatos.

2) **Pós-jogo com score e link**

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

multi_agents:
  - id: "pos_jogo_whatsapp"
    tools:
      - "sportsradar_resumo_evento"
      - "whatsapp_send_text_message"
    local_tools_configuration:
      whatsapp_cloud:
        phone_number_id: "5511999999999"
```

- Fluxo sugerido: obter resumo final do jogo e enviar mensagem com placar
  e link do boxscore/ingressos.

## 🧭 Quando usar qual

- Quer cobertura ampla com eventos detalhados? → Sportsradar.
- Busca futebol com rotas simples e custo menor? → Sportmonks ou
  API-Sports.
- Precisa odds e mercados rápidos? → Enetpulse.
- Prefere schema XML/JSON resiliente com auth por query? → GoalServe.
- Necessita standings e scouting ricos via rotas customizáveis? → Opta.

## 🚦 Tratamento de erros (padrão)

- 401/403: mensagem explícita de autenticação falhou (sem mostrar
  bearer).
- 429: aviso de limite e orientação para tentar mais tarde.
- Outros erros: mensagem genérica, mantendo lista de `dict`.
- Sempre retorna lista, mesmo em erro, para padronizar o fluxo dos
  agentes.

## 🧾 Workflows reais (Twitter + LLM + WhatsApp)

> Baseado no padrão de nodes usado em `app/yaml/system/rag-config-
> modelo.yaml` (modes `set` + `agent` + `whatsapp_send`). O LLM vem do
> pool global (bloco `llm` do YAML), não como tool.

1) **Pré-jogo** — coletar buzz e avisar gestor

```yaml
security_keys:
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4.1-mini"

workflows:
  - id: "sent_pre_jogo"
    nodes:
      - id: "preparar_consulta"
        mode: "set"
        reads: ["input_text"]
        params:
          assign:
            vars.twitter.query: "{input_text}"

      - id: "coletar_tweets"
        mode: agent
        reads: ["vars.twitter.query"]
        prompt:
          system: |
            Busque tweets recentes sobre {vars.twitter.query}.
            Use twitter_search e retorne lista normalizada.
        tools: [twitter_search]

      - id: "resumir_sentimento"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Resuma sentimento geral, tópicos e riscos em 5 linhas.
        tools: []

      - id: "enviar_whatsapp"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Envie via whatsapp_send_text_message para {vars.destino}.
        tools: [whatsapp_send_text_message]
```

## 🧾 Workflows para esportes (consultas + push)

### Workflow 1 — Briefing pré-jogo + alerta

Pipeline de nodes para pegar fixtures do dia (API-Sports), resumir e
avisar no WhatsApp.

```yaml
security_keys:
  APISPORTS_API_KEY: "${APISPORTS_API_KEY}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4.1-mini"

workflows:
  - id: "workflow_pre_jogo_alerta"
    enabled: false
    nodes:
      - id: "preparar_data"
        mode: "set"
        reads: ["input_text"]
        params:
          assign:
            vars.data: "{input_text}"

      - id: "buscar_fixtures"
        mode: agent
        reads: ["vars.data"]
        prompt:
          system: |
            Liste partidas na data {vars.data}.
            Traga time A, time B e horário.
        tools: [apisports_partidas_por_data]

      - id: "resumir_partidas"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Gere resumo curto das partidas (3-5 linhas).
        tools: []

      - id: "enviar_whatsapp"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Envie via whatsapp_send_text_message para {vars.destino}.
        tools: [whatsapp_send_text_message]

    local_tools_configuration:
      apisports:
        endpoints:
          fixtures_by_date: "/fixtures"
      whatsapp_cloud:
        phone_number_id: "5511999999999"
        api_version: "v20.0"
```

### Workflow 2 — Pós-jogo com resumo e odds

Usa Sportsradar para resumo e Enetpulse para odds, entrega síntese em
texto (pode ser usado antes de um push manual).

```yaml
security_keys:
  SPORTSRADAR_API_KEY: "${SPORTSRADAR_API_KEY}"
  ENETPULSE_API_KEY: "${ENETPULSE_API_KEY}"

llm:
  provider: "openai"
  model: "gpt-4o-mini"

workflows:
  - id: "workflow_pos_jogo_analytics"
    enabled: false
    nodes:
      - id: "preparar_id"
        mode: "set"
        reads: ["input_text"]
        params:
          assign:
            vars.evento_id: "{input_text}"

      - id: "resumo_partida"
        mode: agent
        reads: ["vars.evento_id"]
        prompt:
          system: |
            Busque resumo do evento {vars.evento_id}.
        tools: [sportsradar_resumo_evento]

      - id: "buscar_odds"
        mode: agent
        reads: ["vars.evento_id"]
        prompt:
          system: |
            Consulte odds relacionadas ao evento {vars.evento_id}.
        tools: [enetpulse_buscar_odds]

      - id: "consolidar"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Resuma placar, destaques e odds principais em 5 linhas.
        tools: []

    local_tools_configuration:
      enetpulse:
        base_url: "https://api.enetpulse.example"
        routes:
          events: "/v1/events/search"
          fixtures: "/v1/fixtures/search"
          odds: "/v1/odds/search"
          standings: "/v1/standings/search"
          teams: "/v1/teams/search"

```
```

### Workflow 3 — Buzz + push condicional (Twitter → WhatsApp)

Ativa push no WhatsApp apenas quando o usuário pediu. Usa condição com
`mode: "if"` para exemplificar bifurcação simples.

```yaml
security_keys:
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4.1"

workflows:
  - id: "workflow_buzz_condicional"
    enabled: false
    nodes:
      - id: "preparar"
        mode: "set"
        reads: ["input_text"]
        params:
          assign:
            vars.time: "{input_text}"
            vars.enviar_push: { expr: "'whatsapp' in input_text.lower()" }

      - id: "coletar_buzz"
        mode: agent
        reads: ["vars.time"]
        prompt:
          system: |
            Busque tweets recentes sobre {vars.time}.
            Traga tópicos e sentimento geral.
        tools: [twitter_search]

      - id: "resumir"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Resuma em 4 linhas sentimento, elogios e críticas.
        tools: []

      - id: "devo_enviar"
        mode: "if"
        reads: ["vars.enviar_push"]
        condition: "vars.enviar_push == True" # noqa: E712
        true_go_to: "enviar_whatsapp"
        false_go_to: "fim"

      - id: "enviar_whatsapp"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Envie via whatsapp_send_text_message para {vars.destino}.
        tools: [whatsapp_send_text_message]

      - id: "fim"
        mode: "set"
        reads: ["last_output"]
        params:
          assign:
            outgoing.text: "{last_output}"

    local_tools_configuration:
      twitter_search:
        TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
      whatsapp_cloud:
        phone_number_id: "5511999999999"
        api_version: "v20.0"
```
```

2) **Ao vivo** — menções durante a partida

```yaml
security_keys:
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4o"

workflows:
  - id: "sent_live"
    nodes:
      - id: "coletar_mencoes"
        mode: agent
        reads: ["input_text"]
        prompt:
          system: |
            Busque menções recentes ao time {input_text}.
            Use twitter_mentions_search.
        tools: [twitter_mentions_search]

      - id: "resumir_live"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Gere bullets com sentimento, elogios e críticas.
        tools: []

      - id: "enviar_whatsapp"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Envie via whatsapp_send_text_message para {vars.destino}.
        tools: [whatsapp_send_text_message]
```

3) **Pós-jogo** — resumo final ao gestor

```yaml
security_keys:
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4.1"

workflows:
  - id: "sent_pos"
    nodes:
      - id: "coletar_tweets"
        mode: agent
        reads: ["input_text"]
        prompt:
          system: |
            Busque tweets pós-jogo sobre {input_text} (últimas horas).
            Use twitter_search.
        tools: [twitter_search]

      - id: "resumir_pos"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Monte relatório curto: sentimento, destaques, riscos.
        tools: []

      - id: "enviar_whatsapp"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Envie via whatsapp_send_text_message para {vars.destino}.
        tools: [whatsapp_send_text_message]
```
