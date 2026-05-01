# Suite Oficial de Testes

Atualizado com base no runner oficial atual.

## Visão geral

A validação canônica do repositório passa pelo script
scripts/suite_de_testes_padrao.sh. Ele não é só um atalho para pytest.
Ele organiza checkpoints, controla retomada, grava telemetria estruturada
em .sandbox/tmp e separa modos de execução para backend hermético,
frontend oficial, infraestrutura real e validações dedicadas de PDF.

## Por que existe

O projeto tem muitos testes e vários tipos de validação. Se cada pessoa
rodar um conjunto diferente de comandos, o resultado vira ruído. A suíte
oficial existe para padronizar o que significa validar de forma rápida,
validar de forma ampla e validar infraestrutura real sem misturar tudo no
mesmo pacote.

## Explicação conceitual

O runner oficial carrega .venv, define variáveis de ambiente para modo de
validação e monta checkpoints numerados. Sem reset, ele tenta retomar do
último checkpoint compatível já concluído. Durante a rodada, a suíte gera
um diretório próprio com run.log, logs por etapa, sinais recebidos,
metadados e um snapshot JSON estruturado.

Esse desenho resolve três problemas práticos. Primeiro, evita repetir uma
suíte longa inteira quando a falha aconteceu no meio. Segundo, dá um
ponto único de auditoria para terminal, Copilot e investigação forense.
Terceiro, separa gates pequenos, gates amplos e execuções dedicadas, como
PDF real e infraestrutura real.

## Explicação for dummies

Pense nessa suíte como uma inspeção de aeronave com checklist salvo. Se a
inspeção parou na etapa 7, você não precisa recomeçar do parafuso 1.
Você corrige o problema e continua do último ponto confiável.

Ela também funciona como uma caixa-preta. Mesmo que o terminal role
rápido demais, a execução deixa um retrato gravado em arquivo mostrando o
que rodou, o que falhou, quanto tempo levou e onde estão os logs.

## Como usar na prática

Fluxo recomendado no dia a dia:

1. No ciclo implementar, testar e corrigir, use o modo
   --focus-paths do runner oficial apontando para CAMINHOS_DE_TESTE.
2. Para um resumo operacional do repositório, use
   source .venv/bin/activate && ./scripts/suite_de_testes_padrao.sh --status-repo.
3. Para provar infraestrutura real em modo dedicado, use
   source .venv/bin/activate && ./scripts/suite_de_testes_padrao.sh --infra-real-gate.
4. Para fechar a tarefa, use
   source .venv/bin/activate && ./scripts/suite_de_testes_padrao.sh --all-tests.

Regra operacional importante: não redirecione a saída com > arquivo.
O próprio runner já espelha no terminal e grava seus artefatos em
.sandbox/tmp/full_suite_runs.

## Modos principais

### Focus paths

É o modo rápido para o laço local. Ele executa apenas os testes
relacionados aos arquivos ou caminhos informados e evita abrir um gate
amplo cedo demais.

### Final gate

É o gate hermético de backend. Ele serve quando você precisa validar o
núcleo de backend sem trazer infraestrutura real para a mesma rodada.

### Status repo

É a leitura operacional compacta do repositório. Pelo runner oficial,
isso combina backend hermético com a suíte oficial de frontend.

### Infra real gate

É a prova dedicada de infraestrutura real. Ele existe para validar o que
realmente depende de recurso externo sem contaminar o gate hermético.

### All tests

É o fechamento amplo oficial. Esse é o modo que reúne 100% dos testes
automatizados habilitados pela suíte.

### Modos dedicados de PDF

As validações com PDF real não entram por padrão nos pacotes prontos.
Elas rodam em chamadas dedicadas, como:

- --with-pdf-engine-matrix
- --with-pdf-benchmark-catalog
- --with-pdf-real-fixture

Na prática, isso evita que uma validação pesada e específica de PDF vire
custo invisível em toda rodada comum.

## Paralelização e retomada

A suíte tem política explícita de paralelização por bloco. O padrão é
safe, mas o runner também aceita balanced, aggressive e off. Isso quer
dizer que paralelismo não é um detalhe escondido: ele faz parte do
contrato operacional do runner.

A retomada também é explícita. Sem --reset, a suíte tenta continuar do
último checkpoint salvo compatível com o modo atual. Quando você quer
forçar a execução do zero, usa --reset.

## Artefatos que importam

Os artefatos principais ficam em .sandbox/tmp.

Arquivos compartilhados entre rodadas:

- full_suite_state
- full_suite_total_tests.cache
- full_suite_error_history.jsonl
- full_suite_latest_run_dir
- full_suite_latest_run_log
- full_suite_latest_telemetry.json
- full_suite_telemetry_history.jsonl

Artefatos por rodada:

- .sandbox/tmp/full_suite_runs/NOME_DA_RODADA/run.log
- .sandbox/tmp/full_suite_runs/NOME_DA_RODADA/steps/
- .sandbox/tmp/full_suite_runs/NOME_DA_RODADA/signals.log
- .sandbox/tmp/full_suite_runs/NOME_DA_RODADA/run_meta.env
- .sandbox/tmp/full_suite_runs/NOME_DA_RODADA/suite_telemetry.json

## Como auditar uma execução

A ordem correta de leitura é:

1. abrir .sandbox/tmp/full_suite_latest_telemetry.json;
2. verificar run.status, snapshotStatus, totalFailures e steps;
3. se houver falha, abrir o latest run log;
4. ler os logs raw e clean das etapas falhas;
5. conferir signals.log quando houver encerramento anômalo.

Isso importa porque o terminal sozinho não é a fonte final de verdade.
A suíte já persiste a evidência operacional em arquivos próprios.

## Impacto para o usuário

Na rotina de desenvolvimento, essa suíte reduz retrabalho e evita falsas
conclusões do tipo passou aqui, mas falhou no CI. Na rotina de operação,
elas deixa claro quando uma falha é de backend hermético, de frontend,
de infraestrutura real ou de um modo dedicado como PDF.

## Pegadinhas

- pytest direto pode ajudar no diagnóstico, mas não substitui o runner
  oficial.
- --status-repo não substitui o fechamento com --all-tests.
- PDF real e benchmark PDF são execuções dedicadas, não parte automática
  dos pacotes principais.
- Sem --reset, a suíte reaproveita checkpoint salvo; isso é útil, mas
  também exige atenção para saber de onde a rodada retomou.

## Troubleshooting

Se a suíte falhar no meio, não comece adivinhando pelo terminal. Leia a
telemetria da última rodada e os logs da etapa falha.

Se houver erro visível em vermelho, traceback, FAILED ou ERROR, trate
isso como evidência real e reconcilie com run.log e com os arquivos da
etapa.

Se a execução parecer lenta demais, verifique primeiro o modo usado,
política de paralelização, presença de browser, contract, slow,
infraestrutura real ou validações dedicadas de PDF.

## Evidência no código

- scripts/suite_de_testes_padrao.sh
- scripts/run_frontend_suite.sh
- tests/unit/test_suite_de_testes_padrao_cli.py

## Lacunas no código

Não foi encontrado um painel único da própria aplicação que exponha a
telemetria da suíte sem leitura direta dos arquivos em .sandbox/tmp.
Hoje a auditoria operacional da suíte ainda depende desses artefatos
locais gerados pelo runner.
