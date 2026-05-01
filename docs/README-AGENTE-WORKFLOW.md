# Workflow Agentic Determinístico

Este manual explica o workflow como mecanismo de execução determinística do produto. O objetivo não é ensinar a caçar arquivo. É mostrar como o YAML de workflow vira decisão, validação, grafo executável e trilha operacional confiável.

## 1. Escopo e Seleção de Workflow

Workflow é a espinha dorsal usada quando o produto precisa dizer explicitamente em que ordem as coisas acontecem, quais decisões desviam o fluxo e quais dados entram e saem de cada passo. O ponto de partida do runtime é selected_workflow. Ele decide qual item de workflows fica ativo. workflows_defaults existe para empurrar configuração comum sem duplicar texto, e local_tools_configuration existe para ajustar comportamento de ferramenta no escopo do workflow, não no documento inteiro.

A decisão de qual workflow roda não é um detalhe de conveniência. Ela muda o grafo inteiro. Um selected_workflow errado significa montar o processo errado desde o primeiro node, mesmo quando o YAML parece sintaticamente válido.

## 2. Modos de Execução

O runtime trabalha com dois modos. O primeiro é o fluxo node-driven, em que a sequência nasce do próprio tipo de node e de campos internos como router.go_to_node, true_go_to, false_go_to, loop_more_label e loop_done_label. O segundo é o edge-first, em que workflows[].edges assume a responsabilidade pela transição e o grafo passa a obedecer relações declarativas.

Essa separação existe porque o produto precisa suportar dois tipos de modelagem. Em alguns fluxos, a ordem implícita entre nodes basta. Em outros, o mais importante é declarar claramente cada aresta e reduzir ambiguidade sobre transição. O runtime registra isso em log e trata WORKFLOW_EDGE_TRANSITION como parte observável do comportamento.

## 3. Contrato `workflows[].edges` (Edge-first)

Quando workflows[].edges está presente com conteúdo, o workflow entra em edge-first. Cada aresta usa from, to, when e default para dizer de onde o fluxo sai, para onde vai, qual condição ativa o salto e qual caminho serve como padrão. O valor desse desenho é tirar da cabeça do leitor a obrigação de inferir transições olhando múltiplos nodes ao mesmo tempo.

Na prática, from e to modelam o grafo. when declara decisão condicional. default impede dead end quando mais de uma hipótese não casa. O validador checa conflitos, referências inexistentes, defaults duplicados e erros de modelagem antes da execução real.

## 4. Estado e Metadados do Workflow

O workflow não passa estado de forma mágica. Ele mantém mensagens, variables, context, last_output e metadata como contrato vivo da execução. Dois campos operacionais merecem atenção especial: workflow_path e execution_trace. workflow_path conta o caminho visitado pelo grafo. execution_trace registra eventos como início, sucesso ou falha por node. max_iterations atua como freio contra loops descontrolados.

Esse estado não existe apenas para o resultado final. Ele existe para diagnóstico. Sem metadata, workflow_path e execution_trace, o operador vê só que o fluxo falhou. Com eles, fica possível localizar em que node a falha apareceu e se o problema foi decisão, dado, expressão ou integração.

## 5. Campos Comuns de Node

Todo node começa pelas mesmas perguntas. Qual é seu id? Qual mode ele usa? O que ele lê? O que ele escreve? Há prompt.system? Existe retry_policy? Existe human_approval? Há output_schema? Em alguns casos também entra auto_retry.

O valor desses campos não está na sintaxe em si. Está na clareza operacional. O runtime sabe que pode exigir prompt.system para agent, router, planner e executor. Sabe também que retries e saídas estruturadas não são globais por acidente; eles precisam ser declarados no lugar certo para o node certo.

## 6. Referência Completa de Nodes

### Node `agent`

O node agent existe para produzir raciocínio ou resposta com apoio de prompt.system. Ele é o ponto em que o workflow pede interpretação ao modelo. Quando a saída precisa seguir formato rígido, output_schema entra como contrato. Quando o cenário aceita repetição controlada, auto_retry e retry_policy ajudam a distinguir erro recuperável de erro estrutural.

### Node `router`

O router existe para classificar e decidir caminho. Ele precisa de router.allowed_labels para deixar claro quais saídas são aceitáveis. Em modo node-driven, router.go_to_node define para qual node cada label aponta. router.fallback_node cobre o caso em que a classificação não resolve o caminho esperado. Em edge-first, o router continua decidindo, mas a transição final passa para as edges.

### Node `planner`

O planner quebra um problema em passos menores. settings.output_key diz onde o plano será salvo, settings.cursor_key guarda progresso e settings.coerce_json ajuda quando o retorno precisa ser normalizado como estrutura em vez de texto cru. Esse node existe para separar decisão tática de execução efetiva.

### Node `executor`

O executor consome o plano e executa passo a passo. settings.failure_policy define o que fazer quando uma etapa falha. settings.retry_policy trata repetição do próprio executor. loop_more_label e loop_done_label existem para o caso node-driven em que a execução precisa continuar ou encerrar conforme o resultado. Em edge-first, a lógica do executor permanece, mas as setas saem de workflows[].edges.

### Node `if`

O if existe para bifurcação determinística. condition define a expressão booleana. true_go_to e false_go_to definem os destinos em modo node-driven. Em edge-first, o valor da condição continua existindo, mas os saltos são modelados em arestas explícitas.

### Node `set`

O set existe para materializar atribuição simples no estado. params.assign descreve o que será escrito. Ele é útil quando o workflow precisa produzir dados intermediários sem chamar modelo nem tool externa.

### Node `merge`

O merge junta ramos ou coleções de dados. params.strategy explica como a união acontece e params.aliases ajuda a nomear entradas diferentes antes da recomposição. Ele existe para impedir que a convergência do fluxo vire concatenação implícita e opaca.

