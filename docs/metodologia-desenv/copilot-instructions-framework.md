# Estratégia de Governança do Copilot na pasta .github

Atualizado com base na estrutura real do repositório e nas práticas
recomendadas pela documentação oficial do GitHub Copilot.

## Visão geral

A pasta .github deste repositório não foi montada como um depósito de
prompts soltos. Ela funciona como uma arquitetura de governança para o
Copilot. A estratégia separa regras globais, regras por tipo de arquivo,
agentes de trabalho e módulos reutilizáveis de procedimento.

A ideia central é simples: cada camada resolve um problema diferente.
O arquivo global define o que vale para quase tudo. As path
instructions evitam poluir a regra global com detalhes de stack. Os
agents organizam fluxos longos de trabalho. As skills reaproveitam
procedimentos recorrentes.

## Por que existe

A documentação do GitHub recomenda que instruções de repositório sejam
relevantes, amplas o suficiente para o escopo em que atuam e livres de
conflito desnecessário. Em repositórios grandes, colocar tudo no mesmo
arquivo piora a resposta do Copilot em vez de melhorar.

Este repositório leva essa ideia um passo adiante. Em vez de usar a
pasta .github apenas para customizar resposta, ela é usada para reduzir
exploração desnecessária, padronizar validação, separar especialidades e
registrar aprendizado operacional.

## Explicação conceitual

Pela documentação oficial do GitHub, o Copilot reconhece três famílias
principais de instruções de repositório: repository-wide instructions,
path-specific instructions e agent instructions. Elas são combinadas por
precedência, e o GitHub recomenda evitar conjuntos conflitantes.

Este repositório usa essas bases, mas adiciona uma camada operacional
própria. Por isso, nem todo arquivo em .github é uma custom instruction
nativa do GitHub.com. Parte da pasta funciona como memória operacional,
parte como backlog de governança, parte como definição de modos/agentes
para o ambiente de desenvolvimento.

## Explicação for dummies

Pense na pasta .github como a sinalização de uma fábrica. Existe uma
placa geral dizendo como a fábrica funciona. Existem placas específicas
para cada setor. Existem líderes especializados para tipos diferentes de
trabalho. E existem cadernos internos com lições, falhas e regras que
precisam ser revisadas de tempos em tempos.

Se tudo fosse uma única placa gigante, ninguém saberia o que vale para
qual contexto. Se cada setor inventasse sua própria regra sem coordenação,
a fábrica viraria caos. A estratégia boa é dividir por responsabilidade,
mas sem criar vinte documentos quase iguais.

## O que o GitHub reconhece nativamente

Segundo a documentação oficial do GitHub Copilot, as superfícies nativas
mais importantes são estas:

- Repository-wide instructions:
  ficam em .github/copilot-instructions.md e guardam contexto amplo do
  repositório, comandos, arquitetura, validação e convenções que se
  aplicam à maior parte das tarefas.
