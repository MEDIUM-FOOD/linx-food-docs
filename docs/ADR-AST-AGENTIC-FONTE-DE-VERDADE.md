# ADR: AST como Fonte de Verdade no Escopo Agentic

Status: Aceita

Data: 2026-03-20

## 1. Contexto

O módulo de assembly agentic já opera sobre um fluxo tipado e validado:
1. parse do YAML para `AgenticDocumentAST`;
2. validação estrutural e semântica por alvo;
3. compilação para fragmento;
4. merge do fragmento no YAML final.

As evidências diretas no código e na documentação técnica lida são:
1. `src/config/agentic_assembly/ast/document.py` define o envelope canônico `AgenticDocumentAST` e o método `to_fragment()`.
2. `src/config/agentic_assembly/assembly_service.py` usa `validate` e `_select_fragment()` para produzir o fragmento compilado antes do `confirm`.
3. `docs/README-AGENTE-WORKFLOW.md`, `docs/README-AGENTE-SUPERVISOR.md` e `docs/README-DEEPAGENTS-SUPERVISOR.md` já documentam o mapeamento AST para os contratos YAML executados.

O problema prático do contrato anterior era este:
1. o YAML era tratado como verdade final de runtime;
2. a AST era tratada como verdade de edição e validação;
3. isso mantinha dois centros de decisão conceitual para o mesmo escopo agentic.

Na prática, isso abre espaço para drift entre:
1. schema exposto para UI;
2. AST aceita por `draft` e `validate`;
3. fragmento compilado em `confirm`;
4. YAML persistido e consumido pelo runtime.

## 2. Decisão

Fica decidido que, no escopo agentic controlado pelo módulo de assembly, a AST passa a ser a fonte de verdade do contrato.

Isso significa:
1. a AST é a referência oficial para edição, validação, schema, diff, revisão e aprovação;
2. o YAML persistido nesse escopo passa a ser artefato compilado a partir da AST;
3. o YAML compilado continua sendo o formato lido pelo runtime atual, mas deixa de ser tratado como contrato independente dentro do escopo governado;
4. fora do escopo agentic governado por esta ADR, o sistema continua YAML-first.

## 3. Escopo fechado de governança

Esta ADR vale apenas para o escopo agentic montado por `AgenticAssemblyService`.

Lista fechada de chaves raiz sob governança AST-first:
1. `selected_workflow`
2. `workflows_defaults`
3. `workflows`
4. `selected_supervisor`
5. `multi_agents`
6. `tools_library`

### 3.1 Subcontratos governados por alvo

#### Workflow

`workflows[]` passa a ser governado por `WorkflowAST`, `WorkflowNodeAST` e `WorkflowEdgeAST`.

Campos governados de `workflows[]`:
1. `id`
2. `name`
3. `description`
4. `enabled`
5. `settings`
6. `local_tools_configuration`
7. `local_mcp_configuration`
8. `nodes`
9. `edges`

Campos governados de `workflows[].nodes[]`:
1. `id`
2. `mode`
3. `prompt`
4. `reads`
5. `writes`
6. `tools`
7. `params`
8. `settings`
9. `router`
10. `retry_policy`
11. `human_approval`

Campos adicionais governados por modo quando presentes:
1. `output_schema`
2. `auto_retry`
3. `failure_policy`
4. `condition`
5. `true_go_to`
6. `false_go_to`
7. `source_mode`

Campos governados de `workflows[].edges[]`:
1. `from`
2. `to`
3. `when`
4. `default`

#### Supervisor clássico

`multi_agents[]` em modo clássico passa a ser governado por `SupervisorAST` e `AgentAST`.

Campos governados de `multi_agents[]`:
1. `id`
2. `name`
3. `description`
4. `enabled`
5. `execution`
6. `prompt`
7. `planner`
8. `agents`
9. `tools_library`
10. `local_tools_configuration`
11. `local_mcp_configuration`
12. `tenants`

Campos governados de `multi_agents[].agents[]`:
1. `id`
2. `name`
3. `description`
4. `prompt`
5. `system_prompt`
6. `tools`
7. `config`
8. `tools_config`
9. `limits`
10. `execution`
11. `local_mcp_configuration`

#### DeepAgent supervisor

No alvo `deepagent_supervisor`, `multi_agents[]` continua sendo a chave compilada de runtime, mas a origem de verdade passa a ser `DeepAgentSupervisorAST`.

