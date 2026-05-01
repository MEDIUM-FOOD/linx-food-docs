# Manual de Provisionamento WhatsApp Cloud API

## 1. O que esta feature faz

Este manual explica o provisionamento de números WhatsApp Business na plataforma. Na prática, esse fluxo permite registrar um número novo na Meta, solicitar o código de verificação, ativar o número, assumir o webhook do tenant e deixar o canal pronto para operação sem depender de passos manuais espalhados no portal da Meta.

O fluxo real do produto não é uma automação genérica. Ele está dividido entre um serviço assíncrono que fala com a Meta Graph API e um router HTTP que protege, valida, registra estado local e aplica regras de tenant.

## 2. Que problema ele resolve

Sem esse fluxo, o onboarding de WhatsApp vira um processo frágil: a Meta devolve um identificador técnico do número, o cliente recebe um código por SMS ou voz, o webhook precisa ser assumido no ambiente certo e o estado local do tenant precisa acompanhar cada etapa.

O módulo resolve isso separando o problema em duas partes:

1. Falar com a Meta do jeito correto.
2. Manter o diretório local e a operação do tenant coerentes com o que aconteceu na Meta.

O ganho prático é previsibilidade operacional. O time sabe quando um número ainda está pendente de verificação, quando já virou ativo e quando o webhook foi realmente assumido pelo ambiente atual.

## 3. Componentes principais

### 3.1 Credenciais e perfil do número

O contrato de credenciais fica em `MetaGraphCredentials`. Ele carrega `access_token`, `app_id`, `whatsapp_business_account_id`, `graph_api_version` e `base_url`.

O perfil do número fica em `ClientPhoneProfile`. Esse objeto concentra `country_code`, `phone_number`, `display_name`, `legal_name`, `email`, `verification_channel`, `timezone` e `website`. O método `phone_e164()` transforma país e telefone na forma internacional usada ao longo do fluxo.

### 3.2 Serviço assíncrono de integração com a Meta

`WhatsAppProvisionerAsync` é a camada que executa as chamadas externas.

Ele concentra as etapas externas do onboarding:

1. Registrar o número na Meta.
2. Pedir o código de verificação.
3. Validar o código recebido.
4. Ativar o número.
5. Configurar o webhook.
6. Garantir o template padrão.

O papel desse serviço é técnico. Ele não decide regra de tenant nem persistência local. Ele executa a conversa com a Meta e devolve os fatos necessários para a camada acima.

### 3.3 Gerenciador multi-tenant

`MultiTenantWhatsAppManager` é a camada que liga a operação Meta ao diretório do produto.

Ele resolve:

1. Credenciais do tenant com `get_credentials`.
2. Configuração de webhook do tenant com `get_webhook_config`.
3. Início do onboarding com `start_provision`.
4. Conclusão do onboarding com `finalize_provision`.
5. Atalhos operacionais como `provision_full_flow`.
6. Operações auxiliares de webhook e template usadas pelo router de takeover.

O ponto importante é este: o manager já sabe buscar configuração do tenant, aplicar fallback de webhook quando necessário e devolver um resumo operacional simples para o restante do sistema.

### 3.4 Router HTTP

O boundary público está em `whatsapp_provision_router`.

Esse router faz o trabalho de borda:

1. Exigir permissão `PROVISION_WHATSAPP`.
2. Extrair `correlation_id` para logging.
3. Resolver o contexto do tenant e do `client_code`.
4. Validar telefone, `phone_number_id`, SMS e regras de reentrada.
5. Persistir o estado do número no `ClientDirectory`.
6. Traduzir falhas internas em respostas HTTP operacionais.

## 4. Conceitos que importam para entender o fluxo

### 4.1 `client_code`

É o identificador do cliente dentro do diretório multi-tenant. Ele define de quais credenciais Meta, webhook e metadados de canal o fluxo vai depender.

### 4.2 `phone_number_id`

É o identificador técnico que a Meta devolve quando aceita o registro do número. O produto usa esse valor como elo entre a etapa de início e a etapa de verificação final.

### 4.3 Estado local do número

O diretório local não serve só como cadastro. Ele é a memória operacional do onboarding.

Os estados confirmados no código são:

1. `pending_verification` quando o `/start` já registrou o número e disparou o código.
2. `active` quando o `/verify` ou o `/import-existing` concluem o processo.

### 4.4 Idempotência

Os endpoints de verificação e takeover aceitam `X-Idempotency-Key`. Isso existe para proteger a operação contra replay de navegador, timeout e reenvio da mesma ação.

### 4.5 Correlação

Cada endpoint extrai `correlation_id` de `user_data` e cria logger correlacionado. Isso permite seguir a mesma execução do início ao fim sem adivinhar qual tentativa gerou determinado efeito.

## 5. Fluxo principal de onboarding de número novo

### 5.1 Descoberta do `client_code`

O fluxo começa com a listagem de `client_codes` autorizados para o tenant. Essa etapa existe para impedir que um operador tente provisionar número em um cliente que não pertence ao contexto autenticado.

### 5.2 Início do onboarding com `/start`