- Path-specific instructions:
  ficam em .github/instructions/*.instructions.md e tiram do arquivo
  global aquilo que só vale para certos caminhos, stacks ou tipos de
  arquivo.
- Agent instructions:
  vivem em AGENTS.md, CLAUDE.md ou GEMINI.md e orientam agentes com foco
  mais operacional.

Ponto importante: a documentação oficial não trata .agent.md e SKILL.md
como formatos nativos do GitHub.com para repository custom instructions.
Esses formatos fazem parte da camada operacional usada neste repositório
em ambiente de agente e editor.

## Regra de precedência

A ordem relevante da documentação oficial é:

1. instruções pessoais;
2. instruções path-specific aplicáveis;
3. instruções repository-wide;
4. agent instructions;
5. instruções da organização.

Na prática, isso significa que uma regra crítica para todo o repositório
não deve ficar escondida apenas em um agent ou em uma skill. Se ela
precisa valer de forma ampla, deve morar em uma superfície oficial e
abrangente.

## Como este repositório organiza a pasta .github

A estratégia local pode ser lida em quatro blocos.

- Regras oficiais amplas:
  definem o contrato geral do repositório e são superfície oficial.
- Regras oficiais por caminho:
  especializam por stack e tipo de arquivo e também são superfície
  oficial.
- Orquestração por agent mode:
  separa fluxos de trabalho por especialidade e é camada operacional do
  repositório.
- Memória e auditoria:
  registram lições, defeitos de governança e backlog operacional; são
  artefatos internos de governança.

## Estratégia de cada arquivo de instrução em .github

### Núcleo global e memória de governança

- .github/copilot-instructions.md:
  é a constituição operacional do repositório. Deve receber apenas regras
  transversais, amplas e realmente úteis para quase toda tarefa.
  Não deve acumular detalhe de stack, diretório ou fluxo muito
  especializado.
- .github/lessons-instructions.md:
  é a memória institucional de lições aprendidas. Deve ser atualizado
  quando houver correção recorrente, feedback forte do usuário ou regra
  preventiva nova. Não deve substituir o arquivo global.
- .github/bad-instructions.md:
  é o backlog de defeitos da própria governança. Serve para registrar
  conflito, redundância, ambiguidade e arquivos vazios. Não é o lugar de
  escrever regra nova.
- .github/error-log-instruction.md:
  registra incidentes concretos de execução. É útil para preservar
  evidência histórica de erro real, mas não deve virar manual geral de
  desenvolvimento.
- .github/error-backlog-instructions.md:
  guarda backlog detalhado de falhas observadas e correções estruturais
  pendentes. Ele funciona como memória operacional, não como contrato
  global de como programar.
- .github/regression-logs-instructions.md:
  funciona como diário de regressões recorrentes. Deve ser usado quando a
  mesma família de defeito reaparece e precisa ser tratada como padrão de
  risco.

Leitura estratégica: os três primeiros arquivos são governança e
metagovernança. Os três últimos são memória operacional baseada em erro.
Eles ajudam o agente a errar menos, mas não substituem as custom
instructions oficiais do GitHub.

### Path-specific instructions

- .github/instructions/docs.instructions.md:
  cobre documentação em docs e app/docs. Existe para separar regras de
  público-alvo, didática e anti-fragmentação do restante do repositório.
- .github/instructions/html.instructions.md:
  cobre UI e frontend em app/ui. Existe para isolar UX, contratos
  visuais e validação de frontend sem poluir o arquivo global.
- .github/instructions/python.instructions.md:
  cobre backend Python e testes Python. Concentra política de tipagem,
  suíte shell e validação do backend.
- .github/instructions/sh.instructions.md:
  cobre scripts shell. Mantém uma regra mínima, direta e específica para
  permissão de execução e uso correto de .sh.
- .github/instructions/yaml.instructions.md:
  cobre YAMLs do produto e assembly agentic. Existe porque YAML possui
  contrato próprio e sensível demais para ser tratado como detalhe do
  backend genérico.

Leitura estratégica: essas cinco instruções implementam exatamente a
ideia recomendada pelo GitHub de tirar do arquivo global o que só vale
para um subconjunto de arquivos.

## Estratégia dos agents

Antes de tudo, há um esclarecimento importante. Pela documentação
oficial, agent instructions nativas em GitHub.com vivem em AGENTS.md,
CLAUDE.md ou GEMINI.md. Já este repositório usa .github/agents/*.agent.md
como camada operacional de modos especializados dentro do ambiente de
agente.

Em termos de estratégia, isso é válido e útil. Em termos de portabilidade,
porém, regras críticas não devem depender apenas desses arquivos.

- planejar.agent.md:
  transforma pedidos em plano executável e verificável. Existe separado
  porque planejar bem é diferente de implementar bem.
- investigar.agent.md:
  faz leitura forense do código e do wiring real. Existe para evitar
  implementação baseada em nome de arquivo ou intuição.
- implementar.agent.md:
  executa mudanças seguras, incrementais e verificadas. Existe para
  manter foco em entrega sem misturar com auditoria ampla.
- criar-testes.agent.md:
  acrescenta proteção automatizada onde ela reduz risco real. Existe
  porque criar testes úteis exige critério diferente de apenas rodar
  testes.
- executar-testes.agent.md:
  fecha o loop de estabilização pela suíte oficial. Existe porque testar
  bem é um fluxo próprio, com artefatos e diagnóstico.
- corrigir-erros-com-log.agent.md:
  faz debugging guiado por log, stack trace e evidência. Existe para
  separar causalidade de simples patching.
- documentar.agent.md:
  produz documentação de feature e assunto com didática alta. Existe
  porque documentação boa é um trabalho distinto de implementar.
- sincronizar-documentacao.agent.md:
  mantém docs alinhados ao runtime real. Existe para evitar dispersão e
  documentos donos concorrentes.
- tutorial-101.agent.md:
  converte conhecimento técnico em onboarding progressivo. Existe porque
  tutorial e manual técnico não cumprem o mesmo papel.
- inventario-yaml.agent.md:
  mapeia superfícies YAML e seus contratos. Existe porque YAML é espinha
  dorsal do produto e pede leitura própria.
- validar-instructions.agent.md:
  audita a própria governança do Copilot. Existe para proteger o
  repositório contra degradação normativa.

## Estratégia das skills

Skills não são a mesma coisa que agents. O agent organiza um modo de
trabalho inteiro. A skill encapsula um procedimento reutilizável e mais
estreito.

A melhor prática aqui é a mesma defendida pelo GitHub para instruções:
modularidade, escopo claro, baixa duplicação e reaproveitamento real.

- .github/skills/planejar/SKILL.md:
  módulo reutilizável para planejamento crítico. Hoje está implementado e
  é boa opção quando o resultado esperado é plano, não código.
- .github/skills/refatorar/SKILL.md:
  módulo reutilizável para refatoração segura baseada em evidência. Hoje
  está implementado e reforça leitura prévia e hierarquia de verdade.
- .github/skills/testar/SKILL.md:
  módulo reutilizável para estratégia de validação e encerramento. Hoje
  está implementado e centraliza raio de impacto e gate final.
- .github/skills/testes-criar/SKILL.md:
  slot reservado para criação de testes. O arquivo está vazio, então hoje
  representa intenção arquitetural, não governança ativa.
- .github/skills/testes-executar/SKILL.md:
  slot reservado para execução de testes. O arquivo está vazio, então
  hoje representa intenção arquitetural, não governança ativa.

Ponto crítico: skill vazia cria falsa sensação de cobertura normativa.
Estratégia boa não é ter muitos arquivos. Estratégia boa é ter poucos
arquivos com contrato real.

## O que esta pasta faz bem

Há quatro decisões fortes neste desenho.

1. Separa regra global de regra por stack.
2. Separa execução, investigação, teste e documentação em modos
   especializados.
3. Mantém memória explícita de erro, lição e regressão.
4. Trata a própria governança como algo auditável, e não como texto
   sagrado.

Essas decisões estão alinhadas com o espírito da documentação oficial do
GitHub: escopo correto, contexto útil, baixa duplicação e instruções
relevantes para reduzir exploração desnecessária.

## Cuidados estratégicos importantes

### 1. Não confundir mecanismo oficial com camada interna

Se a meta é influenciar Copilot code review, Copilot cloud agent ou uso
em GitHub.com, o que tem garantia oficial é copilot-instructions,
path-specific instructions e agent instructions no formato suportado pelo
GitHub.

Conclusão prática: uma regra crítica do repositório não deve existir só
em .agent.md ou SKILL.md.

### 2. Evitar conflito entre camadas

O GitHub recomenda evitar instruções conflitantes. Neste repositório,
como a governança é densa, a disciplina correta é:

- regra ampla no arquivo global;
- detalhe de stack no arquivo com applyTo;
- fluxo de trabalho no agent;
- procedimento reaproveitável na skill;
- aprendizado e auditoria em arquivos de memória.

### 3. Lembrar do limite prático do code review

A documentação oficial indica que o Copilot code review considera apenas
os primeiros 4.000 caracteres de cada arquivo de instrução. Isso não
limita chat e cloud agent do mesmo jeito, mas muda a estratégia.

Conclusão prática: tudo o que for essencial para revisão automática deve
aparecer cedo e com redação direta nas superfícies oficiais.

### 4. Arquivo vazio é dívida normativa

Uma skill vazia ou uma instrução sem contrato operacional não ajuda o
Copilot. Ela só aumenta ambiguidade. Se o arquivo não orienta nenhuma
ação real, ele deve ser preenchido ou removido.

## Como decidir onde uma nova regra deve entrar

Use esta régua simples.

- Se vale para quase todo o repositório:
  use .github/copilot-instructions.md.
- Se vale só para Python, YAML, HTML, docs ou shell:
  use .github/instructions/*.instructions.md.
- Se descreve um fluxo de trabalho inteiro:
  use .github/agents/*.agent.md.
- Se descreve um procedimento estreito e reaproveitável:
  use .github/skills/**/SKILL.md.
