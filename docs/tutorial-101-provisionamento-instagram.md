# Tutorial 101: Provisionamento Instagram

Este tutorial explica o fluxo real de provisionamento Instagram com base no backend atual.

## 1. Para quem é este tutorial

Público-alvo:

- iniciante;
- desenvolvedor de negócio;
- desenvolvedor de plataforma.

Ao final, você deve conseguir:

- entender o fluxo do endpoint de provisionamento;
- localizar validações, permissão e integração com a Graph API;
- entender onde o vínculo do canal é persistido;
- validar o pós-provisionamento no canal.

## 2. Glossário rápido

- `provisionamento`: etapa que liga uma conta Instagram Business ao sistema.
- `instagram_business_account_id`: identificador da conta de negócio no Meta.
- `page_id`: identificador da página Facebook associada.
- `callback_url`: URL pública usada para webhook.
- `verify_token`: segredo de validação do webhook.
- `ClientDirectory`: camada que registra o vínculo local da conta provisionada.
- `InstagramProvisionerAsync`: serviço que conversa com a Graph API.
- `provision.instagram`: permissão exigida pelo endpoint.
- `channel_id`: identificador lógico do canal no sistema.
- `correlation_id`: identificador de rastreio do fluxo.

## 3. Explicação simples

Pense no provisionamento como cadastrar uma nova linha telefônica na central.

Primeiro, o sistema valida se os dados da conta fazem sentido. Depois, tenta conversar com a Meta para confirmar perfil, inscrição de página e configuração de webhook. Por fim, grava localmente o vínculo dessa conta com o tenant.

Sem esse passo, o canal Instagram existe no mundo externo, mas não está operacional dentro da plataforma.

## 4. Onde o fluxo mora no código

Os pontos principais observados no código são:

- `src/api/routers/instagram_provision_router.py`: recebe a requisição, valida e persiste o registro.
- `src/channel_layer/services/instagram_onboarding.py`: encapsula chamadas para a Graph API.
- `src/security/client_directory.py`: registra e consulta a conta Instagram provisionada.
- `src/api/security/permissions.py`: define a permissão `provision.instagram`.
- `src/api/routers/channel_router.py`: expõe as operações do canal depois do provisionamento.

## 5. Fluxo real do endpoint

O caminho confirmado no backend é este:

1. o cliente chama `POST /api/instagram/provision/start`;
2. o endpoint exige a permissão `provision.instagram`;
3. o payload é validado antes de qualquer chamada externa;
4. o `client_code` é extraído da autenticação, não do corpo da requisição;
5. o router cria `InstagramProvisionerAsync`;
6. o serviço consulta o perfil da conta de negócio;
7. o serviço inscreve a página para mensagens;
8. o serviço configura o webhook;
9. o router calcula hashes de segredos e grava o vínculo no `ClientDirectory`;
10. a resposta volta com `success=true`, `status=active`, `ig_username` e `page_id`.

## 6. Configurações e decisões importantes

No comportamento atual:

- `graph_api_version` pode ser enviado no request;
- se esse campo vier ausente, o fluxo usa `v20.0`;
- não existe endpoint dedicado de `verify` no router de provisionamento;
- não existe endpoint dedicado de `takeover` no router de provisionamento.

## 7. O que já está pronto

| Área | Evidência | Status | Impacto prático | Próximo passo mínimo |
| ---- | --------- | ------ | ---------------- | -------------------- |
| Endpoint de provisionamento | `instagram_provision_router.py` | pronto | provisiona com validação e persistência | manter contrato do request consistente |
| Permissão dedicada | `permissions.py` | pronto | restringe o uso a perfis autorizados | revisar papéis por tenant |
| Integração com Graph API | `instagram_onboarding.py` | pronto | executa o onboarding essencial | ampliar métricas por chamada externa |
| Persistência local | `client_directory.py` | pronto | mantém vínculo tenant x conta Instagram | revisar obrigatoriedade de campos |
| Hash de segredos | `instagram_provision_router.py` | pronto | evita salvar segredo em claro | manter política de segurança |
| Retry explícito | não encontrado no serviço | parcial | aumenta risco em falha transitória de rede | aplicar helper central de retry |
| Fluxos verify e takeover | não encontrados | ausente | limita migração incremental | criar endpoints separados quando fizer sentido |

