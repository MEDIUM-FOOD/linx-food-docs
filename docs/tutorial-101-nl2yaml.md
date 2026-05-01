# Tutorial 101: NL2YAML

## 1. O que NL2YAML significa neste projeto

NL2YAML aqui não significa pedir qualquer YAML para uma IA e aceitar o resultado como verdade.

No código atual, NL2YAML significa usar a esteira governada do assembly agentic para transformar um objetivo em linguagem natural em AST canônica validada e, só no fim, em YAML final.

Esse detalhe importa porque o produto não trata YAML e AST como dois contratos soltos. O assembly usa a AST como fonte de verdade e o YAML final como artefato compilado e confirmado.

## 2. O problema que essa feature resolve

Sem NL2YAML governado, cada criação de workflow ou supervisor agentic cairia em um dos dois extremos ruins:

1. edição manual lenta, difícil para quem ainda não domina a sintaxe;
2. geração automática solta, sem validação semântica e sem controle de publicação.

A feature existe para resolver essa tensão. Ela acelera a criação, mas sem abrir mão de validação, perguntas bloqueantes, diff e confirmação explícita.

## 3. A ideia central em linguagem simples

Pense no NL2YAML como uma linha de projeto técnico em quatro etapas:

1. verificar se o pedido tem contexto mínimo;
2. gerar um rascunho estruturado;
3. validar se esse rascunho faz sentido no contrato real;
4. só então montar o YAML final e decidir se ele será salvo.

O sistema não pula da intenção para o arquivo final. Ele passa por uma montagem governada.

## 4. Os quatro estágios que você precisa decorar

### 4.1 Preflight

É a checagem inicial. O objetivo é responder se o ambiente e o pedido estão prontos para avançar.

### 4.2 Draft

É a geração do rascunho AST. Nesta etapa, o sistema ainda pode devolver perguntas em vez de avançar.

### 4.3 Validate

É a validação semântica forte do payload AST. Aqui o sistema verifica se o documento montado respeita o contrato do assembly e do alvo escolhido.

### 4.4 Confirm

É a etapa que compila o fragmento governado, faz merge no `base_yaml` e produz o YAML final. Quando `apply=false`, ela gera preview. Quando `apply=true`, ela persiste o resultado no root governado.

## 5. Como o fluxo direto funciona de verdade

O serviço principal é `AgenticAssemblyService`.

O método `objective_to_yaml` executa uma ordem fixa confirmada no código:

1. `preflight`;
2. `draft`;
3. `validate`;
4. `confirm` em dry-run.

Isso significa que o caminho direto `objective-to-yaml` não é um atalho mágico. Ele é só uma composição oficial das quatro etapas canônicas.

## 6. O que pode sair da resposta

Quando o fluxo roda, a resposta pode seguir dois caminhos principais.

### 6.1 Caminho feliz

Quando tudo fecha, a resposta devolve YAML final, versão textual do YAML, diff preview e o rastro das decisões tomadas.

### 6.2 Caminho bloqueado

Quando falta contexto ou quando o draft ou a validação travam, a resposta devolve perguntas, diagnósticos e um `blocking_stage` explícito.

O valor prático disso é enorme: a UI e a operação não precisam adivinhar por que o fluxo parou.

## 7. Como o alvo é escolhido

O assembly trabalha com três espinhas dorsais principais:

1. workflow;
2. supervisor clássico de agente;
3. supervisor deepagent.

Quando o payload chega com `target=auto`, o `IntentParser` ajuda a resolver qual dessas estruturas faz mais sentido para o pedido. Isso reduz ambiguidade, mas não elimina a necessidade de contexto claro.

## 8. Os modos de geração

No contrato atual, o draft pode ser pedido em três modos:

1. `heuristic`;
2. `llm_schema`;
3. `auto`.

### 8.1 `heuristic`

É o caminho mais determinístico. Ele não depende do provider estruturado do LLM para ser útil.

### 8.2 `llm_schema`

É o caminho que tenta gerar o rascunho via LLM estruturado, com envelope JSON validado.

### 8.3 `auto`

É o modo que tenta escolher melhor estratégia. No código atual, o fallback heurístico não deve acontecer por impulso escondido. Ele depende de opt-in explícito.

## 9. O que acontece quando faltam informações

Se o pedido ainda não está pronto para virar YAML final, o fluxo não inventa respostas.

Os sinais mais importantes são:

1. `questions`, quando ainda falta decisão humana concreta;
2. `diagnostics`, quando existe problema técnico ou contratual;
3. `blocking_stage`, quando já é possível dizer em qual etapa o fluxo travou.

Isso é especialmente importante para times juniores, porque evita tratar um rascunho incompleto como se fosse um YAML publicável.