- Se é aprendizado após erro real:
  use .github/lessons-instructions.md.
- Se é defeito da própria governança:
  use .github/bad-instructions.md.
- Se é incidente ou regressão observada:
  use error-log, error-backlog ou regression-logs.

## Exemplos de uso prático

- Se a regra nova for sobre como validar backend Python, ela deve ir para
  python.instructions.md, não para o arquivo global.
- Se a regra nova for sobre como conduzir investigação forense, ela deve
  fortalecer investigar.agent.md, não docs.instructions.md.
- Se surgiu uma lição após o usuário corrigir uma falsa premissa, ela
  deve entrar em lessons-instructions.md, não em error-backlog.
- Se a intenção é criar um modo especializado de onboarding, o lugar é um
  agent ou tutorial, não uma path instruction de stack.

## Impacto para o repositório

Essa arquitetura reduz a quantidade de contexto que o agente precisa
redescobrir a cada sessão. Também melhora a previsibilidade do trabalho,
porque cada camada deixa claro o que ela governa e o que ela não governa.

O benefício prático mais importante é este: o Copilot deixa de depender
só do prompt do momento e passa a operar dentro de uma estrutura de
regras, memória e especialização.

## Limites e pegadinhas

- Nem todo arquivo em .github é uma custom instruction nativa do GitHub.
- Nem toda regra importante hoje está na superfície mais portátil.
- Skills vazias não devem ser tratadas como cobertura real.
- Quanto mais densa a governança, maior a necessidade de auditoria contra
  redundância e conflito.

