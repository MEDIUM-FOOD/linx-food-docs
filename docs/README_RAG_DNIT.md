# Manual técnico do pipeline RAG DNIT

## 1. O que esta feature faz

O caso DNIT é a especialização do pipeline RAG para documentos técnicos rodoviários com alto peso de código normativo, siglas, tabelas e terminologia de engenharia.

No código atual, essa especialização não é um produto isolado. Ela nasce da combinação de quatro blocos de runtime:

1. detecção de domínio na ingestão;
2. vocabulário lexical adaptado para o schema DNIT;
3. expansão de query antes do retrieval;
4. retrieval híbrido com fallback operacional.

O valor prático dessa feature é reduzir um erro clássico de RAG genérico: entender o tema da pergunta, mas falhar no documento técnico exato, na norma específica ou na tabela correta.

## 2. Que problema ela resolve

Documentos DNIT têm um tipo de dificuldade que busca vetorial pura trata mal:

1. códigos exatos como normas e métodos;
2. termos técnicos raros e siglas do domínio;
3. necessidade de priorizar tabelas e metadados específicos;
4. diferença entre um documento normativo principal e um documento apenas relacionado ao tema.

A especialização DNIT existe para aumentar precisão semântica e precisão lexical ao mesmo tempo.

## 3. Visão executiva

Para a liderança, o ganho é previsibilidade de resposta em um domínio onde erro documental custa caro. A feature reduz risco de devolver norma errada, melhora rastreabilidade do motivo de uma recuperação e transforma o RAG em algo mais auditável para operação técnica.

## 4. Visão comercial

Comercialmente, o caso DNIT mostra que a plataforma não depende só de similaridade semântica genérica. Ela consegue incorporar vocabulário de domínio, priorizar códigos técnicos e usar sinais operacionais para melhorar recuperação. A promessa correta é robustez maior em consulta técnica especializada. A promessa incorreta seria afirmar acerto automático em qualquer pergunta sem depender de boa ingestão e boa configuração de vocabulário.

## 5. Visão estratégica

Estratégicamente, o caso DNIT prova um desenho reutilizável: o runtime de expansão por domínio é registrado em um registry, recebe vocabulário adaptado e injeta enriquecimento antes do retrieval. Isso prepara a plataforma para outros domínios especializados sem hardcode no pipeline principal.

## 6. Conceitos que importam

### 6.1 Processador de domínio

Na ingestão, o `DNITDomainProcessor` decide se um documento deve ser tratado como DNIT. Ele olha padrões normativos, título, caminho de origem, tipo inferido e contexto primário do documento.

### 6.2 Vocabulário de expansão

A expansão DNIT trabalha com vocabulário técnico, frases exatas, normas, tabelas e aliases. Esse vocabulário pode vir do YAML, de auto_config ou de snapshot BM25 adaptado ao schema esperado pela etapa de expansão.

### 6.3 Hints de recuperação

A etapa DNIT não adiciona só termos. Ela também publica hints como priorização de metadados de norma, priorização de tabelas e indicadores de saúde do auto_config, por exemplo truncamento e scan lento.

### 6.4 Query híbrida

O retrieval híbrido do projeto parte da pergunta original e acrescenta apenas termos técnicos selecionados que ainda não estavam na consulta. A ideia é aumentar recall lexical sem duplicar ruído desnecessário.

## 7. Como o pipeline funciona por dentro

## 7.1 Ingestão e aceitação do documento

O `DNITDomainProcessor` possui padrões explícitos para identificar documentos DNIT, como códigos de norma, nome institucional e tipos documentais relevantes.

O ponto mais importante no código é este: o processador não aceita qualquer documento só porque o corpo menciona uma norma. Ele privilegia evidência primária no título e no caminho, e ainda distingue documentos normativos de documentos relevantes não normativos.

Isso reduz falso positivo na indexação de domínio.

## 7.2 Configuração do comportamento DNIT

No bootstrap do processador, o código lê configurações reais como:

