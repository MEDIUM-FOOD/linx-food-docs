# 🗄️ SQL Dinâmico - Documentação Completa

> **Ferramenta**: `dyn_sql<query_id>`
> **Versão**: 2.0
> **Categoria**: domain_tools
> **Fabricante**: Plataforma de Agentes de IA Custom

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Por Que Usar](#-por-que-usar)
3. [Configuração YAML](#-configuração-yaml)
4. [Casos de Uso Corporativos](#-casos-de-uso-corporativos)
5. [Exemplos Práticos](#-exemplos-práticos)
6. [Parâmetros e Tipos](#-parâmetros-e-tipos)
7. [Múltiplas Conexões](#-múltiplas-conexões)
8. [Boas Práticas](#-boas-práticas)
9. [Pré-requisitos](#-pré-requisitos)
10. [Checklist operacional MSSQL](#-checklist-operacional-mssql)
11. [Troubleshooting](#-troubleshooting)
12. [Migração de Versões](#-migração-de-versões)

---

## 🎯 Visão Geral

**SQL Dinâmico** permite executar queries SQL parametrizadas configuradas
via YAML ou publicadas por tenant na tabela `integrations.sql_query_registry`,
sem escrever código Python para cada consulta.

### Como Funciona

```
Agente solicita dados
        ↓
Sistema procura a query no YAML e, se não achar, no registro por tenant
        ↓
Substitui parâmetros (@p1, @p2, etc.)
        ↓
Executa no banco configurado
        ↓
Retorna resultado ao agente
```

### Características Principais

- **Configuração via YAML**: Queries definidas no arquivo de config
- **Cadastro por tabela**: Queries publicadas por `query_code` no registro de integrações
- **Lazy Loading**: Carrega sob demanda (economia de memória)
- **Parametrização inline**: `@p1`, `@p2`, etc.
- **Múltiplas conexões**: Vários bancos no mesmo agente
- **Pool de conexões**: Performance otimizada
- **Segurança**: SQL Injection protegido por parametrização
- **Multi-banco**: PostgreSQL, MySQL, SQLite, SQL Server

---

## 💡 Por Que Usar?

### Antes (SQL Executor Tradicional)

**Problemas**:

- Query fixa no código
- Difícil parametrizar
- Uma conexão por vez
- Requer código Python

### Depois (SQL Dinâmico)

**Soluções**:

- Queries no YAML
- Parametrização inline
- Múltiplas conexões
- 100% YAML

### Comparação

| Característica | SQL Tradicional | SQL Dinâmico |
|----------------|-----------------|--------------|
| Configuração | Código Python | YAML |
| Parametrização | Manual | Automática (@p1) |
| Múltiplas Queries | Difícil | Simples |
| Múltiplos Bancos | Complexo | Nativo |
| Lazy Loading | Não | Sim |
| Segurança | Manual | Automática |

---

## ⚙️ Configuração YAML

O YAML continua sendo a primeira fonte. Quando `dyn_sql<query_id>` aponta
para uma query que existe em `tools_config.sql_dynamic.queries`, o runtime
usa essa definição e não consulta o banco de integrações.

Quando o `query_id` não existe no YAML, ele passa a ser interpretado como
`query_code` de `integrations.sql_query_registry`. Nesse caso, a sessão
precisa ter `user_session.tenant_id`, a query precisa estar ativa e publicada
para agentes, e a conexão associada precisa estar ativa e marcada como
somente leitura.

Em termos práticos, isso permite manter YAMLs menores: o agente declara
`dyn_sql<clientes_ativos>`, e a plataforma busca a query aprovada do tenant
quando ela não está no próprio YAML.

### Estrutura Básica

```yaml
# 1. Declarar ferramentas no agente
multi_agents:
  - id: "meu_agente"
    tools:
      - "dyn_sql<query1>"    # Nome da query
      - "dyn_sql<query2>"    # Outra query

# 2. Configurar conexões e queries
local_tools_configuration:
  sql_dynamic:
    # Conexões disponíveis
    connections:
      nome_conexao:
        secret_key: "CHAVE_DB_URL"  # Do arquivo .keys.yaml
        pool_size: 5                # Opcional
        timeout: 30                 # Opcional

    # Queries disponíveis
    queries:
      query1:
        connection: "nome_conexao"
        description: "O que essa query faz"
        sql: "SELECT * FROM tabela WHERE id = @p1"
        parameters:
          - name: "id"
            type: "integer"
            description: "ID do registro"
```

### Parâmetros de Conexão

| Parâmetro | Obrigatório | Padrão | Descrição |
|-----------|-------------|--------|-----------|
| `secret_key` |  Sim | - | Chave da URL no .keys.yaml |
| `pool_size` |   Não | 5 | Tamanho do pool de conexões |
| `timeout` |   Não | 30 | Timeout em segundos |
| `echo` |   Não | false | Log de SQL (debug) |

### Parâmetros de Query

| Parâmetro | Obrigatório | Descrição |
|-----------|-------------|-----------|
| `connection` |  Sim | Nome da conexão a usar |
| `description` | ⚠️ Recomendado | Descrição para o agente |
| `sql` |  Sim | Query SQL com @p1, @p2... |
| `parameters` | ⚠️ Recomendado | Lista de parâmetros |

---

## Pré-requisitos

### Nota centralizada sobre drivers

Para SQL Server, a conexão depende de um driver DBAPI compatível
instalado no ambiente. O sistema aceita DSNs SQLAlchemy com prefixos
`mssql`, `mssql+pyodbc` ou `mssql+pymssql`, mas a execução só funciona
quando o driver correspondente estiver disponível no ambiente.

---

## Checklist operacional MSSQL

Use este checklist antes de ativar uma conexão SQL Server.

1) Validar se o driver DBAPI está instalado no ambiente de execução.
2) Confirmar que a URI de conexão usa um prefixo SQLAlchemy compatível
  com o driver disponível.
3) Testar a conexão com uma consulta simples no banco.
4) Validar se a sintaxe usada no `call_template` é aceita pelo SQL
  Server; se necessário, definir o template específico no YAML.
5) Registrar logs com `correlation_id` durante o teste de conexão.

---

## 💼 Casos de Uso Corporativos

### 1. Dashboard de Vendas

**Objetivo**: Relatórios de vendas sob demanda

```yaml
tools:
  - "dyn_sql<vendas_mes>"
  - "dyn_sql<top_produtos>"
  - "dyn_sql<vendas_loja>"

local_tools_configuration:
  sql_dynamic:
    connections:
      erp:
        secret_key: "ERP_DB_URL"

    queries:
      vendas_mes:
        connection: "erp"
        description: "Vendas totais do mês"
        sql: |
          SELECT
            SUM(valor_total) as total,
            COUNT(*) as qtd_vendas
          FROM vendas
          WHERE EXTRACT(MONTH FROM data_venda) = @p1
            AND EXTRACT(YEAR FROM data_venda) = @p2
        parameters:
          - name: "mes"
            type: "integer"
          - name: "ano"
            type: "integer"
```

**Uso**:

```
Agente: "Qual o total de vendas de setembro/2025?"
→ dyn_sql<vendas_mes>(mes=9, ano=2025)
→ Resultado: R$ 1.234.567,00
```

---

### 2. Consulta de Estoque

**Objetivo**: Verificar disponibilidade de produtos

```yaml
tools:
  - "dyn_sql<checar_estoque>"
  - "dyn_sql<produtos_baixo_estoque>"

local_tools_configuration:
  sql_dynamic:
    connections:
      estoque:
        secret_key: "ESTOQUE_DB_URL"

    queries:
      checar_estoque:
        connection: "estoque"
        description: "Verifica estoque de um produto"
        sql: |
          SELECT
            p.nome,
            e.quantidade,
            e.estoque_minimo,
            CASE
              WHEN e.quantidade < e.estoque_minimo THEN 'CRÍTICO'
              WHEN e.quantidade < e.estoque_minimo * 2 THEN 'BAIXO'
              ELSE 'OK'
            END as status
          FROM produtos p
          JOIN estoque e ON p.id = e.produto_id
          WHERE p.codigo = @p1
        parameters:
          - name: "codigo_produto"
            type: "string"
```

---

### 3. Análise de Clientes

**Objetivo**: Insights sobre comportamento de clientes

```yaml
tools:
  - "dyn_sql<clientes_inativos>"
  - "dyn_sql<clientes_vip>"

local_tools_configuration:
  sql_dynamic:
    connections:
      crm:
        secret_key: "CRM_DB_URL"

    queries:
      clientes_inativos:
        connection: "crm"
        description: "Clientes sem compra há X dias"
        sql: |
          SELECT
            c.id,
            c.nome,
            c.email,
            MAX(v.data_venda) as ultima_compra,
            CURRENT_DATE - MAX(v.data_venda) as dias_inativo
          FROM clientes c
          LEFT JOIN vendas v ON c.id = v.cliente_id
          GROUP BY c.id, c.nome, c.email
          HAVING CURRENT_DATE - MAX(v.data_venda) > @p1
          ORDER BY dias_inativo DESC
          LIMIT @p2
        parameters:
          - name: "dias"
            type: "integer"
            default: 90
          - name: "limite"
            type: "integer"
            default: 50
```

---

### 4. Monitoramento de Performance

**Objetivo**: KPIs em tempo real

```yaml
tools:
  - "dyn_sql<kpis_diarios>"

local_tools_configuration:
  sql_dynamic:
    connections:
      analytics:
        secret_key: "ANALYTICS_DB_URL"

    queries:
      kpis_diarios:
        connection: "analytics"
        description: "KPIs do dia"
        sql: |
          WITH vendas_hoje AS (
            SELECT
              COUNT(*) as num_vendas,
              SUM(valor_total) as receita,
              AVG(valor_total) as ticket_medio
            FROM vendas
            WHERE DATE(data_venda) = @p1
          ),
          vendas_ontem AS (
            SELECT
              COUNT(*) as num_vendas,
              SUM(valor_total) as receita
            FROM vendas
            WHERE DATE(data_venda) = @p1 - INTERVAL '1 day'
          )
          SELECT
            vh.*,
            ROUND((vh.receita - vo.receita) / vo.receita * 100, 2)
              as crescimento_pct
          FROM vendas_hoje vh, vendas_ontem vo
        parameters:
          - name: "data"
            type: "date"
```

---

## 📚 Exemplos Práticos

### Exemplo 1: Consulta Simples

**Buscar cliente por ID**

```yaml
queries:
  buscar_cliente:
    connection: "crm"
    description: "Busca cliente por ID"
    sql: "SELECT * FROM clientes WHERE id = @p1"
    parameters:
      - name: "id_cliente"
        type: "integer"
        description: "ID do cliente"
```

**Execução**:

```
→ dyn_sql<buscar_cliente>(id_cliente=123)
→ Retorna dados do cliente #123
```

---

### Exemplo 2: Query com JOIN

**Pedidos com detalhes**

```yaml
queries:
  pedidos_cliente:
    connection: "erp"
    description: "Pedidos de um cliente"
    sql: |
      SELECT
        p.id,
        p.data_pedido,
        p.status,
        p.valor_total,
        c.nome as cliente_nome
      FROM pedidos p
      JOIN clientes c ON p.cliente_id = c.id
      WHERE p.cliente_id = @p1
        AND p.data_pedido >= @p2
      ORDER BY p.data_pedido DESC
    parameters:
      - name: "cliente_id"
        type: "integer"
      - name: "data_inicio"
        type: "date"
```

---

### Exemplo 3: Agregações

**Relatório de vendas por categoria**

```yaml
queries:
  vendas_categoria:
    connection: "erp"
    description: "Vendas por categoria de produto"
    sql: |
      SELECT
        cat.nome as categoria,
        COUNT(v.id) as num_vendas,
        SUM(v.quantidade) as qtd_vendida,
        SUM(v.valor_total) as receita_total,
        AVG(v.valor_total) as ticket_medio
      FROM vendas v
      JOIN produtos p ON v.produto_id = p.id
      JOIN categorias cat ON p.categoria_id = cat.id
      WHERE v.data_venda BETWEEN @p1 AND @p2
      GROUP BY cat.nome
      ORDER BY receita_total DESC
    parameters:
      - name: "data_inicio"
        type: "date"
      - name: "data_fim"
        type: "date"
```

---

## 🔧 Parâmetros e Tipos

### Tipos Suportados

| Tipo YAML | Tipo SQL | Exemplo |
|-----------|----------|---------|
| `string` | VARCHAR/TEXT | `"João Silva"` |
| `integer` | INTEGER | `123` |
| `float` | DECIMAL/NUMERIC | `99.99` |
| `date` | DATE | `"2025-09-30"` |
| `datetime` | TIMESTAMP | `"2025-09-30 14:30:00"` |
| `boolean` | BOOLEAN | `true` / `false` |
| `array` | ARRAY (PostgreSQL) | `[1, 2, 3]` |

### Sintaxe de Parâmetros

```sql
-- Formato: @p1, @p2, @p3, etc.
SELECT * FROM tabela WHERE coluna = @p1

-- NÃO use:
-- ${param}   ❌
-- :param     ❌
-- {param}    ❌
```

### Valores Padrão

```yaml
parameters:
  - name: "limite"
    type: "integer"
    default: 10          # Valor padrão se não informado
    description: "Máximo de resultados"
```

---

## 🔀 Múltiplas Conexões

### Cenário: Multi-Database

**Sistema com vários bancos**

```yaml
local_tools_configuration:
  sql_dynamic:
    # Conexão 1: ERP (PostgreSQL)
    connections:
      erp:
        secret_key: "ERP_POSTGRES_URL"
        pool_size: 10

      # Conexão 2: CRM (MySQL)
      crm:
        secret_key: "CRM_MYSQL_URL"
        pool_size: 5

      # Conexão 3: Analytics (SQLite)
      analytics:
        secret_key: "ANALYTICS_SQLITE_URL"

    queries:
      # Query no ERP
      vendas_erp:
        connection: "erp"    # ← PostgreSQL
        sql: "SELECT * FROM vendas WHERE id = @p1"

      # Query no CRM
      cliente_crm:
        connection: "crm"    # ← MySQL
        sql: "SELECT * FROM clientes WHERE email = @p1"

      # Query no Analytics
      metricas:
        connection: "analytics"  # ← SQLite
        sql: "SELECT * FROM metricas WHERE data = @p1"
```

**Agente pode usar todos**:

```yaml
tools:
  - "dyn_sql<vendas_erp>"
  - "dyn_sql<cliente_crm>"
  - "dyn_sql<metricas>"
```

---

## Boas Práticas

### 1. Nomenclatura Clara

```yaml
#  BOM
queries:
  vendas_mes_loja:        # Claro e descritivo
  estoque_produto:
  clientes_inativos_90d:

#   EVITE
queries:
  q1:                     # Vago
  consulta:               # Genérico
  teste:                  # Temporário
```

---

### 2. Descrições Completas

```yaml
#  BOM
queries:
  vendas_mes:
    description: "Retorna total de vendas e quantidade de transações
                  para um mês/ano específico"
    sql: "..."
    parameters:
      - name: "mes"
        type: "integer"
        description: "Mês (1-12)"
      - name: "ano"
        type: "integer"
        description: "Ano (YYYY)"

#   EVITE
queries:
  vendas:
    sql: "..."  # Sem descrição
```

---

### 3. Limite de Resultados

```yaml
#  BOM - Sempre use LIMIT
sql: |
  SELECT * FROM clientes
  WHERE ativo = true
  ORDER BY data_cadastro DESC
  LIMIT @p1

#   EVITE - Sem limite (pode retornar milhões)
sql: "SELECT * FROM clientes"
```

---

### 4. Índices no Banco

```sql
-- Crie índices para colunas frequentemente usadas em WHERE
CREATE INDEX idx_vendas_data ON vendas(data_venda);
CREATE INDEX idx_vendas_cliente ON vendas(cliente_id);
CREATE INDEX idx_produtos_codigo ON produtos(codigo);
```

---

### 5. Pool de Conexões

```yaml
connections:
  meu_db:
    secret_key: "DB_URL"
    pool_size: 10         # Para alta concorrência
    timeout: 60           # Queries lentas
```

**Recomendações**:

- **Baixa carga**: pool_size = 5
- **Média carga**: pool_size = 10
- **Alta carga**: pool_size = 20

---

## 🔍 Troubleshooting

### Erro: "Query não encontrada"

**Causa**: Nome da query errado

```yaml
# tools declara:
tools:
  - "dyn_sql<vendas_mes>"  # ← Nome esperado

# Mas queries tem:
queries:
  vendas_mensal:           #   Nome diferente!
```

**Solução**: Sincronizar nomes

```yaml
tools:
  - "dyn_sql<vendas_mes>"

queries:
  vendas_mes:             #  Nome igual
```

---

### Erro: "Conexão não encontrada"

**Causa**: secret_key não existe em .keys.yaml

```yaml
# config.yaml
connections:
  meu_db:
    secret_key: "DB_URL"   # ← Busca essa chave

# .keys.yaml (faltando!)
security_keys:
  OUTRA_CHAVE: "..."      #   Não tem DB_URL
```

**Solução**: Adicionar em .keys.yaml

```yaml
security_keys:
  DB_URL: "postgresql://user:pass@host/db"
```

---

### Erro: "SQL Syntax Error"

**Causa**: SQL inválido ou parâmetro errado

```sql
--   ERRADO
SELECT * FROM vendas WHERE data = @p1 AND @p2

--  CORRETO
SELECT * FROM vendas WHERE data BETWEEN @p1 AND @p2
```

**Dica**: Teste a query direto no banco antes de usar

---

### Erro: "Stored procedure não executa"

**Causa**: Sintaxe padrão `CALL` não é suportada pelo SGBD alvo.

**Soluções**:

- Definir `call_template` com a sintaxe específica do banco no YAML.
- Validar a chamada da procedure diretamente no banco antes de usar na tool.

---

### Erro: "Timeout"

**Causa**: Query muito lenta

**Soluções**:

1. Aumentar timeout

```yaml
connections:
  meu_db:
    timeout: 120  # 2 minutos
```

1. Otimizar query

```sql
-- Adicionar índices
-- Limitar resultados
-- Evitar SELECT *
```

---

## 🔄 Migração de Versões

### v1.0 → v2.0

**Mudanças**:

- Adicionado: Parametrização inline (@p1)
- Adicionado: Múltiplas conexões
- Adicionado: Lazy loading
- ⚠️ Mudança: Formato do YAML

**Antes (v1.0)**:

```yaml
local_tools_configuration:
  sql:
    connection_string: "postgresql://..."
    queries:
      - name: "vendas"
        sql: "SELECT * FROM vendas"
```

**Depois (v2.0)**:

```yaml
local_tools_configuration:
  sql_dynamic:
    connections:
      main:
        secret_key: "DB_URL"
    queries:
      vendas:
        connection: "main"
        sql: "SELECT * FROM vendas WHERE id = @p1"
```

---

## 📞 Suporte e Recursos

### Documentação Relacionada

- [Guia Central de Tools](../GUIA-USUARIO-TOOLS.md)
- [Lista de Ferramentas](alfabetica.md)
- [Por Finalidade](por_finalidade.md)

### Bancos observados no contrato atual

- PostgreSQL
- MySQL / MariaDB
- SQLite
- Microsoft SQL Server
- ⚠️ Outros dialetos SQLAlchemy podem funcionar, mas isso nao foi validado de forma dedicada no codigo atual.

### Evidência no código

- `src/agentic_layer/supervisor/tool_loader.py`
- `src/agentic_layer/supervisor/tools_factory.py`
- `src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py`

### Lacunas no código

Não encontrado no código:

- matriz formal de compatibilidade por banco para cada dialeto SQLAlchemy;
- documento único de exemplos oficiais versionados só para SQL dinâmica.

Onde deveria estar:

- `src/agentic_layer/tools/domain_tools/dynamic_sql_tools/`
- `docs/tools/`

---

<div align="center">

**[⬆️ Voltar ao Topo](#-sql-dinâmico---documentação-completa)** | **[🏠 Página Principal](../README.md)**

</div>
