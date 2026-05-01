**Produto:** Plataforma de Agentes de IA

# Manual de Provisionamento WhatsApp Cloud API

## 📋 Visão Geral

Este manual descreve o sistema de provisionamento automático de números WhatsApp Business através da Meta Graph API, projetado para portais SaaS multi-tenant.

## 🎯 Objetivo

Automatizar 100% do onboarding de números WhatsApp sem interação manual com o portal da Meta, permitindo que clientes ativem números diretamente através da interface web do seu sistema.

---

## 🏗️ Arquitetura

### Componentes Principais

#### 1. `WhatsAppProvisionerAsync`
**Classe de serviço pura** que executa operações na Meta Graph API.

**Características:**
-  Autossuficiente (sem herança ou injeções complexas)
-  Assíncrona (`async/await`)
-  Gerencia conexões HTTP internamente
-  Logging com correlação usando `create_logger_with_correlation`
-  Tratamento robusto de erros HTTP e rede

**Métodos:**
- `register_phone()` - Registra número no WABA
- `request_verification()` - Solicita código SMS/VOICE
- `verify_code()` - Valida código recebido
- `activate_number()` - Ativa para envio de mensagens
- `configure_webhook()` - Configura webhook da aplicação
- `create_default_template()` - Cria template transacional

#### 2. `MultiTenantWhatsAppManager`
**Gerenciador multi-tenant** que carrega configurações do `customers.config`.

**Características:**
-  Lê credenciais do diretório multi-tenant (`ClientDirectory`)
-  Suporta múltiplos clientes SaaS com cache em memória
-  Registra `profile_cache_initialized_at` em UTC para integração com o
    sistema de invalidação global
-  Expõe `start_provision()` e `finalize_provision()` para orquestrar o fluxo
-  `provision_full_flow()` continua disponível como atalho
-  Métodos auxiliares garantem webhook e template em cenários idempotentes

#### 3. `MetaGraphCredentials`
**Dataclass** que armazena credenciais da Meta.

**Campos:**
- `access_token` - Token de longa duração
- `app_id` - ID da aplicação Meta
- `whatsapp_business_account_id` - ID do WABA
- `graph_api_version` - Versão da API (default: v20.0)

#### 4. `ClientPhoneProfile`
**Dataclass** com dados fornecidos pelo cliente.

**Campos:**
- `country_code` - Código do país (ex: "55")
- `phone_number` - Número sem formatação
- `display_name` - Nome exibido no perfil
- `legal_name` - Razão social
- `email` - Email comercial
- `verification_channel` - "SMS" ou "VOICE"
- `timezone` - Fuso horário (default: America/Sao_Paulo)
- `website` - URL do site (opcional)

---

## 📁 Arquivos de Configuração

### `security/customers.config`

Localizado na mesma pasta que `users.config`.

```json
### Uso Básico (Recomendado)

```python
import asyncio
from src.channel_layer.services.whatsapp_meta_onboarding import (
    ClientPhoneProfile,
    MultiTenantWhatsAppManager,
)
from src.security.client_directory import ClientDirectory


async def provisionar_numero() -> None:
    directory = ClientDirectory.instance()
    manager = MultiTenantWhatsAppManager(directory)

    profile = ClientPhoneProfile(
        country_code="55",
        phone_number="11987654321",
        display_name="Meu Negócio",
        legal_name="Meu Negócio LTDA",
        email="contato@meunegocio.com",
    )

    inicio = await manager.start_provision(
        client_code="cliente_demo",
        profile=profile,
    )

    directory.register_whatsapp_phone(
        phone_number_id=inicio["phone_number_id"],
        client_code="cliente_demo",
        phone_e164=inicio["phone_e164"],
        status="pending_verification",
        metadata={"provision_pending": True},
    )

    codigo_sms = input("Informe o código SMS recebido: ")

    resultado = await manager.finalize_provision(
        client_code="cliente_demo",
        phone_number_id=inicio["phone_number_id"],
        phone_e164=inicio["phone_e164"],
        verification_code=codigo_sms,
        idempotency_key="demo-uuid-123",
    )

    print(f" Número ativado: {resultado['phone_e164']}")


