# MCP (Model Context Protocol) na Plataforma de Agentes de IA

Produto: Plataforma de Agentes de IA

## Visão geral

O MCP é o mecanismo que permite adicionar ferramentas externas à Plataforma de Agentes de IA sem trocar o fluxo principal de execução. Ele funciona como uma extensão do catálogo de tools já existente, ativada por configuração no YAML. Quando habilitado, as tools MCP são carregadas e somadas às tools locais do agente ou workflow. Isso mantém a previsibilidade do sistema, mas amplia o repertório disponível quando há servidores MCP configurados. A configuração é feita por escopo, com precedência clara entre global, supervisor, agente e workflow.

## Por que existe

O MCP existe para permitir que a Plataforma de Agentes de IA consuma tools expostas por servidores externos de forma controlada e configurável. Isso evita duplicação de integrações e acelera casos de uso que dependem de fontes externas. Ele também permite ligar ou desligar servidores MCP sem impactar o restante do catálogo. Em termos operacionais, o MCP traz flexibilidade sem romper a governança já definida no YAML.

## Explicação conceitual

Na Plataforma de Agentes de IA, o MCP é resolvido como uma camada complementar de tools. O sistema carrega as tools locais, resolve as tools MCP do escopo ativo e faz o merge evitando conflitos de nome. Se ocorrer conflito, a tool MCP é ignorada e um warning é registrado no log. As tools MCP ficam em cache por tenant para reduzir custo de descoberta. A habilitação e os detalhes de transporte são definidos no YAML, e podem ser sobrescritos por escopo.

## Explicação for dummies

Pense no MCP como um conjunto extra de ferramentas que você pode plugar na Plataforma de Agentes de IA quando precisa de algo fora do catálogo padrão. Você não troca as ferramentas que já usa; você só adiciona novas opções. É como ter uma caixa de ferramentas extra que pode ser ligada e desligada conforme o projeto. Se duas ferramentas tiverem o mesmo nome, o sistema mantém a ferramenta local e ignora a do MCP. Para não ter surpresa, tudo isso é configurado no YAML. Assim, o consultor consegue ativar ou desativar MCP sem tocar no código.

## Como o usuário recebe essa feature

1. O MCP é ativado no YAML com o bloco `global_mcp_configuration`.
2. O escopo pode ser refinado com `local_mcp_configuration` no supervisor, agente ou workflow.
3. Os servidores MCP são declarados em `servers`, com transporte e parâmetros específicos.
4. As tools MCP passam a ficar disponíveis junto das tools locais no agente ou workflow.
5. Caso haja conflito de nome, o sistema registra warning e mantém a tool local.

## Exemplo de uso

Caso de uso: um agente de documentação precisa consultar fontes externas de forma oficial.

Passos práticos:

1. O YAML define um servidor MCP remoto de documentação AWS no bloco global.
2. Um agente especialista lista as tools MCP de documentação no seu bloco `tools`.
3. O agente executa chamadas de busca e leitura de documentação via MCP.
4. O resultado entra no fluxo normal do supervisor, sem mudança de arquitetura.

Referências de configuração já existentes:

- Exemplo de MCP global e agentes em [app/yaml/rag-config-mrctito-dnit.yaml](app/yaml/rag-config-mrctito-dnit.yaml).
- Exemplo de MCP global em [app/yaml/system/rag-config-modelo.yaml](app/yaml/system/rag-config-modelo.yaml).

## Exemplo de erro

Sintoma: MCP habilitado sem servidores configurados.

Efeito: a resolução de MCP falha porque não há servidores disponíveis no escopo.

Onde verificar: bloco `global_mcp_configuration` e a lista `servers` no YAML ativo.

## O MCP de exemplo que usamos aqui

O exemplo ativo nos YAMLs é o servidor `aws_knowledge_mcp`, configurado com transporte HTTP remoto. Ele é usado para fornecer tools de documentação AWS no agente especialista de documentação. As tools listadas no agente incluem `search_documentation`, `read_documentation`, `recommend`, `list_regions` e `get_regional_availability`. Esse exemplo mostra como consumir documentação técnica de forma direta via MCP, mantendo o fluxo da Plataforma de Agentes de IA intacto.

