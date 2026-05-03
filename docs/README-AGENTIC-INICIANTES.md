# Guia para Iniciantes: YAML Agentic com Validação AST

## 1. Para quem é este guia

Este material foi escrito para desenvolvedores iniciantes que precisam trabalhar com:

1. Geração de configuração em YAML (inclusive por LLM).
2. Validação estrutural e semântica antes de colocar em produção.
3. Fluxo `draft -> validate -> confirm` da montagem assistida.

Objetivo prático:

1. Entender os termos técnicos sem depender de conhecimento prévio.
2. Evitar os erros mais comuns de modelagem.
3. Ter um roteiro claro para evoluir configurações com segurança.

## 2. Visão geral em linguagem simples

No projeto, o YAML é a configuração que descreve comportamento de agentes e workflows.

A AST (Abstract Syntax Tree) é a versão tipada e validada dessa configuração.

Pipeline mental:

1. Você escreve (ou pede para a LLM escrever) YAML.
2. O sistema faz parse e transforma em AST.
3. O contrato estrutural rejeita campos inválidos, tipos errados e formatos fora do padrão.
4. A validação semântica checa regras de negócio (referências, IDs, tools existentes, etc.).
5. Só então a configuração é confirmada e aplicada.

Resumo:

1. YAML é entrada humana.
2. AST é o contrato técnico canônico do módulo de assembly; o YAML final continua sendo o contrato executado no runtime.
3. Diagnósticos são feedback objetivo para correção.

## 3. Glossário de termos e jargões

| Termo | Significado simples | Exemplo prático |
| --- | --- | --- |
| `YAML` | Formato de texto para configuração | Definir `workflows`, `nodes` e `tools_library` |
| `AST` | Representação tipada da configuração | `AgenticDocumentAST`, `WorkflowAST`, `ToolDefinitionAST` |
| `Contrato estrutural` | Regras de forma e tipo dos campos | `strategy` obrigatório em `tools_library` |
| `Contrato semântico` | Regras de consistência de negócio | Node usando tool inexistente gera erro |
| `Parser` | Camada que lê YAML/JSON e monta AST | Parser de `tools_library` identifica item inválido |
| `Validador` | Camada que aplica regras e gera diagnóstico | Detecta ID duplicado e backend inválido |
| `Diagnóstico` | Mensagem estruturada de erro/aviso | `TOOLS_LIBRARY_ID_DUPLICADO` |
| `JSON Schema` | Especificação para UI/formulário validar payload | Endpoint `/config/assembly/schema` |
| `Target` | Tipo de montagem desejada | `workflow`, `agent_supervisor`, `deepagent_supervisor` |
| `Merge` | Combinação de fragmento AST com YAML base | `confirm` compila e une resultado final |

## 4. Estratégia recomendada para iniciantes

Trabalhe sempre em ciclos curtos e verificáveis:

1. Comece pequeno: monte um caso mínimo funcional.
2. Rode `validate` cedo para receber diagnósticos rápidos.
3. Corrija erros estruturais antes dos semânticos.
4. Só use `confirm` com `apply=true` quando `is_valid=true`.

Sequência prática:

1. Escolha o `target`.
2. Defina IDs estáveis e únicos (`workflow`, `node`, `agent`, `tool`).
3. Adicione `tools_library` com contrato novo (`strategy`).
4. Valide em modo estrito.
5. Faça revisão humana.
6. Aplique.

## 5. Fluxo operacional (`draft -> validate -> confirm`)

### 5.1 `draft`

Uso:

1. Gerar rascunho AST com ajuda da LLM.
2. Obter perguntas pendentes para completar campos obrigatórios.

Saída importante:

1. `ast_draft`
2. `diagnostics`
3. `questions`

Estratégias de geração (`generation_mode`):

1. `auto`: tenta primeiro LLM estruturado. Se esse ramo falhar, vier inválido ou insuficiente, o fluxo pode tentar reparo conservador e cair para a heurística local quando o fallback estiver habilitado.
2. `llm_schema`: obriga LLM estruturado (sem fallback).
3. `heuristic`: usa apenas heurística local.