asyncio.run(provisionar_numero())
```

---

## 🧭 Jornada de uso (passo a passo)

1. **Pré-requisitos prontos**
   - WABA, token e permissões definidos no `customers.config`.
2. **Início do onboarding (start)**
   - UI chama `/api/whatsapp/provision/start`.
   - Backend registra o número e solicita o código SMS/VOICE.
3. **Cliente recebe o código**
   - Usuário final digita o código no portal.
4. **Conclusão do onboarding (verify)**
   - Backend valida o código, ativa o número e configura webhook.
5. **Validação funcional**
   - Teste de webhook e envio de mensagem de prova.
6. **Operação contínua**
   - Canal entra em produção com rastreio por `correlation_id`.

---

## 🧩 Quando usar

- **Novo número**: quando o cliente ainda não possui número ativo.
- **Migração**: quando o número já existe na Meta e precisa ser vinculado.
- **Takeover**: quando o webhook precisa ser assumido pelo ambiente atual.

---

##  Exemplos de uso

### Exemplo feliz (start + verify)

1. `POST /api/whatsapp/provision/start`
2. Usuário recebe código SMS.
3. `POST /api/whatsapp/provision/verify` com `phone_number_id` válido.
4. Número ativo com webhook configurado.

### Exemplo de erro (código inválido)

- Se o código não confere, o `verify` retorna erro e o número segue como
  `pending_verification`. A orientação é reiniciar a etapa `verify` usando
  o mesmo `phone_number_id` até o SMS correto.

---

## 📌 Sobre o onboarding do Instagram

O fluxo completo de Instagram Direct está documentado em
`docs/README-INSTAGRAM-PROVISIONING.md`.
```

**Backend executa automaticamente:**
```python
resultado = await manager.finalize_provision(
        client_code="cliente_demo",
        phone_number_id="109876543210",
        phone_e164="+5511988888777",
        verification_code="123456",
        idempotency_key=idempotency_key,
)

# resultado = {
#   "phone_number_id": "109876543210",
#   "phone_e164": "+5511988888777",
#   "status": "active",
#   "webhook_configured": True,
#   "template_created": True
# }

directory.register_whatsapp_phone(
        phone_number_id=resultado["phone_number_id"],
        client_code="cliente_demo",
        phone_e164=resultado["phone_e164"],
        status=resultado["status"],
        metadata={
                "webhook_configured": resultado["webhook_configured"],
                "template_created": resultado["template_created"],
                "provision_pending": False,
        "profile_cache_initialized_at": (
            manager.profile_cache_initialized_at.isoformat()
        ),
        }
)
```

### Fluxo B: Trazer Número Já Existente na Meta

1. **Importação (`/import-existing`)** — registra localmente um número já
     ativo, associando `phone_number_id` e metadados ao tenant.
2. **Takeover (`/takeover`)** — assume o webhook para o ambiente atual e
     garante o template padrão (opcional). Também suporta `X-Idempotency-Key`.

**Exemplo de importação:**
```json
POST /api/whatsapp/provision/import-existing
{
    "phone_e164": "+5511988888777",
    "phone_number_id": "109876543210",
    "client_code": "cliente_demo",
    "assume_active": true
}
```

**Exemplo de takeover:**
```json
POST /api/whatsapp/provision/takeover
{
    "phone_e164": "+5511988888777",
    "client_code": "cliente_demo",
    "ensure_template": true
}
```

Em ambos os passos, a UI envia o cabeçalho `X-API-Key` e pode transmitir uma
`X-Idempotency-Key` fixa por operação para evitar efeitos duplicados quando há
replays ou timeouts no navegador.

### 🔄 Reciclando o Cache de Perfis
- Sempre que credenciais ou metadados do cliente forem ajustados no diretório,
    utilize o botão **"🧠 Limpar caches em memória"** no painel administrativo
    para forçar o descarte do cache `profile_cache_initialized_at`.
- O timestamp é comparado com o sinal global publicado pelo endpoint
    `/admin/cache/clear-all`; ao detectar mudança, o manager carrega as
    credenciais novamente antes de prosseguir com o onboarding.

