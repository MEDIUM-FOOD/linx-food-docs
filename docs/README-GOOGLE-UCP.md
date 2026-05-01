Produto: Plataforma de Agentes de IA

# Integração com Google UCP

## Visão geral
Esta documentação descreve a integração da Plataforma de Agentes de IA com o Google Universal Commerce Protocol (UCP) de ponta a ponta. O foco é explicar o discovery do Business Profile, os endpoints REST do UCP, os contratos de payloads, o fluxo pós-compra e as dependências operacionais com Merchant Center e PDV. O objetivo é garantir que o checkout siga o contrato do UCP e que a Plataforma de Agentes de IA consiga orquestrar a sessão com segurança e previsibilidade. Este material não substitui a especificação oficial, mas mostra o que está efetivamente implementado no repositório. Sempre considere os requisitos oficiais do UCP como referência normativa.

## Por que existe
A integração UCP permite que plataformas e agentes iniciem e completem o checkout de forma programática. Isso reduz fricção no processo de compra e habilita experiências guiadas por agentes, mantendo o lojista como Merchant of Record. A Plataforma de Agentes de IA precisa publicar capacidades e operar endpoints alinhados ao protocolo para que a integração seja reconhecida. Além disso, o PDV deve responder com dados completos e determinísticos para garantir consistência entre catálogo, preços e cobrança. Esta documentação unifica a visão operacional para consultores e equipes que fazem onboarding de comércio.

## Explicação conceitual
O UCP define um conjunto de capacidades, entidades e operações para checkout. A Plataforma de Agentes de IA publica um Business Profile em /.well-known/ucp com informações de versão, serviços e capacidades. O checkout é realizado via endpoints REST definidos pelo UCP, com payloads que incluem itens, comprador, moeda e pagamento. A Plataforma de Agentes de IA não calcula preços; ele solicita ao PDV as informações de itens, totais e status do checkout. A resposta do PDV deve retornar o envelope UCP, itens completos e estado da sessão. Essa abordagem preserva a lógica do lojista e reduz divergências entre catálogo e cobrança. O fluxo pós-compra é disparado por eventos do ERP, que encaminham o pedido para o webhook da plataforma UCP, mantendo o mesmo contrato de dados.

## Explicação for dummies
Pense no UCP como uma ficha padrão que descreve como comprar em uma loja. A Plataforma de Agentes de IA publica essa ficha para que qualquer plataforma saiba onde começar a compra e quais passos seguir. Quando alguém quer comprar, a Plataforma de Agentes de IA passa o carrinho para o PDV e recebe de volta tudo pronto: itens completos, preços e status. Se faltar alguma informação, o PDV avisa o que precisa ser corrigido e o comprador tenta de novo. Quando o pagamento é aprovado, o PDV confirma o pedido e a Plataforma de Agentes de IA apenas repassa essa confirmação. Depois da compra, o ERP manda os eventos do pedido e a Plataforma de Agentes de IA dispara um aviso para a plataforma. No fim, a Plataforma de Agentes de IA funciona como o mensageiro confiável entre quem compra e quem vende.

## Conceitos importantes (explicação simples)
Esta seção explica conceitos que aparecem no fluxo e podem gerar dúvidas na integração.

### Webhook de eventos de pedido
Webhook é um mecanismo em que um sistema envia um aviso automático para outro quando algo importante acontece. No UCP, o ERP informa � Plataforma de Agentes de IA que um pedido mudou, e a Plataforma de Agentes de IA dispara esse aviso para a plataforma UCP. Isso evita consultas constantes e garante que a plataforma receba o evento logo após a mudança. Aqui, o webhook serve para notificar eventos pós-compra, como criação, envio ou cancelamento. É essencial que a URL do webhook esteja correta para que o evento não se perca.

### RFC 8414 (metadados OAuth)
RFC 8414 é um padrão que descreve como descobrir automaticamente os endpoints de um servidor OAuth. Em vez de configurar manualmente os endereços de autorização e token, a plataforma busca um endpoint bem conhecido e obtém a lista oficial. Isso reduz erro de configuração e facilita integração entre sistemas. Na Plataforma de Agentes de IA, esses metadados são publicados em /.well-known/oauth-authorization-server. A integração depende de variáveis de ambiente que descrevem os endpoints disponíveis.

### Idempotência
Idempotência é a garantia de que repetir a mesma requisição não cria efeitos duplicados. Em checkout, isso evita que um pagamento seja processado duas vezes se o cliente repetir a chamada. O UCP utiliza o header Idempotency-Key para esse controle. Na Plataforma de Agentes de IA, o header é propagado nos endpoints mutáveis (Create, Update, Complete e Cancel). A validação final de semântica idempotente depende do comportamento do PDV.

## Responsabilidades por componente
1) Plataforma de Agentes de IA
  - Publica o Business Profile e o OAuth metadata.
  - Orquestra os endpoints REST do UCP.
  - Encaminha o checkout para o PDV e retorna a resposta.
  - Recebe eventos do ERP e dispara webhook de pedido.
2) PDV
  - Calcula preços, taxas, disponibilidade e status do checkout.
  - Retorna line_items completos e totals recalculados.
  - Aplica regras de cancelamento e finalização.
3) ERP
  - Emite eventos pós-compra com o payload do pedido.
