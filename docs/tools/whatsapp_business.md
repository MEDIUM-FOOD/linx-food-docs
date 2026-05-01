# 📱 WhatsApp Business API - Guia Completo

## 🎯 Visão Geral

**WhatsApp Business API** permite que empresas integrem o WhatsApp em seus sistemas de atendimento, automação e notificações, alcançando **2+ bilhões de usuários** globalmente.

---

## 📋 Índice

1. [O que é WhatsApp Business API](#o-que-é-whatsapp-business-api)
2. [Diferenças: API vs App](#diferenças-api-vs-app)
3. [Requisitos e Setup](#requisitos-e-setup)
4. [Configuração YAML](#configuração-yaml)
5. [Tipos de Mensagens](#tipos-de-mensagens)
6. [Templates Aprovados](#templates-aprovados)
7. [Webhooks e Eventos](#webhooks-e-eventos)
8. [Exemplos Práticos](#exemplos-práticos)
9. [Compliance e Políticas](#compliance-e-políticas)
10. [Troubleshooting](#troubleshooting)
11. [Onboarding SaaS automatizado](#onboarding-saas-automatizado)

---

## 🤔 O que é WhatsApp Business API?

### Conceito

**WhatsApp Business API** é a interface oficial para empresas de **médio/grande porte** integrarem WhatsApp em seus sistemas, com:

 **Múltiplos usuários** simultâneos
 **Automação** completa com IA
 **Integração** com CRM/sistemas
 **Templates** pré-aprovados pela Meta
 **Webhooks** para eventos em tempo real
 **Métricas** e analytics

### Casos de Uso

1. **Atendimento ao Cliente**
   - Chatbot 24/7
   - Escalação para humanos
   - Histórico centralizado

2. **Notificações Transacionais**
   - Confirmações de pedido
   - Atualizações de entrega
   - Lembretes de pagamento
   - Alertas de sistema

3. **Marketing**
   - Campanhas promocionais
   - Novos produtos

   - Catálogo de produtos
   - Pagamentos integrados
---
## 🔄 Diferenças: API vs App

| Característica | WhatsApp Business App | WhatsApp Business API |
|----------------|----------------------|------------------------|
| **Público** | Pequenos negócios | Médias/grandes empresas |
| **Usuários simultâneos** |   1-4 dispositivos |  Ilimitados |
| **Automação** | ⚠️ Limitada |  Completa |
| **Integração sistemas** |   Não |  Sim |
| **Templates pré-aprovados** | ⚠️ Básicos |  Customizados |
| **Webhooks** |   Não |  Sim |
| **Custo** | 🆓 Grátis | 💰 Pago (conversas) |
| **Aprovação Meta** |   Não requer |  Requer |

---

## 🚀 Requisitos e Setup

### 1. Pré-requisitos

**Documentação necessária:**
-  Facebook Business Manager ativo
-  Conta Meta verificada

### 2. Processo de Setup
**Passo 1:** Criar conta no Meta Business Suite
→ Criar conta empresarial
→ Adicionar informações da empresa
```

**Passo 2:** Registrar número WhatsApp
```
Meta Business Suite → WhatsApp → Começar
→ Adicionar número de telefone
→ Verificar via SMS/chamada
```

**Passo 3:** Escolher provedor
```
Opções:
1. Twilio (mais popular)
2. 360Dialog
3. MessageBird
4. Vonage
5. WhatsApp Cloud API (direto Meta)
```

**Passo 4:** Obter credenciais
```
→ Phone Number ID
→ WhatsApp Business Account ID
→ Access Token (permanente)
```

### 3. Custo (Conversações)

**Modelo de cobrança:** Por conversa de 24 horas

| Tipo | Descrição | Custo (Brasil) |
|------|-----------|----------------|
| **Marketing** | Campanhas promocionais | R$ 0,36/conv |
| **Utilitário** | Transacionais (pedido, entrega) | R$ 0,14/conv |
| **Autenticação** | OTP, códigos 2FA | R$ 0,05/conv |
| **Serviço** | Resposta a cliente (<24h) | 🆓 Grátis |

**⚠️ Janela de 24h:** Após cliente iniciar conversa, empresa tem 24h para responder gratuitamente.

---

## ⚙️ Configuração YAML

### Configuração Básica

```yaml
tools_library:
  - name: "whatsapp_enviar_mensagem"
    type: "whatsapp_business"
    description: "Envia mensagem via WhatsApp Business API"
    config:
      # Credenciais Meta
      provider: "meta_cloud_api"  # ou "twilio", "360dialog"
      phone_number_id: "${WHATSAPP_PHONE_NUMBER_ID}"
      access_token: "${WHATSAPP_ACCESS_TOKEN}"
      business_account_id: "${WHATSAPP_BUSINESS_ACCOUNT_ID}"

      # Configurações
      webhook_url: "https://api.empresa.com/webhooks/whatsapp"
      webhook_verify_token: "${WEBHOOK_VERIFY_TOKEN}"

      # Limites (rate limiting)
      max_messages_per_second: 80  # Limite Meta Cloud API
      max_messages_per_day: 100000



tools_library:
    type: "whatsapp_business"
    config:
      provider: "twilio"
      account_sid: "${TWILIO_ACCOUNT_SID}"
      auth_token: "${TWILIO_AUTH_TOKEN}"
      from_number: "whatsapp:+5511999999999"

      # Webhook Twilio
      webhook_url: "https://api.empresa.com/webhooks/twilio"
```

---

## 💬 Tipos de Mensagens

### 1. Mensagem de Texto Simples

```yaml
- name: "enviar_texto"
  type: "whatsapp_business"
  config:
    action: "send_message"
    message_type: "text"
    parameters:
      to: "{{ cliente_telefone }}"  # +5511999999999
      text: |
        Olá {{ cliente_nome }}! 👋

        Seu pedido #{{ numero_pedido }} foi confirmado!
        Previsão de entrega: {{ data_entrega }}
```

---

### 2. Mensagem com Template Aprovado

```yaml
- name: "enviar_confirmacao_pedido"
  type: "whatsapp_business"
  config:
    action: "send_template"
    template_name: "pedido_confirmado"
    language: "pt_BR"
    parameters:
      to: "{{ cliente_telefone }}"
      components:
        - type: "body"
          parameters:
            - type: "text"
              text: "{{ cliente_nome }}"
            - type: "text"
              text: "{{ numero_pedido }}"
            - type: "text"
              text: "{{ valor_total }}"

        - type: "button"
          sub_type: "url"
          index: 0
          parameters:
            - type: "text"
              text: "{{ numero_pedido }}"  # Para URL dinâmica
```

**Template no Meta (pré-aprovado):**
```
Olá {{1}}! 🎉

Seu pedido #{{2}} foi confirmado!
Valor: R$ {{3}}

Acompanhe seu pedido:
[Rastrear Pedido] → https://empresa.com/pedido/{{button_param}}
```

---

### 3. Mensagem com Imagem

```yaml
- name: "enviar_comprovante"
  type: "whatsapp_business"
  config:
    action: "send_message"
    message_type: "image"
    parameters:
      to: "{{ cliente_telefone }}"
      image:
        link: "https://cdn.empresa.com/comprovantes/{{ id }}.jpg"
        caption: "Comprovante do seu pedido #{{ numero_pedido }}"
```

---

### 4. Mensagem com Documento (PDF)

```yaml
- name: "enviar_boleto"
  type: "whatsapp_business"
  config:
    action: "send_message"
    message_type: "document"
    parameters:
      to: "{{ cliente_telefone }}"
      document:
        link: "https://cdn.empresa.com/boletos/{{ boleto_id }}.pdf"
        filename: "Boleto_{{ numero_pedido }}.pdf"
        caption: "Seu boleto está anexado 👆"
```

---

### 5. Mensagem com Botões Interativos

```yaml
- name: "menu_atendimento"
  type: "whatsapp_business"
  config:
    action: "send_message"
    message_type: "interactive"
    sub_type: "button"
    parameters:
      to: "{{ cliente_telefone }}"
      body: |
        Olá! Como posso ajudar?

        Escolha uma opção:

      buttons:
        - id: "rastrear_pedido"
          title: "📦 Rastrear Pedido"

        - id: "falar_atendente"
          title: "👤 Falar com Atendente"

        - id: "ver_promocoes"
          title: "🎁 Ver Promoções"
```

---

### 6. Mensagem com Lista (até 10 itens)

```yaml
- name: "listar_produtos"
  type: "whatsapp_business"
  config:
    action: "send_message"
    message_type: "interactive"
    sub_type: "list"
    parameters:
      to: "{{ cliente_telefone }}"
      body: "Confira nossos produtos em promoção!"
      button: "Ver Produtos"

      sections:
        - title: "Eletrônicos"
          rows:
            - id: "prod_001"
              title: "Notebook Dell"
              description: "R$ 3.499 - 20% OFF"

            - id: "prod_002"
              title: "Mouse Logitech"
              description: "R$ 149 - 15% OFF"

        - title: "Informática"
          rows:
            - id: "prod_003"
              title: "Teclado Mecânico"
              description: "R$ 499 - 10% OFF"
```

---

### 7. Mensagem com Localização

```yaml
- name: "enviar_localizacao_loja"
  type: "whatsapp_business"
  config:
    action: "send_message"
    message_type: "location"
    parameters:
      to: "{{ cliente_telefone }}"
      location:
        latitude: -23.5505
        longitude: -46.6333
        name: "Loja Centro SP"
        address: "Av. Paulista, 1000 - São Paulo"
```

---

### 8. Mensagem com Contato

```yaml
- name: "enviar_contato_vendedor"
  type: "whatsapp_business"
  config:
    action: "send_message"
    message_type: "contact"
    parameters:
      to: "{{ cliente_telefone }}"
      contacts:
        - name:
            formatted_name: "João Silva - Vendedor"
            first_name: "João"
            last_name: "Silva"
          phones:
            - phone: "+5511988887777"
              type: "WORK"
          emails:
            - email: "joao.silva@empresa.com"
              type: "WORK"
```

---

## 📝 Templates Aprovados

### Como Funcionam

1. **Criar template** no Meta Business Suite
2. **Aguardar aprovação** (24-48h)
3. **Usar no sistema** via template_name

### Categorias de Templates

**1. Utilitário (Transacionais):**
-  Confirmação de pedido
-  Atualização de entrega
-  Lembrete de pagamento
-  Confirmação de reserva

**2. Marketing:**
-  Promoções
-  Lançamentos
-  Eventos

**3. Autenticação:**
-  Códigos OTP
-  Verificação 2FA

---

### Exemplo: Template de Confirmação de Entrega

**Nome:** `entrega_realizada`
**Categoria:** Utilitário
**Idioma:** Português (Brasil)

**Corpo:**
```
Olá {{1}}! ✅

Seu pedido #{{2}} foi entregue com sucesso!

Data/Hora: {{3}}
Recebido por: {{4}}

Avalie sua experiência:
```

**Botões:**
```
[⭐ Avaliar Entrega] → https://empresa.com/avaliar/{{button_param}}
[📞 Suporte] → Botão de telefone
```

**Uso no YAML:**
```yaml
- name: "notificar_entrega"
  type: "whatsapp_business"
  config:
    action: "send_template"
    template_name: "entrega_realizada"
    language: "pt_BR"
    parameters:
      to: "{{ cliente_telefone }}"
      components:
        - type: "body"
          parameters:
            - type: "text"
              text: "{{ cliente_nome }}"
            - type: "text"
              text: "{{ numero_pedido }}"
            - type: "text"
              text: "{{ data_hora_entrega }}"
            - type: "text"
              text: "{{ nome_recebedor }}"

        - type: "button"
          sub_type: "url"
          index: 0
          parameters:
            - type: "text"
              text: "{{ pedido_id }}"
```

---

## 🔔 Webhooks e Eventos

### Configurar Webhook

```yaml
webhooks:
  whatsapp:
    url: "https://api.empresa.com/webhooks/whatsapp"
    verify_token: "${WEBHOOK_VERIFY_TOKEN}"

    # Eventos subscritos
    events:
      - "messages"        # Mensagens recebidas
      - "message_status"  # Status de entrega
      - "message_template_status_update"  # Status templates
```

---

### Eventos Recebidos

**1. Mensagem Recebida:**
```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "changes": [{
      "value": {
        "messages": [{
          "from": "5511999999999",
          "id": "wamid.xxx",
          "timestamp": "1234567890",
          "type": "text",
          "text": {
            "body": "Oi, quero fazer um pedido"
          }
        }]
      }
    }]
  }]
}
```

**Processar no YAML:**
```yaml
- name: "processar_mensagem_recebida"
  type: "webhook_handler"
  config:
    event_type: "whatsapp_message"

    # Extrair dados
    extract:
      cliente_telefone: "entry[0].changes[0].value.messages[0].from"
      mensagem: "entry[0].changes[0].value.messages[0].text.body"
      timestamp: "entry[0].changes[0].value.messages[0].timestamp"

    # Ações
    actions:
      # Buscar histórico do cliente
      - name: "buscar_cliente"
        type: "sql_tool_v2"
        config:
          query: |
            SELECT * FROM clientes
            WHERE telefone = :telefone

      # Passar para agente IA
      - name: "responder_cliente"
        type: "agentic_supervisor"
        config:
          system_prompt: |
            Você é um atendente virtual.
            Cliente: {{ cliente_nome }}
            Histórico: {{ historico }}

            Responda a mensagem: "{{ mensagem }}"
```

---

**2. Status de Entrega:**
```json
{
  "entry": [{
    "changes": [{
      "value": {
        "statuses": [{
          "id": "wamid.xxx",
          "status": "delivered",  // sent, delivered, read, failed
          "timestamp": "1234567890",
          "recipient_id": "5511999999999"
        }]
      }
    }]
  }]
}
```

**Status possíveis:**
- `sent` - Enviada para WhatsApp
- `delivered` - Entregue ao dispositivo
- `read` - Lida pelo destinatário
- `failed` - Falha no envio

---

**3. Botão Clicado:**
```json
{
  "entry": [{
    "changes": [{
      "value": {
        "messages": [{
          "from": "5511999999999",
          "type": "interactive",
          "interactive": {
            "type": "button_reply",
            "button_reply": {
              "id": "rastrear_pedido",
              "title": "📦 Rastrear Pedido"
            }
          }
        }]
      }
    }]
  }]
}
```

---

## 💡 Exemplos Práticos

### Exemplo 1: Confirmação de Pedido Automática

```yaml
multi_agents:
  - id: "supervisor"
    system_prompt: |
      Ao confirmar pedido, envie notificação WhatsApp.

tools_library:
  # Confirmar pedido no banco
  - name: "confirmar_pedido"
    type: "sql_tool_v2"
    config:
      query: |
        UPDATE pedidos
        SET status = 'confirmado',
            data_confirmacao = NOW()
        WHERE id = :pedido_id

  # Buscar dados do pedido
  - name: "buscar_dados_pedido"
    type: "sql_tool_v2"
    config:
      query: |
        SELECT
          p.id,
          p.numero_pedido,
          p.valor_total,
          p.data_prevista_entrega,
          c.nome as cliente_nome,
          c.telefone
        FROM pedidos p
        JOIN clientes c ON p.id_cliente = c.id
        WHERE p.id = :pedido_id

  # Enviar WhatsApp
  - name: "notificar_whatsapp"
    type: "whatsapp_business"
    config:
      action: "send_template"
      template_name: "pedido_confirmado"
      language: "pt_BR"
      parameters:
        to: "{{ telefone }}"
        components:
          - type: "body"
            parameters:
              - type: "text"
                text: "{{ cliente_nome }}"
              - type: "text"
                text: "{{ numero_pedido }}"
              - type: "text"
                text: "{{ valor_total }}"
              - type: "text"
                text: "{{ data_prevista_entrega }}"
```

**Resultado:**
```
👤 Cliente faz pedido no site

🤖 Sistema:
1. Registra pedido
2. Confirma no banco
3. Busca dados
4. Envia WhatsApp automaticamente

📱 Cliente recebe em 5 segundos:
"Olá Maria! 🎉
Seu pedido #45821 foi confirmado!
Valor: R$ 249,90
Previsão entrega: 05/10/2025
[Rastrear Pedido]"
```

---

### Exemplo 2: Chatbot de Atendimento 24/7

```yaml
multi_agents:
  - id: "supervisor"
    system_prompt: |
      Você é um assistente virtual de e-commerce via WhatsApp.

      Você pode:
      • Rastrear pedidos
      • Consultar produtos
      • Processar devoluções
      • Esclarecer dúvidas

      Se não souber responder, escale para atendente humano.

tools_library:
  # Receber mensagem WhatsApp
  - name: "webhook_whatsapp"
    type: "webhook_handler"
    config:
      event_type: "whatsapp_message"
      extract:
        telefone: "{{ from }}"
        mensagem: "{{ text.body }}"

  # Buscar cliente
  - name: "buscar_cliente"
    type: "sql_tool_v2"
    config:
      query: |
        SELECT id, nome, email, telefone
        FROM clientes
        WHERE telefone = :telefone

  # Buscar pedidos do cliente
  - name: "buscar_pedidos"
    type: "sql_tool_v2"
    config:
      query: |
        SELECT numero_pedido, status, valor_total, data_pedido
        FROM pedidos
        WHERE id_cliente = :cliente_id
        ORDER BY data_pedido DESC
        LIMIT 5

  # Consultar rastreamento
  - name: "rastrear_pedido"
    type: "api_call"
    config:
      url: "https://api.correios.com/rastreio/{{ codigo_rastreio }}"
      method: "GET"
      headers:
        Authorization: "Bearer ${CORREIOS_API_KEY}"

  # Responder no WhatsApp
  - name: "responder_whatsapp"
    type: "whatsapp_business"
    config:
      action: "send_message"
      message_type: "text"
      parameters:
        to: "{{ telefone }}"
        text: "{{ resposta_ia }}"

  # Transferir para humano
  - name: "escalar_atendente"
    type: "human_gate"
    config:
      approvers: ["atendimento@empresa.com"]
      notification_channels:
        - type: "slack"
          channel: "#atendimento-whatsapp"
          message: |
            🆘 Cliente precisa de atendente

            Cliente: {{ cliente_nome }}
            Telefone: {{ telefone }}
            Mensagem: "{{ mensagem }}"
            Histórico: {{ historico_conversa }}
```

**Fluxo:**
```
📱 Cliente: "Oi, quero rastrear meu pedido"

🤖 Bot:
1. Busca cliente por telefone
2. Busca últimos pedidos
3. Responde: "Olá Maria! Vi que você tem 2 pedidos:
   #45821 - Em transporte 📦
   #44103 - Entregue ✅
   Qual deseja rastrear?"

📱 Cliente: "O 45821"

🤖 Bot:
1. Consulta API Correios
2. Responde: "Seu pedido #45821 está a caminho!
   Status: Em rota de entrega
   Previsão: Hoje até 18h
   Última atualização: 14:32"

📱 Cliente: "Preciso trocar o endereço"

🤖 Bot:
1. Detecta que não consegue resolver
2. Escala para humano
3. Responde: "Entendo! Vou te conectar com um atendente.
   Aguarde um momento 😊"

📧 Atendente recebe notificação Slack
👤 Assume conversa no WhatsApp
```

---

### Exemplo 3: Campanhas de Marketing

```yaml
workflows:
  - name: "campanha_black_friday"
    description: "Enviar ofertas via WhatsApp"

    steps:
      # 1. Buscar clientes elegíveis
      - name: "buscar_clientes_campanha"
        type: "sql_tool_v2"
        config:
          query: |
            SELECT
              c.nome,
              c.telefone,
              c.categoria_cliente,
              COALESCE(SUM(p.valor_total), 0) as total_gasto
            FROM clientes c
            LEFT JOIN pedidos p ON c.id = p.id_cliente
            WHERE c.opt_in_marketing = 1
              AND c.telefone IS NOT NULL
              AND p.data_pedido >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
            GROUP BY c.id
            HAVING total_gasto > 500
            ORDER BY total_gasto DESC

      # 2. Segmentar por valor
      - name: "segmentar_clientes"
        type: "pandas_filter_rows"
        config:
          segments:
            - name: "vip"
              filter: "total_gasto >= 2000"
              desconto: "30%"

            - name: "premium"
              filter: "total_gasto >= 1000 and total_gasto < 2000"
              desconto: "20%"

            - name: "regular"
              filter: "total_gasto < 1000"
              desconto: "15%"

      # 3. Enviar mensagens personalizadas
      - name: "enviar_campanha"
        type: "whatsapp_business"
        config:
          action: "send_template"
          template_name: "campanha_black_friday"
          language: "pt_BR"

          # Para cada segmento
          for_each: "{{ segmentos }}"

          parameters:
            to: "{{ cliente_telefone }}"
            components:
              - type: "body"
                parameters:
                  - type: "text"
                    text: "{{ cliente_nome }}"
                  - type: "text"
                    text: "{{ desconto }}"

              - type: "button"
                sub_type: "url"
                index: 0
                parameters:
                  - type: "text"
                    text: "{{ cupom_desconto }}"

          # Rate limiting
          rate_limit: 10  # 10 mensagens/segundo
          batch_size: 1000
```

**Template:**
```
🔥 BLACK FRIDAY EXCLUSIVA! 🔥

Olá {{1}}!

Como cliente especial, você ganhou {{2}} OFF
em TUDO do site!

Cupom: {{button_param}}
Válido até: 30/11/2025

[Aproveitar Agora] → https://loja.com/bf/{{button_param}}
```

**Resultado:**
-  10.000 clientes contatados
-  Taxa de abertura: 98% (WhatsApp)
-  Taxa de conversão: 12%
-  ROI: 450%

---

### Exemplo 4: Lembretes Automáticos

```yaml
workflows:
  - name: "lembretes_pagamento"
    schedule: "0 9 * * *"  # Todos os dias às 9h

    steps:
      # Buscar boletos vencendo
      - name: "buscar_boletos_vencer"
        type: "sql_tool_v2"
        config:
          query: |
            SELECT
              b.id,
              b.numero_boleto,
              b.valor,
              b.data_vencimento,
              c.nome,
              c.telefone
            FROM boletos b
            JOIN clientes c ON b.id_cliente = c.id
            WHERE b.status = 'pendente'
              AND b.data_vencimento BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 3 DAY)

      # Enviar lembrete
      - name: "enviar_lembrete"
        type: "whatsapp_business"
        config:
          action: "send_template"
          template_name: "lembrete_pagamento"
          language: "pt_BR"

          for_each: "{{ boletos }}"

          parameters:
            to: "{{ telefone }}"
            components:
              - type: "body"
                parameters:
                  - type: "text"
                    text: "{{ nome }}"
                  - type: "text"
                    text: "{{ valor }}"
                  - type: "text"
                    text: "{{ data_vencimento }}"

              - type: "button"
                sub_type: "url"
                index: 0
                parameters:
                  - type: "text"
                    text: "{{ numero_boleto }}"
```

---

## ⚖️ Compliance e Políticas

### Políticas Meta (Obrigatórias)

**1. Opt-in Obrigatório:**
```yaml
# Registrar consentimento
- name: "registrar_opt_in"
  type: "sql_tool_v2"
  config:
    query: |
      INSERT INTO consentimentos
      (id_cliente, canal, data_opt_in, ip, origem)
      VALUES (:id, 'whatsapp', NOW(), :ip, :origem)
```

**2. Opt-out Facilitado:**
```yaml
# Processar SAIR/PARAR
- name: "processar_opt_out"
  type: "webhook_handler"
  config:
    condition: "mensagem in ['SAIR', 'PARAR', 'STOP']"
    actions:
      - type: "sql_tool_v2"
        query: |
          UPDATE clientes
          SET opt_in_marketing = 0
          WHERE telefone = :telefone

      - type: "whatsapp_business"
        action: "send_message"
        text: "Você foi descadastrado. Para voltar, envie SIM."
```

**3. Horários Permitidos:**
```yaml
# Não enviar em horários inadequados
config:
  allowed_hours:
    start: "08:00"
    end: "20:00"

  timezone: "America/Sao_Paulo"
```

**4. Limite de Mensagens:**
```yaml
config:
  max_marketing_messages_per_user_per_day: 3
```

---

### LGPD (Lei Geral de Proteção de Dados)

```yaml
# Dados armazenados
lgpd_compliance:
  dados_coletados:
    - telefone
    - nome
    - mensagens (histórico)

  finalidade: "Atendimento ao cliente e notificações transacionais"

  base_legal: "Consentimento e legítimo interesse"

  retencao: "12 meses após último contato"

  direitos:
    - acesso: "Cliente pode solicitar dados"
    - retificacao: "Cliente pode corrigir dados"
    - exclusao: "Cliente pode excluir dados"
```

---

## 🔧 Troubleshooting

### Problema 1: Mensagem Não Entregue

**Sintomas:** Status `failed`

**Causas possíveis:**
1. Número inválido/não existe
2. Usuário bloqueou empresa
3. Template não aprovado
4. Limite de taxa excedido

**Solução:**
```yaml
- name: "tratar_falha_envio"
  type: "error_handler"
  config:
    on_error: "whatsapp_send_failed"

    actions:
      # Verificar motivo
      - type: "log_error"
        message: "Falha WhatsApp: {{ error_code }} - {{ error_message }}"

      # Se número inválido
      - condition: "error_code == 131026"  # Número inválido
        then:
          - type: "sql_tool_v2"
            query: |
              UPDATE clientes
              SET telefone_invalido = 1
              WHERE telefone = :telefone

      # Tentar canal alternativo
      - type: "send_email"
        fallback: true
```

---

### Problema 2: Template Rejeitado

**Sintomas:** Template não aprovado pela Meta

**Causas:**
- Linguagem promocional muito agressiva
- Falta de contexto claro
- Botões inadequados
- Violação de políticas

**Solução:**
```
  Ruim:
"🔥 COMPRE JÁ! 90% OFF! ÚLTIMA CHANCE!"

 Bom:
"Olá {{1}}! Sua promoção personalizada está disponível.
Desconto de {{2}} em produtos selecionados.
Válido até: {{3}}"
```

---

### Problema 3: Webhooks Não Recebidos

**Causa:** URL inacessível ou resposta lenta

**Solução:**
```yaml
webhooks:
  whatsapp:
    # Responder RÁPIDO (< 5 segundos)
    response_timeout: 5000  # ms

    # Processar em background
    async: true
    queue: "whatsapp_events"

    # Retry automático
    retry:
      max_attempts: 3
      backoff: "exponential"
```

---

### Problema 4: Custos Altos

**Causa:** Muitas mensagens marketing/utilitárias pagas

**Soluções:**
```yaml
# 1. Maximizar janela gratuita 24h
- name: "responder_dentro_janela"
  config:
    condition: "time_since_last_message < 86400"  # 24h
    use_free_window: true

# 2. Usar autenticação (mais barata)
- name: "enviar_otp"
  config:
    message_type: "authentication"  # R$ 0,05 vs R$ 0,36

# 3. Agrupar notificações
- name: "agrupar_notificacoes"
  config:
    batch_notifications: true
    batch_interval: 3600  # 1 hora

    template: |
      Olá! Você tem {{ count }} atualizações:

      {{ lista_atualizacoes }}
```

---

## 📊 Métricas e Analytics

```yaml
analytics:
  whatsapp:
    metrics:
      - name: "taxa_entrega"
        formula: "delivered / sent"
        target: "> 95%"

      - name: "taxa_leitura"
        formula: "read / delivered"
        target: "> 80%"

      - name: "tempo_resposta"
        formula: "AVG(response_time)"
        target: "< 5 min"

      - name: "taxa_conversao"
        formula: "conversions / messages_sent"
        target: "> 10%"

      - name: "custo_por_conversao"
        formula: "total_cost / conversions"
        target: "< R$ 5"
```

---

## 🧭 Onboarding SaaS automatizado

### 1. Visão geral do fluxo multi-tenant

Quando a Plataforma de Agentes de IA é oferecido como SaaS, cada cliente informa o
telefone corporativo diretamente no portal e espera começar a disparar
mensagens imediatamente. O objetivo deste fluxo é automatizar tudo que a
Meta exige para liberar o número:

1. Registrar o telefone dentro do **WhatsApp Business Account (WABA)**
   principal da plataforma SaaS.
2. Disparar e validar o **código de verificação** enviado ao aparelho.
3. Associar o número ao **aplicativo Cloud API** do provedor.
4. Configurar **webhook** apontando para o endpoint Multicanal já
   existente na Plataforma de Agentes de IA.
5. Criar um **template transacional** padrão para homologação e testes.

### 2. Pré-requisitos obrigatórios

- Business Manager verificado, com o WABA que será compartilhado entre
  os clientes.
- Aplicativo Cloud API publicado, com as permissões
  `whatsapp_business_management`, `business_management` e
  `pages_messaging`.
- Token permanente de sistema (System User) que mantenha as permissões
  citadas.
- URL pública e segura (HTTPS) para o webhook já desenvolvido no produto.
- Arquivo YAML do tenant atualizado com `security_keys.whatsapp` (mesmo
  que temporariamente vazio) para receber as credenciais do novo número.

### 3. Roteiro operacional (automação oficial)

O arquivo `src/channel_layer/services/whatsapp_meta_onboarding.py`
implementa todo o fluxo usando a Graph API. A execução é via CLI e deve
ocorrer dentro do ambiente virtual (`.venv`) do projeto.

```bash
source .venv/bin/activate
python -m src.channel_layer.services.whatsapp_meta_onboarding \
  --yaml-config app/yaml/rag-config.yaml \
  --access-token "$META_SYSTEM_USER_TOKEN" \
  --app-id "$META_APP_ID" \
  --business-account-id "$META_BUSINESS_ID" \
  --whatsapp-business-account-id "$META_WABA_ID" \
  register-phone \
  --country-code 55 \
  --phone-number 11987654321 \
  --display-name "Loja XPTO" \
  --legal-name "Loja XPTO LTDA" \
  --email atendimento@lojaxpto.com.br
```

1. **register-phone** → retorna `phone_number_id` e o status inicial.
2. **request-code** → `python ... request-code --phone-number-id <ID>`
   dispara SMS/voz para o aparelho.
3. **verify-phone** → confirma o código recebido.
4. **activate** → associa o número ao produto WhatsApp Cloud e libera o
   envio de mensagens.
5. **configure-webhook** → aponta o callback da SaaS para receber eventos
   (`messages`, `message_template_status` e outros campos desejados).
6. **update-profile** (opcional) → ajusta texto "Sobre", email e links
   exibidos aos clientes finais.
7. **create-template** (opcional) → registra template transacional inicial
   capaz de acionar a jornada de boas-vindas.

Todos os comandos retornam JSON estruturado. Guarde os campos
`phone_number_id`, `certificate` (quando existir) e qualquer identificador
adicional, pois serão usados na configuração do tenant.

### 4. Persistindo credenciais na Plataforma de Agentes de IA

Após a ativação, valorize as chaves no YAML do cliente conforme o modelo
abaixo. Isso garante que os workflows e a camada de canais consigam usar
o novo número.

```yaml
security_keys:
  whatsapp:
    channels:
      atendimento_whatsapp:
        access_token: "${WHATSAPP_ACCESS_TOKEN_CLIENTE}"
        phone_number_id: "${WHATSAPP_PHONE_NUMBER_ID}"
        api_version: "v20.0"
        waba_id: "${META_WABA_ID}"
        business_account_id: "${META_BUSINESS_ID}"
        webhook_verify_token: "token-seguro-definido-no-cli"
```

> **Boas práticas:** mantenha tokens e IDs em `security/keys/<tenant>.json`
> ou em um cofre seguro, e injete referências criptografadas no YAML
> principal.

### 5. Checklist final para o consultor

- Validar que o webhook recebeu o evento `messages` de teste enviado pelo
  CLI ou pela própria Meta.
- Enviar mensagem utilizando o template criado para confirmar aprovação
  imediata.
- Atualizar a documentação de handover do cliente com o `phone_number_id`
  e o `verify_token` utilizados.
- Habilitar o fluxo de atendimento na Plataforma de Agentes de IA, apontando o canal
  `atendimento_whatsapp` (ou ID equivalente) nos workflows ativos.
- Orientar o cliente sobre o SLA de aprovação de novos templates e sobre
  o modelo de cobrança por sessão de conversa definidos pela Meta.

## 📚 Recursos Adicionais

### Documentação Oficial

- 📘 **Meta Cloud API:** https://developers.facebook.com/docs/whatsapp/cloud-api
- 📗 **Twilio API:** https://www.twilio.com/docs/whatsapp
- 📙 **Políticas Meta:** https://www.whatsapp.com/legal/commerce-policy

### Ferramentas

- **Postman Collection:** Templates API prontos
- **WhatsApp Business Manager:** https://business.facebook.com
- **Template Tester:** Testar templates antes de submeter

### Casos de Sucesso

**🏪 E-commerce:**
- Taxa abertura: 98% (vs 20% email)
- Taxa conversão: 12% (vs 2% email)
- ROI: 450%

**🏥 Healthcare:**
- Lembretes consulta: 85% comparecimento (vs 60%)
- Satisfação: 4.7/5.0
- Redução no-shows: 40%

**🏦 Bancos:**
- Autenticação 2FA: 99.2% entrega
- Resolução fraudes: 3x mais rápido
- Satisfação: 4.8/5.0

---

## 🎯 Conclusão

**WhatsApp Business API** é essencial para:
-  Alcançar clientes onde eles estão (2+ bilhões)
-  Automatizar atendimento 24/7
-  Taxas de abertura/conversão muito superiores
-  Experiência mobile-first
-  Integração completa com sistemas

**Comece:**
1. Configure conta no Meta Business
2. Escolha provedor (Twilio/Cloud API)
3. Crie templates
4. Integre com seu YAML
5. Monitore métricas

---

**🚀 Próximo Passo:** Configure seu primeiro chatbot WhatsApp!

## Evidência no código

- `src/channel_layer/services/whatsapp_meta_onboarding.py`

## Lacunas no código

Não encontrado no código:
- manual técnico dedicado da tool `whatsapp_business` dentro de `src/agentic_layer/tools/` com contrato YAML versionado nesta mesma granularidade;
- documento de caso de uso em `docs/casos_uso/atendimento_cliente.md`.

Onde deveria estar:
- `src/agentic_layer/tools/`
- `docs/casos_uso/`

**📧 Suporte:** Consulte [README-HUMAN-IN-THE-LOOP](../README-HUMAN-IN-THE-LOOP.md) e o [guia central de tools](../GUIA-USUARIO-TOOLS.md).
