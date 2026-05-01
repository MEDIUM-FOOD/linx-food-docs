# ADR: Gate G1 das Integrações na Fase 1

Status: Superada

Data: 2026-04-25

## Nota de supersessao

Esta ADR foi superada pela decisao mais recente de hard cut do catalogo builtin para banco.

Documento substituto:
- `docs/ADR-BIG-BANG-CATALOGO-BUILTIN-TOOLS-BANCO.md`

Motivo da supersessao:
1. a decisao anterior congelava o runtime agentic para uma fase futura;
2. a decisao atual muda o escopo e passa a exigir corte integral sem legado, sem fallback e sem paralelo;
3. manter as duas ADRs como aceitas ao mesmo tempo criaria conflito operacional e risco de implementacao contraditoria.

## 1. Contexto

O plano de integrações abriu uma dúvida importante sobre a publicação agentic.

As evidências lidas no código mostram que o runtime atual ainda depende do catálogo builtin em arquivo:
1. `src/config/config_cli/configuration_factory.py` ainda injeta o catálogo raiz quando ele vem vazio.
2. `src/config/agentic_assembly/tool_resolver.py` ainda usa fallback para o catálogo builtin em arquivo.
3. O histórico desta iniciativa também já registrou uma segunda fase futura para migrar esse arquivo para tabelas de banco por tenant.

O risco prático é simples:
1. se a fase 1 tentar mexer agora no runtime agentic,
2. o projeto pode acabar com uma convivência híbrida mal definida,
3. com duas fontes de verdade e risco de vazamento entre tenants.

## 2. Decisão

Fica decidido que, na fase 1 do módulo de integrações, o runtime agentic permanece exatamente como está hoje.

Isso significa:
1. o catálogo builtin em arquivo continua sendo usado pelo runtime atual sem mudança estrutural nesta fase.
2. Nenhuma das trilhas de runtime agentic do plano original entra em execução agora.
3. A migração desse catálogo para tabelas de banco por tenant fica reservada para a fase 2.

Decisão textual do usuário que fecha o Gate G1:

"Deixe como está atualmente porque iremos ter uma segunda fase para cuidar disso. Esse arquivo será migrado para tabelas no banco por tenant. Mas não agora."

## 3. Escopo da Fase 1

Esta decisão libera apenas o bloco comum do plano:
1. schema `integrations`;
2. persistência;
3. CRUD administrativo;
4. importação Swagger/OpenAPI;
5. endpoints de teste;
6. endpoint dedicado de NL2SQL;
7. endpoint de serviço para importação remota;
8. interfaces administrativas dentro da Área Plataforma.

Ficam fora da fase 1:
1. overlay tenant-aware sobre o catálogo agentic;
2. hard cut removendo o catálogo builtin em arquivo do runtime;
3. qualquer mudança que altere a publicação agentic hoje em produção.

## 4. Consequência prática

Na prática, a fase 1 entrega governança administrativa sem alterar o modo como os agentes descobrem tools hoje.

Isso reduz risco porque:
1. o cadastro novo pode nascer com boa modelagem, testes e UI;
2. a equipe não força uma mudança transversal no catálogo antes da hora;
3. a fase 2 passa a começar de uma base administrativa pronta e auditável.

## 5. Guard rails operacionais

Durante a fase 1:
1. é proibido alterar `configuration_factory.py` para trocar a fonte de verdade do catálogo agentic;
2. é proibido alterar `tool_resolver.py` para projetar ativos do banco no catálogo efetivo;
3. é proibido criar sincronização automática do catálogo builtin em arquivo com as novas tabelas;
4. os campos `publish_to_agents` podem existir nas tabelas, mas não passam a publicar nada no runtime nesta fase.

## 6. Validação da decisão

Esta ADR é considerada aplicada quando:
1. o plano consolidado referencia explicitamente a manutenção do runtime atual na fase 1;
2. a execução da fase 1 não toca no wiring agentic;
3. as tarefas de runtime agentic ficam registradas como fase 2.

## 7. Rollback

Se esta decisão precisar ser revista ainda na fase 1, o correto não é improvisar um meio-termo.

O correto é:
1. abrir nova decisão técnica explícita;
2. escolher conscientemente a trilha tenant-aware ou a trilha hard cut;
3. replanejar o restante do rollout antes de alterar o runtime.
