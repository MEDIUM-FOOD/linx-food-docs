# Tutorial 101: YAML, Execução Local e Linguagem Natural para SQL

Este tutorial foi feito para desfazer uma confusão recorrente no projeto: achar que subir a aplicação, resolver YAML e perguntar em linguagem natural para SQL são o mesmo problema. Não são. Eles se encostam, mas cada um resolve uma camada diferente.

## 1. Primeiro conceito: o YAML decide o que é possível

Toda execução começa pelo contrato YAML. Ele define qual supervisor ou workflow está ativo, que catálogo de tools entra em jogo, quais credenciais são resolvidas e qual comportamento de domínio fica habilitado. Sem esse contrato íntegro, a execução local pode até subir, mas o caso real não roda do jeito esperado.

## 2. Segundo conceito: execução local não é só "subir a API"

A plataforma tem papéis diferentes. O launcher operacional local existe porque API, worker e scheduler têm responsabilidades diferentes. Quando a pessoa sobe apenas um processo solto sem entender o papel dele, o diagnóstico fica enganoso: parece que o sistema quebrou, quando na verdade o papel necessário nem estava rodando.

## 3. Terceiro conceito: NL para SQL tem dois caminhos

O primeiro caminho é o mais controlado: dyn_sql e proc_sql. Nesse caso, a pergunta em linguagem natural acaba escolhendo uma query ou procedure já homologada. O segundo caminho é schema_rag_sql, em que o sistema usa metadados de schema para ajudar um modelo a gerar SQL novo.

A diferença prática é enorme. dyn_sql é mais previsível e governado. schema_rag_sql é mais flexível, mas depende de schema metadata, vector store correto e contexto semântico suficiente.

## 4. Como os três temas se conectam

O YAML define se o runtime terá acesso às tools certas. A execução local garante que os papéis necessários estão vivos. E o caminho de NL para SQL só funciona quando os dois primeiros já estão coerentes. O erro mais comum é começar pela pergunta em linguagem natural sem garantir contrato e infraestrutura.

## 5. Quando usar dyn_sql e quando usar schema_rag_sql

Use dyn_sql quando a empresa já conhece a consulta desejada e quer só preenchimento seguro de parâmetros.

Use proc_sql quando a operação aprovada já vive em procedure e precisa ser executada com governança.

Use schema_rag_sql quando o problema exige composição nova de consulta a partir de metadados do banco, sabendo que esse caminho cobra mais contexto, mais validação e mais disciplina operacional.

## 6. Como pensar execução local sem cair em improviso

A pergunta certa não é "qual comando sobe tudo?". A pergunta certa é "qual papel eu preciso validar agora?". Se a dúvida é contrato HTTP, a API basta. Se a dúvida é job assíncrono, o worker entra. Se a dúvida é manutenção temporal, o scheduler precisa existir. Quando o caso mistura YAML agentic com operação longa, normalmente a validação mais fiel exige mais de um papel ativo.

## 7. Onde o fluxo AST entra

Quando a mudança é agentic e governada, o caminho oficial pode passar por endpoints como /config/assembly/objective-to-yaml e /config/assembly/validate. Isso não existe para embelezar autoria. Existe para impedir que uma configuração pareça razoável ao humano, mas quebre semanticamente no runtime.

## 8. Erros comuns neste tema

1. Achar que qualquer pergunta em português já vira SQL gerado na hora.
2. Subir só a API e concluir que o fluxo completo está saudável.
3. Mandar tools_library preenchida manualmente no YAML.
4. Tratar placeholder não resolvido como warning aceitável.
5. Ignorar a diferença entre query homologada e SQL realmente gerado.

## 9. Exemplo prático guiado

Imagine um time que quer perguntar "quais pedidos atrasados existem para o cliente X?". Se a consulta já é conhecida, o caminho mais seguro é dyn_sql, porque a governança fica na query aprovada. Se a pergunta muda muito e exige composição nova, schema_rag_sql pode fazer sentido, mas só depois que schema metadata estiver indexado, o YAML estiver correto e a execução local tiver os papéis necessários de fato funcionando.

## 10. Explicação 101

Pense assim. O YAML é a ficha técnica da operação. A execução local é a equipe que vai colocar a operação de pé. E NL para SQL é a forma como você quer conversar com o banco. Se a ficha está errada ou a equipe certa não apareceu, a conversa nem começa direito.

## 11. Leitura complementar

- [README-CONFIGURACAO-YAML.md](./README-CONFIGURACAO-YAML.md)
- [README-SERVICE-API.md](./README-SERVICE-API.md)
- [README-DYNAMIC-SQL-TOOLS.md](./README-DYNAMIC-SQL-TOOLS.md)
- [README-SQL-SCHEMA-RAG-TOOL.md](./README-SQL-SCHEMA-RAG-TOOL.md)
- [README-AST-AGENTIC-DESIGNER.md](./README-AST-AGENTIC-DESIGNER.md)