4) Merchant Center
  - Garante políticas de devolução e suporte ao cliente.
  - Informa elegibilidade via atributos do feed.

## Como o usuário recebe essa feature
1) A Plataforma de Agentes de IA publica automaticamente o Business Profile UCP em /.well-known/ucp. A implementação está em [src/api/routers/ucp_router.py](src/api/routers/ucp_router.py).
2) A Plataforma de Agentes de IA expõe endpoints de Checkout Sessions em {base}/checkout-sessions conforme configuração. A implementação está em [src/api/routers/ucp_checkout_router.py](src/api/routers/ucp_checkout_router.py).
3) A Plataforma de Agentes de IA expõe metadados OAuth em /.well-known/oauth-authorization-server para discovery RFC 8414. A implementação está em [src/api/routers/ucp_router.py](src/api/routers/ucp_router.py).
4) A Plataforma de Agentes de IA expõe um endpoint para eventos pós-compra do ERP em {base}/order-events. A implementação está em [src/api/routers/ucp_order_event_router.py](src/api/routers/ucp_order_event_router.py).
5) O contrato de integração com o PDV é definido em [src/ucp/pdv_checkout_api.py](src/ucp/pdv_checkout_api.py), que formaliza o formato de request e response que o PDV deve atender.
6) O diagnóstico do manifesto UCP pode ser feito via tool ucp_discovery_tool, descrita em [docs/GUIA-USUARIO-TOOLS.md](docs/GUIA-USUARIO-TOOLS.md).

## Endpoints UCP expostos pela Plataforma de Agentes de IA
Esta seção descreve o contrato de endpoints para discovery, checkout e eventos pós-compra. O path base é definido por UCP_REST_BASE_PATH e, por padrão, é /ucp/v1.

1) Discovery
  - GET /.well-known/ucp: Business Profile UCP.
  - GET /.well-known/oauth-authorization-server: metadados OAuth (RFC 8414).

2) Checkout Sessions (REST)
  - POST {base}/checkout-sessions: cria uma sessão de checkout.
  - GET {base}/checkout-sessions/{checkout_id}: consulta a sessão.
  - PUT {base}/checkout-sessions/{checkout_id}: atualiza a sessão.
  - POST {base}/checkout-sessions/{checkout_id}/complete: conclui a sessão.
  - POST {base}/checkout-sessions/{checkout_id}/cancel: cancela a sessão.

3) Eventos pós-compra
  - POST {base}/order-events: recebe eventos do ERP e encaminha webhook.

## Passo a passo para abrir e configurar o Merchant Center
1) Garantir conta Merchant Center ativa, em bom estado, com produtos aprovados para free listings.
2) Confirmar que os produtos já são enviados por feed, Content API ou Merchant API.
3) Configurar políticas obrigatórias no Merchant Center antes da integração UCP.
4) Atualizar o feed com atributos de elegibilidade e avisos regulatórios.
5) Validar mapeamento de IDs entre Merchant Center e a API de checkout.

Estas etapas são pré-requisitos definidos pelo Google para habilitar o UCP e precisam estar concluídas antes de publicar o Business Profile e expor o checkout.

## Passo a passo for dummies (configuração manual no site do Google)
Este passo a passo é o “manual do básico” para quem vai criar e configurar a conta no site do Google. Ele não substitui a documentação oficial e não detalha telas específicas, porque a interface pode mudar. O objetivo é garantir que você saiba o que precisa existir antes de integrar o UCP.

### Visão geral simples
Pense no Merchant Center como o “cadastro oficial da loja” dentro do Google. Sem ele, o Google não sabe quais produtos existem nem quais políticas a sua loja segue. A integração UCP só funciona quando esse cadastro está completo, porque o checkout precisa dessas informações para aparecer nas superfícies do Google.

### Passo a passo (sem detalhes de tela)
1) Criar a conta no Merchant Center e confirmar que ela está ativa.
2) Enviar o catálogo de produtos usando feed ou API, garantindo que os itens apareçam corretamente.
3) Configurar as políticas obrigatórias de devolução e suporte ao cliente.
4) Marcar os produtos elegíveis para UCP no feed com os atributos exigidos.
5) Validar se os IDs do catálogo batem com os IDs usados pela API de checkout.
6) Revisar o status da conta e corrigir avisos antes do go‑live.

### Explicação for dummies
Se o Merchant Center fosse uma “ficha de cadastro”, você precisa preencher o nome da loja, informar quais produtos vende e dizer como o cliente devolve algo se precisar. Sem isso, o Google não libera o botão de comprar. Depois, você precisa garantir que o código do produto no catálogo seja o mesmo usado no checkout. É como usar o mesmo número de pedido no caixa e no estoque: se for diferente, ninguém encontra o produto certo.

### Onde conferir o passo a passo oficial
Use as páginas oficiais abaixo como referência obrigatória, porque elas são a fonte de verdade e mudam com frequência:
- https://developers.google.com/merchant/ucp/guides/merchant-center
- https://developers.google.com/merchant/ucp/guides/business-profile
- https://developers.google.com/merchant/ucp/guides/checkout

## Políticas obrigatórias (Return Policy e Customer Support)
Retorno e suporte são exigências de Merchant of Record e aparecem no checkout e na confirmação de pedido.

