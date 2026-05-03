# API Dinâmica

Este documento cobre a família dyn_api, usada para expor endpoints HTTP
aprovados para agentes e workflows.

## O que essa família resolve

Ela evita criar uma integração nova em código para cada endpoint externo
que possa ser descrito por contrato.

No runtime atual, dyn_api também pode resolver operação publicada em
registro governado por tenant quando ela não está no YAML local.

## Leitura relacionada

- Guia geral de tools: [GUIA-USUARIO-TOOLS.md](./GUIA-USUARIO-TOOLS.md)
- Catálogo governado por tenant: [README-INTEGRACOES-GOVERNADAS.md](./README-INTEGRACOES-GOVERNADAS.md)
- SQL dinâmico e procedures: [README-DYNAMIC-SQL-TOOLS.md](./README-DYNAMIC-SQL-TOOLS.md)
- Manual conceitual especializado: [README-CONCEITUAL-DYNAMIC-API-TOOLS.md](./README-CONCEITUAL-DYNAMIC-API-TOOLS.md)
- Manual técnico especializado: [README-TECNICO-DYNAMIC-API-TOOLS.md](./README-TECNICO-DYNAMIC-API-TOOLS.md)
- Tools por finalidade: [tools/por_finalidade.md](./tools/por_finalidade.md)
- Catálogo alfabético: [tools/alfabetica.md](./tools/alfabetica.md)

## Leitura especializada recomendada

Este manual continua sendo a porta de entrada do tema dyn_api. Para aprofundamento por finalidade, use a trilha especializada abaixo.

- [README-CONCEITUAL-DYNAMIC-API-TOOLS.md](./README-CONCEITUAL-DYNAMIC-API-TOOLS.md): explica dyn_api como capability governada, incluindo valor executivo, comercial e estratégico.
- [README-TECNICO-DYNAMIC-API-TOOLS.md](./README-TECNICO-DYNAMIC-API-TOOLS.md): detalha a arquitetura de runtime, o fallback governado, autenticação, boundaries administrativos, troubleshooting e limites operacionais.

## Visão executiva

Executivamente, dyn_api reduz custo de integração repetida. Em vez de
pedir um deploy novo para cada endpoint externo homologado, o produto
pode publicar operações governadas por tenant com contrato, autenticação
e guardrails já descritos.

## Visão comercial

Comercialmente, essa família ajuda a vender velocidade com controle. O
cliente percebe ganho porque novas integrações simples deixam de exigir
um conector dedicado em código, mas a plataforma continua impondo
catálogo, autenticação válida e filtro por publicação para agentes.

## Visão estratégica

Estrategicamente, dyn_api é uma peça da tese YAML-first governada. Ela
permite expandir capacidade agentic por configuração e catálogo
persistido, sem transformar cada nova API externa em ramo especial do
runtime.

## Sintaxe pública no YAML

- dyn_api<endpoint_id>

O modo parametrizado é o caminho mais seguro porque entrega apenas o
endpoint necessário ao agente.

## Ordem real de resolução

O runtime tenta primeiro tools_config.api_dynamic.endpoints.
Se o endpoint não existir no YAML, ele tenta o registro governado por
tenant.

No caminho persistido, o código exige:

- user_session.tenant_id;
- publish_to_agents=true;
- protocol_type=rest_json.

Se houver auth_profile_id, o perfil de autenticação também precisa ser
resolvido.

## Onde a configuração vive

O bloco principal dessa família é tools_config.api_dynamic.

As partes mais importantes são:

- endpoints;
- authentications.

O endpoint pode descrever método, URL, parâmetros de path, query, body,
headers, timeout e autenticação.

## Como a execução é montada

![Como a execução é montada](assets/diagrams/docs-readme-dynamic-api-tools-diagrama-01.svg)

## Guardrails importantes

- nome interno com prefixo esperado;
- endpoint_id obrigatório;
- URL obrigatória;
- autenticação válida quando o endpoint declara authentication;
- validação de parâmetros antes da chamada externa;
- retry e timeout no cliente HTTP.

No registro governado, dyn_api falha fechado para protocol_type fora de
rest_json.

## Autenticação e cache

Quando o endpoint exige autenticação, a família usa
AuthenticationManager.

Na prática, isso traz três efeitos importantes:

- token pode ser reaproveitado;
- placeholders em headers podem usar security_keys;
- erro de autenticação fica separado de erro de rede ou de cadastro.

## Contrato de parâmetros

A factory monta schema dinâmico para path, query e body.
Isso reduz inventação de campos fora do contrato.

Em linguagem simples: se o agente tentar mandar parâmetro que não existe
ou esquecer um obrigatório, a falha deve acontecer antes da API externa.

## Quando usar dyn_api e quando não usar

- Use dyn_api quando o endpoint for conhecido e o contrato for
    declarativo.
- Evite dyn_api quando a integração exigir fluxo altamente customizado,
    transformação pesada de resposta ou política de segurança que o YAML
    não consiga descrever com clareza.

## Como validar

1. Confirme se o endpoint está no YAML ou no registro governado.
2. Se vier do registro, confirme tenant_id, publish_to_agents e
     protocol_type.
3. Confirme se método, URL e timeout fazem sentido.
4. Confirme se a autenticação foi resolvida.
5. Em runtime, siga o correlation_id para separar erro de cadastro,
     autenticação e rede.

## Troubleshooting

### A tool dyn_api não aparece para o agente

Causa provável: o endpoint não foi publicado no YAML local nem no
registro governado com `publish_to_agents=true`.

Como confirmar: valide primeiro a trilha de resolução em
`tools_config.api_dynamic.endpoints` e depois o cadastro governado por
tenant.

### A operação existe, mas falha antes da chamada externa

Causa provável: schema dinâmico rejeitou parâmetro ausente, nome de campo
inválido ou autenticação mal resolvida.

Como confirmar: compare o payload informado com o contrato montado pela
factory e confirme se headers, query e body exigidos foram declarados.

### O cadastro governado existe, mas continua ignorado

Causa provável: `protocol_type` fora de `rest_json`, ausência de
`tenant_id` no contexto ou perfil de autenticação não resolvido.

Como confirmar: revise a linha do registro governado e confirme se ela
está ativa, publicada e compatível com o resolver.

## Checklist de entendimento

- Entendi quando dyn_api usa YAML local e quando cai no registro governado.
- Entendi por que `publish_to_agents` e `protocol_type` importam.
- Entendi a diferença entre erro de contrato, erro de autenticação e erro de rede.
- Entendi quando dyn_api acelera a entrega e quando ainda faz mais sentido um conector dedicado.

## Evidência no código

- src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py
- src/agentic_layer/tools/domain_tools/dynamic_tool_registry_resolver.py
- src/agentic_layer/tools/domain_tools/dynamic_api_tools/auth_manager.py
- src/agentic_layer/tools/domain_tools/dynamic_api_tools/http_client.py
- src/agentic_layer/supervisor/tool_loader.py
- src/integrations/repository.py

## Lacunas no código

Não encontrado no código.

Onde deveria estar:

- uma visão administrativa única para diagnosticar endpoint, auth
    profile e publicação para agentes no mesmo painel;
- um inventário consolidado das operações dyn_api disponíveis por
    tenant.
