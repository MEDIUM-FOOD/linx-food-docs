# 🛍️ Manual de Tools de Varejo e Marketplaces

## Visão geral

Concentra as ferramentas de catálogo, marketplace e estoque já disponíveis
(Magalu Hub, Amazon SP-API, Plugg.to, Linx) para consultas de produtos,
comparação de ofertas e alertas operacionais.

## Por que existe

- Destacar o pacote varejo em um único lugar, sem depender de buscas em
  listas alfabéticas.
- Facilitar combinações rápidas (busca, comparação, alerta) com exemplos
  prontos.
- Padronizar pré-requisitos de configuração, evitando erros de autenticação
  ou de parâmetros obrigatórios.

## Explicação conceitual

- Todas as tools recebem `query` obrigatória e retornam lista de dicionários
  normalizados (produto/atributos/preço/estoque quando expostos pela API).
- Falhas críticas exibem mensagens claras para 401/403 e 429, sem vazar
  bearer/API Key; retries com backoff já vêm configurados.
- Logs sempre carregam `correlation_id` do YAML, preservando o rastro da
  sessão multi-tenant.
- As tools são descobertas automaticamente pelo builder e ficam disponíveis
  no catálogo carregado pelo sistema.

## Explicação for dummies

Peça ao agente algo como "busque PS5 nos marketplaces" e ele consulta Magalu
Hub, Amazon ou Plugg.to, devolvendo listas de produtos já normalizadas.

## Catálogo varejo (onde buscar o detalhe)

- `magalu_hub_buscar_produtos` — Magalu Hub; termo/SKU;
  ver [catálogo por finalidade](por_finalidade.md)
- `amazon_sp_api_buscar_produtos` — Amazon SP-API; termo/SKU;
  ver [catálogo por finalidade](por_finalidade.md)
- `pluggto_buscar_produtos` — Plugg.to; catálogo multicanal;
  ver [catálogo por finalidade](por_finalidade.md)
- `linx_product_search` — Linx; consulta produtos ERP;
  ver [catálogo por finalidade](por_finalidade.md)

## Como o usuário recebe essa feature

1. Adicione as chaves em `security_keys` do YAML (veja a seção do provider).
2. Configure `global_tools_configuration` para parâmetros comuns e credenciais
  por provider (ex.: `base_url`, `search_path`, `phone_number_id`).
3. Inclua a tool no bloco `multi_agents` ou workflow desejado.
4. Gere/atualize o catálogo via tools_library_builder quando incluir novas
   factories (já padrão no projeto).

## Entendendo `global_tools_configuration` e `local_tools_configuration`

### O que é `global_tools_configuration`

Bloco na raiz do YAML que concentra configurações **padrão por tool**. É o
local certo para credenciais e parâmetros compartilhados entre agentes e
workflows. Exemplo típico: `base_url` e `search_path` que valem para todo o
ambiente.

### O que é `local_tools_configuration`

Bloco dentro de um agente (`multi_agents[]`) ou de um workflow. Serve para
**override local**, quando um único agente/workflow precisa de ajustes
específicos (por exemplo, timeout menor, query_param diferente ou endpoint
regional).

### Ordem de precedência

1. `local_tools_configuration` do agente (mais específico)
2. `local_tools_configuration` do workflow
3. `global_tools_configuration` na raiz do YAML (fallback)

Se um parâmetro existir no bloco local, ele sobrescreve o global. Use local
apenas quando houver motivo real; caso contrário, mantenha no global.

## Exemplos práticos

### Exemplo 1 — consulta multicanal com filtro local

```yaml
security_keys:
  MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"
  AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

global_tools_configuration:
  magalu_hub: {}
  amazon_sp_api:
    base_url: "https://sellingpartnerapi-na.amazon.com"
    search_path: "/products/search"

multi_agents:
  - id: "catalogo_varejo"
    tools:
      - "magalu_hub_buscar_produtos"
      - "amazon_sp_api_buscar_produtos"

# Prompt: "Busque 'smartwatch', filtre preço < 800 e resuma em 5 itens".
```

### Exemplo 2 — alerta de estoque baixo

```yaml
security_keys:
  PLUGGTO_API_KEY: "${PLUGGTO_API_KEY}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

global_tools_configuration:
  pluggto:
    base_url: "https://api.plugg.to"
    search_path: "/products/search"
  whatsapp_cloud:
    phone_number_id: "${WHATSAPP_PHONE_ID}"

multi_agents:
  - id: "alerta_estoque_varejo"
    tools:
      - "pluggto_buscar_produtos"
      - "whatsapp_send_text_message"

# Prompt: "Ache SKUs com estoque < 5 e envie alerta para +551199999999".
```

### Exemplo 3 — SKU específico com tempo de resposta curto

```yaml
security_keys:
  AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

global_tools_configuration:
  amazon_sp_api:
    base_url: "https://sellingpartnerapi-eu.amazon.com"
    search_path: "/products/search"
    query_param: "sku"

multi_agents:
  - id: "sku_fast_lane"
    tools:
      - "amazon_sp_api_buscar_produtos"
    local_tools_configuration:
      amazon_sp_api:
        timeout: 10
        retry_attempts: 3

# Prompt: "Busque SKU B00TEST123 e devolva atributos essenciais".
```

### Exemplo 4 — comparação rápida com resumo executivo

```yaml
security_keys:
  MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"
  PLUGGTO_API_KEY: "${PLUGGTO_API_KEY}"

global_tools_configuration:
  magalu_hub: {}
  pluggto:
    base_url: "https://api.plugg.to"
    search_path: "/products/search"

multi_agents:
  - id: "comparador_preco"
    tools:
      - "magalu_hub_buscar_produtos"
      - "pluggto_buscar_produtos"
```