1) Return policy:
  - Configurar política de devolução no Merchant Center.
  - Informar custo de devolução, prazo de devolução e link para a política completa.
  - Se necessário, usar return_policy_label no feed para aplicar políticas por subconjunto de produtos.
  - Em contas avançadas do Merchant Center, a política de devolução deve ser definida em cada subconta.
2) Customer support info:
  - Configurar informações de suporte ao cliente no Merchant Center.
  - Essas informações são usadas no link “Contact Merchant” da confirmação de pedido.

Sem essas políticas configuradas, a integração UCP não atende aos requisitos do Google.

## Requisitos do feed para habilitar UCP
O Google exige atributos específicos no feed para determinar elegibilidade e exibir avisos legais.

1) native_commerce:
  - Atributo booleano que opta o produto para o checkout.
  - Se estiver ausente ou false, o produto não fica elegível para UCP.
2) consumer_notice (quando aplicável):
  - Obrigatório para produtos que exigem aviso regulatório.
  - Subatributos exigidos:
    - consumer_notice_type: legal_disclaimer, safety_warning ou prop_65.
    - consumer_notice_message: texto do aviso, até 1000 caracteres; HTML permitido apenas para as tags b, br e i.
3) Mapeamento de IDs:
  - O id do feed precisa ser o mesmo ID usado pela API de checkout.
  - Se os IDs não forem iguais, usar o atributo merchant_item_id para mapear o ID do checkout.
4) Fonte recomendada dos atributos:
  - O Google recomenda usar um feed suplementar para evitar impacto no feed principal.

Além disso, categorias restritas não devem ser habilitadas para checkout; nesses casos o atributo native_commerce deve ficar vazio ou false.

## Observações para cenário multi-tenant (vários restaurantes)
Quando a Plataforma de Agentes de IA atende múltiplos restaurantes, é necessário refletir essa separação no Merchant Center.

1) Avaliar uso de conta avançada do Merchant Center com subcontas por restaurante.
2) Cada subconta deve ter return policy e customer support info próprios.
3) Cada feed deve manter IDs consistentes com a API do PDV daquele restaurante.
4) Se houver divergência de IDs entre feed e PDV, usar merchant_item_id por subconta.
5) native_commerce e consumer_notice devem ser definidos por item, respeitando elegibilidade de cada restaurante.

Essas regras evitam cruzamento de políticas e garantem consistência entre catálogo e checkout.

## Checklist operacional (pré-go-live)
Use esta lista para validar se a integração está pronta, sem etapas de marketing.

1) Merchant Center
  - Conta ativa, em bom estado e com produtos aprovados para free listings.
  - Return policy configurada com custo, prazo e link público.
  - Customer support info configurado e publicado.
2) Feed e elegibilidade
  - native_commerce definido como true para produtos elegíveis.
  - consumer_notice definido quando houver aviso regulatório obrigatório.
  - ID do feed mapeado para o ID usado pela API de checkout (ou merchant_item_id configurado).
3) Integração UCP
  - Business Profile publicado em /.well-known/ucp com versão e capabilities corretas.
  - Base path REST configurado e acessível externamente.
4) PDV/ERP
  - Create/Get/Update/Complete/Cancel retornam checkout completo com totals e status.
  - Mensagens de erro retornam códigos claros quando o checkout não é elegível.
5) Multi-tenant
  - Subcontas (quando aplicável) com políticas e suporte próprios.
  - Feeds segmentados por restaurante e IDs consistentes com o PDV correspondente.

## O que está implementado no código
- Business Profile em /.well-known/ucp com versão, serviços REST e capacidades configuráveis.
  Implementação em [src/api/routers/ucp_router.py](src/api/routers/ucp_router.py).
- Rotas REST de checkout session mapeadas com base path configurável.
  Implementação em [src/api/routers/ucp_checkout_router.py](src/api/routers/ucp_checkout_router.py).
- Create, Get, Update, Complete e Cancel de checkout conectados ao microserviço de PDV, com validação do payload e retorno do objeto atualizado.
  Implementação em [src/api/routers/ucp_checkout_router.py](src/api/routers/ucp_checkout_router.py).
- Contrato tipado para requests e responses do PDV, incluindo fulfillment delivery/pickup e destinos.
  Implementação em [src/ucp/pdv_checkout_api.py](src/ucp/pdv_checkout_api.py).
- Endpoint para disparo de eventos de pedido via ERP, que encaminha webhook conforme payload ou configuração.
  Implementação em [src/api/routers/ucp_order_event_router.py](src/api/routers/ucp_order_event_router.py) e [src/ucp/order_event_dispatcher.py](src/ucp/order_event_dispatcher.py).
- Endpoint de metadados OAuth em /.well-known/oauth-authorization-server conforme RFC 8414.
  Implementação em [src/api/routers/ucp_router.py](src/api/routers/ucp_router.py).
- Documentação de endpoints do serviço em [docs/README-SERVICE-API.md](docs/README-SERVICE-API.md).

Observação importante: todos os endpoints de checkout UCP passam pelo PDV e dependem de configuração correta no .env.

