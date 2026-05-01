# Manual técnico da AST Agentic Designer

## 1. O que esta feature é

A AST Agentic Designer é a camada governada usada para editar, validar, compilar e publicar a parte agentic do YAML do produto.

No código atual, isso significa uma coisa muito importante: no escopo agentic, o sistema não trabalha tratando o YAML bruto como contrato solto. Ele lê o YAML, converte para modelos tipados, valida esses modelos, compila o fragmento governado e só então produz o YAML final.

## 2. Que problema ela resolve

Sem essa camada, qualquer edição visual, NL2YAML ou automação de montagem agentic cairia em dois riscos graves:

1. editar YAML cru sem saber se a estrutura realmente fecha com o runtime;
2. misturar parse, validação e persistência no mesmo passo, dificultando diagnóstico.

A AST existe para quebrar esse problema em etapas governadas, cada uma com responsabilidade clara.

## 3. A ideia central em linguagem simples

Pense no YAML agentic como um documento final de obra, e na AST como a planta técnica organizada desse documento.

A planta é mais segura para revisar, validar e alterar do que sair mexendo direto na obra pronta. No projeto, a AST cumpre exatamente esse papel.

Em termos práticos:

1. o parser lê o YAML e monta objetos tipados;
2. o validator confere se esses objetos fazem sentido estrutural e semanticamente;
3. o compiler gera o fragmento governado do alvo escolhido;
4. o assembly service decide se isso vira preview ou publicação.

## 4. O que entra no escopo governado

O escopo agentic governado observado no código cobre principalmente:

1. workflows;
2. supervisores clássicos de agente;
3. supervisores deepagent;
4. tools_library do documento e dos supervisores;
5. seletores como `selected_workflow` e `selected_supervisor`.

Fora desse escopo, o sistema continua tratando o YAML como YAML-first. Dentro dele, a AST é a referência canônica de edição e validação.

## 5. Modelos e contratos

Os contratos públicos do assembly ficam em `models.py`.

Os pontos mais importantes confirmados ali são:

1. alvos oficiais do assembly: `auto`, `workflow`, `agent_supervisor` e `deepagent_supervisor`;
2. modos oficiais de geração de draft: `auto`, `llm_schema` e `heuristic`;
3. severidades de diagnóstico: `error`, `warning` e `info`;
4. respostas tipadas para `preflight`, `draft`, `validate`, `confirm`, `objective-to-yaml`, `schema` e `catalog`.

Isso é importante porque a UI e os integradores não precisam adivinhar formato de resposta. O contrato já existe como modelo tipado.

## 6. Como o assembly funciona por dentro

O centro da orquestração é `AgenticAssemblyService`.

Ele organiza o ciclo do assembly em etapas canônicas:

1. `draft`;
2. `validate`;
3. `confirm`;
4. composição direta via `objective_to_yaml`.

No construtor do serviço, o código deixa clara a arquitetura:

1. loaders e parsers para ler e converter YAML;
2. schema service para exposição externa do contrato;
3. tool resolver para catálogo efetivo;
4. validator semântico;
5. compiler de documento;
6. drift detector e diff preview;
7. parser de intenção e gerador de draft LLM.

Esse arranjo mostra que o assembly não é um endpoint simples. É um pipeline de montagem governada.

## 7. Papel de cada etapa

### 7.1 Draft

O draft recebe prompt ou `base_yaml`, resolve alvo, monta catálogo efetivo e produz um rascunho AST com preview inicial.

Se vier prompt, o serviço tenta montar AST a partir da linguagem natural. Se não vier prompt, ele pode partir do YAML existente.

### 7.2 Validate

A validação pega um payload AST e decide se ele é semanticamente válido para o alvo escolhido.

Ela não para em checagem de tipo. Ela também considera catálogo efetivo, referências e regras de consistência do domínio agentic.

### 7.3 Confirm

O confirm é a etapa de consolidação. Ele recompõe o documento AST, aplica respostas a perguntas pendentes, roda validação, calcula drift, compila o fragmento governado e faz merge no `base_yaml`.

Quando `apply=false`, ele funciona como preview. Quando `apply=true`, ele segue para persistência.

### 7.4 Objective to YAML

Esse fluxo não é um quarto contrato independente. Ele é a composição oficial que encadeia `preflight`, `draft`, `validate` e `confirm` em dry-run.

## 8. Como a validação é organizada

A validação agregada fica em `DocumentSemanticValidator`.

O desenho é simples e forte:

1. workflow usa `WorkflowSemanticValidator`;
2. supervisor clássico usa `SupervisorSemanticValidator`;
3. deepagent usa `DeepAgentSemanticValidator`;
4. tools_library passa por `ToolsSemanticValidator`.