Campos adicionais governados no modo deepagent:
1. `deepagent_memory.enabled`
2. `deepagent_memory.backend`
3. `deepagent_memory.redis.url`
4. `deepagent_memory.redis.key_prefix`
5. `deepagent_memory.redis.ttl_seconds`

Regra complementar:
1. `execution.type=deepagent` deixa de ser convenção documental e passa a ser parte do contrato governado.

#### Catálogo de tools

`tools_library[]` de raiz e `multi_agents[].tools_library[]` passam a ser governados por `ToolDefinitionAST`.

Campos governados de cada item:
1. `strategy`
2. `id`
3. `description`
4. `category`
5. `status`
6. `tags`
7. `aliases`
8. `config`
9. `metadata`
10. `impl` quando `strategy=direct`
11. `factory_impl` quando `strategy=factory`
12. `tool_name` quando `strategy=factory`
13. `factory_function` quando `strategy=factory`
14. `factory_returns` quando `strategy=factory`

## 4. Exceções explícitas

Esta ADR fecha o escopo exatamente nestes termos. Ficam fora da governança AST-first nesta fase:
1. `target`: existe no payload AST para orquestração do assembly, mas não é chave de runtime persistida no YAML final.
2. `deepagent_multi_agents`: existe no envelope AST como coleção de staging para compilação, mas não é chave persistida; o runtime continua recebendo `multi_agents`.
3. `local_tools_configuration` na raiz do `AgenticDocumentAST`: o campo existe no envelope, mas não é emitido por `to_fragment()` nem por `_select_fragment()` como chave raiz compilada.
4. `global_tools_configuration` na raiz do `AgenticDocumentAST`: mesmo motivo da exceção anterior.
5. `memory` na raiz do `AgenticDocumentAST`: o campo existe no envelope, mas não faz parte do fragmento compilado governado pelo assembly nesta fase.
6. `workflows_defaults` tem governança de presença e ownership na raiz, mas seu conteúdo interno ainda é mapa opaco; nenhuma lista adicional de subchaves fica congelada por esta ADR até existir AST tipada dedicada.
7. `workflows[].tools_library`: o contrato YAML comum menciona esse bloco, mas `WorkflowAST` não o declara e o assembly não o governa como subcontrato tipado nesta fase.
8. `multi_agents[].agents[].local_tools_configuration`: o contrato YAML comum menciona override por agente, mas `AgentAST` não declara essa chave; portanto ela fica fora da governança AST-first desta fase.
9. Qualquer bloco YAML fora do escopo agentic do assembly continua YAML-first.

## 5. Guard rails operacionais

Enquanto o runtime ainda consome YAML compilado diretamente, estes cuidados são obrigatórios:
1. `selected_workflow` e `selected_supervisor` devem continuar coerentes com `enabled` até a seleção ativa depender exclusivamente da camada AST end-to-end.
2. Se a AST não modela uma chave, essa chave não pode ser promovida implicitamente para o escopo governado.
3. Nenhum campo novo entra no escopo governado sem atualização conjunta de AST, parser, validator, compiler, schema, documentação e testes.

O efeito prático desta regra é simples:
1. o runtime ainda lê YAML;
2. mas, no escopo agentic governado, esse YAML deve ser tratado como saída compilada da AST, não como fonte primária para desenho de contrato.

## 6. Risco principal e mitigação

Risco principal: ampliar a decisão para YAMLs não agentic.

Mitigação:
1. manter a lista fechada de chaves raiz da seção 3;
2. manter a lista fechada de exceções da seção 4;
3. rejeitar mudanças que tentem usar esta ADR para blocos de ingestão, RAG, segurança, storage, banco, observabilidade ou qualquer outro domínio fora do assembly agentic.

## 7. Validação para entrada em vigor

Para considerar esta decisão aplicada com segurança em código e documentação:
1. esta ADR deve permanecer alinhada ao manual `docs/README-AST-AGENTIC-DESIGNER.md`;
2. o verificador `scripts/docs/verify_agentic_ast_docs_sync.py` deve permanecer verde;
3. a suíte mínima do fluxo `draft -> validate -> confirm` deve permanecer verde;
4. qualquer expansão do escopo governado exige atualização explícita desta ADR.

## 8. Rollback

Se a equipe decidir reverter a decisão, o rollback formal é este:
1. voltar a tratar o YAML agentic como contrato primário de definição;
2. reduzir a AST novamente ao papel de representação auxiliar de edição/validação;
3. remover ou revisar as seções de governança AST-first do manual e desta ADR;
4. preservar a lista de exceções para evitar falso escopo durante a reversão.
