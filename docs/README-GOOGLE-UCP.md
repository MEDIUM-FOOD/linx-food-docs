# Integração com Google UCP

Produto: Plataforma de Agentes de IA

## O que esta feature é

Esta integração expõe a Plataforma de Agentes de IA como um endpoint compatível com o Universal Commerce Protocol, permitindo discovery de capacidades, operação de checkout e recebimento de eventos pós-compra.

No código atual, isso é implementado por três frentes complementares:

1. publicação do manifesto UCP e dos metadados OAuth em rotas well-known;
2. binding REST de checkout sessions com repasse para o PDV;
3. endpoint de eventos de pedido disparados pelo ERP.

## Que problema ela resolve

Sem essa camada, a plataforma até poderia conversar com sistemas internos de comércio, mas não se apresentaria de forma padronizada para um ecossistema externo que espera o contrato UCP. Isso criaria integração sob medida, maior custo operacional e risco de divergência entre o que o protocolo espera e o que o backend realmente entrega.

A camada UCP resolve isso ao separar discovery, checkout e eventos em contratos explícitos.

## Explicação simples

Pense no UCP como uma linguagem comum de checkout.

A Plataforma de Agentes de IA publica um cartão de apresentação dizendo o que suporta. Quando alguém inicia a compra, a plataforma recebe a requisição UCP, encaminha ao PDV, recebe a resposta estruturada e devolve no formato esperado. Depois da compra, o ERP pode avisar eventos do pedido e a plataforma repassa isso para o webhook configurado.

## Conceitos necessários

### Business Profile

É o manifesto publicado em `/.well-known/ucp`. Ele informa versão do protocolo, endpoint REST e capacidades habilitadas.

### Metadados OAuth

São publicados em `/.well-known/oauth-authorization-server` para permitir discovery automático do servidor OAuth.

### Checkout Session

É a sessão de compra exposta no binding REST. Ela pode ser criada, consultada, atualizada, concluída e cancelada.

### Idempotência

Os endpoints mutáveis usam o header `Idempotency-Key` para evitar efeitos duplicados quando a mesma requisição é repetida.

## Como a feature funciona por dentro

O ponto de discovery fica em `ucp_router.py`.

Ali, `UcpBusinessProfileBuilder` monta o manifesto com base no host atual e nas variáveis de ambiente. O builder calcula:

1. a versão do protocolo;
2. o endpoint REST final com base em `UCP_REST_BASE_PATH`;
3. a lista de capacidades habilitadas;
4. as chaves de assinatura, quando configuradas.

O ponto de checkout fica em `ucp_checkout_router.py`.

Esse router registra cinco operações:

1. criar sessão;
2. consultar sessão;
3. atualizar sessão;
4. concluir sessão;
5. cancelar sessão.

Cada uma dessas operações valida payload, resolve `correlation_id`, lê a chave de idempotência quando aplicável e então segue por um de dois caminhos:

1. cliente PDV normal, quando a configuração do PDV está disponível;
2. repositório fallback, quando o router detecta que deve operar nesse modo.

O ponto de pós-compra fica em `ucp_order_event_router.py`.

Esse router recebe o payload do ERP, valida `event_type` e `checkout_id`, resolve configuração do webhook e despacha o evento com `OrderEventDispatcher`. Quando o payload é inválido ou o webhook está mal configurado, o router falha de forma explícita.

## Papel do contrato PDV

A plataforma não calcula o checkout sozinha. O contrato operacional real está em `pdv_checkout_api.py`.

Esse módulo define:

1. requests de create, update e complete;
2. response tipada do checkout;
3. entidades como buyer, items, line_items, payments, totals, links e fulfillment;
4. classes de erro para falha de configuração, falha de chamada e resposta inválida do PDV.

Na prática, isso significa que a plataforma faz o papel de orquestração protocolar, enquanto o PDV continua sendo a fonte de verdade do checkout calculado.

## Endpoints expostos

Os endpoints confirmados no código são estes:

