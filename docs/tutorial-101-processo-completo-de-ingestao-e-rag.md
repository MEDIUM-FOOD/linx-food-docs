# Tutorial 101: Como o Acervo Nasce e Como a Resposta Sai

Este tutorial existe para ensinar a história completa entre ingestão e RAG como fluxo de produto. Ele não é um inventário da pasta de ingestão nem um catálogo de classes do QA. Ele é uma trilha para entender por que o sistema separa preparação de acervo e resposta ao usuário.

## 1. O problema que este tutorial resolve

Muita gente entra no projeto achando que RAG é só pegar chunks e mandar para o modelo. Isso esconde metade do trabalho. Antes da resposta existir, alguém precisou extrair documento, limpar ruído, quebrar conteúdo em partes úteis, registrar metadados, sincronizar stores e manter observabilidade.

Sem esse entendimento, qualquer resposta ruim parece problema de prompt. Quase nunca é tão simples.

## 2. O que é ingestão, em linguagem simples

Ingestão é o processo de transformar fonte bruta em acervo consultável. Ela lida com PDF, página web, planilha, JSON e outras origens. O trabalho dela não acaba quando consegue ler texto. Ela precisa produzir algo que depois possa ser recuperado com qualidade.

## 3. O que é RAG, em linguagem simples

RAG é o processo de usar esse acervo para responder perguntas com evidência. Ele não deveria responder "do nada". Ele primeiro tenta entender a pergunta, depois recuperar contexto útil e só então gerar a resposta final.

## 4. Por que os dois pipelines são separados

Ingestão e RAG resolvem problemas diferentes. Ingestão está preocupada com produção, limpeza, particionamento, persistência e atualização do acervo. RAG está preocupado com entendimento da pergunta, recuperação, ordenação da evidência e redação final.

Misturar os dois leva a diagnósticos ruins. Uma resposta ruim pode ser culpa de chunks ruins. E chunks bons não garantem resposta boa se retrieval ou geração estiverem mal orientados.

## 5. Como a história completa acontece

A história começa quando a API aceita um pedido de ingestão. Se o trabalho é pesado, ela publica continuidade assíncrona. O worker executa a ingestão real, passa por dispatcher, processor e persistência. Quando o acervo fica pronto, o RAG pode usá-lo. A consulta chega, o runtime moderno valida se o contexto mínimo existe, escolhe a estratégia de retrieval e só depois chama o modelo para gerar resposta.

Em linguagem prática: primeiro a plataforma prepara o terreno. Depois ela consulta o terreno.

## 6. O que faz a ingestão ser boa ou ruim

Uma ingestão boa extrai conteúdo suficiente, preserva informação útil, quebra em partes recuperáveis, registra metadados e mantém stores sincronizados. Uma ingestão ruim até pode parecer concluída, mas deixa acervo incoerente ou pouco recuperável. É aí que começam os falsos diagnósticos sobre o RAG.

## 7. O que faz o RAG ser bom ou ruim

Um RAG bom entende a natureza da pergunta, escolhe estratégia adequada, busca evidência relevante e deixa claro o que usou para responder. Um RAG ruim pode falhar por muita coisa: pergunta mal formulada, retrieval fraco, cache inadequado, acervo inconsistente ou geração final sem contexto suficiente.

## 8. Como o operador deve investigar

A pergunta certa é: o problema nasceu na produção do acervo ou na consulta do acervo? Se a ingestão falhou, o operador precisa olhar status, detalhe operacional e sincronização do dataset. Se a ingestão foi boa, a suspeita passa para retrieval, filtros, estratégia de consulta e geração.

## 9. Erros comuns de entendimento

1. Tratar ingestão concluída como sinônimo de acervo útil.
2. Tratar HTTP 202 como resultado final.
3. Achar que toda resposta ruim vem do modelo.
4. Ignorar BM25, store vetorial e banco relacional como conjunto operacional único.
5. Tentar depurar o fim do fluxo sem confirmar o começo.

## 10. Exemplo prático guiado

Imagine um PDF técnico com imagem, tabela e texto corrido. A ingestão precisa decidir como extrair, se faz OCR, como quebrar em partes e como persistir metadados. Depois, um usuário faz uma pergunta sobre esse conteúdo. O RAG precisa recuperar justamente os trechos que preservaram o sentido certo. Se a extração perdeu a tabela ou o chunking quebrou a lógica do parágrafo, a resposta final sai ruim mesmo com retrieval e LLM saudáveis.

## 11. Explicação 101

Pense numa biblioteca. A ingestão é o trabalho de catalogar e guardar os livros do jeito certo. O RAG é o bibliotecário que vai buscar o material para responder uma pergunta. Se os livros foram catalogados mal, o bibliotecário sofre. Se o bibliotecário procura mal, o acervo bom também não ajuda.

## 12. Leitura complementar

- [README-INGESTAO.md](./README-INGESTAO.md)
- [README-RAG.md](./README-RAG.md)
- [PIPELINE-INGESTAO-RAG.md](./PIPELINE-INGESTAO-RAG.md)
- [README-ARQUITETURA.md](./README-ARQUITETURA.md)