### Node `function`

O function resolve computação local controlada. params.expression cobre casos simples. params.script cobre casos mais ricos. timeout_seconds impede função longa ou travada. result_var define onde o resultado final fica. Esse node existe para evitar que cada transformação pequena vire tool externa desnecessária.

### Node `transform`

O transform ajusta payloads e mensagens entre etapas. filter_messages existe para limpar ruído, json_map para remapear campos, regex_replace para correções dirigidas e string_ops para transformações textuais menores. Ele existe porque parte importante do workflow não é decidir, e sim preparar dado para a próxima etapa.

### Node `rule_router`

O rule_router existe quando a decisão precisa ser declarada como regras explícitas. params.rules concentra as condições e labels. params.default_label cobre o caminho padrão. Ele é útil quando a equipe quer regras legíveis e auditáveis sem delegar toda a escolha ao modelo.

### Node `tool`

O tool aciona uma ferramenta já resolvida no catálogo. params.tool_id identifica qual capacidade entra. params.arguments descreve os insumos. params.extract controla como o resultado útil volta para o estado. Esse node existe para manter integração externa dentro do grafo sem dissolver o contrato do workflow.

### Node `schema_validator`

O schema_validator protege fronteiras de dado. params.schema descreve o formato esperado, params.source diz o que será validado e params.on_error define a reação. Ele evita que um payload inválido siga viagem e que o problema só apareça muito depois, em outro node mais difícil de diagnosticar.

### Node `sub_workflow`

O sub_workflow existe para delegar uma parte do processo a outro workflow. params.workflow_id escolhe o fluxo filho. inherit_variables decide quanto do estado sobe junto. result_path define onde a resposta do filho será gravada. Ele serve para modularizar sem abrir mão do controle declarativo.

### Node `whatsapp_media_resolver`

O whatsapp_media_resolver prepara mídia antes do envio. payload_path indica de onde ler o conteúdo. cache_ttl_seconds controla reaproveitamento. allow_url_fallback diz se a mídia pode continuar por URL quando o upload direto não for viável. channel explicita o canal de destino. Esse node existe porque mídia exige tratamento diferente de texto.

### Node `whatsapp_send`

O whatsapp_send é o ponto terminal de expedição. payload_path indica o pacote a enviar. write_path registra resultado operacional. product_list_key ajuda no caso de lista de produtos e text_key define o campo textual principal. Ele existe para que o envio final permaneça governado pelo grafo em vez de espalhado em chamadas soltas.

## 7. Fluxo principal, sucesso e erro

O caminho feliz começa em selected_workflow, passa pela carga do contexto ativo, entra na análise de integridade e só então permite que a etapa de criação dinâmica do grafo monte o StateGraph. Se a integridade falha, o runtime não tenta executar parcialmente. Ele registra erro estruturado, resume os problemas mais importantes e interrompe a inicialização.

No erro, a regra prática é separar três famílias. Erro de contrato, como node sem id ou mode inválido. Erro de modelagem, como destino inexistente, edge inconsistente ou regra legada. Erro de execução, que só aparece depois que o workflow íntegro já foi montado. Misturar essas famílias atrasa diagnóstico.

## 8. Matriz de Compatibilidade

A compatibilidade mais importante do workflow não é entre arquivos. É entre intenção declarada e modo de execução. selected_workflow precisa apontar para um fluxo válido. workflows_defaults precisa combinar com o shape dos workflows reais. local_tools_configuration precisa respeitar o que o catálogo permite. output_schema faz sentido sobretudo em nodes que geram resposta estruturada. retry_policy e auto_retry devem entrar onde o runtime realmente os entende. human_approval precisa aparecer nos pontos em que a pausa tem sentido operacional.

O contraste central é node-driven versus edge-first. Node-driven é mais curto para fluxos simples. Edge-first é mais explícito para grafos ricos e auditoria de transição. Os dois coexistem, mas não devem ser misturados sem intenção clara.

## 9. Observabilidade e troubleshooting

A melhor forma de investigar workflow é seguir esta sequência.

1. Confirmar selected_workflow e motivo de seleção.
2. Ler o relatório de integridade para ver node_count, router_nodes_checked, error_count e warning_count.
3. Verificar workflow_path e execution_trace para reconstruir o percurso.
4. Se houver edge-first, observar as transições registradas como WORKFLOW_EDGE_TRANSITION.
5. Só depois aprofundar no node específico.

Em linguagem simples: primeiro confirme se o grafo certo foi montado. Depois veja se ele estava íntegro. Só então discuta por que um passo individual falhou.

## 10. Explicação 101

Workflow aqui é como um roteiro de atendimento com bifurcações explícitas. Em vez de deixar cada passo decidir sozinho para onde ir de forma escondida, o sistema registra o caminho, valida a estrutura antes de começar e guarda rastros suficientes para alguém entender a história completa depois.

## 11. Evidências no código

- [src/agentic_layer/workflow/agent_workflow.py](../src/agentic_layer/workflow/agent_workflow.py): seleção do contexto ativo, análise de integridade e montagem do grafo executável.
- [src/agentic_layer/workflow/integrity_analyzer.py](../src/agentic_layer/workflow/integrity_analyzer.py): validações estáticas do contrato antes da execução.
- [src/agentic_layer/workflow/edge_compiler.py](../src/agentic_layer/workflow/edge_compiler.py): compilação das edges declarativas em conexões do StateGraph.
- [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py): seleção do target e montagem do documento agentic.
- [src/config/agentic_assembly/validators/workflow_semantic_validator.py](../src/config/agentic_assembly/validators/workflow_semantic_validator.py): validação forte da AST com regras do runtime.
