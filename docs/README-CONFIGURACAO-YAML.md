# YAML como Contrato Operacional

Este manual explica o YAML como contrato vivo da plataforma. O foco não é listar chaves em ordem alfabética. O foco é mostrar como a configuração entra no sistema, quais transformações ela sofre, onde a plataforma falha fechado e por que isso é essencial para operação confiável.

## 1. Visão geral

A plataforma é YAML-first, mas isso não significa que o YAML cru vai direto para execução. O documento passa por carga, resolução de origem, injeção de contexto, expansão de placeholders, normalização estrutural e, no escopo agentic, validação governada e possível compilação AST.

Em termos práticos, o YAML é a intenção operacional do produto. O runtime existe para provar se essa intenção é executável com segurança.

## 2. O problema que este módulo resolve

Sem uma resolução central, cada endpoint teria sua própria leitura de YAML, seus próprios defaults e suas próprias exceções. Isso criaria divergência entre o que o documento parece dizer e o que o sistema realmente executa.

O resolvedor central existe para evitar exatamente isso. Ele garante que as mesmas regras valham para YAML vindo por encrypted_data, yaml_inline_content ou yaml_config_path. Também garante que contexto do cliente, security_keys, tools_library e user_session entrem sempre pelo fluxo canônico.

## 3. Conceitos necessários para entender o módulo

Resolução significa transformar uma entrada bruta em configuração executável.

Normalização significa ajustar a estrutura para o contrato canônico, rejeitando layouts antigos em vez de mascará-los.

Placeholder é um valor simbólico que será substituído por segredo ou configuração concreta depois do enriquecimento do contexto.

No escopo agentic, a AST é a representação tipada do YAML governado. Ela existe para que edição, validação e compilação falem a mesma língua. Quando a feature flag FEATURE_AGENTIC_AST_ENABLED está ativa, esse fluxo também fica exposto por endpoints como /config/assembly/validate.

## 4. Como o YAML funciona por dentro

O fluxo começa na borda HTTP. O sistema escolhe a origem do documento, carrega o conteúdo e injeta user_session com correlation_id. Depois disso, enriquece client_context, resolve security_keys, garante store para chaves, tenta expandir placeholders e só então normaliza a estrutura.

Nesse processo, tools_library ocupa um papel crítico. O cliente não envia o catálogo pronto. O contrato observado no código é fechado: tools_library precisa existir na raiz e chegar vazia para receber a injeção do catálogo builtin persistido. tools_library ausente ou preenchida vira erro explícito.

No escopo agentic, o YAML pode seguir ainda para draft, validate e confirm. Quando isso acontece, o sistema registra metadata.agentic_assembly.governed_hashes para detectar drift entre o que foi validado e o que ficou persistido depois.

## 5. Pipeline principal de resolução

1. Escolher a origem do YAML.
2. Carregar o documento em memória.
3. Injetar user_session e correlation_id.
4. Enriquecer client_context e security_keys.
5. Expandir placeholders com base no contexto real.
6. Garantir o contrato de tools_library.
7. Normalizar a estrutura canônica.
8. Se houver escopo agentic, validar pelo fluxo governado, inclusive /config/assembly/validate quando apropriado.

Esse pipeline existe para impedir que uma configuração aparentemente aceitável só quebre muito depois, dentro de um domínio já difícil de investigar.

## 6. Decisões técnicas importantes

A primeira decisão importante é falhar fechado quando o contrato crítico está errado. user_session fora do lugar, placeholder essencial não resolvido, tools_library preenchida manualmente ou chaves legadas incompatíveis devem interromper o fluxo.

A segunda decisão é separar resolução geral de governança agentic. Nem todo YAML precisa da mesma profundidade de validação, mas quando o documento entra em workflows, multi_agents ou tools_library governado, a AST deixa de ser opcional.

A terceira decisão é tratar vector_store.if_exists como política do dataset vivo, não como detalhe local do provider vetorial. Isso evita desalinhamento entre BM25, banco relacional e store vetorial.

## 7. O que acontece em caso de sucesso

No caminho feliz, o YAML entra por uma das origens aceitas, recebe user_session, contextos do cliente, chaves de segurança, catálogo de tools e normalização estrutural. Se houver parte agentic governada, ela ainda recebe validação ou compilação consistente. O resultado é um documento pronto para o runtime, não apenas um texto bem formatado.

## 8. O que acontece em caso de erro

Os erros mais importantes aparecem em quatro famílias.

1. Erro de origem: payload ausente, inválido ou fonte não suportada.
2. Erro de contrato: user_session mal posicionado, tools_library inválida ou estrutura legada.
3. Erro de segredo: placeholder sem resolução ou security_keys insuficientes.
4. Erro de governança: documento agentic que não passa pelo modelo AST quando deveria.

O comportamento desejado não é sucesso parcial. É erro claro, com contexto suficiente para o operador saber se o problema está no YAML, no ambiente ou no fluxo governado.

## 9. Configurações que realmente mudam comportamento

FEATURE_AGENTIC_AST_ENABLED decide se o slice governado do assembly AST fica exposto e utilizável.

tools_library decide se o catálogo builtin poderá ser injetado corretamente.

vector_store.if_exists decide a política do dataset vivo em ingestão.

user_session.correlation_id decide a identidade lógica da execução.

Essas chaves são mais do que dados. Elas mudam o comportamento estrutural do sistema.

## 10. Observabilidade e diagnóstico

A sequência mais eficiente de diagnóstico é esta.

1. Descobrir de onde o YAML veio.
2. Verificar se user_session e correlation_id foram injetados.
3. Verificar se client_context e security_keys foram resolvidos.
4. Verificar se placeholders críticos sobraram.
5. Verificar se tools_library chegou vazia e foi enriquecida.
6. Se houver escopo agentic, verificar a trilha de validação e metadata.agentic_assembly.governed_hashes.

Em termos simples: primeiro confirme se o documento foi realmente preparado. Só depois discuta o que o domínio fez com ele.

## 11. Exemplo prático guiado

Imagine um operador enviando um YAML agentic por linguagem natural. A API resolve a origem do pedido, prepara o YAML base e, com FEATURE_AGENTIC_AST_ENABLED ativa, o fluxo governado pode usar endpoints como /config/assembly/objective-to-yaml e /config/assembly/validate. O documento resultante recebe hash governado em metadata.agentic_assembly.governed_hashes. A partir daí, o sistema consegue dizer se o YAML usado em runtime ainda corresponde ao que foi validado.

## 12. Explicação 101

Pense no YAML como a ordem de produção de uma fábrica. Se a ordem vem incompleta, com campo errado ou segredo faltando, a fábrica não deveria começar a montar metade do produto e torcer pelo melhor. O resolvedor existe para impedir esse tipo de improviso.

## 13. Evidências no código

- [src/api/routers/config_resolution.py](../src/api/routers/config_resolution.py): resolução central das origens de YAML.
- [src/config/config_cli/configuration_factory.py](../src/config/config_cli/configuration_factory.py): carga, enriquecimento, injeção de tools_library e normalização final.
- [src/utils/yaml_schema_normalizer.py](../src/utils/yaml_schema_normalizer.py): guarda estrutural do contrato canônico.
- [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py): parse, validação e compilação do escopo agentic.
- [src/config/agentic_assembly/drift_detector.py](../src/config/agentic_assembly/drift_detector.py): lógica de detecção de deriva governada.
- [src/ingestion_layer/vector_stores/base.py](../src/ingestion_layer/vector_stores/base.py): política de vector_store.if_exists no runtime de ingestão.