---

## 💻 Exemplos de Código

### Uso Básico (Recomendado)

```python
import asyncio
from src.channel_layer.services.whatsapp_meta_onboarding import (
    ClientPhoneProfile,
    MultiTenantWhatsAppManager,
)
from src.security.client_directory import ClientDirectory

async def provisionar_numero():
    directory = ClientDirectory.instance()

## 📣 Onboarding Instagram Direct (Graph API)

### 🎯 Objetivo
Provisionar contas Instagram Business para receber e enviar mensagens via
Graph API, configurando webhook e registrando a conta no diretório multi-
tenant sem passos manuais na Meta.

###  Pré-requisitos
- Permissão `provision.instagram` na `X-API-Key` usada na chamada.
- IDs Meta: `instagram_business_account_id`, `page_id`, `app_id` e
    `app_secret` válidos com `instagram_manage_messages`.
- `access_token` da página (long-lived recomendado).
- `callback_url` público HTTPS para receber webhooks e `verify_token`
    compartilhado.
- Opcional: `channel_id` para vincular ao `tenant_channels`/YAML.

### 🔄 Fluxo resumido
1. **Chamada `POST /api/instagram/provision/start`** (router
     `instagram_provision_router`): envia payload com IDs, token e webhook.
2. **Validações**: campos saneados (`_sanitize_identifier`, `_validate_url`,
     `_validate_token`) e permissão verificada via `require_permission`.
3. **Configuração Meta** (`InstagramProvisionerAsync`):
     - `fetch_business_profile` lê username/display name.
     - `subscribe_page_to_messages` habilita recebimento de mensagens.
     - `configure_webhook` registra `callback_url` + `verify_token`.
4. **Registro local** (`ClientDirectory.register_instagram_account`): salva
     `instagram_business_account_id`, `page_id`, `ig_username` e status
     `active`, guardando apenas hashes de `access_token`, `app_secret` e
     `verify_token` em `metadata` (segurança).
5. **Resposta**: retorna `success`, `ig_username`, `page_id`, status e
     mensagem de sucesso.

### 📨 Exemplo de requisição
```json
POST /api/instagram/provision/start
Headers:
    X-API-Key: <chave do tenant com provision.instagram>
Body:
{
    "instagram_business_account_id": "17841400000000000",
    "page_id": "112233445566778",
    "page_name": "Plataforma de Agentes de IA Corp",
    "display_name": "Loja Plataforma de Agentes de IA",
    "email": "contato@prometeu.com.br",
    "access_token": "EAAJ...",
    "app_id": "123456789012345",
    "app_secret": "app-secret-met",
    "callback_url": "https://prometeu.com/webhook/instagram",
    "verify_token": "seu-token-secreto",
    "channel_id": "instagram_prometeu_sac",
    "graph_api_version": "v20.0"
}
```

### 🔍 O que é persistido
- Diretório armazena hashes de `access_token`, `app_id|app_secret` e
    `verify_token` para evitar vazamento.
- `metadata` mantém `callback_url`, `graph_api_version`, `page_name` e
    `registered_by` (email do usuário que chamou).
- `channel_id` (quando enviado) facilita casamento com `tenant_channels` e
    escolha do `yaml_path` para webhooks.

### 🧪 Boas práticas
- Reutilize a mesma `X-Idempotency-Key` ao reemitir a requisição em caso de
    timeout para não duplicar registros.
- Após provisionar, teste webhook via rota de teste `/channels/instagram/
    webhook/test` para validar assinatura e entrega.
- Se trocar token/segredo, reenvie o fluxo `start` para atualizar hashes e
    reconfigurar o webhook na Meta.
    manager = MultiTenantWhatsAppManager(directory)

    profile = ClientPhoneProfile(
        country_code="55",
        phone_number="11987654321",
        display_name="Meu Negócio",
        legal_name="Meu Negócio LTDA",
        email="contato@meunegocio.com"
    )

    inicio = await manager.start_provision(
        client_code="cliente_demo",
        profile=profile,
    )

    # Persistir estágio pendente no diretório da sua aplicação
    directory.register_whatsapp_phone(
        phone_number_id=inicio["phone_number_id"],
        client_code="cliente_demo",
        phone_e164=inicio["phone_e164"],
        status="pending_verification",
        metadata={"provision_pending": True}
    )

    codigo_sms = input("Informe o código SMS recebido: ")

    resultado = await manager.finalize_provision(
        client_code="cliente_demo",
        phone_number_id=inicio["phone_number_id"],
        phone_e164=inicio["phone_e164"],
        verification_code=codigo_sms,
        idempotency_key="demo-uuid-123"
    )

    print(f" Número ativado: {resultado['phone_e164']}")

asyncio.run(provisionar_numero())
```

