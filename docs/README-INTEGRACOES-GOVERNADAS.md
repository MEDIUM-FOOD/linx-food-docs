# Integrações Governadas

Este documento cobre o módulo de cadastros governados usado para APIs,
queries SQL, procedures e perfis técnicos reutilizáveis.

## O que o módulo resolve

Ele separa duas coisas diferentes:

- cadastro técnico, como conexão SQL e perfil de autenticação;
- cadastro funcional, como operação HTTP, query SQL e procedure.

Na prática, isso evita repetir segredo, melhora rastreabilidade e deixa
claro o que é infraestrutura e o que é ativo de negócio.

## Leitura relacionada

- Guia geral de tools: [GUIA-USUARIO-TOOLS.md](./GUIA-USUARIO-TOOLS.md)
- APIs dinâmicas governadas: [README-DYNAMIC-API-TOOLS.md](./README-DYNAMIC-API-TOOLS.md)
- SQL dinâmico e procedures: [README-DYNAMIC-SQL-TOOLS.md](./README-DYNAMIC-SQL-TOOLS.md)
- Catálogo alfabético de tools: [tools/alfabetica.md](./tools/alfabetica.md)
- Catálogo por finalidade: [tools/por_finalidade.md](./tools/por_finalidade.md)

## Visão executiva

Executivamente, este módulo reduz um problema caro de escala: segredo,
endpoint, query e procedure deixam de ficar espalhados entre times,
ambientes e YAMLs locais. O ganho real é governança com reaproveitamento.
O cadastro técnico fica separável do ativo funcional, e o teste seguro
permite validar contrato sem abrir um console irrestrito de produção.

## Visão comercial

Comercialmente, integrações governadas ajudam a vender velocidade com
controle. O cliente percebe que a plataforma consegue homologar APIs e
operações SQL sem depender de código novo em toda pequena evolução, mas
também não abre mão de autenticação, rastreabilidade e guardrails.

## Visão estratégica

Estrategicamente, esse catálogo é o elo entre administração humana e
execução agentic. Ele prepara ativos que depois podem virar dyn_api,
dyn_sql ou apoio a NL2SQL, sem duplicar a lógica de cadastro, teste e
publicação em cada superfície nova.

## Trilhas HTTP observadas no runtime

O módulo já aparece no boundary administrativo em três grupos.

- /admin/integrations/technical para conexões SQL e auth profiles.
- /admin/integrations/functional para grupos, operações HTTP, queries e
  procedures.
- /admin/integrations/actions para importação Swagger e testes seguros.

Além disso, o portal do cliente reutiliza o mesmo caso de uso de
importação remota em /client-portal/integrations/swagger-import.

## Como o catálogo se divide

### Cadastros técnicos

Guardam infraestrutura reutilizável.

- conexão SQL governada;
- perfil de autenticação governado.

### Cadastros funcionais

Guardam o ativo que o negócio realmente quer publicar ou testar.

- grupo funcional;
- operação HTTP governada;
- query SQL governada;
- procedure SQL governada.

### Ações seguras

Não são cadastro em si, mas fazem parte da operação do catálogo.

- importar Swagger ou OpenAPI;
- testar operação HTTP;
- testar query SQL;
- testar procedure SQL.

## Guardrails reais do teste seguro

O runtime administrativo não executa teste de integração de forma cega.
Ele aplica barreiras explícitas antes de falar com API externa ou banco.

No teste HTTP, o comportamento observado hoje é este:

- GET, HEAD e OPTIONS passam como modo seguro padrão;
- POST, PUT, PATCH e DELETE ficam bloqueados até o operador marcar
  allow_non_idempotent;
- timeout nunca passa do teto administrativo de 60 segundos;
- parâmetro obrigatório da rota precisa estar presente antes da chamada;
- o preview de resposta mascara headers sensíveis, como Authorization e
  x-api-key.

No teste SQL, também existem limites concretos:

- query SQL passa pelo guardrail de leitura para evitar escrita acidental;
- timeout também é limitado a 60 segundos;
- o preview de linhas é limitado a 200 registros;
- o retorno mostra um recorte operacional, não um dump livre da base.

Em linguagem simples: o módulo foi desenhado para validar cadastro,
conectividade e contrato, não para virar um console irrestrito de
produção.

## Importação remota e reaproveitamento do portal do cliente

A importação Swagger não existe em dois fluxos independentes.
O portal do cliente reutiliza o mesmo caso de uso administrativo de
importação, mudando a porta HTTP, mas não criando uma segunda regra de
negócio paralela.