No modo `auto`, o serviço agrega diagnósticos dos três alvos e consolida um relatório único.

O ganho prático disso é separar o contrato por espinha dorsal sem perder um ponto central de orquestração.

## 9. Como o schema é exposto para UI e tooling

`AgenticAssemblySchemaService` entrega os JSON Schemas e o catálogo estático consumido por interfaces e validações externas.

Ele expõe schemas para:

1. workflow;
2. agent supervisor;
3. deepagent supervisor;
4. common;
5. document.

Também expõe catálogo operacional com:

1. modos de workflow;
2. catálogo efetivo de tools;
3. safe functions;
4. execution modes;
5. middlewares deepagent oficiais e de plataforma.

Isso importa porque o designer visual não precisa duplicar conhecimento interno do runtime. Ele pode consumir o schema oficial produzido pelo backend.

## 10. Como o merge canônico funciona

O `DocumentCompiler` é o componente que aplica o fragmento AST no documento base sem tocar seções fora do alvo.

O comportamento confirmado no código é este:

1. para workflow, ele substitui apenas `selected_workflow`, `workflows_defaults`, `workflows` e `tools_library` do recorte governado;
2. para supervisores, ele trata `selected_supervisor`, faz merge controlado de `multi_agents` e também atualiza `tools_library` por id;
3. fora desses casos, ele aplica o fragmento inteiro.

Esse detalhe é um dos mais importantes do assembly. O objetivo não é reescrever o YAML inteiro. O objetivo é alterar apenas o slice governado pelo alvo atual.

## 11. Por que isso fortalece a arquitetura

A AST Agentic Designer fortalece a arquitetura em quatro pontos:

1. separa parsing de validação;
2. separa validação de compilação;
3. separa preview de persistência;
4. reduz edição manual cega de YAML agentic.

Na prática, isso melhora segurança de evolução, legibilidade do erro e capacidade de criar tooling visual ou NL sem quebrar o contrato do runtime.

## 12. O que acontece em caso de sucesso

Quando o ciclo fecha corretamente:

1. o YAML ou o prompt viram AST coerente;
2. o relatório de validação vem sem erros bloqueantes;
3. o compiler produz fragmento governado consistente;
4. o merge entrega `final_yaml` previsível para preview ou publicação.

## 13. O que acontece em caso de erro

Os erros mais importantes observáveis nessa camada são estes:

### 13.1 Erro de parse

O YAML ou o payload AST não conseguem ser convertidos para o modelo esperado.

### 13.2 Erro semântico

A estrutura parece válida, mas quebra regras de consistência do alvo, de catálogo ou de contrato agentic.

### 13.3 Drift no YAML governado

O confirm e a validação podem detectar divergência entre o estado governado esperado e o YAML base atual.

### 13.4 Alvo incompatível

Um fragmento, payload ou operação tenta agir sobre um alvo diferente do contrato atual do documento.

## 14. Como diagnosticar problemas reais

A ordem mais segura de investigação é:

1. o alvo foi resolvido corretamente?
2. o parser conseguiu montar a AST?
3. o validator apontou erro estrutural ou semântico?
4. o fragmento compilado faz sentido para o alvo?
5. o merge final preservou o restante do YAML fora do escopo governado?

Essa ordem impede um erro comum: tentar corrigir o YAML final antes de entender se o problema nasceu no draft, no validate ou no compiler.

## 15. Explicação 101

Se você esquecer todos os nomes técnicos, lembre desta frase:

A AST Agentic Designer é o jeito seguro de mexer no pedaço agentic do YAML sem editar no escuro.

Ela funciona como uma camada intermediária que entende a estrutura, valida a coerência e só depois monta a saída final.

## 16. Limites e pegadinhas

Os limites mais importantes confirmados no desenho atual são:

1. AST e YAML governado não devem evoluir separadamente;
2. preview não é publicação;
3. validação estrutural não substitui validação semântica;
4. o schema de UI não é decorativo, ele é parte do contrato real do assembly.

## 17. Evidências no código

1. `src/config/agentic_assembly/models.py`: contratos públicos do assembly.
2. `src/config/agentic_assembly/assembly_service.py`: orquestração principal do ciclo agentic.
3. `src/config/agentic_assembly/schema_service.py`: schemas JSON e catálogo para UI e tooling.
4. `src/config/agentic_assembly/validators/document_validator.py`: validação agregada por alvo.
5. `src/config/agentic_assembly/compilers/document_compiler.py`: merge canônico do fragmento governado no YAML base.