## O que ainda precisa ser implementado
Esta seção descreve, de forma objetiva, o que falta para completar a integração UCP na Plataforma de Agentes de IA. O objetivo é orientar o time de produto e integração, evitando suposições e garantindo previsibilidade. As pendências listadas abaixo são baseadas no comportamento atual do código e no contrato oficial do UCP. Sempre que possível, a implementação deve reaproveitar o cliente do PDV existente e manter o padrão de logs com correlation_id.

Em termos simples, pense que a Plataforma de Agentes de IA já conversa com o PDV em todas as etapas do checkout. A propagação do header de idempotência já está implementada nos endpoints mutáveis. O que falta é alinhar a semântica final com o PDV e padronizar mensagens de erro para garantir previsibilidade operacional em produção.

### Pendências obrigatórias do fluxo UCP
1) Validar e documentar a semântica de idempotência com o PDV para Create, Update, Complete e Cancel (propagação do header já implementada na Plataforma de Agentes de IA).
2) Padronizar mensagens de erro e códigos HTTP quando o PDV retornar estados não elegíveis, especialmente no Cancel e Complete.

### Pendências de contrato e integração com PDV
1) Confirmar quais campos de fulfillment o PDV aceita no Update, incluindo destinos, grupos e opções, conforme a extensão oficial.
2) Confirmar se o PDV exige campos adicionais para buyer, payment ou risk_signals além do schema UCP.
3) Confirmar o comportamento do PDV quando o checkout está em estado finalizado, para garantir que a Plataforma de Agentes de IA apenas repasse a semântica correta.

### Pendências de observabilidade e operação
1) Medir tempo de resposta do PDV e registrar métricas operacionais com correlation_id.
2) Definir política operacional de fallback quando o PDV estiver indisponível (quando usar em sandbox e quando bloquear em produção).
3) Documentar o SLA esperado do PDV e limites de rate limit para evitar bloqueios.

## Recomendação do fabricante versus comportamento observado
Comportamento observado no código:
- O Business Profile contém version, services e capabilities configuráveis via variáveis de ambiente.
- Os endpoints REST do checkout existem e estão mapeados com base path configurável.
- O contrato PDV exige respostas completas de checkout, incluindo totals, payment e links.

Recomendação do fabricante:
- O UCP define que o checkout deve seguir o capability dev.ucp.shopping.checkout com version 2026-01-11.
- O status do checkout deve seguir o ciclo incomplete, requires_escalation, ready_for_complete, complete_in_progress, completed, canceled.
- O continue_url deve ser fornecido quando o status for requires_escalation.

## Configurações operacionais
Configurações de Business Profile e REST UCP:
- UCP_PROTOCOL_VERSION
- UCP_REST_BASE_PATH
- UCP_CAPABILITIES
- UCP_SIGNING_KEYS_JSON

Configurações de integração com o PDV:
- UCP_PDV_BASE_URL
- UCP_PDV_CREATE_CHECKOUT_PATH
- UCP_PDV_GET_CHECKOUT_PATH_TEMPLATE
- UCP_PDV_UPDATE_CHECKOUT_PATH_TEMPLATE
- UCP_PDV_COMPLETE_CHECKOUT_PATH_TEMPLATE
- UCP_PDV_CANCEL_CHECKOUT_PATH_TEMPLATE
- UCP_PDV_TIMEOUT_SECONDS
- UCP_PDV_RETRY_ATTEMPTS
- UCP_PDV_BACKOFF_BASE_SECONDS

Configuração de fallback sandbox:
- UCP_FALLBACK_ENABLED
- UCP_FALLBACK_DSN
- UCP_FALLBACK_SCHEMA
- UCP_FALLBACK_POOL_MIN_SIZE
- UCP_FALLBACK_POOL_MAX_SIZE
- UCP_FALLBACK_POOL_MAX_IDLE
- UCP_FALLBACK_POOL_TIMEOUT_SECONDS
- UCP_FALLBACK_RETRY_ATTEMPTS
- UCP_FALLBACK_RETRY_MIN_SECONDS
- UCP_FALLBACK_RETRY_MAX_SECONDS
- DATABASE_VAREJO_* (compatibilidade legada; usado quando UCP_FALLBACK_* não estiver preenchido)

Configuração de webhook de eventos de pedido:
- UCP_ORDER_WEBHOOK_URL

Configuração de metadados OAuth (RFC 8414):
- UCP_OAUTH_AUTH_SERVER_METADATA_JSON
- UCP_OAUTH_ISSUER
- UCP_OAUTH_AUTHORIZATION_ENDPOINT
- UCP_OAUTH_TOKEN_ENDPOINT
- UCP_OAUTH_REVOCATION_ENDPOINT
- UCP_OAUTH_SCOPES_SUPPORTED
- UCP_OAUTH_RESPONSE_TYPES_SUPPORTED
- UCP_OAUTH_GRANT_TYPES_SUPPORTED
- UCP_OAUTH_TOKEN_ENDPOINT_AUTH_METHODS_SUPPORTED

Os nomes acima são lidos do .env global pelo SystemConfigManager. O ponto de leitura está em [src/config/config_api/system_config_manager.py](src/config/config_api/system_config_manager.py).

## Contrato de payloads UCP e PDV
Esta seção descreve o contrato mínimo esperado pela Plataforma de Agentes de IA, com base nos modelos tipados do código. Ela não substitui o schema oficial do UCP, mas orienta integração e validação operacional.

