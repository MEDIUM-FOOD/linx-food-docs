# ADR: Big Bang do Catalogo Builtin de Tools para Banco

Status: Aceita

Data: 2026-04-25

## 1. Contexto

O runtime agentic atual ainda depende diretamente do antigo catálogo builtin em arquivo.

As evidencias lidas no codigo mostram isso de forma objetiva:
1. `src/config/config_cli/configuration_factory.py` ainda injeta o catalogo a partir do JSON quando `tools_library` vem vazio no YAML.
2. `src/config/agentic_assembly/tool_resolver.py` ainda usa fallback do JSON e ainda recompõe catalogo por AST em runtime.
3. `src/agentic_layer/tools/tools_library_cache.py` ainda trata o arquivo como fonte canônica do cache.
4. `src/agentic_layer/tools/tools_library_builder.py` ainda gera o arquivo em disco como saida principal.

O problema pratico e direto:
1. o catalogo builtin ja participa do comportamento real do runtime;
2. se banco e JSON conviverem ao mesmo tempo, o sistema passa a ter duas fontes de verdade;
3. isso destrói a governanca de `disabled`, reabre risco de tool reaparecer por outro caminho e inviabiliza o corte sem legado.

## 2. Decisao

Fica decidido que a migracao do catalogo builtin sera feita como corte integral.

Isso significa:
1. `integrations.builtin_tool_registry` passa a ser a unica fonte de verdade do catalogo builtin.
2. o catálogo builtin em arquivo deixa de ser fonte operacional do runtime.
3. o runtime nao pode manter dual-read, dual-write, fallback, compatibilizacao paralela nem overlay AST em tempo de execucao.
4. o builder oficial passa a sincronizar a tabela do banco, e nao mais um arquivo JSON como artefato canonico do runtime.
5. o estado `status` passa a ser a unica fonte de verdade para habilitar e desabilitar tools builtin.
6. rows ja desabilitadas manualmente nao podem ser religadas automaticamente pelo builder ou por endpoint de sincronizacao.

## 3. Escopo da decisao

Esta decisao cobre integralmente:
1. a persistencia da tabela global `integrations.builtin_tool_registry`;
2. o builder/sync do catalogo builtin;
3. o cache do catalogo builtin;
4. a injecao da `ConfigurationFactory`;
5. o `ToolCatalogResolver` e qualquer outro wiring runtime do catalogo;
6. as rotas administrativas globais do catalogo builtin;
7. a UI administrativa da Area Plataforma para esse catalogo;
8. scripts, YAMLs, testes, docs e instrucoes que ainda apontem o JSON como contrato atual.

## 4. Regras obrigatorias do corte

Durante a execucao desta ADR:
1. e proibido manter o catálogo builtin em arquivo como fallback de seguranca.
2. e proibido manter `_load_ast_discovered_catalog()` no runtime como mecanismo de recomposicao.
3. e proibido introduzir leitura dupla de banco e JSON.
4. e proibido introduzir escrita dupla em banco e JSON como transicao.
5. e proibido criar feature flag de compatibilidade para escolher entre JSON e banco.
6. a exclusao fisica deve ficar restrita a rows obsoletas que nao existem mais no conjunto builtin descoberto pelo builder.
7. rows ainda descobertas pelo codigo devem ser desabilitadas, nao deletadas.
8. a sincronizacao manual disparada por endpoint ou UI deve reutilizar o programa oficial de leitura das tools builtin e preservar `status='disabled'` quando esse estado ja existir.

## 5. Consequencia pratica

Na pratica, a equipe deixa de tratar o catalogo builtin como artefato local em disco e passa a tratá-lo como cadastro global da plataforma.

Isso traz as seguintes consequencias:
1. a governanca administrativa de enable, disable, filtro, batch e sincronizacao passa a refletir o estado real do runtime;
2. o runtime agentic deixa de depender de fallback opaco em arquivo;
3. scripts, testes, docs e YAMLs precisam ser alinhados no mesmo ciclo para nao perpetuar a verdade antiga.

## 6. Rollout e rollback

O rollout e incremental apenas na branch de trabalho, mas o comportamento de producao muda em corte unico.

Regras operacionais:
1. nenhuma etapa parcial desta migracao deve ser mergeada em `main` antes do corte completo estar verde.
2. a troca em producao acontece depois da sincronizacao inicial da tabela e do deploy do build novo que le apenas banco.
3. o rollback oficial e rollback de versao da aplicacao.
4. a tabela nova pode permanecer existente no banco apos rollback de versao, desde que o runtime revertido nao a consuma.
5. e proibido usar reativacao de leitura do JSON como saida emergencial.

## 7. Validacao da decisao