1. `extract_tables`;
2. `remove_headers_footers`;
3. `validate_norma_codes`;
4. `min_confidence_threshold`;
5. `min_indicator_hits`.

Também existem blocos avançados para busca agressiva de norma, detecção de normas relacionadas, referências de página e análise estrutural.

## 7.3 Registro runtime do domínio

O domínio DNIT é registrado em `DomainQueryExpansionRuntimeRegistry`. Esse registry informa duas coisas para o restante do pipeline:

1. qual adapter de vocabulário BM25 deve ser usado;
2. qual step de expansão deve ser instanciado.

Isso evita espalhar condição especial de DNIT pelo retrieval engine inteiro.

## 7.4 Merge do vocabulário em runtime

`DomainQueryExpansionConfigService` aplica o snapshot BM25 sobre a configuração do domínio. O serviço garante estrutura mínima, mescla vocabulário e stats, persiste backend e timestamp do snapshot e ativa a etapa de expansão quando o domínio está elegível.

O significado prático é importante: o vocabulário usado na query não depende só do YAML manual. Ele pode ser enriquecido dinamicamente pelo snapshot lexical produzido pelo ecossistema BM25.

## 7.5 Adaptação do snapshot BM25 para o schema DNIT

`BM25DnitVocabularyAdapter` converte blocos brutos de vocabulário BM25 para o formato que `DnitQueryExpansionStep` entende.

No código atual, essa adaptação:

1. transforma termos relacionados em `technical_terms`;
2. transforma sinônimos e variantes em `aliases`;
3. mescla listas e mapas sem duplicidade;
4. preserva seções genéricas adicionais do snapshot.

Isso é o elo entre o mundo lexical bruto do BM25 e o contrato semântico que a expansão DNIT consome.

## 7.6 Expansão da query

`DnitQueryExpansionStep` é a etapa que analisa a pergunta e decide se deve enriquecer `QueryFeatures`.

Quando habilitada, ela:

1. procura termos técnicos, frases exatas, códigos de norma, labels de tabela e aliases;
2. calcula confiança da expansão;
3. seleciona termos sem duplicidade e dentro do limite configurado;
4. injeta esses termos em `technical_terms`, `keywords` e `context_hints`.

Os hints confirmados no código incluem exemplos como:

1. `prioritize_norma_metadata`;
2. `prioritize_domain_tables`;
3. `boost_domain_terms`;
4. `auto_config_truncated`;
5. `auto_config_slow_scan`;
6. `auto_config_limit_hit`.

Esses sinais ajudam o retrieval a não tratar a pergunta técnica como texto genérico.

## 7.7 Montagem da query híbrida

No `RetrievalEngine`, `_build_hybrid_query` pega a pergunta original e acrescenta os `technical_terms` que ainda não existem no texto.

Esse detalhe é importante porque evita duplicação inútil. A query só cresce quando a expansão realmente adiciona contexto novo.

## 7.8 Execução do retrieval híbrido

`execute_hybrid_processor` decide se usa hybrid nativo do vector store ou fallback manual.

O fluxo confirmado é este:

1. checar se hybrid está desligado, não nativo ou indisponível;
2. tentar hybrid nativo quando suportado;
3. cair para o retriever híbrido manual quando necessário;
4. em último caso, voltar para o processador tradicional.

Isso significa que o caso DNIT não depende de um único backend específico para existir. Ele tenta usar o melhor modo disponível e preserva fallback operacional.

## 8. O que acontece em caso de sucesso

Quando o pipeline está saudável:

1. documentos DNIT relevantes entram no domínio correto durante a ingestão;
2. o vocabulário de expansão é carregado ou montado em runtime;
3. a pergunta recebe termos técnicos e hints úteis;
4. a query híbrida chega mais rica ao retrieval;
5. o retrieval combina evidência lexical e evidência semântica com fallback seguro.

## 9. O que acontece em caso de erro

Os cenários confirmados no código mais relevantes são:

### 9.1 Documento rejeitado como DNIT

Se não houver evidência primária suficiente no título ou no caminho, o processador pode rejeitar o documento. Isso evita indexação oportunista de material apenas tangencial.

