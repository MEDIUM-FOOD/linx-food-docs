# Manual de Provisionamento e Operação Instagram Direct

## 1. O que esta feature faz

Este manual explica como a plataforma provisiona contas Instagram Business, configura o webhook do aplicativo e registra o canal no diretório central do tenant.

O fluxo real confirmado no código é objetivo: o backend valida o payload, chama a Instagram Graph API para ler o perfil, assinar a página para mensagens e configurar o webhook, e por fim grava a conta no `ClientDirectory` com os metadados necessários para operação posterior.

## 2. Que problema ela resolve

Sem esse módulo, o onboarding de Instagram Direct dependeria de configuração manual espalhada entre Meta, portal interno e diretório multi-tenant. Isso gera risco operacional em três pontos:

1. Credencial correta, mas webhook apontando para o ambiente errado.
2. Conta ativa na Meta, mas não registrada no diretório da plataforma.
3. Segredo e token usados em produção sem rastreabilidade local.

O fluxo existe para transformar esse onboarding em um processo repetível, auditável e isolado por tenant.

## 3. Componentes principais

### 3.1 Serviço de onboarding Instagram

`InstagramProvisionerAsync` encapsula as chamadas externas para a Graph API.

Os comportamentos confirmados no serviço são:

1. Validar presença de `access_token`, `app_id` e `app_secret` já na inicialização.
2. Abrir um cliente HTTP com base URL na versão configurada da Graph API.
3. Ler o perfil principal da conta com `fetch_business_profile`.
4. Assinar a página em mensagens com `subscribe_page_to_messages`.
5. Configurar o webhook do aplicativo com `configure_webhook`.
6. Tratar falhas HTTP e de rede com logging correlacionado.

O papel desse serviço é exclusivamente integrar com a Meta. Ele não decide autenticação do usuário, persistência local ou regra de tenant.

### 3.2 Router público de provisionamento

O boundary HTTP fica em `instagram_provision_router`.

Ele concentra:

1. Permissão `PROVISION_INSTAGRAM`.
2. Validação do payload com `InstagramProvisionStartRequest`.
3. Chamada ao `InstagramProvisionerAsync`.
4. Registro da conta no `ClientDirectory`.
5. Tradução de erros externos em respostas HTTP operacionais.

## 4. Conceitos que importam para entender

### 4.1 `instagram_business_account_id`

É o identificador técnico da conta Instagram Business que será associada ao canal da plataforma.

### 4.2 `page_id`

É a página Facebook associada à conta Instagram. O código confirma que a assinatura de mensagens é feita no escopo da página, não diretamente no identificador da conta Instagram.

### 4.3 `callback_url` e `verify_token`

Esses dois valores governam o webhook.

1. `callback_url` precisa ser URL HTTP ou HTTPS válida.
2. `verify_token` precisa cumprir comprimento mínimo e é usado no handshake de verificação.

### 4.4 `channel_id`

É o identificador lógico do canal dentro da plataforma. Ele permite associar a conta provisionada a um canal de operação e ao YAML correto do tenant.

## 5. Fluxo real de provisionamento

O endpoint público confirmado é `POST /api/instagram/provision/start`.

O fluxo executável segue esta ordem:

1. O router valida campos como IDs, email, token, URL de callback, segredo e versão da API.
2. O `client_code` é resolvido a partir do contexto autenticado em `user_data`.
3. O serviço de onboarding lê o perfil da conta Instagram Business.
4. O serviço assina a página para mensagens.
5. O serviço configura o webhook do aplicativo.
6. O router registra a conta no `ClientDirectory`.
7. A resposta HTTP devolve sucesso, identificadores principais e status ativo.

## 6. O que o payload precisa trazer

Os campos obrigatórios confirmados no schema do router são:

1. `instagram_business_account_id`.
2. `page_id`.
3. `display_name`.
4. `email`.
5. `access_token`.
6. `app_id`.
7. `app_secret`.
8. `callback_url`.
9. `verify_token`.

Os campos opcionais confirmados são:

1. `page_name`.
2. `channel_id`.
3. `graph_api_version`, com default operacional `v20.0`.

## 7. Persistência local e segurança

Depois do provisionamento externo, o router grava a conta no `ClientDirectory`.

O ponto de segurança mais importante é este: o metadata persistido não guarda os segredos em claro. O código calcula hash do `access_token`, do token de aplicação e do `verify_token` antes de registrar o canal.