Esta ADR e considerada aplicada quando:
1. o runtime nao ler mais o catálogo builtin legado em arquivo.
2. o runtime nao chamar mais `_load_ast_discovered_catalog()`.
3. o builder sincronizar apenas a tabela do banco.
4. a API administrativa global do catalogo builtin estiver publicada.
5. a UI administrativa possuir botao de sincronizacao manual, com preservacao do estado `disabled`.
6. scripts, docs, YAMLs, testes e instrucoes ativos nao apontarem mais o JSON como fonte canonica.
7. a suite oficial fechar verde no gate final e na regressao ampla final.

## 8. Relacao com decisoes anteriores

Esta ADR supera explicitamente a decisao anterior de manter o runtime atual intacto.

Documento superado:
- `docs/ADR-INTEGRATIONS-GATE-G1-FASE1-RUNTIME-ATUAL.md`

Motivo:
1. o escopo anterior era de uma fase administrativa sem corte do runtime;
2. o escopo atual exige substituicao completa do catalogo builtin sem legado;
3. manter ambas como aceitas geraria contradicao direta no mesmo dominio.

## 9. Runbook operacional do corte

Este runbook existe para executar a troca de forma disciplinada, sem fallback escondido e sem reabrir o JSON antigo por conveniencia.

### 9.1. Preparacao da janela

Antes do corte:
1. confirmar que a release candidata ja passou pelo gate minimo oficial do repositorio;
2. confirmar que a busca residual em `src`, `tests`, `scripts` e `app` esta zerada para `tools_library.json` e `_load_ast_discovered_catalog`;
3. confirmar que a versao candidata ja contem o runtime lendo apenas banco, a API admin global e a UI administrativa do catalogo builtin;
4. confirmar que o rollback por versao da aplicacao esta disponivel e documentado para a janela.

Se qualquer item acima falhar, o corte nao comeca.

### 9.2. Ordem obrigatoria de execucao

Durante a janela:
1. aplicar o bootstrap oficial do schema `integrations` pelo fluxo suportado em `scripts/upgrade_integrations_schema.py`;
2. executar o sync inicial do catalogo builtin usando o builder oficial em `src/agentic_layer/tools/tools_library_builder.py`, com sincronizacao forcada do estado descoberto para a tabela global;
3. validar o resultado do sync inicial antes do deploy:
	- total de tools descobertas maior que zero;
	- verificacao de paths sem pendencias;
	- validacao de imports sem falhas;
	- nenhuma row previamente `disabled` religada automaticamente;
4. implantar a versao nova da aplicacao, que le apenas banco;
5. executar o smoke administrativo do catalogo builtin na API e na UI da Area Plataforma;
6. executar a sincronizacao manual pela surface administrativa para comprovar que o fluxo de operacao usa o mesmo mecanismo oficial do builder.

### 9.3. Sinais objetivos de sucesso

O corte so e considerado saudavel quando todos os sinais abaixo forem verdadeiros:
1. a listagem administrativa do catalogo builtin retorna rows do banco com `status` coerente;
2. uma row previamente `disabled` continua `disabled` depois do sync inicial e depois do sync manual;
3. a API administrativa global conclui batch enable, batch disable, delete de obsoletas e sync manual sem depender de `tenant_id`;
4. o runtime agentic continua materializando `tools_library` a partir do banco;
5. o gate minimo oficial fecha verde;
6. a busca residual continua zerada nos diretorios executaveis do produto.

### 9.4. Condicoes que obrigam abortar a janela

Abortar o corte e preparar rollback por versao se ocorrer qualquer um destes sinais:
1. o bootstrap do schema falhar;
2. o builder reportar falha de importacao, path nao verificavel ou total de tools inconsistente;
3. o sync inicial religar row `disabled`;
4. a UI ou a API administrativa nao conseguir listar o catalogo builtin apos o deploy;
5. o gate minimo oficial falhar;
6. a busca residual voltar a mostrar consumidor executavel do JSON legado.

## 10. Checklist objetivo de corte e rollback

### 10.1. Checklist de corte

Marcar cada item como concluido durante a janela:
1. release candidata aprovada no gate minimo oficial;
2. busca residual zerada em `src`, `tests`, `scripts` e `app`;
3. schema `integrations` confirmado pelo bootstrap oficial;
4. sync inicial do builder executado com sucesso;
5. deploy da versao DB-only concluido;
6. smoke administrativo da API concluido;
7. smoke administrativo da UI concluido;
8. sync manual pela UI confirmado sem religar `disabled`;
9. gate minimo oficial executado novamente apos o corte;
10. evidencias de log e paths dos artefatos da rodada salvos no registro da release.

### 10.2. Checklist de rollback por versao

Se o corte precisar voltar:
1. reverter a aplicacao para o build anterior;
2. nao reativar leitura do JSON legado;
3. nao introduzir flag de compatibilidade para escolher fonte de catalogo;
4. manter a tabela nova como artefato inativo do release revertido, sem tentar adaptá-la ao runtime antigo;
5. validar que a versao revertida voltou a responder conforme o contrato anterior dela;
6. registrar a causa do rollback e a evidência que bloqueou a continuidade da janela.