### Uso Avançado (Passo a Passo)

```python
async def onboarding_manual():
    creds = MetaGraphCredentials(
        access_token="EAA...",
        app_id="123456",
        whatsapp_business_account_id="789012"
    )

    async with WhatsAppProvisionerAsync(creds) as svc:
        # Etapa 1: Registrar
        reg = await svc.register_phone(profile)
        phone_id = reg["id"]

        # Etapa 2: Solicitar código
        await svc.request_verification(phone_id)

        # Aguardar input do usuário
        codigo = input("Digite o código SMS: ")

        # Etapa 3: Verificar
        await svc.verify_code(phone_id, codigo)

        # Etapa 4: Ativar
        await svc.activate_number(phone_id)

        # Etapa 5: Webhook
        await svc.configure_webhook(
            "https://meuapp.com/webhook",
            "token123"
        )

        # Etapa 6: Template
        await svc.create_default_template("boas_vindas")
```

---

## 🌐 Integração com Interface Web

### Endpoint FastAPI (Exemplo)

```python
from fastapi import APIRouter, HTTPException, Header
from pydantic import BaseModel

from src.channel_layer.services.whatsapp_meta_onboarding import (
    ClientPhoneProfile,
    MultiTenantWhatsAppManager,
)
from src.security.client_directory import ClientDirectory

router = APIRouter(prefix="/api/whatsapp/provision")


class StartRequest(BaseModel):
    telefone: str
    client_code: str
    channel_id: str | None = None


class VerifyRequest(BaseModel):
    telefone: str
    client_code: str
    phone_number_id: str
    codigo_sms: str
    channel_id: str | None = None


@router.post("/start")
async def iniciar_provisionamento(req: StartRequest):
    directory = ClientDirectory.instance()
    manager = MultiTenantWhatsAppManager(directory)

    profile = ClientPhoneProfile(
        country_code="55",
        phone_number=req.telefone,
        display_name="Nome exibido",
        legal_name="Nome exibido",
        email="contato@cliente.com",
    )

    inicio = await manager.start_provision(
        client_code=req.client_code,
        profile=profile,
    )

    directory.register_whatsapp_phone(
        phone_number_id=inicio["phone_number_id"],
        client_code=req.client_code,
        phone_e164=inicio["phone_e164"],
        status="pending_verification",
        metadata={"provision_pending": True},
    )

    return {
        "success": True,
        "phone_number_id": inicio["phone_number_id"],
        "message": "SMS enviado para o número informado.",
    }


@router.post("/verify")
async def verificar_codigo(
    req: VerifyRequest,
    x_idempotency_key: str | None = Header(default=None),
):
    directory = ClientDirectory.instance()
    manager = MultiTenantWhatsAppManager(directory)

    registro = directory.get_whatsapp_phone_by_e164(
        req.client_code,
        req.telefone,
    )
    if not registro or registro.get("phone_number_id") != req.phone_number_id:
        raise HTTPException(409, "Fluxo inválido. Reinicie a etapa /start.")

    resultado = await manager.finalize_provision(
        client_code=req.client_code,
        phone_number_id=req.phone_number_id,
        phone_e164=registro.get("phone_e164", req.telefone),
        verification_code=req.codigo_sms,
        idempotency_key=x_idempotency_key,
    )

    directory.register_whatsapp_phone(
        phone_number_id=resultado["phone_number_id"],
        client_code=req.client_code,
        phone_e164=resultado["phone_e164"],
        status=resultado.get("status", "active"),
        metadata={
            "webhook_configured": resultado.get("webhook_configured", False),
            "template_created": resultado.get("template_created", False),
            "provision_pending": False,
        },
    )

    return resultado
```