Na prática, isso reduz divergência entre times.
Quando o operador importa um documento OpenAPI por URI, arquivo ou texto
direto, o sistema centraliza parse, validação, correlação e criação das
operações governadas no mesmo serviço de aplicação.

## Perfis de autenticação e segredo governado

O catálogo técnico não guarda apenas nomes bonitos de conexão.
Ele existe para resolver autenticação e acesso sem espalhar segredo em
cada operação funcional.

Hoje, o teste seguro já reaproveita o perfil governado para montar
headers de autenticação.
Isso inclui cenários como bearer, basic e api_key, usando credencial
criptografada quando necessário.

Em linguagem simples: a operação HTTP não deveria conhecer o segredo em
si. Ela aponta para um perfil técnico, e o runtime monta o header certo
na hora do teste.

## Relação com dyn_api, dyn_sql e NL2SQL

Este módulo não é a mesma coisa que a execução agentic.

Ele prepara o catálogo governado que depois pode alimentar dyn_api e
dyn_sql por tenant. Também se conecta ao endpoint dedicado de NL2SQL
quando o operador quer gerar uma proposta de SQL antes do cadastro
final.

Em linguagem simples: aqui fica a governança e o teste seguro. A
publicação para agentes e a execução do agente ficam em outras camadas.

## O que o módulo não faz sozinho

- não publica tools automaticamente só porque o cadastro existe;
- não expõe segredo cru em payload de resposta;
- não transforma teste seguro em execução cega de produção.
- não ignora confirmação explícita para chamadas HTTP mutáveis.

## Jornada prática resumida

1. Cadastrar a base técnica.
2. Organizar por grupo funcional.
3. Criar ou importar operações HTTP.
4. Cadastrar query ou procedure.
5. Quando necessário, usar NL2SQL para sugerir a query antes do
  cadastro.
6. Executar teste seguro com correlation_id rastreável e guardrails
  ativos.

## Como validar

1. Confirme se o cadastro foi criado na trilha técnica ou funcional
  correta.
2. Confirme se a importação Swagger foi feita pela trilha de actions ou
  pelo portal do cliente.
3. Confirme se o teste HTTP mutável exigiu confirmação explícita.
4. Confirme se o teste SQL respeitou guardrail de leitura, timeout e
  limite de linhas.
5. Confirme se o teste seguro devolveu correlation_id e não vazou
  segredo.
6. Se o objetivo final for agentic, valide depois o uso em dyn_api,
  dyn_sql ou schema_rag_sql.

## Troubleshooting

### A operação existe no catálogo, mas não aparece para agentes

Causa provável: o cadastro funcional foi criado, mas não passou pela
trilha de publicação usada por dyn_api ou dyn_sql.

Como confirmar: diferencie cadastro administrativo de publicação
agentic. O item pode estar válido para teste seguro e ainda não estar
pronto para consumo por agente.

### O teste HTTP devolve bloqueio mesmo com endpoint correto

Causa provável: a operação usa método mutável sem `allow_non_idempotent`
ou há parâmetro obrigatório de rota faltando.

Como confirmar: revise método, flags do teste seguro e presença dos
parâmetros exigidos antes da chamada externa.

### O teste SQL falha mesmo com conexão válida

Causa provável: a query caiu no guardrail de leitura, excedeu timeout ou
tentou devolver mais do que o preview administrativo aceita.

Como confirmar: valide se a query é realmente de leitura e se o retorno
esperado cabe no modo de teste seguro.

## Checklist de entendimento

- Entendi a diferença entre cadastro técnico e cadastro funcional.
- Entendi por que teste seguro não é console livre de produção.
- Entendi como o catálogo administrativo se conecta a dyn_api, dyn_sql e NL2SQL.
- Entendi como diagnosticar falha de importação, autenticação e guardrail.

## Evidência no código

- src/api/routers/admin/integrations_router.py
- src/api/routers/admin/integrations_functional_router.py
- src/api/routers/admin/integrations_actions_router.py
- src/api/routers/client_portal_router.py
- src/api/routers/config_nl2sql_router.py
- src/api/services/admin/integrations_service.py
- src/api/services/admin/integrations_actions_service.py
- src/api/services/client_portal_swagger_import_service.py
- src/api/services/nl2sql_service.py

## Lacunas no código

Não encontrado no código.

Onde deveria estar:

- um painel único de prontidão do catálogo por tenant;
- uma visão administrativa consolidando técnico, funcional, testes e
  publicação agentic no mesmo relatório.