Ajuste fino de retries:

1. `constraints.llm_schema_max_attempts` define quantas tentativas o LLM estruturado fará (`1` a `5`, padrão `2`).
2. `constraints.auto_heuristic_fallback_enabled` liga ou desliga a queda automática para heurística dentro do modo `auto`.
3. `constraints.assembly_repair_max_attempts` define quantas tentativas de reparo conservador serão feitas antes de encerrar o ramo atual.

### 5.2 `validate`

Uso:

1. Validar AST final com regras estruturais e semânticas.
2. Garantir que o runtime consiga executar.

Saída importante:

1. `validation_report.is_valid`
2. `validation_report.diagnostics`
3. `error_count` e `warning_count`

### 5.3 `confirm`

Uso:

1. Compilar AST.
2. Mesclar no YAML base.
3. Aplicar no arquivo final (quando `apply=true`).

Boa prática:

1. Primeiro rodar com `apply=false` para revisar preview/diff.
2. Depois aplicar.

Regra importante:

1. O `confirm` bloqueia aplicação quando existe qualquer erro no consolidado (parse + semântico), exceto com `force=true`.

## 6. Contrato de `tools_library` explicado

Hoje o contrato é estrito e sem formato legado.

Regras centrais:

1. Toda tool precisa de `strategy`.
2. `strategy: direct` exige `impl`.
3. `strategy: factory` exige `factory_impl`, `tool_name`, `factory_function`, `factory_returns`.
4. Campos fora do contrato são rejeitados.

### 6.1 Exemplo `direct` (simples)

Leitura prática do formato `direct`:

1. A tool entra em `tools_library` com `strategy=direct`.
2. Ela precisa expor `id`, `description` e `category`.
3. Campos de catálogo como `status`, `tags`, `aliases`, `config` e `metadata` complementam a publicação.
4. O campo decisivo é `impl`, que aponta para o callable real da calculadora.

### 6.2 Exemplo `factory` (avançado)

Leitura prática do formato `factory`:

1. A tool continua em `tools_library`, mas usa `strategy=factory`.
2. `factory_impl` aponta para a implementação que monta a tool ou o conjunto de tools.
3. `tool_name` identifica a tool lógica publicada.
4. `factory_function` informa qual função da fábrica será chamada.
5. `factory_returns` declara o formato do retorno esperado.

## 7. Exemplo mínimo fim a fim

Leitura prática do exemplo mínimo fim a fim:

1. O `target` aponta para `workflow`.
2. `selected_workflow` escolhe um fluxo básico, como `wf_triagem_basica`.
3. `tools_library` publica uma tool direta chamada `json_parse`.
4. Em `workflows`, o fluxo habilitado declara um node `n1` em modo `agent`.
5. O node recebe um prompt de sistema simples e usa a tool `json_parse`.
6. As arestas ligam `START` ao node e o node a `END`, fechando o grafo mínimo válido.

O que validar neste exemplo:

1. `json_parse` existe em `tools_library`.
2. IDs são únicos.
3. Fluxo tem entrada e saída válidas (`START` e `END`).

## 8. Erros comuns e como evitar

1. Esquecer `strategy` na tool.
Impacto: falha estrutural imediata.
Como evitar: sempre começar pelo esqueleto `direct` ou `factory`.

2. Misturar formato antigo de tool com formato novo.
Impacto: parse/validação rejeitam payload.
Como evitar: usar apenas contrato descrito neste guia e no schema.

3. Referenciar tool não declarada.
Impacto: erro semântico (`WORKFLOW_TOOL_INEXISTENTE` ou equivalente).
Como evitar: validar catálogo efetivo antes de montar nodes/agentes.

4. Definir IDs duplicados.
Impacto: comportamento ambíguo e erro de validação.
Como evitar: padronizar prefixos (`wf_`, `sup_`, `ag_`, `tool_`).

## 9. Checklist de qualidade antes de aplicar

1. `validation_report.is_valid` está `true`.
2. `error_count` está `0`.
3. Warnings foram revisados e aceitos conscientemente.
4. Nenhum campo legado foi usado.
5. Diff de `confirm` foi revisado por humano.

