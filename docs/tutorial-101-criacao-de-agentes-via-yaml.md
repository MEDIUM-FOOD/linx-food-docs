# Tutorial 101: Criar Agentes via YAML Sem Virar Refém do YAML

Este tutorial foi escrito para ensinar criação de agentes como decisão de produto e operação, não como exercício de preencher campos. O YAML importa muito, mas ele é meio, não fim.

## 1. O que significa "criar um agente" neste projeto

Criar um agente aqui significa descrever uma responsabilidade executável. Isso envolve dizer quem coordena a execução, quais ferramentas são permitidas, como memória e limites funcionam e que tipo de resposta o runtime deve produzir.

Em linguagem simples: criar agente não é escrever um nome bonito e um prompt. É desenhar um comportamento governado.

## 2. Quando faz sentido criar um agente novo

Só faz sentido criar um agente novo quando existe especialização real. Se a nova necessidade puder ser absorvida por ajuste pequeno de tool, regra ou workflow, criar mais um agente piora a operação porque espalha responsabilidade e dificulta diagnóstico.

Agente novo vale a pena quando há mudança clara de papel. Exemplo: um especialista que usa ferramentas distintas, opera com limite diferente ou responde por uma etapa crítica que precisa ser identificável sozinha.

## 3. Como o YAML entra nessa história

O YAML é a fonte declarativa da decisão. É ele que liga o agente ao supervisor, às tools, aos limites e ao modo de execução. O runtime não deveria adivinhar intenção que o YAML não expressa. Por isso o documento precisa ser explícito e coerente com a espinha dorsal escolhida.

Na prática, isso evita o erro mais comum de autoria: tentar misturar conceitos de workflow, supervisor clássico e DeepAgent no mesmo lugar como se fossem sinônimos.

## 4. A pergunta certa antes de escrever qualquer chave

Antes de começar, responda estas perguntas.

1. Esse comportamento precisa de supervisor clássico, DeepAgent ou workflow?
2. O agente decide algo ou só executa uma ação específica?
3. Ele precisa de tools? Quais riscos essas tools trazem?
4. A execução é curta ou pode virar operação assíncrona?
5. O suporte conseguirá explicar depois o que ele fez?

Se essas perguntas não estiverem claras, o YAML tende a virar acúmulo de campos sem desenho real.

## 5. Como pensar ferramentas e permissões

Ferramenta não é bônus. Cada tool aumenta superfície de risco. Por isso o agente deve receber só o que precisa. O catálogo builtin define a família técnica disponível. O registro por tenant ou a configuração local define o item concreto. O runtime faz a composição final.

O melhor agente não é o que pode tudo. É o que faz o necessário com o menor raio de risco.

## 6. Como pensar memória

Memória precisa responder a um problema real. Se o agente precisa apenas concluir uma tarefa pontual, memória demais atrapalha. Se ele precisa continuar conversa, manter contexto operacional ou usar revisão humana, a memória passa a ser parte do contrato.

No supervisor clássico, isso aparece no bloco memory. No DeepAgent, a conversa fica mais forte porque entra deepagent_memory e middlewares de memória. O importante é não confundir continuidade útil com acúmulo descontrolado de contexto.

## 7. Como saber se o agente ficou bom

Um agente bem criado responde sim para estas perguntas.

1. O papel dele está claro?
2. O runtime correto ficou explícito?
3. As tools estão justificadas?
4. O limite operacional faz sentido?
5. A falha dele será rastreável?

Se a resposta for não para qualquer uma delas, o problema ainda não é de implementação. É de desenho.

## 8. Erros comuns de autoria

1. Criar um agente para cada microvariação de prompt.
2. Usar tools demais porque "vai que precisa".
3. Deixar o runtime inferir comportamento que deveria estar claro no YAML.
4. Tratar memória como default inocente.
5. Ignorar como o operador vai acompanhar ou revisar a execução.

## 9. Exemplo prático guiado

Imagine um agente de atendimento que precisa consultar dado homologado e responder com segurança. Se o caso é simples, o supervisor clássico pode bastar. Se o caso exige filesystem, revisão humana ou subagentes, o DeepAgent pode ser mais adequado. Em ambos os cenários, a criação correta começa definindo papel, depois tools, depois limites, depois memória e só então refinando resposta.

## 10. Explicação 101

Criar agente por YAML é como definir o cargo de alguém numa empresa. Você não começa escolhendo a cor da mesa. Você começa definindo responsabilidade, acesso, limite e processo de trabalho. O YAML é a ficha desse cargo.

## 11. Leitura complementar

- [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md)
- [README-DEEPAGENTS-SUPERVISOR.md](./README-DEEPAGENTS-SUPERVISOR.md)
- [README-AGENTE-WORKFLOW.md](./README-AGENTE-WORKFLOW.md)
- [GUIA-USUARIO-TOOLS.md](./GUIA-USUARIO-TOOLS.md)