## 10. Quando a publicação acontece de verdade

Gerar preview não é o mesmo que publicar.

No assembly, a persistência real só acontece na confirmação com `apply=true`. É nesse momento que o serviço valida `output_path`, respeita o root governado de saída e grava o resultado.

Na prática, o produto separa duas intenções:

1. ver o que seria gerado;
2. salvar de fato o YAML confirmado.

## 11. O que já está pronto hoje

Pelo código lido, já está claro que o projeto possui:

1. o serviço canônico de assembly;
2. o fluxo direto `objective-to-yaml` com ordem definida;
3. contratos de request e response tipados;
4. modo heurístico;
5. modo LLM estruturado;
6. confirmação com preview e com persistência.

## 12. O que ainda depende de contexto real

Mesmo com a feature implementada, a qualidade do resultado ainda depende de contexto real do tenant e do pedido.

Os fatores mais sensíveis são:

1. clareza do objetivo em linguagem natural;
2. qualidade do `base_yaml` de partida;
3. disponibilidade do provider estruturado quando o modo usa LLM;
4. escolha correta do alvo ou qualidade da inferência em `auto`.

## 13. O que acontece em caso de sucesso

Quando tudo dá certo:

1. o preflight libera o caminho;
2. o draft consegue montar AST útil;
3. a validação semântica passa;
4. o confirm em dry-run produz `final_yaml`;
5. a publicação opcional salva o arquivo no caminho governado.

## 14. O que acontece em caso de erro

Os cenários de falha mais importantes confirmados no código são estes:

### 14.1 Bloqueio no draft

Se o rascunho sair incompleto ou ambíguo, o fluxo para no draft e devolve perguntas em vez de forçar validação e confirmação.

### 14.2 Bloqueio no validate

Se a AST parecer válida à primeira vista, mas quebrar o contrato semântico do assembly, o fluxo para na validação.

### 14.3 Bloqueio no confirm

Se o dry-run não conseguir produzir um YAML final utilizável, o fluxo marca bloqueio em `confirm`.

### 14.4 Feature flag desligada

O boundary do assembly protege os endpoints principais por feature flag. Se ela estiver desligada, o produto não deve fingir disponibilidade da feature.

## 15. Como colocar para funcionar sem se perder

A sequência mental correta para operar a feature é esta:

1. subir a API do projeto;
2. garantir que a flag do assembly agentic esteja ativa no ambiente alvo;
3. entrar pela UI ou endpoint oficial do assembly;
4. começar pelo preview, não pela publicação;
5. só salvar quando o resultado vier sem bloqueios.

## 16. Como pensar a feature com uma analogia

Imagine que você quer construir uma casa.

O texto do usuário é o pedido do cliente. A AST é a planta técnica. A validação é a revisão do engenheiro. O YAML final é o projeto executivo. Publicar com `apply=true` é protocolar esse projeto no lugar certo.

Ninguém sério pega um pedido informal e manda direto para a obra. O assembly existe exatamente para impedir esse tipo de salto.

## 17. Checklist de entendimento

Se você entendeu estes pontos, o essencial do NL2YAML já ficou sólido:

1. entendi que NL2YAML aqui é AST-first, não texto para YAML direto;
2. entendi que o fluxo direto roda `preflight -> draft -> validate -> confirm`;
3. entendi que `questions` e `blocking_stage` são parte do contrato oficial;
4. entendi que preview e publicação são etapas diferentes;
5. entendi que `heuristic`, `llm_schema` e `auto` não significam a mesma coisa.

## 18. Exercícios guiados

### Exercício 1

Objetivo: entender a ordem do pipeline.

Passos:

1. leia o método `objective_to_yaml`;
2. identifique em que momento `validate` é chamado;
3. identifique em que momento `confirm` entra como dry-run.

Resposta esperada: o fluxo direto só chama `confirm` depois que o draft e a validação passam.

### Exercício 2

Objetivo: entender por que o sistema devolve perguntas.

Passos:

1. leia a montagem do draft no service;
2. depois observe como a resposta do fluxo direto trata `questions`.

Resposta esperada: perguntas não são falha cosmética; são o mecanismo oficial para impedir que o backend invente decisões faltantes.

## 19. Evidências no código

1. `src/config/agentic_assembly/assembly_service.py`: serviço principal e ordem oficial do fluxo.
2. `src/config/agentic_assembly/models.py`: contratos tipados de request, response e `blocking_stage`.
3. `src/config/agentic_assembly/nl/intent_parser.py`: resolução de alvo quando o pedido usa `auto`.
4. `src/config/agentic_assembly/nl/llm_draft_generator.py`: geração estruturada do draft via LLM.
5. `src/api/routers/config_assembly_router.py`: boundary HTTP do assembly.