Quando a API recebe `POST /start`, ela faz cinco coisas importantes:

1. Resolve o contexto do tenant e do canal.
2. Confere se o número já existe localmente com `phone_number_id`; se existir, devolve conflito e bloqueia re-onboarding.
3. Monta `ClientPhoneProfile` a partir do perfil do tenant e do telefone informado.
4. Chama `MultiTenantWhatsAppManager.start_provision`.
5. Persiste o telefone como `pending_verification` no diretório local.

No manager, `start_provision` registra o número na Meta, exige que a Meta devolva `phone_number_id` e solicita o código de verificação no canal definido pelo perfil. Se a Meta não devolver o identificador, o fluxo falha fechado.

### 5.3 Verificação e ativação com `/verify`

Quando a API recebe `POST /verify`, o comportamento é mais rigoroso:

1. O número precisa existir localmente, senão o usuário é orientado a reiniciar pelo `/start`.
2. O `phone_number_id` enviado na requisição precisa bater com o valor já salvo localmente.
3. Se houver `X-Idempotency-Key`, a API tenta reaproveitar a resposta anterior antes de refazer a etapa externa.
4. O manager finaliza o fluxo validando código, ativando número, configurando webhook e criando template padrão.
5. O diretório é atualizado com `status=active`, indicadores de webhook e template e metadados de idempotência, quando aplicável.

O comportamento real de `finalize_provision` também confirma uma regra importante: `callback_url` e `verify_token` podem vir da chamada, mas se estiverem ausentes o manager cai para a configuração já registrada do tenant. Se esses valores estiverem vazios, a finalização falha com erro explícito de webhook inválido.

## 6. Fluxos operacionais auxiliares

### 6.1 Importar número já existente com `/import-existing`

Esse endpoint existe para o cenário em que o número já está ativo na Meta, mas ainda não foi registrado no diretório da plataforma.

Ele não refaz onboarding. Ele apenas:

1. Resolve o tenant e o canal.
2. Normaliza o telefone.
3. Registra `phone_number_id`, telefone, dados do tenant e status local.
4. Devolve confirmação de que o número foi importado sem re-onboarding.

Se `assume_active=true`, o status local vira `active`. Caso contrário, o fluxo preserva um estado mais cauteloso.

### 6.2 Remover webhook antigo com `/remove-webhook`

Essa etapa existe para cenários de migração. O objetivo é desinscrever uma configuração anterior antes de assumir o tráfego no ambiente atual.

O router resolve o callback do request ou do diretório do tenant e delega ao manager a remoção da assinatura. O retorno é operacional: ele informa se o webhook foi removido com sucesso ou se a solicitação foi enviada e ainda precisa de validação manual.

### 6.3 Assumir webhook com `/takeover`

O takeover só funciona quando o número já existe localmente no `ClientDirectory`. Essa exigência reduz risco de corte errado em número desconhecido.

O fluxo faz o seguinte:

1. Busca local do número por `phone_e164`.
2. Se o número não existir, responde `404`.
3. Resolve a configuração de webhook do tenant.
4. Chama `ensure_webhook_subscription` para assumir o webhook do ambiente atual.
5. Opcionalmente chama `ensure_template_exists` para garantir o template padrão.
6. Se houver `X-Idempotency-Key`, salva a resposta de takeover no metadata do número.

O uso prático é migração com corte controlado. Primeiro você registra o número localmente, depois assume o webhook quando o ambiente estiver pronto.

## 7. Endpoints públicos e papel de cada um

Os endpoints confirmados no router são:

1. `GET /client-codes` para listar `client_codes` autorizados.
2. `POST /start` para registrar o número e solicitar o código de verificação.
3. `POST /verify` para validar o SMS e ativar o número.
4. `POST /import-existing` para registrar localmente número já ativo na Meta.
5. `POST /remove-webhook` para remover subscrição antiga.
6. `POST /takeover` para assumir o webhook e, se solicitado, garantir template.

## 8. O que entra e o que sai em cada etapa

### 8.1 Entradas críticas

Os dados críticos confirmados no código são:

1. `phone_e164` no formato internacional.
2. `client_code` coerente com o tenant autenticado.
3. `phone_number_id` nas etapas que não podem se apoiar só no telefone.
4. `codigo_sms` no `/verify`.
5. `channel_id` quando a operação precisa se vincular a um canal específico.
6. `X-Idempotency-Key` nas operações em que replay é risco real.

### 8.2 Saídas importantes

As respostas confirmadas no código carregam, conforme a etapa:

1. `phone_number_id`.
2. `phone_e164`.
3. `status` operacional.
4. mensagem humana de progresso ou sucesso.
5. `webhook_configured` e `template_created` quando a etapa mexe nisso.

## 9. O que acontece em caso de sucesso

No caminho feliz, o produto conta uma história operacional coerente:

1. `/start` cria o vínculo com a Meta e grava o número como pendente.
2. O cliente recebe o código por SMS ou voz.
3. `/verify` conclui ativação, webhook e template.
4. O diretório local sai do estado pendente e entra em `active`.
5. Os logs registram início, sucesso e identificadores relevantes com o mesmo `correlation_id`.