### 9.2 Expansão desativada por ausência de vocabulário

Se a etapa DNIT estiver habilitada mas sem vocabulário utilizável, ela se desliga e registra warning. Isso impede expansão vazia fingindo estar ativa.

### 9.3 Fallback da recuperação híbrida

Se o backend não suportar hybrid nativo ou se o retriever híbrido falhar, o engine cai para caminhos tradicionais. Na prática, o sistema continua respondendo, mas perde parte do ganho especializado.

### 9.4 Auto_config degradado

A própria expansão publica hints de truncamento, scan lento e limite atingido. Isso não bloqueia o pipeline, mas informa que a base lexical automática pode estar incompleta ou degradada.

## 10. Observabilidade e diagnóstico

Para investigar problema real no caso DNIT, siga esta ordem:

1. o documento foi aceito pelo `DNITDomainProcessor` ou ficou fora do domínio?
2. a configuração de domínio realmente ativou `query_expansion`?
3. o vocabulário veio do YAML, do auto_config ou do snapshot BM25?
4. os `technical_terms` foram adicionados em `QueryFeatures`?
5. a query híbrida foi enriquecida antes do retrieval?
6. o engine usou hybrid nativo, híbrido manual ou fallback tradicional?

Essa sequência ajuda a separar erro de ingestão, erro de vocabulário, erro de montagem da query e erro de backend de retrieval.

## 11. Impacto técnico

Tecnicamente, o caso DNIT reforça três padrões importantes do projeto:

1. especialização por domínio sem acoplamento duro no retrieval central;
2. merge de configuração em runtime com snapshot lexical;
3. fallback explícito em vez de dependência silenciosa de um backend ideal.

## 12. Impacto estratégico

Esse desenho abre caminho para outros domínios especializados com a mesma forma de integração: registry de domínio, adapter de vocabulário e etapa de expansão dedicada. O DNIT é um caso real de como a plataforma pode crescer por domínio sem virar uma coleção de ifs no core.

## 13. Explicação 101

Pense no caso DNIT como um tradutor técnico acoplado ao RAG.

Primeiro, ele decide se um documento realmente pertence ao mundo DNIT. Depois, ele aprende palavras e relações desse mundo. Quando chega uma pergunta, ele acrescenta contexto técnico útil antes da busca. Por fim, a recuperação combina semelhança semântica com sinais lexicais mais exatos.

Sem isso, a plataforma entende mais ou menos o assunto. Com isso, ela ganha mais chance de achar a norma, a tabela ou o termo certo.

## 14. Limites e pegadinhas

Os limites reais confirmados no código são:

1. a expansão DNIT depende de vocabulário válido;
2. o ganho do caso DNIT não corrige ingestão ruim ou documento fora do domínio;
3. fallback operacional mantém resposta disponível, mas pode reduzir qualidade da recuperação especializada;
4. o registro runtime atual visto no código está focado no domínio `dnit`, não em uma lista ampla de domínios equivalentes.

## 15. Evidências no código

1. `src/ingestion_layer/processors/domain_plugins/dnit_domain_processor.py`: detecção de domínio, thresholds e configuração de extração.
2. `src/qa_layer/rag_engine/domain_query_expansion_registry.py`: registro runtime do domínio DNIT.
3. `src/qa_layer/rag_engine/domain_query_expansion_config_service.py`: merge do snapshot BM25 e criação da etapa de expansão.
4. `src/qa_layer/rag_engine/bm25_dnit_vocabulary_adapter.py`: adaptação do vocabulário BM25 para o schema DNIT.
5. `src/qa_layer/rag_engine/dnit_query_expansion.py`: enriquecimento de `QueryFeatures` com termos e hints de domínio.
6. `src/qa_layer/rag_engine/retrieval_engine.py`: montagem da query híbrida e execução do hybrid com fallback.
7. `tests/unit/qa_layer/rag_engine/test_dnit_query_expansion.py`: evidência automatizada de auto_vocabulary, hints e enriquecimento lexical.