Na prática, isso reduz exposição operacional sem perder rastreabilidade de qual material credencial foi usado no onboarding.

## 8. O que sai na resposta de sucesso

O contrato de resposta confirmado no router devolve:

1. `success`.
2. `instagram_business_account_id`.
3. `ig_username`.
4. `page_id`.
5. `status`.
6. `message`.

Essa resposta é suficiente para a camada de portal saber que a conta foi provisionada e qual identificador ficou ativo na plataforma.

## 9. O que acontece em caso de erro

Os erros confirmados no código se dividem assim:

### 9.1 Erro de validação do payload

Se identificador, URL, token ou email forem inválidos, o router responde com erro de validação antes de chamar a Meta.

### 9.2 Erro HTTP da Instagram Graph API

Se a Meta responder com falha HTTP, o router devolve `502`. O objetivo é deixar claro que a borda da plataforma está saudável, mas a integração externa falhou.

### 9.3 Erro de rede

Se houver problema de conectividade com a Graph API, o router responde `503`.

### 9.4 Erro de contexto autenticado

Se a permissão usada não trouxer `client_code`, o router bloqueia o fluxo com erro operacional, porque sem isso não existe como decidir em qual tenant a conta deve ser registrada.

## 10. Operação do canal depois do provisionamento

Este manual é de onboarding, mas o código e a documentação existente deixam claro o acoplamento com a operação posterior.

Depois que a conta é provisionada:

1. O canal fica registrado no diretório.
2. O webhook do aplicativo pode entregar eventos para o boundary multicanal.
3. O `channel_id` passa a ser usado para resolver o canal e o YAML do tenant.
4. O restante da operação de mensagens segue o stack específico de Instagram do projeto.

## 11. Observabilidade e diagnóstico

O principal ponto de diagnóstico no código é o logger correlacionado criado dentro do `InstagramProvisionerAsync`.

Para investigar problema real, separe a análise em três perguntas:

1. O payload passou pela validação local?
2. A Graph API respondeu com sucesso nas três etapas externas?
3. O registro local no `ClientDirectory` foi concluído?

Se a resposta falhar antes da persistência, o problema está no boundary HTTP ou na Graph API. Se a Meta aceitar o fluxo mas a conta não aparecer no diretório, o problema passa para a gravação local.

## 12. Limites e pegadinhas

Os limites confirmados no código são:

1. O fluxo atual cobre provisionamento inicial; ele não descreve, por si só, toda a operação de envio e recebimento posterior.
2. O router depende de `client_code` vindo da autenticação; ele não tenta adivinhar tenant.
3. O webhook só é configurado se `callback_url` e `verify_token` passarem na validação local.
4. A versão da Graph API é configurável, mas precisa seguir o formato esperado pelo router.

## 13. Troubleshooting rápido

### 13.1 O provisionamento falhou com erro de validação

Causa provável: `callback_url`, email, token ou algum identificador veio fora do contrato aceito pelo schema.

### 13.2 O provisionamento falhou com `502`

Causa provável: a Graph API respondeu com falha HTTP em leitura do perfil, assinatura da página ou configuração do webhook.

### 13.3 O provisionamento falhou com `503`

Causa provável: erro de rede entre a plataforma e a Graph API.

### 13.4 A conta foi aceita, mas o canal não aparece no tenant

Causa provável: problema na gravação via `ClientDirectory` ou na associação com o `client_code` autenticado.

## 14. Explicação 101

Provisionar Instagram Direct aqui significa ligar a conta comercial do cliente ao aplicativo da plataforma e registrar isso no diretório interno.

Pense no fluxo como três travas sucessivas:

1. Validar se os dados que chegaram fazem sentido.
2. Pedir para a Meta aceitar essa conta e esse webhook.
3. Registrar localmente que esse tenant agora tem um canal Instagram ativo.

Sem a primeira trava, o sistema aceita payload ruim. Sem a segunda, o canal não funciona de verdade. Sem a terceira, a operação futura não sabe que a conta existe.

## 15. Evidências no código

1. `src/channel_layer/services/instagram_onboarding.py`: integração assíncrona com a Instagram Graph API.
2. `src/api/routers/instagram_provision_router.py`: endpoint público, validações de payload, registro no diretório e mapeamento de erros.
3. `src/security/client_directory.py`: persistência da conta provisionada no diretório multi-tenant.