## Troubleshooting

Se o Copilot parecer ignorar uma regra, a primeira pergunta correta não é
se o modelo falhou. A primeira pergunta correta é: essa regra está na
camada certa para o efeito esperado?

Se o problema acontecer em code review, confirme se a regra está em uma
superfície oficial, se aparece cedo no arquivo e se não depende apenas de
agent ou skill local.

Se o problema acontecer no editor com modos especializados, confirme se o
agent ou a skill realmente possui contrato materializado e não está vazio.

## Evidência no repositório

- .github/copilot-instructions.md
- .github/instructions/docs.instructions.md
- .github/instructions/html.instructions.md
- .github/instructions/python.instructions.md
- .github/instructions/sh.instructions.md
- .github/instructions/yaml.instructions.md
- .github/agents/
- .github/skills/
- .github/lessons-instructions.md
- .github/bad-instructions.md

## Referências oficiais consideradas

- GitHub Docs: About customizing GitHub Copilot responses
- GitHub Docs: Adding repository custom instructions for GitHub Copilot

## Lacunas atuais observadas

As skills testes-criar e testes-executar existem como intenção de
arquitetura, mas ainda não possuem contrato operacional escrito.

Além disso, a estratégia local usa .agent.md e SKILL.md de forma útil no
ambiente de desenvolvimento, porém isso não substitui a necessidade de
manter regras críticas nas superfícies oficialmente reconhecidas pelo
GitHub quando a meta for portabilidade para GitHub.com, code review e
cloud agent.
