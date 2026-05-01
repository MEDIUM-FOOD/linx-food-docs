# 🔌 API Dinâmica - Documentação Completa

> **Ferramenta**: `dyn_api<endpoint_id>`
> **Versão**: 1.0
> **Categoria**: domain_tools
> **Fabricante**: Plataforma de Agentes de IA Custom

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Por Que Usar](#-por-que-usar)
3. [Configuração YAML](#-configuração-yaml)
4. [Casos de Uso Corporativos](#-casos-de-uso-corporativos)
5. [Exemplos Práticos](#-exemplos-práticos)
6. [Autenticação](#-autenticação)
7. [Retry e Error Handling](#-retry-e-error-handling)
8. [Processamento de Resposta](#-processamento-de-resposta)
9. [Boas Práticas](#-boas-práticas)
10. [Troubleshooting](#-troubleshooting)

---

## 🎯 Visão Geral

**API Dinâmica** permite integrar com APIs REST configuradas via YAML ou
publicadas por tenant na tabela `integrations.api_operation_registry`, sem
escrever código Python para cada endpoint.

### Como Funciona

```
Agente precisa de dados externos
        ↓
Sistema procura o endpoint no YAML e, se não achar, no registro por tenant
        ↓
Monta request com auth + headers + params
        ↓
Executa chamada HTTP (com retry automático)
        ↓
Processa resposta
        ↓
Retorna dados ao agente
```

### Características Principais

- **Configuração via YAML**: Endpoints definidos no config
- **Cadastro por tabela**: Endpoints publicados por `operation_code` no registro de integrações
- **Lazy Loading**: Carrega sob demanda
- **Autenticação automática**: API Key, Bearer, OAuth, Basic
- **Retry inteligente**: Tenta novamente em caso de falha
- **Timeout configurável**: Evita travamentos
- **Processamento de resposta**: Extrai só o necessário
- **Cliente httpx async**: Alta performance

---

## 💡 Por Que Usar?

### Antes (HTTP GET/POST Tradicional)

**Problemas**:

- Configuração no código
- Autenticação manual
- Sem retry automático
- Uma API por vez
- Requer código Python

### Depois (API Dinâmica)

**Soluções**:

- Endpoints no YAML
- Auth automática
- Retry nativo
- Múltiplas APIs
- 100% YAML

### Comparação

| Característica | HTTP Tradicional | API Dinâmica |
|----------------|------------------|--------------|
| Configuração | Código Python | YAML |
| Autenticação | Manual | Automática |
| Retry | Manual | Automático |
| Timeout | Fixo | Configurável |
| Múltiplas APIs | Complexo | Simples |
| Cache de tokens | Não | Sim (OAuth) |

---

## ⚙️ Configuração YAML

O YAML continua sendo a primeira fonte. Quando `dyn_api<endpoint_id>` aponta
para um endpoint que existe em `tools_config.api_dynamic.endpoints`, o runtime
usa essa definição e não consulta o banco de integrações.

Quando o `endpoint_id` não existe no YAML, ele passa a ser interpretado como
`operation_code` de `integrations.api_operation_registry`. Nesse caso, a
sessão precisa ter `user_session.tenant_id`, a operação precisa estar ativa,
publicada para agentes e marcada como `protocol_type=rest_json`.

Em termos práticos, isso permite manter YAMLs menores: o agente declara
`dyn_api<buscar_cliente>`, e a plataforma busca o endpoint aprovado do tenant
quando ele não está no próprio YAML.

### Estrutura Básica

```yaml
# 1. Declarar ferramentas no agente
multi_agents:
  - id: "meu_agente"
    tools:
      - "dyn_api<endpoint1>"    # Nome do endpoint
      - "dyn_api<endpoint2>"    # Outro endpoint

# 2. Configurar endpoints
local_tools_configuration:
  api_dynamic:
    # Configuração global (opcional)
    default_timeout: 30
    retry_attempts: 3
    retry_delay: 2

    # Endpoints disponíveis
    endpoints:
      endpoint1:
        url: "https://api.exemplo.com/recurso"
        method: "GET"
        description: "O que esse endpoint faz"

        # Autenticação (opcional)
        auth:
          type: "api_key"
          secret_key: "API_KEY"

        # Parâmetros (opcional)
        parameters:
          - name: "param1"
            type: "string"
            required: true
```

### Parâmetros Globais

| Parâmetro | Padrão | Descrição |
|-----------|--------|-----------|
| `default_timeout` | 30 | Timeout padrão (segundos) |
| `retry_attempts` | 3 | Tentativas em caso de erro |
| `retry_delay` | 2 | Delay entre tentativas (seg) |
| `verify_ssl` | true | Verificar certificado SSL |

### Parâmetros de Endpoint

| Parâmetro | Obrigatório | Descrição |
|-----------|-------------|-----------|
| `url` |  Sim | URL do endpoint |
| `method` |  Sim | GET, POST, PUT, DELETE, PATCH |
| `description` | ⚠️ Recomendado | Descrição para o agente |
| `auth` |   Não | Configuração de autenticação |
| `parameters` |   Não | Parâmetros da API |
| `headers` |   Não | Headers customizados |
| `timeout` |   Não | Timeout específico |

---

## 💼 Casos de Uso Corporativos

### 1. Integração com ERP

**Objetivo**: Criar pedido no sistema ERP

```yaml
tools:
  - "dyn_api<criar_pedido>"
  - "dyn_api<consultar_pedido>"

local_tools_configuration:
  api_dynamic:
    endpoints:
      criar_pedido:
        url: "https://erp.empresa.com/api/v1/pedidos"
        method: "POST"
        description: "Cria novo pedido no ERP"

        auth:
          type: "bearer"
          secret_key: "ERP_API_TOKEN"

        body_params:
          - name: "cliente_id"
            type: "integer"
            required: true
          - name: "produtos"
            type: "array"
            required: true
          - name: "observacao"
            type: "string"

        headers:
          Content-Type: "application/json"
          X-Sistema: "Plataforma de Agentes de IA"
```

**Uso**:

```
Agente: "Criar pedido para cliente 123 com produto XYZ"
→ dyn_api<criar_pedido>(cliente_id=123, produtos=[{...}])
→ Resultado: Pedido #9876 criado com sucesso
```

---

### 2. Busca de Clima (OpenWeather)

**Objetivo**: Informar clima para tomada de decisão

```yaml
tools:
  - "dyn_api<buscar_clima>"

local_tools_configuration:
  api_dynamic:
    endpoints:
      buscar_clima:
        url: "https://api.openweathermap.org/data/2.5/weather"
        method: "GET"
        description: "Busca clima atual de uma cidade"

        auth:
          type: "query_param"
          param_name: "appid"
          secret_key: "OPENWEATHER_API_KEY"

        parameters:
          - name: "cidade"
            type: "string"
            required: true
            maps_to: "q"
          - name: "unidade"
            type: "string"
            default: "metric"
            maps_to: "units"

        response:
          extract: "main"
          fields:
            - temp
            - feels_like
            - humidity
```

**Uso**:

```
Agente: "Qual o clima em São Paulo?"
→ dyn_api<buscar_clima>(cidade="São Paulo")
→ Resultado: 25°C, sensação 27°C, umidade 65%
```

---

### 3. GitHub Issues

**Objetivo**: Automatizar criação de issues

```yaml
tools:
  - "dyn_api<criar_issue>"
  - "dyn_api<listar_issues>"

local_tools_configuration:
  api_dynamic:
    endpoints:
      criar_issue:
        url: "https://api.github.com/repos/{owner}/{repo}/issues"
        method: "POST"
        description: "Cria issue no GitHub"

        auth:
          type: "bearer"
          secret_key: "GITHUB_TOKEN"

        path_params:
          - name: "owner"
            type: "string"
            required: true
          - name: "repo"
            type: "string"
            required: true

        body_params:
          - name: "titulo"
            type: "string"
            required: true
            maps_to: "title"
          - name: "descricao"
            type: "string"
            maps_to: "body"
          - name: "labels"
            type: "array"
            default: ["bug"]

        headers:
          Accept: "application/vnd.github+json"
          X-GitHub-Api-Version: "2022-11-28"
```

---

### 4. Webhook de Notificação

**Objetivo**: Notificar sistema externo

```yaml
tools:
  - "dyn_api<notificar_slack>"

local_tools_configuration:
  api_dynamic:
    endpoints:
      notificar_slack:
        url: "https://hooks.slack.com/services/{webhook_id}"
        method: "POST"
        description: "Envia mensagem para Slack"

        path_params:
          - name: "webhook_id"
            type: "string"
            required: true

        body_params:
          - name: "mensagem"
            type: "string"
            required: true
            maps_to: "text"
          - name: "canal"
            type: "string"
            maps_to: "channel"
```

---

## 📚 Exemplos Práticos

### Exemplo 1: GET Simples

**JSONPlaceholder - Buscar post**

```yaml
endpoints:
  buscar_post:
    url: "https://jsonplaceholder.typicode.com/posts/{id}"
    method: "GET"
    description: "Busca post por ID"

    path_params:
      - name: "id"
        type: "integer"
        required: true
```

**Execução**:

```
→ dyn_api<buscar_post>(id=1)
→ GET https://jsonplaceholder.typicode.com/posts/1
→ Retorna: { id: 1, title: "...", body: "..." }
```

---

### Exemplo 2: POST com Body

**Criar novo recurso**

```yaml
endpoints:
  criar_usuario:
    url: "https://api.exemplo.com/users"
    method: "POST"
    description: "Cria novo usuário"

    auth:
      type: "api_key"
      header_name: "X-API-Key"
      secret_key: "API_KEY"

    body_params:
      - name: "nome"
        type: "string"
        required: true
      - name: "email"
        type: "string"
        required: true
      - name: "idade"
        type: "integer"
```

**Execução**:

```
→ dyn_api<criar_usuario>(nome="João", email="joao@example.com", idade=30)
→ POST https://api.exemplo.com/users
→ Body: {"nome": "João", "email": "joao@example.com", "idade": 30}
```

---

### Exemplo 3: Query Parameters

**Busca com filtros**

```yaml
endpoints:
  buscar_produtos:
    url: "https://api.loja.com/produtos"
    method: "GET"
    description: "Busca produtos com filtros"

    parameters:
      - name: "categoria"
        type: "string"
      - name: "preco_max"
        type: "float"
        maps_to: "max_price"
      - name: "limite"
        type: "integer"
        default: 10
        maps_to: "limit"
```

**Execução**:

```
→ dyn_api<buscar_produtos>(categoria="eletrônicos", preco_max=500)
→ GET https://api.loja.com/produtos?categoria=eletrônicos&max_price=500&limit=10
```

---

## 🔐 Autenticação

### 1. API Key no Header

```yaml
auth:
  type: "api_key"
  header_name: "X-API-Key"      # Nome do header
  secret_key: "CHAVE_API"        # Chave do .keys.yaml
```

**Request**:

```
GET /api/recurso
X-API-Key: abc123xyz789
```

---

### 2. Bearer Token

```yaml
auth:
  type: "bearer"
  secret_key: "TOKEN_BEARER"
```

**Request**:

```
GET /api/recurso
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

---

### 3. API Key como Query Param

```yaml
auth:
  type: "query_param"
  param_name: "apikey"
  secret_key: "API_KEY"
```

**Request**:

```
GET /api/recurso?apikey=abc123xyz789
```

---

### 4. Basic Authentication

```yaml
auth:
  type: "basic"
  username_key: "API_USERNAME"   # Do .keys.yaml
  password_key: "API_PASSWORD"   # Do .keys.yaml
```

**Request**:

```
GET /api/recurso
Authorization: Basic dXNlcjpwYXNz
```

---

### 5. OAuth 2.0 (Client Credentials)

```yaml
auth:
  type: "oauth2"
  token_url: "https://auth.api.com/oauth/token"
  client_id_key: "OAUTH_CLIENT_ID"
  client_secret_key: "OAUTH_CLIENT_SECRET"
  scopes:
    - "read"
    - "write"
```

**Como funciona**:

1. Sistema busca token automaticamente
2. Cache de token com TTL
3. Renova quando expira
4. Adiciona ao header automaticamente

---

## 🔄 Retry e Error Handling

### Configuração de Retry

```yaml
api_dynamic:
  # Global
  retry_attempts: 5              # Tentar 5 vezes
  retry_delay: 3                 # 3 segundos entre tentativas

  endpoints:
    api_instavel:
      url: "..."
      # Sobrescreve global
      retry_attempts: 10         # 10 tentativas só para esta
      retry_delay: 5             # 5 segundos
```

### Estratégia de Retry

**Códigos que tentam novamente**:

- `429` - Too Many Requests (rate limit)
- `500` - Internal Server Error
- `502` - Bad Gateway
- `503` - Service Unavailable
- `504` - Gateway Timeout

**Códigos que NÃO tentam**:

- `400` - Bad Request (erro do cliente)
- `401` - Unauthorized (auth inválida)
- `403` - Forbidden (sem permissão)
- `404` - Not Found (recurso não existe)

### Backoff Exponencial

```
Tentativa 1: Imediato
Tentativa 2: Aguarda 2s
Tentativa 3: Aguarda 4s
Tentativa 4: Aguarda 8s
Tentativa 5: Aguarda 16s
```

---

## 📦 Processamento de Resposta

### Extrair Campos Específicos

```yaml
endpoints:
  buscar_clima:
    url: "https://api.openweathermap.org/data/2.5/weather"
    method: "GET"

    response:
      extract: "main"            # Extrai só a seção 'main'
      fields:                     # E só esses campos
        - temp
        - humidity
```

**Resposta original**:

```json
{
  "coord": {...},
  "weather": [...],
  "main": {
    "temp": 25.3,
    "feels_like": 27.1,
    "humidity": 65,
    "pressure": 1013
  }
}
```

**Resposta processada**:

```json
{
  "temp": 25.3,
  "humidity": 65
}
```

---

### Transformação de Dados

```yaml
endpoints:
  buscar_usuarios:
    url: "..."

    response:
      transform: "list"          # Retorna como lista
      map_fields:                # Renomeia campos
        id: "user_id"
        name: "user_name"
        email: "user_email"
```

---

### Validação de Resposta

```yaml
endpoints:
  api_critica:
    url: "..."

    response:
      validate:
        required_fields:         # Campos obrigatórios
          - id
          - status
        status_field: "status"   # Campo que indica sucesso
        success_value: "ok"      # Valor esperado
```

---

## Boas Práticas

### 1. Nomenclatura Descritiva

```yaml
#  BOM
endpoints:
  criar_pedido_erp:              # Claro
  buscar_clima_cidade:
  listar_usuarios_ativos:

#   EVITE
endpoints:
  api1:                          # Vago
  chamada:                       # Genérico
  teste:                         # Temporário
```

---

### 2. Descrições Completas

```yaml
#  BOM
endpoints:
  criar_pedido:
    description: "Cria novo pedido no ERP com validação de
                  estoque e cálculo automático de frete"
    parameters:
      - name: "cliente_id"
        type: "integer"
        description: "ID do cliente (obrigatório)"

#   EVITE
endpoints:
  criar_pedido:
    url: "..."  # Sem descrição
```

---

### 3. Timeout Adequado

```yaml
# APIs rápidas
endpoints:
  cache_lookup:
    timeout: 5               # 5 segundos

# APIs lentas (processamento)
endpoints:
  gerar_relatorio:
    timeout: 120             # 2 minutos

# APIs muito lentas
endpoints:
  ml_predict:
    timeout: 300             # 5 minutos
```

---

### 4. Rate Limiting

```yaml
# Se API tem limite de requisições
endpoints:
  api_limitada:
    url: "..."
    retry_attempts: 10       # Mais tentativas
    retry_delay: 60          # 1 minuto entre tentativas
```

---

### 5. Segurança

```yaml
#  BOM - Chaves no .keys.yaml
auth:
  type: "bearer"
  secret_key: "API_TOKEN"    # Referencia chave

#   NUNCA - Hardcode de secrets
auth:
  type: "bearer"
  token: "sk-abc123..."      #   EXPÕE CHAVE!
```

---

## 🔍 Troubleshooting

### Erro: "Endpoint não encontrado"

**Causa**: Nome do endpoint errado

```yaml
# tools declara:
tools:
  - "dyn_api<buscar_clima>"  # ← Nome esperado

# Mas endpoints tem:
endpoints:
  clima:                     #   Nome diferente!
```

**Solução**: Sincronizar nomes

---

### Erro: "Autenticação falhou (401)"

**Causa**: Token inválido ou expirado

**Soluções**:

1. Verificar chave em .keys.yaml
2. Regenerar token na API
3. Verificar formato de auth

```yaml
# Verificar tipo correto
auth:
  type: "bearer"             #  Para tokens JWT
  # type: "api_key"          #  Para chaves simples
```

---

### Erro: "Timeout"

**Causa**: API muito lenta

**Soluções**:

1. Aumentar timeout

```yaml
endpoints:
  api_lenta:
    timeout: 120             # 2 minutos
```

1. Verificar conectividade
2. Contatar responsável pela API

---

### Erro: "Rate Limit (429)"

**Causa**: Muitas requisições

**Solução**: Configurar retry adequado

```yaml
endpoints:
  api_limitada:
    retry_attempts: 5
    retry_delay: 60          # 1 minuto
```

---

### Erro: "SSL Verification Failed"

**Causa**: Certificado inválido

**Solução** (apenas para dev/testes):

```yaml
api_dynamic:
  verify_ssl: false          # ⚠️ Só em dev!
```

---

## 📞 Suporte e Recursos

### Documentação Relacionada

- [Guia Central de Tools](../GUIA-USUARIO-TOOLS.md)
- [Lista de Ferramentas](alfabetica.md)
- [Por Finalidade](por_finalidade.md)

### O que o código garante hoje

- Resolução de endpoint por `endpoint_id`
- Retry para falhas transitórias de rede
- Timeout configurável
- Cliente `httpx` assíncrono
- Cache de token para autenticação dinâmica suportada

### Evidência no código

- `src/agentic_layer/supervisor/tool_loader.py`
- `src/agentic_layer/supervisor/tools_factory.py`
- `src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py`
- `src/agentic_layer/tools/domain_tools/dynamic_api_tools/http_client.py`
- `src/agentic_layer/tools/domain_tools/dynamic_api_tools/auth_manager.py`

### Lacunas no código

Não encontrado no código:

- catálogo de compatibilidade oficial por provedor HTTP documentado em `docs/tools/`;
- suíte pública de exemplos homologados por provedor externo.

Onde deveria estar:

- `src/agentic_layer/tools/domain_tools/dynamic_api_tools/`
- `docs/tools/`

---

<div align="center">

**[⬆️ Voltar ao Topo](#-api-dinâmica---documentação-completa)** | **[🏠 Página Principal](../README.md)**

</div>