### Payload de criação de checkout (POST checkout-sessions)
Campos esperados pela integração com o PDV:
1) line_items: lista de itens com item.id e quantity.
2) buyer: dados do comprador (nome, email, telefone), quando disponíveis.
3) currency: moeda do checkout.
4) payment: dados de pagamento e instrumentos selecionados.

### Payload de atualização de checkout (PUT checkout-sessions/{checkout_id})
Campos esperados pela integração com o PDV:
1) id: identificador do checkout, igual ao checkout_id da rota.
2) line_items: itens com item.id, quantity e, opcionalmente, parent_id.
3) buyer: dados atualizados do comprador.
4) fulfillment: métodos e destinos de entrega/retirada quando aplicável.
5) currency: moeda do checkout.
6) payment: dados de pagamento e instrumentos selecionados.

### Payload de finalização (POST checkout-sessions/{checkout_id}/complete)
Campos esperados pela integração com o PDV:
1) payment_data: dados de pagamento necessários para captura.
2) risk_signals: sinais de risco quando disponíveis.

### Payload de evento pós-compra (POST order-events)
Campos esperados para disparo do webhook:
1) event_type: tipo do evento pós-compra.
2) checkout_id: identificador do checkout afetado.
3) order: objeto de pedido completo quando houver informação disponível.
4) webhook_url: URL opcional para sobrescrever o destino do webhook.

### Resposta esperada do PDV para operações de checkout
Campos principais que devem estar presentes no retorno do PDV:
1) ucp: envelope com versão e capabilities ativas.
2) id: identificador do checkout.
3) line_items: lista com item, price, quantity e totals por item.
4) status: status do checkout conforme ciclo UCP.
5) currency: moeda utilizada no checkout.
6) totals: totais globais recalculados.
7) links: links legais ou informativos exibidos no checkout.
8) payment: handlers de pagamento disponíveis e instrumento selecionado.

Campos recomendados para completude operacional:
1) messages: mensagens de erro, aviso ou informação.
2) expires_at: expiração da sessão.
3) continue_url: URL de handoff quando status exige escalonamento.
4) order: confirmação do pedido quando finalizado.
5) fulfillment: métodos e destinos de entrega/retirada.
6) fulfillment_options e fulfillment_option_id quando aplicável.

O contrato formal está definido em [src/ucp/pdv_checkout_api.py](src/ucp/pdv_checkout_api.py) nas classes PdvCheckoutCreateRequest, PdvCheckoutUpdateRequest, PdvCheckoutCompleteRequest e PdvCheckoutResponse. A Plataforma de Agentes de IA espera respostas compatíveis com esse esquema, alinhadas ao UCP e à extensão de fulfillment.

## Tutorial completo para implementar as APIs do PDV
Este tutorial é focado em um desenvolvedor júnior que precisa implementar a parte do PDV para que o checkout UCP funcione de ponta a ponta. O objetivo é deixar claro o que a Plataforma de Agentes de IA envia, o que ele espera receber e quais variáveis precisam existir no .env para o roteamento funcionar.

### Visão geral
A Plataforma de Agentes de IA atua como orquestrador do checkout UCP. Ele não calcula preços nem decide disponibilidade. Em vez disso, ele encaminha o payload do checkout para o PDV e devolve ao UCP a resposta que o PDV calcular. Isso significa que o PDV precisa expor endpoints HTTP coerentes com o contrato do UCP e responder com dados completos. Se o PDV não responder corretamente, o checkout falha e a Plataforma de Agentes de IA devolve erro para a plataforma UCP.

### Por que existe
Sem o PDV, a Plataforma de Agentes de IA não consegue calcular preço, taxa, desconto e disponibilidade. O UCP exige consistência desses dados, e isso é responsabilidade do lojista. O PDV é a fonte de verdade e precisa validar o carrinho e retornar o estado real da sessão. Esse desenho evita divergências entre catálogo, preço e pagamento. Também garante que qualquer alteração no checkout passe por uma lógica única de negócio.

### Explicação conceitual
Pense nos endpoints do PDV como o motor do checkout. A Plataforma de Agentes de IA recebe o pedido do UCP e repassa ao PDV, que recalcula tudo. O PDV responde com o objeto de checkout completo, incluindo status e totais. A Plataforma de Agentes de IA não altera a resposta; ele apenas devolve esse objeto à plataforma UCP. Por isso o PDV precisa obedecer exatamente ao contrato definido em [src/ucp/pdv_checkout_api.py](src/ucp/pdv_checkout_api.py).

### Explicação for dummies
Imagine que a Plataforma de Agentes de IA é um atendente que só passa mensagens entre o cliente e o caixa. O cliente diz o que quer comprar, o atendente leva ao caixa e volta com a conta certa. Se o caixa não souber calcular o preço ou esquecer um item, o atendente não consegue ajudar. O PDV é esse caixa. Ele precisa receber o pedido e devolver o valor certo com todas as informações que o cliente precisa ver. Sem isso, a compra não fecha.