1. `GET /.well-known/ucp` para o Business Profile;
2. `GET /.well-known/oauth-authorization-server` para discovery OAuth;
3. `POST {base}/checkout-sessions` para criar checkout;
4. `GET {base}/checkout-sessions/{checkout_id}` para consultar checkout;
5. `PUT {base}/checkout-sessions/{checkout_id}` para atualizar checkout;
6. `POST {base}/checkout-sessions/{checkout_id}/complete` para concluir checkout;
7. `POST {base}/checkout-sessions/{checkout_id}/cancel` para cancelar checkout;
8. `POST {base}/order-events` para receber eventos do ERP.

O `base` é definido por `UCP_REST_BASE_PATH` e usa `/ucp/v1` como default quando a variável não está definida.

## O que acontece em caso de sucesso

No caminho feliz:

1. o manifesto UCP é publicado com versão e capacidades válidas;
2. o checkout chega ao router correto;
3. o payload é validado contra os modelos tipados;
4. o PDV responde com checkout compatível;
5. a plataforma devolve o payload final sem campos nulos desnecessários;
6. eventos de pedido são aceitos e despachados com status HTTP 202.

## O que acontece em caso de erro

Os cenários de falha confirmados no código incluem:

1. configuração inválida do protocolo ou das chaves de assinatura, levando a erro 500 no discovery;
2. payload inválido de checkout, levando a erro 422;
3. `checkout_id` inválido, levando a erro 400;
4. resposta inválida do PDV, levando a erro 502;
5. falha de comunicação com o PDV, levando a erro 502;
6. conflito de idempotência no fallback, levando a erro 409;
7. webhook mal configurado para order events, levando a erro 500;
8. falha de despacho do webhook, levando a erro 502.

## Configurações que mudam comportamento

As variáveis mais importantes confirmadas no código são:

1. `UCP_PROTOCOL_VERSION`, que define a versão publicada no manifesto;
2. `UCP_REST_BASE_PATH`, que define o prefixo REST do binding UCP;
3. `UCP_CAPABILITIES`, que define quais capacidades entram no manifesto;
4. `UCP_SIGNING_KEYS_JSON`, que injeta chaves de assinatura no manifesto quando presente.

## Impacto técnico

Essa feature reduz acoplamento entre protocolo público e implementação interna do checkout. O router UCP conhece o contrato do protocolo, enquanto o PDV continua encapsulando regras de preço, itens, pagamentos e estado do checkout.

## Impacto executivo

A plataforma passa a conseguir se apresentar de forma padronizada para integrações de comércio, reduzindo custo de onboarding e risco de interpretação inconsistente do contrato de checkout.

## Impacto comercial

Do ponto de vista comercial, essa integração permite posicionar a plataforma como camada de orquestração pronta para ecossistemas que exigem contrato de checkout padronizado. Isso reduz a objeção clássica de integração proprietária ou de alto custo por cliente.

## Impacto estratégico

A camada UCP fortalece a estratégia de plataforma porque desacopla o protocolo público do motor interno de comércio. Isso abre caminho para troca de PDV, evolução de checkout e novos conectores mantendo o contrato externo estável.

## Troubleshooting

Sintoma: o manifesto well-known não sobe corretamente.

Verifique `UCP_PROTOCOL_VERSION`, `UCP_REST_BASE_PATH`, `UCP_CAPABILITIES` e, se houver, `UCP_SIGNING_KEYS_JSON`.

Sintoma: o checkout responde erro 502.

A causa mais provável é falha de comunicação com o PDV ou payload inválido retornado pelo PDV.

Sintoma: o webhook de order events não dispara.

Verifique se `event_type` e `checkout_id` estão presentes no payload e se a configuração do webhook está válida.

## Evidências no código

1. `src/api/routers/ucp_router.py`: manifesto UCP e metadados OAuth.
2. `src/api/routers/ucp_checkout_router.py`: binding REST de checkout sessions.
3. `src/api/routers/ucp_order_event_router.py`: recebimento e despacho de order events.
4. `src/ucp/pdv_checkout_api.py`: contrato tipado e cliente do PDV para checkout.