---

## 🔐 Segurança

### Credenciais

-  Armazenadas em `security/customers.config` (fora do git)
-  Nunca expor `access_token` no frontend
-  Usar HTTPS para todas as chamadas

### Webhook

-  Validar `verify_token` em produção
-  Verificar assinatura Meta (`X-Hub-Signature-256`)
-  Usar URL pública com certificado SSL válido

---

## 🧪 Testes

### Teste de Importação

```bash
python -c "from src.channel_layer.services.whatsapp_meta_onboarding import WhatsAppProvisionerAsync; print(' OK')"
```

### Teste com Credenciais de Teste

```python
# Usar número de teste da Meta (não cobra)
# Documentação: https://developers.facebook.com/docs/whatsapp/cloud-api/get-started
```

---

## 📊 Diagrama de Fluxo

```
┌─────────────┐
│   Cliente   │
│  (Browser)  │
└──────┬──────┘
       │
       │ 1. Preenche formulário
       │    (telefone, nome, email)
       ▼
┌─────────────────────────────┐
│  Backend FastAPI            │
│  POST /provision/start      │
└──────┬──────────────────────┘
       │
       │ 2. Chama Meta API
       ▼
┌─────────────────────────────┐
│  WhatsAppProvisionerAsync   │
│  - register_phone()         │
│  - request_verification()   │
└──────┬──────────────────────┘
       │
       │ 3. Meta envia SMS
       ▼
┌─────────────┐
│   Cliente   │
│  Recebe:    │
│  "123456"   │
└──────┬──────┘
       │
       │ 4. Digita código
       ▼
┌─────────────────────────────┐
│  Backend FastAPI            │
│  POST /provision/verify     │
└──────┬──────────────────────┘
       │
       │ 5. Executa fluxo completo
       ▼
┌─────────────────────────────┐
│  MultiTenantWhatsAppManager │
│  - verify_code()            │
│  - activate_number()        │
│  - configure_webhook()      │
│  - create_default_template()│
└──────┬──────────────────────┘
       │
       │ 6. Retorna sucesso
       ▼
┌─────────────┐
│   Cliente   │
│    Ativo  │
└─────────────┘
```

---

## 🚨 Tratamento de Erros

### Erros Comuns

| Erro | Causa | Solução |
|------|-------|---------|
| `httpx.RequestError` | Problema de rede | Retry com backoff exponencial |
| `httpx.HTTPStatusError 400` | Dados inválidos | Validar entrada do usuário |
| `httpx.HTTPStatusError 401` | Token inválido | Renovar `access_token` |
| `httpx.HTTPStatusError 429` | Rate limit | Aguardar e tentar novamente |
| `ValueError` | JSON corrompido | Verificar resposta da API |

### Exemplo de Tratamento

```python
try:
    resultado = await manager.provision_full_flow(...)
except httpx.HTTPStatusError as e:
    if e.response.status_code == 400:
        raise HTTPException(400, "Dados inválidos")
    elif e.response.status_code == 429:
        raise HTTPException(429, "Muitas requisições. Aguarde.")
    else:
        raise HTTPException(500, "Erro na Meta API")
except httpx.RequestError:
    raise HTTPException(503, "Serviço indisponível")
```

---

## 📚 Referências

- [Meta Graph API Docs](https://developers.facebook.com/docs/graph-api/)
- [WhatsApp Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api)
- [httpx Documentation](https://www.python-httpx.org/)

---

##  Checklist de Implementação

- [ ] Configurar `security/customers.config`
- [ ] Obter `access_token` de longa duração
- [ ] Criar endpoints FastAPI
- [ ] Implementar página HTML de provisionamento
- [ ] Configurar webhook público com HTTPS
- [ ] Testar com número de teste da Meta
- [ ] Validar fluxo completo em produção
- [ ] Monitorar logs de erro
- [ ] Implementar retry logic
- [ ] Documentar para equipe de suporte