### Como o usuário recebe essa feature
1) O time do PDV implementa as rotas HTTP listadas abaixo.
2) O time configura o .env com a base URL e os caminhos de cada rota.
3) A Plataforma de Agentes de IA passa a chamar o PDV automaticamente em Create, Get, Update, Complete e Cancel.
4) A plataforma UCP recebe a resposta do PDV sem transformação adicional.

### APIs do PDV que precisam existir
Os endpoints do PDV são configurados por variáveis no .env. Cada rota deve aceitar o payload descrito no contrato e retornar PdvCheckoutResponse.

1) Create checkout
Variável: UCP_PDV_CREATE_CHECKOUT_PATH.
A Plataforma de Agentes de IA envia line_items, buyer, currency e payment conforme PdvCheckoutCreateRequest.
A resposta precisa conter o checkout completo com totals e status.

2) Get checkout
Variável: UCP_PDV_GET_CHECKOUT_PATH_TEMPLATE (usa {checkout_id}).
A Plataforma de Agentes de IA consulta o estado da sessão e espera PdvCheckoutResponse.

3) Update checkout
Variável: UCP_PDV_UPDATE_CHECKOUT_PATH_TEMPLATE (usa {checkout_id}).
A Plataforma de Agentes de IA envia id, line_items, buyer, fulfillment, currency e payment.
A resposta deve refletir o novo estado com totals recalculados.

4) Complete checkout
Variável: UCP_PDV_COMPLETE_CHECKOUT_PATH_TEMPLATE (usa {checkout_id}).
A Plataforma de Agentes de IA envia payment_data e, quando disponível, risk_signals.
A resposta deve conter status final e order quando aprovado.

5) Cancel checkout
Variável: UCP_PDV_CANCEL_CHECKOUT_PATH_TEMPLATE (usa {checkout_id}).
A Plataforma de Agentes de IA solicita cancelamento e espera PdvCheckoutResponse com status canceled.

### Variáveis do .env que precisam estar preenchidas
Estas variáveis são obrigatórias para a Plataforma de Agentes de IA conseguir chamar o PDV e publicar o UCP corretamente.

Business Profile e REST UCP:
1) UCP_PROTOCOL_VERSION
2) UCP_REST_BASE_PATH
3) UCP_CAPABILITIES
4) UCP_SIGNING_KEYS_JSON

PDV:
1) UCP_PDV_BASE_URL
2) UCP_PDV_CREATE_CHECKOUT_PATH
3) UCP_PDV_GET_CHECKOUT_PATH_TEMPLATE
4) UCP_PDV_UPDATE_CHECKOUT_PATH_TEMPLATE
5) UCP_PDV_COMPLETE_CHECKOUT_PATH_TEMPLATE
6) UCP_PDV_CANCEL_CHECKOUT_PATH_TEMPLATE
7) UCP_PDV_TIMEOUT_SECONDS
8) UCP_PDV_RETRY_ATTEMPTS
9) UCP_PDV_BACKOFF_BASE_SECONDS

Webhook de eventos pós-compra:
1) UCP_ORDER_WEBHOOK_URL

OAuth:
1) UCP_OAUTH_AUTH_SERVER_METADATA_JSON ou, se vazio, todas as UCP_OAUTH_* abaixo
2) UCP_OAUTH_ISSUER
3) UCP_OAUTH_AUTHORIZATION_ENDPOINT
4) UCP_OAUTH_TOKEN_ENDPOINT
5) UCP_OAUTH_REVOCATION_ENDPOINT
6) UCP_OAUTH_SCOPES_SUPPORTED
7) UCP_OAUTH_RESPONSE_TYPES_SUPPORTED
8) UCP_OAUTH_GRANT_TYPES_SUPPORTED
9) UCP_OAUTH_TOKEN_ENDPOINT_AUTH_METHODS_SUPPORTED

### Exemplos didáticos (sem código)
Exemplo feliz, create:
1) A plataforma UCP envia o carrinho para a Plataforma de Agentes de IA.
2) A Plataforma de Agentes de IA chama o PDV em UCP_PDV_CREATE_CHECKOUT_PATH.
3) O PDV retorna totals e status ready_for_complete.
4) A Plataforma de Agentes de IA devolve o mesmo checkout para a plataforma.

Exemplo de erro, payload inválido:
1) A Plataforma de Agentes de IA envia line_items sem item.id.
2) O PDV responde com erro de validação ou payload incompleto.
3) A Plataforma de Agentes de IA retorna 502 indicando resposta inválida do PDV.

Exemplo de erro, PDV indisponível:
1) A Plataforma de Agentes de IA tenta chamar o PDV.
2) A chamada falha por timeout ou erro HTTP.
3) A Plataforma de Agentes de IA devolve 502 e registra o correlation_id no log.

### Impacto para o usuário
Quando o PDV está bem implementado, o checkout fica consistente e previsível. O comprador vê preços reais, disponibilidade correta e status confiável. O time de integração passa a ter um fluxo único para tratar criação, atualização e finalização de pedidos. Isso reduz retrabalho e melhora a taxa de sucesso do checkout.

### Limites e pegadinhas
1) A Plataforma de Agentes de IA sempre exige resposta compatível com PdvCheckoutResponse.
2) O PDV precisa calcular totals após qualquer mudança; a Plataforma de Agentes de IA não calcula.
3) O template dos endpoints deve aceitar {checkout_id} sem variação.
4) A propagação de Idempotency-Key ocorre em Create, Update, Complete e Cancel; o PDV precisa implementar a semântica idempotente final.
5) Se o PDV retornar JSON inválido, a Plataforma de Agentes de IA responde 502.