## 8. Como colocar para funcionar

Passo 1. Ative o ambiente virtual.

- `source .venv/bin/activate`

Passo 2. Suba a API.

- `python main.py`

Passo 3. Use uma credencial com a permissão correta.

- `X-API-Key` precisa carregar `provision.instagram`.

Passo 4. Chame o endpoint de provisionamento.

- `POST /api/instagram/provision/start`

Campos obrigatórios esperados pelo fluxo:

- `instagram_business_account_id`;
- `page_id`;
- `display_name`;
- `email`;
- `access_token`;
- `app_id`;
- `app_secret`;
- `callback_url`;
- `verify_token`.

Passo 5. Valide a persistência local.

- confirme o registro da conta no `ClientDirectory`.

Passo 6. Valide o pós-provisionamento.

- teste as rotas do canal em `channel_router`, como mensagens e envio manual.

Passo 7. Rode os testes focados.

- `source .venv/bin/activate && PROMETEU_RUNNING_TESTS=1 pytest tests/unit/test_instagram_provision_router.py tests/unit/test_instagram_onboarding.py -q`

## 9. O que esperar no caminho feliz

Quando tudo funciona:

- o endpoint responde com HTTP 200;
- `success=true` aparece no retorno;
- o status vem como `active`;
- a conta fica registrada para o tenant;
- o canal pode seguir para rotas operacionais de mensagem.

## 10. Erros comuns

Sintoma: erro 400 no endpoint.

- hipótese mais provável: payload inválido, como URL, email, token ou ID malformado.

Sintoma: erro 400 por `client_code` ausente.

- hipótese mais provável: a credencial autenticada não traz o vínculo de tenant esperado.

Sintoma: erro 502.

- hipótese mais provável: a Graph API respondeu com falha HTTP.

Sintoma: erro 503.

- hipótese mais provável: falha de rede na chamada externa.

Sintoma: canal provisionado não recebe eventos.

- hipótese mais provável: webhook, callback ou vínculo do canal está incompleto.

## 11. O que não fazer

- não aceite `callback_url` sem validação adequada;
- não persista `access_token` nem `app_secret` em claro;
- não use `client_code` vindo do corpo da requisição;
- não mova chamadas da Meta para dentro do endpoint sem encapsulamento;
- não altere o request model sem atualizar os testes.

## 12. Explicação 101

Se você estiver começando agora, guarde esta versão curta:

o router recebe os dados da conta, o serviço fala com a Meta, e o diretório salva o vínculo local. O objetivo do provisionamento não é só testar token. É deixar o canal realmente pronto para operar dentro da plataforma.

## 13. Checklist final

- sei qual endpoint provisiona Instagram;
- sei qual permissão é exigida;
- sei onde a Graph API é chamada;
- sei onde o vínculo local é persistido;
- sei que segredos não devem ser salvos em claro;
- sei quais testes cobrem o fluxo principal;
- sei diferenciar erro de validação, erro HTTP externo e erro de rede.

## 14. Evidências no código

- `src/api/routers/instagram_provision_router.py`: entrada HTTP e persistência do registro.
- `src/channel_layer/services/instagram_onboarding.py`: integração com a Graph API.
- `src/security/client_directory.py`: registro da conta provisionada.
- `src/api/security/permissions.py`: permissão `provision.instagram`.
- `tests/unit/test_instagram_provision_router.py`: validação do endpoint.
- `tests/unit/test_instagram_onboarding.py`: validação do serviço.