## 10. Arquivos e classes que validam de verdade

Os pontos reais do código para validar mudanças no assembly agentic são:

1. `src/config/agentic_assembly/assembly_service.py` -> `AgenticAssemblyService`.
2. `src/config/agentic_assembly/ast/document.py` -> `AgenticDocumentAST`.
3. `src/config/agentic_assembly/ast/workflow.py` -> `WorkflowAST`, `WorkflowNodeAST`, `WorkflowEdgeAST`.
4. `src/config/agentic_assembly/ast/supervisor.py` -> `SupervisorAST`, `AgentAST`.
5. `src/config/agentic_assembly/ast/deepagent.py` -> `DeepAgentSupervisorAST`.
6. `src/config/agentic_assembly/parsers/` -> `WorkflowParser`, `SupervisorParser`, `DeepAgentParser`, `ToolDefinitionsParser`.
7. `src/config/agentic_assembly/validators/document_validator.py` -> `DocumentSemanticValidator`.
8. `src/config/agentic_assembly/validators/workflow_semantic_validator.py` -> `WorkflowSemanticValidator`.
9. `src/config/agentic_assembly/validators/supervisor_semantic_validator.py` -> `SupervisorSemanticValidator`.
10. `src/config/agentic_assembly/validators/deepagent_semantic_validator.py` -> `DeepAgentSemanticValidator`.
11. `src/config/agentic_assembly/compilers/document_compiler.py` -> `DocumentCompiler`.
12. `src/config/agentic_assembly/schema_service.py` -> `AgenticAssemblySchemaService`.

## 11. O que rodar antes de aplicar

1. Rode `uv run python scripts/docs/verify_agentic_ast_docs_sync.py` para confirmar sincronização entre documentação e AST.
2. Depois execute `uv run env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit/docs/test_agentic_ast_docs_sync.py tests/unit/test_agentic_assembly_service.py tests/unit/test_agentic_assembly_draft_llm_e2e.py tests/unit/test_agentic_assembly_quality_gate.py -q` para validar o fluxo principal.

Se a mudança tocar catálogo de tools, schema estruturado ou recomendação de tools:

1. Execute também `uv run env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit/test_agentic_assembly_tools_contract.py tests/unit/test_agentic_assembly_tool_recommendation.py tests/unit/test_agentic_assembly_structured_llm_client.py -q` para cobrir contrato de tools, recomendação e cliente estruturado.

## 12. Próximas leituras recomendadas

1. [README-AST-AGENTIC-DESIGNER.md](./README-AST-AGENTIC-DESIGNER.md) para contrato técnico completo da AST.
2. [README-AGENTIC-CONTRATO-COMUM.md](./README-AGENTIC-CONTRATO-COMUM.md) para blocos comuns reutilizáveis.
3. [README-SERVICE-API.md](./README-SERVICE-API.md) para visão de endpoints e operação da API.

## 13. Evidência no código

Os pontos que comprovam o fluxo descrito neste guia são:

1. `src/config/agentic_assembly/assembly_service.py`, que executa `draft`, `validate`, `confirm`, detecção de drift e merge final.
2. `src/config/agentic_assembly/drift_detector.py`, que governa `selected_workflow`, `selected_supervisor`, `workflows`, `multi_agents` e hashes do fragmento canônico.
3. `src/config/agentic_assembly/schema_service.py`, que entrega o schema usado pela superfície HTTP e por ferramentas auxiliares.
4. `src/api/routers/config_assembly_router.py`, que expõe o fluxo de assembly na API.

## 14. Lacunas no código

- Não encontrado no código: um material único gerado automaticamente para iniciantes a partir do schema em tempo real.
  Onde deveria estar: geração derivada de `schema_service.py` ou pipeline de documentação sincronizada.
- Não encontrado no código: uma UI nativa do assembly que elimine totalmente a necessidade de interpretar YAML para fluxos básicos.
  Onde deveria estar: `app/ui/` ou ferramenta dedicada de edição assistida.