### Troubleshooting focado no PDV
Sintoma: Erro 500 com "Configuração PDV inválida".
Ação: conferir UCP_PDV_BASE_URL e os caminhos no .env.

Sintoma: Erro 502 com "Resposta PDV inválida".
Ação: validar se a resposta segue PdvCheckoutResponse.

Sintoma: Erro 502 com "Falha ao chamar PDV".
Ação: validar disponibilidade do PDV e timeout configurado.

### Regras de contrato que não podem ser quebradas
1) O PDV é a fonte de verdade para preço, taxa, disponibilidade e horário de atendimento.
2) O PDV deve retornar totals coerentes e já recalculados após cada Update, Complete e Cancel.
3) O PDV deve retornar status compatível com o ciclo do UCP e mensagens claras quando houver bloqueio.
4) O PDV deve fornecer continue_url quando status for requires_escalation.

## Exemplos de uso
Exemplo feliz, criação de checkout:
1) Plataforma inicia checkout chamando o endpoint de criação na Plataforma de Agentes de IA.
2) A Plataforma de Agentes de IA encaminha o payload para o PDV seguindo o contrato.
3) O PDV retorna itens completos, totais e status ready_for_complete.
4) A plataforma usa o response para continuar o fluxo e coletar pagamento.

Exemplo de erro comum, dados incompletos:
1) Plataforma envia item sem informação suficiente para preço real.
2) O PDV retorna status incomplete com messages informando o erro.
3) A plataforma corrige o payload e chama Update Checkout.
4) O PDV reavalia e retorna status pronto ou exige handoff.

Exemplo de atualização com fulfillment:
1) Plataforma envia buyer atualizado e fulfillment indicando delivery.
2) O PDV calcula taxas de entrega e serviço, valida horário e disponibilidade.
3) O PDV retorna totals recalculados e fulfillment com opções válidas.
4) A plataforma segue para complete ou para handoff via continue_url.

Exemplo de erro com handoff:
1) O PDV identifica regra que exige revisão do comprador.
2) O PDV retorna status requires_escalation e continue_url.
3) A plataforma redireciona o comprador para o checkout do lojista.

## Guia e tutorial de integração (passo a passo)
Esta seção consolida o roteiro de integração para consultores e equipes de implantação. Ela combina requisitos de Merchant Center, configuração da Plataforma de Agentes de IA e integração com PDV/ERP.

### Etapa 1: validar pré-requisitos do Merchant Center
1) Confirmar conta ativa e produtos aprovados para free listings.
2) Configurar return policy e customer support info, conforme políticas obrigatórias.
3) Garantir que o feed contém native_commerce e, quando aplicável, consumer_notice.
4) Validar o mapeamento de IDs entre feed e o ID usado no checkout.

### Etapa 2: configurar discovery e OAuth
1) Definir UCP_PROTOCOL_VERSION e UCP_REST_BASE_PATH.
2) Definir UCP_CAPABILITIES com as capacidades habilitadas.
3) Configurar UCP_SIGNING_KEYS_JSON quando houver assinatura de payloads.
4) Preencher as variáveis UCP_OAUTH_* para publicar metadados OAuth completos.

### Etapa 3: integrar o PDV ao checkout
1) Configurar UCP_PDV_BASE_URL e caminhos dos endpoints do PDV.
2) Garantir que o PDV aceite os payloads de create, update e complete.
3) Garantir que o PDV retorne PdvCheckoutResponse com totals, status e payment.
4) Garantir que o PDV forneça continue_url quando houver requires_escalation.

### Etapa 4: integrar eventos pós-compra
1) Configurar UCP_ORDER_WEBHOOK_URL ou informar webhook_url no payload.
2) Garantir que o ERP envie event_type, checkout_id e order.
3) Validar que o webhook é disparado com o correlation_id correto.

### Etapa 5: validar o fluxo fim a fim
1) Verificar o Business Profile em /.well-known/ucp com version e capabilities.
2) Validar o OAuth metadata em /.well-known/oauth-authorization-server.
3) Executar o fluxo de checkout com create, get, update, complete e cancel.
4) Simular eventos pós-compra e validar o recebimento do webhook.

## Passo a passo para testar em sandbox
Este roteiro valida a integração sem impactar ambiente real. Ele serve para confirmar que a Plataforma de Agentes de IA e o PDV estão conversando corretamente e que o checkout segue o contrato do UCP.

### Visão geral
O objetivo do sandbox é garantir que a integração funciona do começo ao fim antes de ir para produção. Isso inclui publicar o Business Profile, responder aos endpoints de checkout, validar o webhook de eventos e confirmar que o PDV devolve o objeto completo de checkout.

### Passo a passo (sem detalhes de tela)
1) Separar um ambiente de teste do PDV com dados controlados.
2) Preencher as variáveis UCP_* no .env desse ambiente com URLs e credenciais de sandbox.
3) Publicar o Business Profile e conferir se o endpoint /.well-known/ucp responde sem erro.
4) Disparar um create de checkout e conferir se o PDV retorna totals e status corretos.
5) Executar update, complete e cancel, validando mudanças de status e totals.
6) Enviar um evento pós-compra e confirmar que o webhook recebe o payload.
7) Revisar logs com correlation_id para garantir rastreabilidade.

