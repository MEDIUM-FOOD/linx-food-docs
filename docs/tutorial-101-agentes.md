# Tutorial 101: Como Pensar Agentes Neste Produto

Este tutorial existe para quem abriu a plataforma pela primeira vez e ficou com a impressão errada de que "agente" aqui é só um prompt com tools. Não é. Neste produto, agente é uma unidade governada de execução que depende de YAML, seleção de supervisor, catálogo de tools, limites, memória e observabilidade.

## 1. O problema que este tutorial resolve

Quem chega agora costuma tropeçar em duas confusões.

1. Achar que qualquer bloco agentic vira execução do mesmo jeito.
2. Achar que o ponto principal do trabalho é descobrir em qual arquivo mexer.

Os dois erros desviam o entendimento do que realmente importa: como uma intenção declarada vira uma execução segura e rastreável.

## 2. O que é um agente aqui, na prática

Um agente aqui não é apenas o modelo. É a combinação de cinco coisas.

1. Uma intenção declarada no YAML.
2. Um supervisor ou workflow que decide como essa intenção será executada.
3. Um conjunto governado de tools.
4. Limites, memória e políticas de execução.
5. Um contrato de entrada e saída observável.

Quando uma dessas partes some, a execução até pode parecer funcionar, mas perde controle.

## 3. Quem decide o comportamento do agente

A decisão começa no YAML. Se o documento entra pelo caminho de supervisor, a seleção passa por multi_agents e selected_supervisor. Se entra por workflow, passa por workflows e selected_workflow. Em ambos os casos, o boundary HTTP resolve o documento, autentica a chamada, injeta correlation_id e só então entrega o caso para o runtime certo.

Em linguagem simples: o agente não nasce no router e não nasce no modelo. Ele nasce quando a configuração certa encontra o runtime certo com o contexto certo.

## 4. O fluxo real de execução

O pedido chega pela API. A borda resolve identidade, permissão e YAML. Depois escolhe se a execução é curta ou se precisa seguir de forma assíncrona. No caso de supervisor, o orquestrador monta o runtime, resolve as tools permitidas e chama o agente. No caso de DeepAgent, o processo ainda monta middlewares governados antes de executar.

O ponto importante é que o fluxo tem etapas com responsabilidades diferentes. A API aceita e contextualiza. O orquestrador compõe. O supervisor executa. A tool integra. Misturar essas responsabilidades é a maneira mais rápida de criar regressão difícil de diagnosticar.

## 5. O que faz um agente dar resposta boa ou ruim

Uma resposta boa depende de três camadas estarem coerentes.

1. Configuração correta.
2. Runtime certo para o caso.
3. Ferramentas e limites adequados.

Uma resposta ruim não significa automaticamente que o prompt estava ruim. Pode ser tool ausente, supervisor errado, permissão bloqueada, memória desligada, input mal resolvido ou execução que deveria ter sido assíncrona.

## 6. Como pensar tools sem se perder

Tool aqui é capacidade governada, não atalho improvisado. O agente não deveria receber acesso direto a recurso externo sem passar por catálogo, precedência de configuração e validação semântica. Isso vale tanto para tool fixa quanto para família dinâmica.

Em termos simples: ferramenta boa para agente não é a que existe no código. É a que entra no runtime pelo contrato certo e pode ser explicada depois.

## 7. O que muda entre supervisor clássico e DeepAgent

O supervisor clássico resolve coordenação direta entre agentes e tools. O DeepAgent entra quando o produto precisa de camadas adicionais de governança, como filesystem, shell, memória durável, subagentes, revisão humana, PII e lista de tarefas.

A decisão entre um e outro não deveria ser estética. Ela depende do tipo de risco operacional que o caso precisa controlar.

## 8. Erros mais comuns de quem está começando

1. Achar que o YAML é só um detalhe de configuração e não o contrato operacional.
2. Tentar colocar regra de negócio dentro do router.
3. Instanciar tool manualmente sem passar pelo catálogo.
4. Ignorar correlation_id e depois não conseguir reconstruir o caso.
5. Confundir aceite assíncrono com conclusão do trabalho.

## 9. Como estudar este tema sem cair em mapa de arquivo

A ordem mais produtiva é esta.

1. Entender a arquitetura macro da plataforma.
2. Entender YAML como contrato.
3. Entender supervisor clássico e DeepAgent como modelos de execução.
4. Só depois olhar APIs, tools e casos específicos.

Isso reduz a chance de decorar peças soltas sem entender a lógica que liga tudo.

## 10. Exemplo prático guiado

Imagine um operador pedindo uma automação com acesso a tools e eventual revisão humana. Se o caso exige governança forte, o YAML aponta para DeepAgent. A API resolve o documento e autentica o pedido. O runtime monta middlewares, seleciona tools, aplica limites e só então executa. Se a operação precisar de aprovação, ela pausa. Se a tool estiver fora do catálogo, a falha aparece cedo. Se tudo correr bem, a resposta sai com rastreabilidade suficiente para suporte.

## 11. Explicação 101

Pense no agente como um funcionário novo. Não basta dizer "vá e resolva". Você precisa dar crachá, acesso certo, ferramentas certas, limite de ação, histórico mínimo e uma forma de revisar decisões sensíveis. O runtime faz esse papel de estrutura e controle.

## 12. Leitura complementar

- [README-ARQUITETURA.md](./README-ARQUITETURA.md)
- [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md)
- [README-DEEPAGENTS-SUPERVISOR.md](./README-DEEPAGENTS-SUPERVISOR.md)
- [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md)
