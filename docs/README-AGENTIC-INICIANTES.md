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
|---|---|---|
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

```yaml
tools_library:
  - strategy: direct
    id: calculator
    description: "Calculadora básica"
    category: custom_tools
    status: active
    tags: ["math"]
    aliases: []
    config: {}
    metadata: {}
    impl: src.agentic_layer.tools.custom_tools.calculator_tool.calculator
```

### 6.2 Exemplo `factory` (avançado)

```yaml
tools_library:
  - strategy: factory
    id: qa_rag
    description: "QA com contexto vetorial"
    category: vector_store_tools
    status: active
    tags: ["qa"]
    aliases: []
    config: {}
    metadata: {}
    factory_impl: src.agentic_layer.tools.vector_store_tools.vectorstore_toolkit.create_vectorstore_toolkit_tools
    tool_name: qa_rag
    factory_function: create_vectorstore_toolkit_tools
    factory_returns: list
```

## 7. Exemplo mínimo fim a fim

```yaml
target: workflow
selected_workflow: wf_triagem_basica
tools_library:
  - strategy: direct
    id: json_parse
    description: "Parse de JSON"
    category: utility_tools
    status: active
    tags: []
    aliases: []
    config: {}
    metadata: {}
    impl: src.agentic_layer.tools.utility_tools.json_parse.json_parse
workflows:
  - id: wf_triagem_basica
    enabled: true
    nodes:
      - id: n1
        mode: agent
        prompt:
          system: "Classifique a mensagem do usuário"
        tools: ["json_parse"]
    edges:
      - from: START
        to: n1
      - from: n1
        to: END
```

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

```bash
uv run python scripts/docs/verify_agentic_ast_docs_sync.py
uv run env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit/docs/test_agentic_ast_docs_sync.py tests/unit/test_agentic_assembly_service.py tests/unit/test_agentic_assembly_draft_llm_e2e.py tests/unit/test_agentic_assembly_quality_gate.py -q
```

Se a mudança tocar catálogo de tools, schema estruturado ou recomendação de tools:

```bash
uv run env PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest tests/unit/test_agentic_assembly_tools_contract.py tests/unit/test_agentic_assembly_tool_recommendation.py tests/unit/test_agentic_assembly_structured_llm_client.py -q
```

## 12. Próximas leituras recomendadas

1. `docs/README-AST-AGENTIC-DESIGNER.md` para contrato técnico completo da AST.
2. `docs/README-AGENTIC-CONTRATO-COMUM.md` para blocos comuns reutilizáveis.
3. `docs/README-SERVICE-API.md` para visão de endpoints e operação da API.

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