# Prompt: "Compare 'geladeira frost free 370l' e traga top 5 por preço".

## Boas práticas e limites

- Sempre preencha `query`; entradas vazias retornam erro claro.
- Respeite 401/403 (token/API Key inválidos) e 429 (rate limit). Ajuste
  `retry_attempts` e `backoff_base` conforme a API.
- Não logue tokens. Os tools já evitam expor bearer; mantenha prompts
  limpos.
- Saída é lista de dicionários; aplique filtros locais (preço, estoque,
  categoria) no agente para não depender de ordenação da API.
- Valide `base_url` e `search_path` dos providers que permitem customização
  (Magalu Hub já é fixo no código; demais podem variar por região/ambiente).

## Troubleshooting

- 401/403: confira chave/token em `security_keys` e permissões do seller.
- 429: reduza frequência, habilite backoff maior ou amplie timeout.
- Resposta vazia: verifique `query_param` e termo de busca; teste direto no
  endpoint da API.
- Divergência de atributos: normalize campos críticos (nome, preço,
  disponibilidade) antes de mesclar catálogos distintos.
- Logs faltando correlação: garanta `user_session.correlation_id` no YAML
  carregado pelo agente.

## Workflows (exemplos práticos)

### Workflow 1 — Alerta de estoque baixo no WhatsApp

Fluxo completo no padrão de nodes LangGraph. Usa Plugg.to para localizar
SKUs críticos e envia WhatsApp com lista resumida.

```yaml
security_keys:
  PLUGGTO_API_KEY: "${PLUGGTO_API_KEY}"
  WHATSAPP_CLOUD_API_TOKEN: "${WHATSAPP_CLOUD_API_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4.1-mini"

workflows:
  - id: "workflow_alerta_estoque_whatsapp"
    enabled: false
    nodes:
      - id: "preparar_contexto"
        mode: "set"
        reads: ["input_text"]
        params:
          assign:
            vars.busca: "{input_text}"

      - id: "buscar_skus"
        mode: agent
        reads: ["vars.busca"]
        prompt:
          system: |
            Busque produtos com texto: {vars.busca}.
            Traga nome, sku, estoque e preço quando houver.
        tools: [pluggto_buscar_produtos]

      - id: "resumir_skus"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Gere lista curta de SKUs com estoque baixo (< 5) em até
            5 linhas. Formato: "SKU - nome - estoque".
        tools: []

      - id: "enviar_whatsapp"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Envie via whatsapp_send_text_message para {vars.destino}.
        tools: [whatsapp_send_text_message]

    local_tools_configuration:
      pluggto:
        base_url: "https://api.plugg.to"
        search_path: "/products/search"
      whatsapp_cloud:
        phone_number_id: "5511999999999"
        api_version: "v20.0"
```

### Workflow 2 — Pré-venda multicanal com catálogo

Localiza itens em Magalu e Amazon e devolve resumo pronto para atender
cliente (sem disparo de canal; saída fica em `last_output`).

```yaml
security_keys:
  MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"
  AMAZON_SP_API_TOKEN: "${AMAZON_SP_API_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4o-mini"

workflows:
  - id: "workflow_prevenda_multicanal"
    enabled: false
    nodes:
      - id: "preparar_consulta"
        mode: "set"
        reads: ["input_text"]
        params:
          assign:
            vars.busca: "{input_text}"

      - id: "consultar_catalogos"
        mode: agent
        reads: ["vars.busca"]
        prompt:
          system: |
            Busque produtos que combinem com: {vars.busca}.
            Traga nome, preço e link quando disponível.
        tools:
          - magalu_hub_buscar_produtos
          - amazon_sp_api_buscar_produtos

      - id: "resumir_oferta"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Monte resposta curta com top 5 itens (nome e preço). Caso
            vazio, informe que não encontrou produtos.
        tools: []

    local_tools_configuration:
      magalu_hub: {}
      amazon_sp_api:
        base_url: "https://sellingpartnerapi-na.amazon.com"
        search_path: "/products/search"
```

### Workflow 3 — Buzz social + vitrine

Combina Twitter para captar interesse e monta texto pronto para postar
no Instagram com produtos do Magalu. A publicação efetiva pode ser feita
manualmente usando `vars.social.copy`.

```yaml
security_keys:
  MAGALU_HUB_API_KEY: "${MAGALU_HUB_API_KEY}"
  TWITTER_BEARER_TOKEN: "${TWITTER_BEARER_TOKEN}"

llm:
  provider: "openai"
  model: "gpt-4o-mini"

workflows:
  - id: "workflow_vitrine_social"
    enabled: false
    nodes:
      - id: "preparar_busca"
        mode: "set"
        reads: ["input_text"]
        params:
          assign:
            vars.termo: "{input_text}"

      - id: "coletar_buzz"
        mode: agent
        reads: ["vars.termo"]
        prompt:
          system: |
            Busque tweets recentes sobre {vars.termo}.
            Resuma principais dores e desejos em 3 bullets.
        tools: [twitter_search]

      - id: "buscar_produtos"
        mode: agent
        reads: ["vars.termo"]
        prompt:
          system: |
            Traga até 5 produtos relacionados a {vars.termo} com
            nome e preço.
        tools: [magalu_hub_buscar_produtos]

      - id: "gerar_copy_instagram"
        mode: agent
        reads: ["last_output"]
        prompt:
          system: |
            Monte texto curto para Instagram com CTA e hashtags.
            Use os produtos listados e mantenha em 3 frases.
        tools: []

      - id: "salvar_saida"
        mode: "set"
        reads: ["last_output"]
        params:
          assign:
            vars.social.copy: "{last_output}"

    local_tools_configuration:
      magalu_hub: {}
```