Em cenários de migração:

1. `/import-existing` registra localmente sem refazer onboarding.
2. `/remove-webhook` limpa o vínculo antigo, quando necessário.
3. `/takeover` passa o webhook para o ambiente atual.

## 10. O que acontece em caso de erro

Os erros confirmados no código se distribuem assim:

### 10.1 Conflito por re-onboarding

Se `/start` encontrar número já registrado com `phone_number_id`, o router devolve conflito. Isso evita duplicar onboarding do mesmo telefone.

### 10.2 Fluxo de verificação sem início válido

Se `/verify` não encontrar registro local, a API responde que o fluxo precisa ser reiniciado pelo `/start`. Se o `phone_number_id` armazenado não bater com o da requisição, a API também bloqueia a continuação.

### 10.3 Erro de Meta ou contrato externo

Se a Meta não devolver `phone_number_id` no registro, a API falha fechado. Se a finalização receber código inválido, o router traduz o problema para erro operacional de código SMS incorreto. Outras falhas externas sobem como erro de integração com a Meta.

### 10.4 Problema de webhook do tenant

Se o manager não conseguir resolver `callback_url` e `verify_token`, a finalização falha com erro explícito de webhook mal configurado. Isso evita ativar número sem conseguir receber eventos no ambiente certo.

### 10.5 Persistência local

Se o `ClientDirectory` falhar ao registrar o telefone pendente ou ativo, o router devolve erro HTTP e registra a exceção com correlação. O objetivo é impedir que o sistema aparente sucesso externo enquanto perde o estado local.

## 11. Observabilidade e diagnóstico

O ponto de partida da investigação é o `correlation_id` do request.

Cada endpoint registra eventos de início, sucesso e exceção. Os nomes dos eventos confirmados no código incluem:

1. `start_provisioning.begin` e `start_provisioning.success`.
2. `verify_and_activate.begin` e `verify_and_activate.success`.
3. `import_existing.begin` e `import_existing.success`.
4. `remove_webhook.begin` e `remove_webhook.success`.
5. `takeover.begin` e `takeover.success`.

Para diagnosticar bem, separe o problema em três perguntas:

1. O tenant e o `client_code` foram resolvidos corretamente?
2. A Meta aceitou a etapa externa e devolveu identificadores válidos?
3. O diretório local foi atualizado com o estado certo?

## 12. Limites e pegadinhas

Alguns limites confirmados no código merecem destaque:

1. O fluxo depende de credenciais Meta válidas por tenant; ele não funciona sem isso.
2. `phone_number_id` não é opcional nas etapas posteriores ao registro inicial.
3. O takeover não substitui o import; ele pressupõe que o número já esteja conhecido localmente.
4. Idempotência protege replay, mas não corrige configuração errada de tenant ou webhook.
5. Este manual é só de WhatsApp. Provisionamento de Instagram é outro fluxo, com outro router e outro serviço.

## 13. Troubleshooting rápido

### 13.1 O `/start` falhou dizendo que o número já existe

Causa provável: o telefone já foi registrado localmente antes. O caminho correto passa a ser verificar se o fluxo precisa de `/verify`, `/import-existing` ou `/takeover`, em vez de refazer onboarding do zero.

### 13.2 O `/verify` diz que o fluxo não foi iniciado

Causa provável: não existe registro local pendente para esse telefone. Confirme se o `/start` concluiu e se o `phone_number_id` retornado foi preservado.

### 13.3 O `/verify` acusa código inválido

Causa provável: SMS incorreto ou expirado. O router já trata esse caso como erro operacional específico.

### 13.4 O webhook não assumiu no takeover

Causa provável: o número não foi importado antes, ou a configuração de webhook do tenant está incompleta. Confirme presença local do número e valores de `callback_url` e `verify_token`.

### 13.5 O fluxo externo parece certo, mas o número não aparece ativo

Causa provável: falha na persistência local. Consulte os logs do endpoint e do diretório para verificar se o `register_whatsapp_phone` concluiu após a chamada externa.

## 14. Explicação 101

Pense nesse fluxo como um cadastro de linha telefônica com três camadas.

1. A Meta entrega o identificador técnico do número e valida que ele existe de verdade.
2. O produto guarda esse número no diretório do cliente para não se perder entre uma etapa e outra.
3. O webhook conecta o número ao ambiente certo, para que as mensagens futuras cheguem nesta plataforma.

Sem essas três camadas juntas, o número até pode existir na Meta, mas a operação do tenant fica cega. Com elas, o onboarding vira um processo rastreável e repetível.

## 15. Evidências no código

1. `src/channel_layer/services/whatsapp_meta_onboarding.py`: credenciais, perfil do telefone, integração assíncrona com a Meta e manager multi-tenant.
2. `src/api/routers/whatsapp_provision_router.py`: endpoints públicos, validações HTTP, idempotência e persistência no diretório.
3. `src/api/service_api.py`: inclusão do router no boundary HTTP da aplicação.