### Explicação for dummies
Pense no sandbox como um “campo de treino” onde nada real acontece. Você envia um carrinho de teste, o PDV calcula o preço e você vê se tudo volta certinho. Se falhar aqui, não dá para liberar produção. Quando todos os passos passam, o fluxo está pronto para a vida real.

## Passo a passo para colocar em produção
Este roteiro orienta a transição do ambiente de teste para o ambiente real, com foco em estabilidade e segurança.

### Visão geral
Em produção, o checkout impacta clientes reais. Por isso, a prioridade é garantir que as variáveis do .env estão corretas, que o PDV responde dentro do tempo esperado e que o Merchant Center está aprovado e sem pendências.

### Passo a passo (sem detalhes de tela)
1) Validar se o Merchant Center está em estado aprovado e sem alertas.
2) Confirmar que o feed de produção contém native_commerce e demais atributos exigidos.
3) Configurar UCP_* no .env de produção com endpoints e credenciais reais.
4) Publicar o Business Profile em produção e validar a resposta do endpoint.
5) Executar um fluxo controlado de checkout com poucos itens para validar totals e status.
6) Confirmar que o webhook de eventos está recebendo notificações do ERP.
7) Monitorar logs e métricas nas primeiras horas de operação.

### Explicação for dummies
Produção é a “loja aberta”. Antes de abrir a porta, você confere se o cadastro está certo, se o caixa funciona e se o endereço está correto. Depois abre em modo controlado, com poucos testes reais. Se tudo estiver ok, mantém o fluxo aberto para todos.

## Como validar sem ambiguidades
1) Discovery
  - O Business Profile deve conter ucp.version, ucp.services e ucp.capabilities.
  - O endpoint OAuth deve listar issuer, authorization_endpoint e token_endpoint.
2) Checkout
  - O PDV deve retornar totals coerentes para cada mudança no checkout.
  - O status deve seguir o ciclo UCP e ser consistente com o evento recebido.
3) Pós-compra
  - O evento deve ser aceito com status 202.
  - O webhook deve receber o payload com event_type e checkout_id.

## Impacto para o usuário
A integração UCP aumenta a previsibilidade de checkout e reduz discrepâncias de preço. Consultores conseguem validar rapidamente se o Business Profile está publicado e se os endpoints estão disponíveis. O fluxo fica transparente: o PDV é responsável pelos valores, e a Plataforma de Agentes de IA apenas orquestra. Isso melhora a governança e a rastreabilidade, especialmente com correlation_id nos logs.

## Limites e pegadinhas
- O Business Profile só publica capabilities presentes em UCP_CAPABILITIES.
- A Plataforma de Agentes de IA não calcula preço, desconto ou imposto sem resposta do PDV.
- O PDV deve responder com totals coerentes, caso contrário o checkout fica inconsistente.
- O continue_url é obrigatório quando status for requires_escalation.
- Create, Get, Update, Complete e Cancel dependem do PDV e de UCP_PDV_*.
- O header Idempotency-Key é propagado nos endpoints mutáveis; a semântica final depende da implementação do PDV.

## Troubleshooting
Sintoma: Business Profile não aparece em /.well-known/ucp.
Ação: confirmar UCP_PROTOCOL_VERSION e UCP_REST_BASE_PATH no .env.

Sintoma: Checkout retorna erro 500 ou 502.
Ação: confirmar se UCP_PDV_* está preenchido e se o PDV está respondendo com PdvCheckoutResponse válido.

Sintoma: PDV retorna erro de estado não elegível.
Ação: confirmar o status atual da sessão e garantir que o fluxo está respeitando o ciclo de estados definido no UCP.

Sintoma: Idempotency-Key não é respeitado pelo PDV.
Ação: validar se o PDV reconhece o header em Create/Update/Complete/Cancel e se a semântica idempotente está implementada no backend do lojista.

Sintoma: Totais inconsistentes no checkout.
Ação: validar se o PDV retorna line_items e totals compatíveis com as regras do UCP.

Sintoma: Handoff não acontece quando deveria.
Ação: garantir que o PDV retorne status requires_escalation e continue_url.

## Referências oficiais
Google UCP:
- https://developers.google.com/merchant/ucp/guides/checkout
- https://developers.google.com/merchant/ucp/guides/checkout/native
- https://developers.google.com/merchant/ucp/guides/checkout/embedded
- https://developers.google.com/merchant/ucp/guides/business-profile
- https://developers.google.com/merchant/ucp/guides/merchant-center
- https://developers.google.com/merchant/ucp/guides/orders
- https://developers.google.com/merchant/ucp
- https://developers.google.com/blog/under-the-hood-universal-commerce-protocol-ucp/

UCP Specification:
- https://ucp.dev/specification/checkout/
- https://ucp.dev/specification/checkout-rest/
- https://ucp.dev/specification/fulfillment/
- https://ucp.dev/specification/overview/

Exemplos oficiais:
- https://github.com/Universal-Commerce-Protocol/samples/tree/main/rest/python/server
