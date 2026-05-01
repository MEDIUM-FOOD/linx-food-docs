# FAQ de Onboarding

Esta FAQ não foi escrita para apontar em qual arquivo alguém deve mexer. Ela foi escrita para responder as dúvidas que realmente aparecem quando alguém tenta entender a plataforma como sistema.

## 1. O que esta plataforma é, na prática?

Ela é uma base operacional para três coisas diferentes que costumam ser confundidas em projetos de IA.

1. Preparar conhecimento a partir de muitas fontes.
2. Consultar esse conhecimento com RAG e geração governada.
3. Executar automações e fluxos agentic mantendo segurança, observabilidade e controle operacional.

Em linguagem simples: o projeto não é apenas um chat e também não é apenas um pipeline de documentos. Ele tenta unir produção de acervo, consulta inteligente e execução governada.

## 2. Por que existem API, worker e scheduler separados?

Porque cada papel resolve um tipo de trabalho com custo e risco diferentes.

- A API recebe pedidos, autentica, resolve contexto e devolve contrato HTTP.
- O worker executa trabalho pesado e assíncrono fora do tempo de resposta do usuário.
- O scheduler cuida do que precisa acontecer por tempo, manutenção e liderança operacional.

Misturar esses três papéis no mesmo processo deixaria o sistema mais simples só na aparência. Na prática, tornaria mais difícil isolar falhas, escalar o que pesa e explicar o que aconteceu em produção.

## 3. O que significa dizer que o projeto é YAML-first?

Significa que a plataforma tenta declarar comportamento no YAML antes de espalhar regra em código ad hoc. Mas isso não quer dizer que o YAML seja executado cru. Antes de virar runtime, ele pode passar por resolução de segredos, normalização estrutural, validação semântica e, no escopo agentic, compilação governada por AST.

A ideia prática é reduzir surpresa. Em vez de a lógica ficar distribuída em múltiplos pontos implícitos, o projeto tenta centralizar intenção e configuração num contrato que o runtime consegue validar.

Quando a intenção nasce em linguagem natural, o fluxo público de geração governada passa por `/config/assembly/objective-to-yaml`.

## 4. Qual a diferença entre ingestão, ETL e RAG?

Essa é a distinção mais importante do projeto.

- Ingestão produz acervo consultável a partir de documentos e fontes remotas.
- ETL transforma e carrega dados estruturados em pipelines próprios.
- RAG consulta um acervo já disponível e decide como recuperar evidência para gerar resposta.

Quando essas três coisas são tratadas como sinônimos, a documentação vira confusa e a operação começa a diagnosticar o lugar errado.

## 5. O que é correlation_id e por que ele é tão importante?

Correlation_id é o identificador lógico que costura a história de uma execução. Ele aparece no boundary HTTP, nos logs, na fila, no worker e nos modelos de leitura operacional.

Sem ele, a investigação de uma execução distribuída vira adivinhação. Com ele, fica possível responder perguntas como: o pedido foi aceito? foi enfileirado? o worker realmente executou? em que etapa morreu? a UI está olhando o run certo?

## 6. Por que o projeto insiste tanto em falhar cedo?

Porque falha silenciosa em sistema corporativo é pior do que erro explícito. Um serviço que aceita uma configuração ruim, entra em um fallback obscuro e entrega um resultado degradado sem deixar claro o que aconteceu cria risco operacional e dificulta auditoria.

A postura do código lido hoje é preferir preflight, validação semântica e bloqueio explícito quando contrato, infraestrutura ou contexto não fecham.

## 7. O que é o runtime moderno do RAG?

É a decisão arquitetural de não tratar a pergunta como uma consulta ingênua sempre igual. O runtime moderno valida contexto mínimo, escolhe estratégia, usa análise da pergunta, executa retrieval com critério e só então gera a resposta.

Na prática, isso separa duas coisas que projetos rasos costumam misturar.

- Procurar evidência.
- Escrever a resposta.

Essa separação melhora diagnóstico. Se uma resposta veio ruim, o operador consegue distinguir melhor se o problema foi evidência insuficiente, estratégia errada ou geração mal orientada.

## 8. O que a AST resolve no escopo agentic?

Ela resolve governança de configuração. Em vez de tratar o YAML agentic como texto livre em todo o ciclo de edição e validação, a AST cria um contrato tipado para draft, validate, confirm, diff e schema.

O ganho prático é previsibilidade. Em fluxos sensíveis, o sistema tenta impedir que uma configuração pareça válida para a UI, mas inválida para o runtime, ou que um fragmento seja aceito sem respeitar o modelo governado.

Esse fluxo não vive só na camada de serviço. A UI administrativa real aparece em `app/ui/static/js/admin-assembly-ast.js`.

## 9. Quando 202 significa sucesso?

Quase nunca significa sucesso final. Em geral, significa apenas que o pedido foi aceito e entregue ao fluxo assíncrono.

O passo seguinte é acompanhar status, task_id, correlation_id e, quando existir, o modelo de leitura operacional específico do domínio. Confundir aceite com conclusão é um dos erros mais comuns em plataformas com fila.

Na ingestão com paralelismo por documento, um dos contratos operacionais mais úteis é `fanout_overview`, porque ele resume a situação agregada do pai e dos filhos.

## 10. Como pensar ferramentas, integrações e canais?

A forma mais segura é separar três níveis.

- Catálogo e governança de tool.
- Runtime que resolve e instancia a tool.
- Canal ou integração externa que consome a capacidade.

Essa separação evita duas confusões comuns: achar que tool documental já está executável no runtime, ou achar que integração externa é apenas um detalhe cosmético do frontend.

## 11. O que mais costuma gerar diagnóstico errado?

Três coisas aparecem o tempo todo.

1. Olhar só o endpoint e ignorar o worker.
2. Olhar só o YAML e ignorar a normalização e o assembly.
3. Olhar só a resposta final e ignorar evidência, status e logs.

Em linguagem simples: o projeto é distribuído demais para ser entendido por um ponto único de observação.

## 12. Como um novo integrante deveria começar a estudar?

A ordem mais produtiva é esta.

1. Entender arquitetura macro.
2. Entender ingestão.
3. Entender RAG.
4. Entender YAML governado e assembly agentic.
5. Entender operação, logs e validação.

A maior parte da confusão desaparece quando essa ordem é respeitada.

## 13. Explicação 101

Pense na plataforma como uma empresa com três áreas.

- Uma área recebe pedidos e organiza contexto.
- Outra executa o trabalho pesado fora do atendimento.
- Outra mantém a operação periódica em ordem.

Enquanto isso, o conhecimento produzido por uma parte do sistema pode ser usado por outra, e tudo precisa deixar rastro suficiente para que alguém consiga reconstruir a história do processo depois.

## 14. Evidências no código

- [src/api/service_api.py](../src/api/service_api.py): boundary HTTP e middlewares principais.
- [app/runners/api_runner.py](../app/runners/api_runner.py): bootstrap do processo API.
- [app/runners/worker_runner.py](../app/runners/worker_runner.py): bootstrap do processo worker.
- [app/runners/scheduler_runner.py](../app/runners/scheduler_runner.py): bootstrap do processo scheduler.
- [src/services/ingestion_service.py](../src/services/ingestion_service.py): fachada do domínio de ingestão.
- [src/qa_layer/content_qa_system.py](../src/qa_layer/content_qa_system.py): fachada do domínio de consulta.
- [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py): governança do escopo agentic.
- [app/ui/static/js/admin-assembly-ast.js](../app/ui/static/js/admin-assembly-ast.js): UI administrativa do fluxo de assembly AST.