Há também um exemplo local chamado `agora_mcp`, configurado com transporte `stdio` e mantido desativado nos YAMLs porque depende de execução local. Ele fica como referência de formato e não impacta a operação quando permanece desativado.

## Proxy HTTP para MCP stdio

A Plataforma de Agentes de IA expõe servidores MCP em modo `stdio` por meio de um endpoint HTTP interno. Essa camada permite que clientes remotos consumam tools MCP sem modificar o servidor MCP original. Cada chamada abre uma sessão isolada e encerra ao final da requisição, preservando o isolamento multi-tenant. O catálogo de tools fica em cache por tenant pelo tempo configurado em `cache_ttl_s`.

Parâmetros esperados no endpoint:

- `yaml_config`: caminho do arquivo YAML que contém o bloco MCP.
- `mcp_scope`: define o escopo de resolução, como `global`, `agent` ou `workflow`.
- Quando `mcp_scope` é `agent`, é obrigatório informar `supervisor_id` e `agent_id`.
- Quando `mcp_scope` é `workflow`, é obrigatório informar `workflow_id`.

Permissões aplicadas:

- Listagem de tools exige `mcp.tools.list`.
- Execução de tools exige `mcp.tools.invoke`.

Limites operacionais:

- Apenas servidores MCP com transporte `stdio` são expostos pelo proxy.
- Em caso de nomes duplicados sem `tool_name_prefix`, a primeira tool prevalece.

Exemplos onde o agente de documentação AWS já está configurado:

- [app/yaml/rag-config-mrctito-dnit.yaml](app/yaml/rag-config-mrctito-dnit.yaml).
- [app/yaml/rag-config-mrctito-food.yaml](app/yaml/rag-config-mrctito-food.yaml).
- [app/yaml/agent-linx-food-cockpit-prd.yaml](app/yaml/agent-linx-food-cockpit-prd.yaml).

## Impacto para o usuário

O MCP amplia o leque de ferramentas disponíveis sem alterar o modelo de operação da Plataforma de Agentes de IA. Isso reduz tempo de integração, melhora cobertura de fontes externas e mantém governança. Para consultores, significa mais opções de solução com menor esforço de implementação. Para o usuário final, significa respostas mais completas quando a demanda depende de fontes externas.

## Limites e pegadinhas

- MCP habilitado sem servidores configurados causa falha de resolução.
- Conflito de nome faz a tool MCP ser ignorada, com warning no log.
- O cache de tools MCP é por tenant; mudanças nos servidores podem exigir limpeza de cache.
- Um servidor MCP desabilitado no YAML é ignorado pelo resolver.

## Troubleshooting

Sintoma: tools MCP não aparecem no agente.

Diagnóstico: verificar se `enabled` está `true` no escopo correto e se há servidores em `servers`.

Solução: habilitar o bloco correto e conferir se o servidor MCP está ativo.

Sintoma: tool MCP esperada não executa.

Diagnóstico: conferir conflito de nome com tool local e procurar warnings de conflito.

Solução: usar `tool_name_prefix` ou ajustar o nome da tool MCP.

## Referências internas

- Configuração MCP no YAML: [docs/README-CONFIGURACAO-YAML.md](docs/README-CONFIGURACAO-YAML.md).
- Resolução e merge de tools MCP: [src/agentic_layer/mcp/mcp_tools_resolver.py](src/agentic_layer/mcp/mcp_tools_resolver.py).
- Resolução de config MCP por escopo: [src/agentic_layer/mcp/mcp_config_resolver.py](src/agentic_layer/mcp/mcp_config_resolver.py).
- Merge de tools locais e MCP: [src/agentic_layer/supervisor/tools_factory.py](src/agentic_layer/supervisor/tools_factory.py).
