**Produto:** Plataforma de Agentes de IA

# 📘 Documentação Completa:  Pipeline RAG Avançado para Documentos Técnicos DNIT

## 🎯 Sumário Executivo

Este documento apresenta uma análise técnica completa do sistema RAG (Retrieval-Augmented Generation) **Plataforma de Agentes de IA**, especializado no processamento de documentos técnicos do DNIT (Departamento Nacional de Infraestrutura de Transportes). O sistema implementa técnicas avançadas de ingestão, processamento multimodal, recuperação híbrida e geração aumentada, superando significativamente as limitações de sistemas RAG convencionais.

## Leitura complementar

Para o compêndio completo de ingestão e RAG, consulte
[docs/PIPELINE-INGESTAO-RAG.md](docs/PIPELINE-INGESTAO-RAG.md).

---

## 📋 Índice

1. [Visão Geral do Problema](#1-visão-geral-do-problema)
2. [Por Que RAG Simples Falha em Documentos DNIT](#2-por-que-rag-simples-falha-em-documentos-dnit)
3. [Arquitetura do Pipeline de Ingestão](#3-arquitetura-do-pipeline-de-ingestão)
4. [Processamento Multimodal Avançado](#4-processamento-multimodal-avançado)
    4.4 [Configuração YAML (visão, OCR e embedding visual)](#44-configuração-yaml-visão-ocr-e-embedding-visual)
5. [Pipeline RAG Inteligente](#5-pipeline-rag-inteligente)
6. [Técnicas Avançadas Implementadas](#6-técnicas-avançadas-implementadas)
7. [Casos de Uso e Exemplos Práticos](#7-casos-de-uso-e-exemplos-práticos)
8. [Explicação para Gestores](#8-explicação-para-gestores-versão-executiva)

---

## 1. Visão Geral do Problema

### 1.1 Contexto:  Documentos Técnicos de Engenharia

Documentos técnicos do DNIT apresentam desafios únicos que sistemas RAG convencionais não conseguem resolver adequadamente:

- **📐 Normas Técnicas**:  DNIT-031/2006-ES, NBR 7180, ABNT NBR 16697
- **🔢 Códigos Específicos**:  Especificações de materiais, procedimentos de ensaio
- **📊 Diagramas e Tabelas**: Informações críticas em formato visual
- **📄 PDFs Escaneados**: Documentos antigos sem camada de texto
- **🏗️ Terminologia Técnica**: Vocabulário especializado de engenharia civil

### 1.2 O Desafio da Recuperação Precisa

Em documentos DNIT, a precisão na recuperação é crítica:

-   **Falso Negativo**: Não encontrar "DNIT-031/2006-ES" quando usuário busca por essa norma específica
-   **Falso Positivo**:  Retornar normas irrelevantes quando usuário pergunta sobre "pavimentação asfáltica"
-   **Perda de Contexto**: Ignorar diagramas e tabelas que contêm informações essenciais

---

## 2. Por Que RAG Simples Falha em Documentos DNIT

### 2.1 Limitações de RAG Convencional

Um sistema RAG básico (apenas busca vetorial + LLM) apresenta falhas críticas:

####   **Problema 1: Busca Vetorial Pura Falha em Códigos Exatos**

**Exemplo Real:**

```yaml
Pergunta: "Qual a norma DNIT-031/2006-ES especifica sobre granulometria?"

RAG Simples:
- Embeddings vetoriais convertem "DNIT-031/2006-ES" em vetor semântico
- Sistema pode retornar documentos "semanticamente similares" mas que falam de outras normas
- MISS: O documento exato com DNIT-031/2006-ES pode não ser o top-1

Resultado:   FALHA - Resposta genérica ou norma errada
```

**Por Que Falha:**
- Embeddings vetoriais capturam **significado semântico**, não **match exato**
- Códigos como "DNIT-031/2006-ES" são tratados como tokens, não como identificadores únicos
- Distância vetorial pode priorizar documentos "sobre normas" em vez do documento da norma específica

####   **Problema 2: Perda de Informação Visual**

**Exemplo Real:**

```yaml
Pergunta: "Qual o esquema de instalação da defensa metálica tipo DMS?"

RAG Simples:
- Extrai apenas TEXTO do PDF
- Diagrama técnico com dimensões e ângulos → IGNORADO
- Tabela com especificações de materiais → IGNORADA

Resultado:   FALHA - LLM responde "não tenho informações suficientes"
```

**Impacto:**
- 60-80% da informação crítica em manuais técnicos está em **diagramas, tabelas e fotos**
- RAG sem processamento multimodal = Sistema cego para conteúdo visual

####   **Problema 3: Documentos Escaneados (OCR Necessário)**

**Exemplo Real:**

```yaml
Documento: Manual antigo do DNIT escaneado (imagem de página)

RAG Simples:
- Tenta extrair texto → String vazia ou garbage
- Chunk vazio → Não é indexado
- Documento inteiro → PERDIDO

Resultado:   FALHA - Sistema ignora documentos históricos valiosos
```

####   **Problema 4: Terminologia Técnica Não Compreendida**

**Exemplo Real:**

```yaml
Pergunta: "Qual o CBR mínimo para subleito de pavimento flexível?"

RAG Simples:
- "CBR" (California Bearing Ratio) → Embedding genérico
- Pode retornar documentos sobre "capacidade de suporte" mas não especificamente CBR
- Valores numéricos (CBR ≥ 2%) → Perdidos na vetorização

Resultado:   PARCIAL - Resposta imprecisa, falta valores específicos
```

### 2.2 Casos de Uso Reais que Falham

| Caso de Uso | RAG Simples | Resultado |
|-------------|-------------|-----------|
| 🔍 Buscar norma por código exato | Busca vetorial apenas |   Pode não retornar o documento certo |
| 📐 Interpretar diagrama técnico | Ignora imagens |   Informação visual perdida |
| 📄 Processar PDF escaneado | Sem OCR |   Documento inteiro inacessível |
| 🔢 Extrair valor numérico de especificação | Texto vetorizado |   Números perdidos na semântica |
| 📊 Consultar dados em tabela | Tabela → texto corrido |   Estrutura e relações perdidas |

---

## 2.3 Plano de avaliação (cientistas de dados)

- Dataset de teste: queries rotuladas por perfil (normas por codigo,
    tabelas/diagramas, PDFs escaneados, narrativas tecnicas), com
    documentos relevantes marcados.
- Metricas-chave: recall@10 para codigos/siglas, precisao@5 em
    narrativas, acuracia de campos numericos/tabelas, cobertura de
    paginas com OCR (pags OCR processadas / pags alvo) e % de consultas
    que acionam multimodal.
- Coleta: usar logs/telemetria para extrair scores, fallback acionado e
    latencia; amostrar chunks anotados com metadados de OCR e dominio.
- Consultas exemplo: "DNIT-031/2006-ES granulometria", "esquema
    defensa metalica DMS (diagrama)", "CBR minimo subleito", "tabela
    de dosagem concreto".
- Iteracao: ajustar OCR/multimodal na ingestao, pesos BM25/vetor/FTS no
    RAG e repetir o mesmo conjunto rotulado para medir ganho.

---

## Anexo: Dicionário das chaves de configuração DNIT (jsonb no Postgres)

As chaves abaixo correspondem ao objeto JSON armazenado em
`manual_config` da tabela `ingestion_domain_processors` quando o
backend de domínio está configurado para Postgres. Apenas estas chaves
são lidas pelas classes atuais do processador DNIT e da etapa de
expansão de queries.

- `enabled`: habilita ou desabilita o processador DNIT.
- `priority`: ordena a execução (menor valor = maior prioridade).
- `min_indicator_hits`: mínimo de indicadores DNIT para aceitar o
    documento.
- `min_confidence_threshold`: razão mínima de confiança na detecção do
    domínio.
- `validate_norma_codes`: valida formato de código DNIT antes de usar.
- `extract_tables`: habilita extração de tabelas durante a ingestão.
- `remove_headers_footers`: remove headers e footers ao extrair
    conteúdo.

### `advanced_extraction`

- `aggressive_norma_search`: busca mais ampla por códigos DNIT.
- `detect_related_normas`: lista códigos DNIT adicionais como
    relacionados.
- `extract_page_references`: tenta preencher metadado de página nos
    chunks.
- `analyze_document_structure`: coleta pistas de estrutura (tópicos,
    contagem de tabelas/figuras).

### `quality_control`

- `validate_against_schema`: quando true, valida metadados contra o
    schema DNIT; se falhar, aplica fallback básico e descarta os campos
    inválidos. Quando false, mantém os metadados extraídos mesmo que
    incompletos ou fora do padrão.
- `fallback_on_validation_error`: em validação inválida, aplica
    metadados de fallback.
- `detailed_extraction_logging`: habilita logging detalhado da
    extração.

### `query_expansion`

- `enabled`: ativa a expansão de termos DNIT na análise de consultas.
- `max_terms`: número máximo de termos adicionados à consulta.
- `min_confidence`: confiança mínima para aplicar expansão.
- `fallback_strategy`: comportamento quando a confiança não atinge o
    mínimo (ex.: `keywords_only`).
- `technical_terms_key`, `exact_phrases_key`, `normas_key`,
    `tables_key`, `aliases_key`: caminhos dentro de `vocabulary` que
    definem onde cada lista está armazenada.

#### `query_expansion.vocabulary`

- `normas`: lista de normas com `codigo` e `aliases` reconhecidos.
- `tables`: tabelas importantes, com `label` e `related_terms` para
    detecção.
- `aliases`: mapa de apelidos com `variants` apontando para um termo
    `preferred`.
- `exact_phrases`: frases exatas que, se presentes na consulta, são
    adicionadas como sinal forte.
- `technical_terms`: termos técnicos adicionados à consulta para reforço
    do contexto DNIT.

### Geração automática de vocabulário DNIT

- A ingestão calcula estatísticas por documento (domínios, keywords e
    expansões observadas em chunks) e grava na telemetria.
- Essas estatísticas são consolidadas no `auto_config` do domínio DNIT,
    populando o vocabulário de query expansion sem scripts externos.
- O `manual_config` continua valendo como fonte principal; o
    `auto_config` apenas adiciona termos descobertos para cobrir mais
    variações de consulta.

## 3. Arquitetura do Pipeline de Ingestão

### 3.1 Visão Geral do Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIPELINE DE INGESTÃO                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📥 INPUT SOURCES                                               │
│  ├─ 📄 PDFs (digitais + escaneados)                            │
│  ├─ 📁 Azure Blob Storage                                       │
│  ├─ ☁️  Google Drive                                            │
│  ├─ 🌐 Web Scraping                                             │
│  ├─ 📝 Office (DOCX, PPTX, XLSX)                               │
│  └─ 🎥 YouTube (transcrições)                                   │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  🔍 DETECÇÃO & ROTEAMENTO                                       │
│  ├─ ContentClientFactory (auto-detect formato)                 │
│  ├─ Repository Pattern (local/Azure/GDrive)                    │
│  └─ ContentProcessorFactory (seleciona processador)            │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  🎯 PROCESSAMENTO ESPECIALIZADO                                 │
│  ├─ PDFProcessor (PyMuPDF + OCR + Multimodal)                  │
│  ├─ DomainProcessingResolver (DNIT-specific)                   │
│  ├─ MultimodalContentProcessor                                 │
│  │   ├─ PDFImageExtractor                                       │
│  │   ├─ TesseractOCR / EasyOCR                                 │
│  │   └─ ImageDescriptor (OpenAI Vision / Local)               │
│  └─ ChunkingStrategies (Page/Semantic/Recursive)              │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  📦 ENRIQUECIMENTO & METADADOS                                  │
│  ├─ Extração de códigos DNIT (regex avançado)                  │
│  ├─ Classificação de páginas (keywords técnicas)               │
│  ├─ Contexto de imagens (texto antes/depois)                   │
│  ├─ Deduplicação inteligente (hash + similaridade)             │
│  └─ Metadados estruturados (source_type, domain, page_num)     │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  💾 INDEXAÇÃO VETORIAL                                          │
│  ├─ VectorStoreFactory (InMemory/Qdrant/Azure Search)          │
│  ├─ Embeddings (OpenAI text-embedding-3-large)                 │
│  ├─ BM25 Index (sparse retrieval paralelo)                     │
│  └─ Metadata Filters (domínio, tipo, data)                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Componentes Críticos do Pipeline

#### 3.2.1 **PDFContentProcessor** - Processador Inteligente de PDFs

**Localização**: `src/ingestion_layer/processors/pdf_processor.py`

**Responsabilidades:**

Antes do processamento, o dispatcher seleciona o client `PdfMultimodalAdapterClient`
para arquivos `.pdf`. Esse adaptador delega o documento ao `PDFContentProcessor`,
que concentra parsing, OCR, tabelas, sinais de página, metadados e processamento
de domínio.

1. **Detecção Automática de Qualidade**
   ```python
   # Detecta se PDF é escaneado ou digital
   if text_density < threshold:
       trigger_ocr = True
   ```

2. **OCR Adaptativo**
   ```python
   # Escolhe melhor engine OCR disponível
   if tesseract_available:
       text = pytesseract.image_to_string(page_image)
   elif easyocr_available:
       text = easyocr_reader.readtext(page_image)
   ```

3. **Extração de Metadados**
   - Número de página
   - Densidade de texto (caracteres/área)
   - Presença de imagens
   - Códigos DNIT detectados
    - Limpeza de cabeçalhos/rodapés repetidos no builder de páginas

**Configuração YAML:**

```yaml
processing:
  pdf:
    # OCR Settings
    allow_scanned_documents: true
    ocr_warning_threshold: 0.05
    force_ocr_if_text_density_below: 0.05

    # Domain Processing (DNIT-specific)
    domain_processing:
      enabled:  true
      domain: "dnit_engineering"
      code_patterns:
        - "\\bDNIT-\\d{3}/\\d{4}-[A-Z]{2}\\b"
        - "\\bNBR\\s*\\d{3,5}\\b"
        - "\\bLei\\s*n[ºo]? \\s*\\d+/? \\d*\\b"
```

#### 3.2.2 **DomainProcessingResolver** - Processamento Específico DNIT

**Localização**:  `src/ingestion_layer/domain_processing/`

**Funcionalidades:**

1. **Detecção de Códigos Normativos**
   ```python
   code_patterns = [
       r"\bDNIT-\d{3}/\d{4}-[A-Z]{2}\b",  # DNIT-031/2006-ES
       r"\bNBR\s*\d{3,5}\b",               # NBR 7180
       r"\bABNT\s*NBR\s*\d+\b"            # ABNT NBR 16697
   ]
   ```

2. **Classificação de Páginas Relevantes**
   ```python
   focus_keywords = [
       "pavimentação", "granulometria", "CBR",
       "defensa", "sinalização", "drenagem"
   ]

   if keyword_matches >= min_threshold:
       page. priority = "high"
   ```

3. **Chunking Específico de Domínio**
   - Respeita limites de seções técnicas
   - Preserva tabelas intactas
   - Mantém referências cruzadas

---

## 4. Processamento Multimodal Avançado

### 4.1 Arquitetura Multimodal

#### Visão geral
O processamento multimodal existe para resolver um problema típico de acervos
técnicos (como DNIT): parte do conhecimento está em **imagens** (diagramas,
plantas, figuras, fotos de obra, tabelas renderizadas), não apenas no texto.
A Plataforma de Agentes de IA transforma esse conteúdo visual em sinais que o RAG consegue usar:
texto extraído por OCR, descrição gerada por modelo de visão e embedding visual
persistido no vector store.

#### Explicação for dummies
Imagine um PDF com um diagrama importante. Se você só “cortar o texto” do PDF,
o sistema fica cego para o diagrama. O multimodal faz três coisas bem simples:
1) lê o texto que estiver escrito dentro da imagem (OCR),
2) pede para um modelo explicar o que está na figura (descrição),
3) cria um número grande (embedding) que funciona como “impressão digital” da imagem.
Com isso, a busca consegue achar o diagrama certo quando o usuário pergunta.

```
┌─────────────────────────────────────────────────────────────────┐
│             MULTIMODAL CONTENT PROCESSOR                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📄 PDF Document                                                │
│  ⬇️                                                              │
│  🖼️  Image Extraction (PDFImageExtractor)                       │
│  │   ├─ PyMuPDF para extração de imagens embedded              │
│  │   ├─ Filtro por tamanho mínimo (evita ícones/logos)         │
│  │   ├─ Metadados:  page_number, position, dimensions           │
│  │   └─ Formato: JPEG, PNG preservado                          │
│  ⬇️                                                              │
│  🔍 OCR Processing (se imagem contém texto)                     │
│  │   ├─ Engine local (Tesseract / EasyOCR)                      │
│  │   ├─ Engine cloud (AWS Textract / Azure DI / Google Vision)  │
│  │   └─ Executa somente quando habilitado por YAML              │
│  ⬇️                                                              │
│  🤖 Image Description (AI-powered)                              │
│  │   ├─ Providers: OpenAI / Azure OpenAI / Gemini / Bedrock     │
│  │   ├─ Fallback: local (determinístico)                        │
│  │   └─ Context-aware: usa texto ao redor                       │
│  ⬇️                                                              │
│  🧮 Vision Embedding (opcional)                                 │
│  │   ├─ Providers: local / OpenAI / Azure OpenAI / Vertex AI / Bedrock │
│  │   └─ Persistência do vetor habilita busca visual no RAG       │
│  ⬇️                                                              │
│  📝 Context Enrichment                                          │
│  │   ├─ Texto ANTES da imagem (250 chars)                      │
│  │   ├─ Texto DEPOIS da imagem (250 chars)                     │
│  │   ├─ Referências:  "Figura 1", "Tabela 3"                    │
│  │   └─ Keywords técnicas extraídas                            │
│  ⬇️                                                              │
│  💾 Enhanced Chunk                                              │
│      ├─ Texto original                                          │
│      ├─ + Texto extraído via OCR                                │
│      ├─ + Descrição da imagem                                   │
│      ├─ + Contexto textual                                      │
│      └─ Metadata:  image_id, has_visual_content                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Componentes Multimodais (contrato real)

**Contrato multimodal**
- Use o contrato central em
    [src/ingestion_layer/interfaces/multimodal_interfaces.py](src/ingestion_layer/interfaces/multimodal_interfaces.py)
    para `ImageData`/`MultimodalChunk` nas extensões DNIT.
- O wrapper em
    [src/ingestion_layer/multimodal/interfaces/multimodal_interfaces.py](src/ingestion_layer/multimodal/interfaces/multimodal_interfaces.py)
    existe apenas por compatibilidade; evite definir dataclasses locais.

#### 4.2.1 **PDFImageExtractor**

**Código**:  `src/ingestion_layer/multimodal/pdf_image_extractor.py`

**Funcionalidades:**

1. **Extração Inteligente**
   ```python
   async def extract_images(self, document_path: Path) -> list[ImageData]:
       doc = fitz.open(str(document_path))
       images = []

       for page_num in range(len(doc)):
           page = doc[page_num]
           page_images = await self._extract_page_images(page, page_num)
           images.extend(page_images)

       return images
   ```

2. **Filtros de Qualidade**
   - Tamanho mínimo:  1500 bytes
   - Dimensões mínimas: 40x40 pixels
   - Máximo 8 imagens por página
   - Formatos suportados: JPEG, PNG, GIF, BMP, TIFF

3. **Metadados Capturados**
   ```python
   @dataclass
   class ImageData:
       image_id: str
       image_bytes: bytes
       format: str
       width: int
       height: int
       page_number: int
       position_in_page: int
   ```

#### 4.2.2 **OCR Processors**

**Código**: `src/core/multimodal_runtime/ocr_processors.py`

**O que existe de verdade (sem chute):**
O OCR multimodal do PDF é escolhido pela fila ordenada no YAML, no bloco:
`ingestion.content_profiles.type_specific.pdf.multimodal.ocr.base.options`.

Engines suportados hoje:
- `tesseract`
- `rapidocr`
- `easyocr`
- `gemini_flash_page_ocr`
- `aws_textract`
- `azure_document_intelligence`
- `google_vision`
- `disabled`

**Explicação conceitual**
OCR é útil quando a “imagem” na verdade contém texto (ex.: um print de tabela,
uma legenda dentro do diagrama ou um PDF escaneado). O pipeline só executa OCR
se `ingestion.content_profiles.type_specific.pdf.multimodal.ocr.enabled=true` e se a imagem passa por heurísticas básicas (tamanho e
proporção) para evitar gastar processamento com ícones e logos.

**Explicação for dummies**
OCR é o “copiar e colar” de uma imagem. Ele tenta achar letras e números dentro
da figura e transformar em texto normal. Isso ajuda muito quando o documento
tem informações que não foram extraídas como texto pelo parser do PDF.

#### 4.2.3 **Image Descriptors**

**Código**: `src/core/multimodal_runtime/image_descriptors.py`

**O que existe de verdade (sem inventar):**
O descritor de imagens do PDF é escolhido pela fila ordenada no YAML em:
`ingestion.content_profiles.type_specific.pdf.multimodal.image_description.base.options`.

Providers suportados hoje:
- `openai`
- `azure_openai` (Microsoft)
- `gemini` (Google)
- `bedrock` (AWS)
- `clip_endpoint` (endpoint custom)
- `local` (fallback determinístico)
- `disabled`

**Explicação conceitual**
A descrição complementa o OCR. Ela é usada quando a imagem tem informação
visual que não vira texto (ex.: um desenho de instalação, uma seta apontando
para um ponto, um diagrama de camadas). O pipeline passa também um contexto
textual “ao redor” da imagem para reduzir alucinação e manter a descrição
mais alinhada ao documento.

**Explicação for dummies**
Se OCR é “ler letras”, a descrição é “entender o desenho”. É como pedir para
alguém olhar a figura e explicar o que ela mostra, com foco técnico.

### 4.3 Exemplos de Enriquecimento Multimodal

#### Exemplo 1: Diagrama Técnico

**Input:** Página PDF com diagrama de instalação de defensa metálica

**Processamento:**

1. **Extração**:
   - Imagem detectada:  800x600px, JPEG
   - Posição: Centro da página 15

2. **OCR**:
   ```
   Texto extraído: "DMS - Defensa Metálica Simples
   Altura: 0,70m
   Afastamento: 0,30m
   Código: DNIT-031/2006-ES"
   ```

3. **Descrição (modelo de visão)**:
   ```
   "Diagrama técnico mostrando vista frontal e lateral de defensa
   metálica simples.  Especifica altura de instalação de 70cm e
   afastamento de 30cm do bordo da pista. Inclui detalhes de
   ancoragem e espaçamento entre postes."
   ```

4. **Contexto Textual**:
   ```
   Antes: "... instalação deve seguir especificações da norma..."
   Depois: "...materiais devem atender aos requisitos de resistência..."
   ```

5. **Chunk Final**:
   ```markdown
   ## Instalação de Defensa Metálica (Página 15)

   A instalação deve seguir especificações da norma...

   [IMAGEM:  Diagrama técnico mostrando vista frontal e lateral de
   defensa metálica simples. Especifica altura de instalação de 70cm
   e afastamento de 30cm do bordo da pista...]

   OCR Extraído:
   DMS - Defensa Metálica Simples
   Altura: 0,70m
   Afastamento: 0,30m
   Código:  DNIT-031/2006-ES

   ... materiais devem atender aos requisitos de resistência...

   Metadata: {
     "has_visual_content": true,
     "image_count": 1,
     "technical_codes": ["DNIT-031/2006-ES"],
     "page_number": 15
   }
   ```

---

### 4.4 Configuração YAML (visão, OCR e embedding visual)

#### Visão geral
No DNIT, recomendamos tratar o multimodal como uma configuração **por coleção**:
coleções com muitos diagramas e PDFs escaneados se beneficiam mais; coleções já
100% textuais podem manter o multimodal desligado para reduzir custo e latência.

#### Explicação for dummies
Você escolhe, no YAML, se quer que o sistema “enxergue imagens”. Se ligar, você
também escolhe *quem* vai fazer isso (OpenAI, Microsoft, Google, AWS) e qual
OCR usar (local ou cloud). Se desligar, o sistema continua funcionando, só não
vai extrair informação extra das figuras.

#### Quais filas o DNIT realmente usa no PDF

| Fila | Chave YAML | O que controla |
| --- | --- | --- |
| Parsing principal | `content_profiles.type_specific.pdf.processing.parsing.base.options` | Quem lê o texto base do PDF |
| OCR document-level | `content_profiles.type_specific.pdf.processing.ocr.document_preprocessing.base.options` | Se o documento inteiro pode passar por OCRmyPDF antes do parsing |
| OCR por página | `content_profiles.type_specific.pdf.processing.ocr.base.options` | Quem tenta recuperar páginas com pouco texto |
| Tabelas | `content_profiles.type_specific.pdf.processing.tables.base.options` | Quem tenta extrair tabelas |
| OCR multimodal | `content_profiles.type_specific.pdf.multimodal.ocr.base.options` | Quem lê texto dentro das imagens extraídas |
| Descrição de imagem | `content_profiles.type_specific.pdf.multimodal.image_description.base.options` | Quem descreve o conteúdo visual das imagens |
| Embedding visual | `content_profiles.type_specific.pdf.multimodal.vision_embedding.base.options` | Quem gera vetor visual para busca multimodal |

#### Onde ver a matriz completa de engines e chaves

- O documento dono do mecanismo completo de engines do PDF está em [README-INGESTAO.md](README-INGESTAO.md).
- O tutorial 101 com explicação passo a passo e diagramas do pipeline está em [tutorial-101-ingestao-pdf.md](tutorial-101-ingestao-pdf.md).

#### Limites e pegadinhas
- **Custos**: descrição de imagem e OCR cloud podem aumentar custo por documento.
- **Latência**: pipelines com muitos PDFs/imagens podem ficar mais lentos.
- **Credenciais**:
    - Azure DI exige `security_keys.AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT` e `security_keys.AZURE_DOCUMENT_INTELLIGENCE_API_KEY`.
    - Google Vision e Vertex AI usam ADC (Application Default Credentials) do ambiente.
    - AWS Bedrock/Textract exigem credenciais AWS disponíveis no runtime.

#### Troubleshooting
- Se o OCR não roda no PDF: confira `ingestion.content_profiles.type_specific.pdf.multimodal.ocr.enabled=true` e a fila em `ingestion.content_profiles.type_specific.pdf.multimodal.ocr.base.options`.
- Se a descrição não aparece no PDF: confira `ingestion.content_profiles.type_specific.pdf.multimodal.image_description.enabled=true` e a fila em `ingestion.content_profiles.type_specific.pdf.multimodal.image_description.base.options`.
- Se o embedding visual está vazio: confira `ingestion.content_profiles.type_specific.pdf.multimodal.vision_embedding.enabled=true`, a fila em `ingestion.content_profiles.type_specific.pdf.multimodal.vision_embedding.base.options`, o bloco comum `ingestion.content_profiles.type_specific.pdf.multimodal.vision_embedding.runtime` e o subbloco do provider ativo, como `ingestion.content_profiles.type_specific.pdf.multimodal.vision_embedding.google_genai`.
- Se a dúvida for sobre imagem fora de PDF, use o escopo global `ingestion.multimodal_ai.ocr`, `ingestion.multimodal_ai.image_description` e `ingestion.multimodal_ai.vision_embedding`.
- Se houver dúvida sobre qual chave de credencial pertence a cada engine, consulte [README-INGESTAO.md](README-INGESTAO.md), que agora traz a matriz completa por fila.

## 5. Pipeline RAG Inteligente

### 5.1 Arquitetura do RAG Avançado

```
┌─────────────────────────────────────────────────────────────────┐
│                  INTELLIGENT RAG ORCHESTRATOR                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❓ QUERY INPUT                                                 │
│  └─ "Qual a especificação DNIT-031/2006-ES para granulometria?"│
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  🧠 QUERY ANALYSIS                                              │
│  ├─ QueryAnalyzer:  detecta tipo (technical_code)                │
│  ├─ DnitQueryExpansion: expande termos técnicos                 │
│  │   └─ "granulometria" → ["distribuição granulométrica",      │
│  │       "curva granulométrica", "faixa granulométrica"]        │
│  └─ AdaptiveRouter: seleciona estratégia (HYBRID)              │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  🔍 HYBRID RETRIEVAL                                            │
│  ├─ BM25 Retriever (sparse, keyword-based)                     │
│  │   ├─ Vocabulário DNIT expandido                              │
│  │   ├─ Tokenização: preserva códigos intactos                  │
│  │   └─ Score: TF-IDF com boost para códigos                    │
│  │                                                               │
│  ├─ Vector Retriever (dense, semantic)                         │
│  │   ├─ Embeddings:  OpenAI text-embedding-3-large              │
│  │   ├─ MMR (Maximum Marginal Relevance) para diversidade      │
│  │   └─ Metadata filters:  domain="dnit"                         │
│  │                                                               │
│  └─ RRF Fusion (Reciprocal Rank Fusion)                        │
│      ├─ Combina scores BM25 + Vetor                             │
│      ├─ Peso configurável (default: 60% vetor, 40% BM25)        │
│      └─ Ranking final balanceado                                │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  🎯 RE-RANKING                                                  │
│  ├─ Neural Reranker (cross-encoder)                            │
│  ├─ Modelo:  ms-marco-MiniLM-L-6-v2                             │
│  ├─ Relevância contextual: query ↔ documento                   │
│  └─ Top-K final: melhores 5 documentos                         │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│  💬 GENERATION                                                  │
│  ├─ Prompt Engineering: contexto técnico                        │
│  ├─ LLM: GPT-4o (Azure OpenAI)                                 │
│  ├─ Temperature: 0.1 (baixa criatividade)                      │
│  └─ Sources: citações com page_number                          │
│                                                                  │
│  ⬇️                                                              │
│                                                                  │
│   RESPONSE                                                    │
│  ├─ Answer:  resposta estruturada                                │
│  ├─ Sources: documentos com scores                              │
│  ├─ Confidence: 0.87                                            │
│  └─ Metadata: estratégia usada, tokens, duração                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Componentes do RAG

#### 5.2.1 **Query Analyzer**

**Código**:  `src/qa_layer/rag_engine/query_analyzer.py`

**Classificação de Queries:**

```python
@dataclass
class QueryFeatures:
    query_type: QueryType  # technical_code, conceptual, numerical, etc.
    data_type: DataType    # specification, procedure, regulation
    complexity: float      # 0.0 (simples) a 1.0 (complexa)
    entities: list[str]    # ["DNIT-031/2006-ES", "granulometria"]
    has_exact_codes: bool  # True se contém código normativo
    needs_visual:  bool     # True se pergunta sobre diagrama/tabela
```

**Regras de Detecção:**

```python
# Detecta código técnico exato
if re.search(r'\bDNIT-\d{3}/\d{4}-[A-Z]{2}\b', query):
    query_type = QueryType.TECHNICAL_CODE
    strategy = RetrievalStrategy.HYBRID  # BM25 essential!
```

#### 5.2.2 **DNIT Query Expansion**

**Código**: `src/qa_layer/rag_engine/dnit_query_expansion.py`

**Vocabulário Técnico Expandido:**

```yaml
vocabulary:
  pavimentação:
    synonyms:  ["pavimento", "revestimento asfáltico", "capa asfáltica"]
    related:  ["base", "sub-base", "subleito"]

  granulometria:
    synonyms: ["distribuição granulométrica", "curva granulométrica"]
    related: ["peneiramento", "faixa granulométrica", "classificação"]

  defensa:
    synonyms: ["barreira", "guarda-corpo", "proteção lateral"]
    related: ["DMS", "DMD", "defensa metálica"]
    codes: ["DNIT-031/2006-ES", "NBR 6971"]
```

**Processo de Expansão:**

```python
def expand_query(query: str) -> str:
    expanded_terms = []

    for term in query.split():
        if term in vocabulary:
            # Adiciona sinônimos
            expanded_terms.extend(vocabulary[term]['synonyms'])

            # Adiciona códigos relacionados
            if 'codes' in vocabulary[term]:
                expanded_terms.extend(vocabulary[term]['codes'])

    return original_query + " " + " ".join(expanded_terms)
```

**Exemplo:**
```
Input:   "Qual especificação para granulometria de base?"
Output: "Qual especificação para granulometria distribuição
         granulométrica curva granulométrica de base DNIT-031/2006-ES
         DNIT-139/2010-ES?"

##### 5.2.2.1 Integração com auto_config de domínio

- **Origem dos dados**: `DomainAutoConfigAggregator` captura sinais dos
    manifestos (termos, códigos, headings, tabelas) e grava em
    `auto_config.vocabulary` e `auto_config.metadata` (manifest_count,
    truncated_by_limit, scan_duration_ms, tables).
- **Propagação**: `DomainConfigurationService` mescla o `auto_config`
    com o YAML (sem sobrepor o `manual_config`) e injeta no bloco
    `domain_specific_rag.domains.<domínio>.query_expansion`, incluindo
    `auto_detection_keywords` e `stats.auto_config`.
- **Consumo no RAG**:
    - A etapa de expansão ativa automaticamente se houver vocabulário
        (manual ou auto) e usa termos/códigos/headings para matches.
    - Hints de saúde: `auto_config_truncated` é adicionado ao
        `context_hints` quando o rebuild truncou, sem bloquear a execução.
    - Hints de performance/limite: `auto_config_slow_scan` é adicionado
        quando `scan_duration_ms` ultrapassa o limiar inicial de 5000 ms
        (ajustável por domínio); `auto_config_limit_hit` aparece quando
        `manifest_count` atinge o `max_manifests_scan`. São apenas
        sinalizações, não quebram o fluxo.
    - Hints de tabela: quando o auto_config indica presença de tabelas ou
        a query sugere tabela/quadro, adiciona `prioritize_domain_tables`.

Tabela de sinais no runtime (DNIT):

| Hint                         | Gatilho (estatística do auto_config)      |
|-----------------------------|-------------------------------------------|
| auto_config_truncated       | truncated_by_limit = true                 |
| auto_config_slow_scan       | scan_duration_ms > 5000 (limiar padrão)   |
| auto_config_limit_hit       | manifest_count >= max_manifests_scan      |
| prioritize_domain_tables    | tables.total_tables > 0 ou query cita     |
|                             | tabela/quadro/planilha                    |

Observação: os limiares são configuráveis por domínio; para novos
domínios basta reaproveitar o mesmo mecanismo de hints e ajustar o
limiar de `slow_scan_threshold_ms` conforme a volumetria esperada. O
auto_config continua complementar ao manual_config: se houver ajustes
manuais, eles prevalecem, e o vocabulário automático só amplia recall.
- **Efeito prático**:
    - Mais recall inicial por expansão com vocabulário descoberto nos
        manifestos.
    - Menos falsos negativos na detecção de domínio via
        `auto_detection_keywords`.
    - Observabilidade segura: truncamento e estatísticas não quebram o
        fluxo; servem para priorizar e monitorar.
```

#### 5.2.3 **BM25 Retriever**

**Código**: `src/qa_layer/rag_engine/bm25_retriever.py`

**Por Que BM25 é Essencial:**

BM25 (Best Match 25) é um algoritmo de ranking baseado em **frequência de termos** e **comprimento do documento**.  É superior a TF-IDF para busca de códigos exatos.

**Fórmula BM25:**

```
Score(D,Q) = Σ IDF(qi) × (f(qi,D) × (k1 + 1)) / (f(qi,D) + k1 × (1 - b + b × |D|/avgdl))

Onde:
- f(qi,D): frequência do termo qi no documento D
- |D|: comprimento do documento D
- avgdl: comprimento médio dos documentos
- k1, b: parâmetros (default: k1=1.5, b=0.75)
```

**Vantagens para DNIT:**

1. **Match Exato**: "DNIT-031/2006-ES" aparece no doc → Score alto
2. **Raridade Conta**: Códigos normativos são raros → IDF alto
3. **Normalização**: Documentos longos não são super-penalizados

**Implementação:**

```python
class BM25Retriever:
    def __init__(self, documents: list[str]):
        self.tokenized_docs = [self._tokenize(doc) for doc in documents]
        self.bm25 = BM25Okapi(self.tokenized_docs)

    def _tokenize(self, text: str) -> list[str]:
        # Preserva códigos DNIT intactos
        tokens = []
        for token in text.split():
            if re.match(r'DNIT-\d+/\d+-[A-Z]+', token):
                tokens.append(token)  # Código completo
            else:
                tokens.extend(token.lower().split('-'))  # Tokeniza normalmente
        return tokens

    async def retrieve(self, query: str, top_k: int = 10) -> list[Document]:
        query_tokens = self._tokenize(query)
        scores = self.bm25.get_scores(query_tokens)

        top_indices = np.argsort(scores)[::-1][:top_k]

        return [
            {
                "content": self.documents[i],
                "score": scores[i],
                "metadata": {"bm25_score": scores[i]}
            }
            for i in top_indices
        ]
```

#### 5.2.4 **Hybrid Retriever**

**Código**: `src/qa_layer/rag_engine/retrievers. py`

**Fusão RRF (Reciprocal Rank Fusion):**

```python
def reciprocal_rank_fusion(
    results_bm25: list[Document],
    results_vector: list[Document],
    k: int = 60
) -> list[Document]:
    """
    RRF combina rankings de múltiplas fontes.

    Score(doc) = Σ 1/(k + rank_i(doc))

    k=60: constante para suavizar ranking
    """
    scores = defaultdict(float)

    # Score BM25
    for rank, doc in enumerate(results_bm25, start=1):
        doc_id = doc['metadata']['document_id']
        scores[doc_id] += 1 / (k + rank)

    # Score Vetorial
    for rank, doc in enumerate(results_vector, start=1):
        doc_id = doc['metadata']['document_id']
        scores[doc_id] += 1 / (k + rank)

    # Ordena por score final
    sorted_docs = sorted(scores.items(), key=lambda x: x[1], reverse=True)

    return [document_map[doc_id] for doc_id, score in sorted_docs]
```

**Exemplo de Fusão:**

```yaml
Query: "DNIT-031/2006-ES granulometria"

BM25 Results:
1. Doc_A (DNIT-031/2006-ES) - score: 15.3
2. Doc_B (granulometria geral) - score: 8.7
3. Doc_C (outras normas) - score: 5.2

Vector Results:
1. Doc_B (semanticamente sobre granulometria) - score: 0.91
2. Doc_A (DNIT-031 mencionado) - score: 0.85
3. Doc_D (procedimentos de ensaio) - score: 0.78

RRF Fusion:
1. Doc_A:  1/61 + 1/62 = 0.0164 + 0.0161 = 0.0325  TOP
2. Doc_B: 1/62 + 1/61 = 0.0161 + 0.0164 = 0.0325  EMPATE
3. Doc_C:  1/63 = 0.0159
4. Doc_D: 1/63 = 0.0159

Resultado Final: Doc_A e Doc_B no topo (exatamente o que queremos!)
```

#### 5.2.5 **Neural Reranker**

**Código**: `src/qa_layer/rag_engine/reranker.py`

**Cross-Encoder para Relevância:**

Enquanto embeddings vetoriais geram representações **independentes** de query e documento, o cross-encoder avalia a **relevância conjunta**.

```python
class NeuralReranker:
    def __init__(self):
        self.model = CrossEncoder('cross-encoder/mmarco-mMiniLMv2-L12-H384-v1')

    async def rerank(
        self,
        question:  str,
        documents: list[Document],
        top_k:  int = 5
    ) -> list[Document]:
        """
        Reordena documentos por relevância real à query.
        """
        # Cria pares (query, documento)
        pairs = [(question, doc['content']) for doc in documents]

        # Modelo avalia relevância de cada par
        scores = self.model.predict(pairs)

        # Ordena por score
        ranked_docs = [
            {**doc, 'rerank_score': score}
            for doc, score in sorted(
                zip(documents, scores),
                key=lambda x: x[1],
                reverse=True
            )
        ]

        return ranked_docs[: top_k]
```

**Diferença Crítica:**

| Método | Como Funciona | Quando Usar |
|--------|---------------|-------------|
| **Bi-Encoder** (embeddings) | `embed(Q)` e `embed(D)` separados → cosseno | Primeira filtragem, milhões de docs |
| **Cross-Encoder** (reranker) | `score(Q, D)` conjunto → relevância direta | Refinamento final, top-100 docs |

**Modelo atual (multilíngue PT‑BR) + fallback:**

```yaml
reranker:
  enabled: true
  model: "cross-encoder/mmarco-mMiniLMv2-L12-H384-v1"
  fallback_model: "cross-encoder/ms-marco-MiniLM-L-6-v2"
  top_k: 8
```

**Exemplo:**

```yaml
Query: "Como instalar defensa metálica em curva?"

Após Hybrid Retrieval (top 10):
1. Doc_A: Manual DNIT defensas - score: 0.85
2. Doc_B: Procedimentos instalação - score: 0.82
3. Doc_C:  Especificação materiais - score: 0.78
...

Após Reranking (top 5):
1. Doc_B: Procedimentos instalação → 0.93 (subiu!) ✅
2. Doc_A: Manual DNIT defensas → 0.91
3. Doc_G: Instalação em curvas → 0.89 (entrou no top!) ✅
4. Doc_C: Especificação materiais → 0.76
5. Doc_D:  Manutenção defensas → 0.71
```

---

## 6. Técnicas Avançadas Implementadas

### 6.1 Domain Processing

**Código**: `src/ingestion_layer/domain_processing/`

**Resolver Específico DNIT:**

```python
class DnitDomainProcessor:
    """
    Processador especializado para documentos técnicos DNIT.
    """

    # Padrões de códigos normativos
    CODE_PATTERNS = [
        r"\bDNIT-\d{3}/\d{4}-[A-Z]{2}\b",     # DNIT-031/2006-ES
        r"\bNBR\s*\d{3,5}\b",                  # NBR 7180
        r"\bABNT\s*NBR\s*\d+\b",              # ABNT NBR 16697
        r"\bDNER-ME\s*\d{3}/\d{2}\b",         # DNER-ME 164/94
    ]

    # Keywords técnicas prioritárias
    FOCUS_KEYWORDS = [
        "pavimentação", "granulometria", "CBR", "ISC",
        "defensa", "barreira", "proteção", "sinalização",
        "drenagem", "obras de arte", "pontes", "viadutos",
        "terraplenagem", "compactação", "umidade ótima"
    ]

    def detect_codes(self, text: str) -> list[str]:
        """Extrai todos os códigos normativos do texto."""
        codes = []
        for pattern in self.CODE_PATTERNS:
            matches = re.findall(pattern, text)
            codes.extend(matches)
        return list(set(codes))  # Remove duplicatas

    def classify_page(self, page_text: str) -> dict:
        """Classifica relevância da página."""
        codes = self.detect_codes(page_text)
        keyword_count = sum(1 for kw in self.FOCUS_KEYWORDS if kw in page_text.lower())

        return {
            "priority":  "high" if (codes or keyword_count >= 3) else "normal",
            "detected_codes": codes,
            "keyword_matches": keyword_count,
            "has_technical_content": bool(codes or keyword_count > 0)
        }
```

**Impacto:**
-  Páginas com códigos DNIT → Prioridade alta na indexação
-  Páginas com keywords técnicas → Chunks maiores preservados
-  Páginas genéricas (índice, capa) → Chunks menores ou ignoradas

### 6.2 Query Expansion

**Estratégia de Expansão:**

```python
@dataclass
class ExpansionRule:
    term: str
    synonyms: list[str]
    related_terms: list[str]
    normative_codes: list[str]
    boost_weight: float = 1.0

DNIT_VOCABULARY = {
    "pavimentação": ExpansionRule(
        term="pavimentação",
        synonyms=["pavimento", "revestimento", "capa asfáltica"],
        related_terms=["base", "sub-base", "subleito", "CBUQ"],
        normative_codes=["DNIT-031/2006-ES", "DNIT-139/2010-ES"],
        boost_weight=1.5
    ),

    "granulometria":  ExpansionRule(
        term="granulometria",

        synonyms=["distribuição granulométrica", "curva granulométrica"],
        related_terms=["peneiramento", "faixa", "graduação"],
        normative_codes=["NBR 7181", "DNER-ME 083/98"],
        boost_weight=1.3
    ),

    "CBR": ExpansionRule(
        term="CBR",
        synonyms=["Índice Suporte Califórnia", "ISC"],
        related_terms=["capacidade de suporte", "resistência"],
        normative_codes=["NBR 9895", "DNER-ME 049/94"],
        boost_weight=2.0  # Código crítico!
    )
}
```

**Processo de Expansão:**

```python
def expand_dnit_query(query: str) -> str:
    expanded = []
    tokens = query.lower().split()

    for token in tokens:
        if token in DNIT_VOCABULARY:
            rule = DNIT_VOCABULARY[token]

            # Adiciona termo original com boost
            expanded.append(f"{token}^{rule.boost_weight}")

            # Adiciona sinônimos
            for syn in rule.synonyms:
                expanded.append(syn)

            # Adiciona códigos normativos (alto boost!)
            for code in rule.normative_codes:
                expanded. append(f"{code}^3. 0")
        else:
            expanded.append(token)

    return " ".join(expanded)
```

**Exemplo:**

```yaml
Query Original: "Especificação CBR para pavimento flexível"

Query Expandida (interna, para BM25):
"Especificação^1.0 CBR^2.0 Índice_Suporte_Califórnia ISC
NBR_9895^3.0 DNER-ME_049/94^3.0 capacidade_de_suporte
para pavimento^1.5 revestimento capa_asfáltica DNIT-031/2006-ES^3.0
flexível^1.0"

Resultado: BM25 encontra documentos que mencionam:
 "CBR" explicitamente (score alto)
 "NBR 9895" (norma do ensaio CBR) (score muito alto!)
 Termos relacionados como "capacidade de suporte"
```

### 6.3 Metadata Filtering

**Filtros Estruturados:**

```python
@dataclass
class DocumentMetadata:
    # Identificação
    document_id: str
    source_type: str  # pdf, confluence, web
    domain: str       # dnit_engineering, general

    # Classificação
    content_type: str       # specification, procedure, regulation
    technical_level: str    # basic, intermediate, advanced

    # Rastreabilidade
    page_number: int | None
    section:  str | None
    normative_codes: list[str]

    # Temporal
    document_date: datetime | None
    version: str | None

    # Multimodal
    has_visual_content: bool
    image_count: int
    has_tables: bool

    # Qualidade
    ocr_applied: bool
    text_density: float
    confidence_score: float
```

**Aplicação de Filtros:**

```python
# Query:  "Normas DNIT sobre defensas publicadas após 2010"

metadata_filters = {
    "domain": "dnit_engineering",
    "content_type": "specification",
    "normative_codes": {"$contains": "DNIT"},
    "document_date": {"$gte": datetime(2010, 1, 1)},
    "technical_level": {"$in": ["intermediate", "advanced"]}
}

# Sistema aplica filtros ANTES da busca vetorial
filtered_docs = vector_store.search_similar(
    query="defensas metálicas",
    top_k=20,
    filters=metadata_filters
)
```

### 6.4 Chunking Strategies

**Código**: `src/ingestion_layer/processors/chunking_strategies.py`

#### Strategy 1: Page-Based (para PDFs estruturados)

```python
class PageBasedChunkingStrategy:
    """Respeita limites de página do documento original."""

    def chunk(self, document: BaseDocument) -> list[ContentChunk]:
        chunks = []

        # Detecta marcadores de página
        page_pattern = r'\[Página (\d+)\]'
        pages = re.split(page_pattern, document.content)

        for i in range(1, len(pages), 2):
            page_num = int(pages[i])
            page_content = pages[i+1]. strip()

            # Cria chunk por página
            if len(page_content) >= self.min_chunk_size:
                chunks.append(ContentChunk(
                    content=page_content,
                    metadata={
                        "page_number":  page_num,
                        "chunk_strategy": "page_based"
                    }
                ))

        return chunks
```

#### Strategy 2: Semantic (baseada em mudança de tópico)

```python
class SemanticChunkingStrategy:
    """Quebra texto quando detecta mudança semântica."""

    def __init__(self, embeddings_model):
        self.embeddings = embeddings_model
        self.similarity_threshold = 0.75

    def chunk(self, document: BaseDocument) -> list[ContentChunk]:
        sentences = document.content.split('.  ')
        sentence_embeddings = self.embeddings.embed_documents(sentences)

        chunks = []
        current_chunk = [sentences[0]]

        for i in range(1, len(sentences)):
            # Calcula similaridade com chunk atual
            chunk_embedding = np.mean([sentence_embeddings[j] for j in range(len(current_chunk))], axis=0)
            similarity = cosine_similarity([chunk_embedding], [sentence_embeddings[i]])[0][0]

            if similarity > self.similarity_threshold:
                # Tópico similar → adiciona ao chunk atual
                current_chunk.append(sentences[i])
            else:
                # Tópico mudou → fecha chunk e inicia novo
                chunks.append(ContentChunk(
                    content='.  '.join(current_chunk),
                    metadata={"chunk_strategy": "semantic"}
                ))
                current_chunk = [sentences[i]]

        # Adiciona último chunk
        if current_chunk:
            chunks. append(ContentChunk(
                content='. '.join(current_chunk),
                metadata={"chunk_strategy": "semantic"}
            ))

        return chunks
```

#### Strategy 3: DNIT-Specific (domain-aware)

```python
class DnitChunkingStrategy:
    """Chunking especializado para documentos DNIT."""

    SECTION_MARKERS = [
        r'^\d+\.\s+[A-Z]',           # 1. OBJETIVO
        r'^\d+\.\d+\s+',             # 1.1 Subseção
        r'^ANEXO\s+[A-Z]',           # ANEXO A
        r'^Tabela\s+\d+',            # Tabela 1
        r'^Figura\s+\d+',            # Figura 1
    ]

    def chunk(self, document: BaseDocument) -> list[ContentChunk]:
        lines = document.content.split('\n')
        chunks = []
        current_section = []
        current_section_title = None

        for line in lines:
            # Detecta início de nova seção
            if any(re.match(pattern, line. strip()) for pattern in self. SECTION_MARKERS):
                # Salva seção anterior
                if current_section:
                    chunks.append(self._create_chunk(
                        current_section,
                        current_section_title
                    ))

                # Inicia nova seção
                current_section = [line]
                current_section_title = line.strip()
            else:
                current_section.append(line)

            # Força quebra se chunk ficou muito grande
            if self._get_size(current_section) > self.max_chunk_size:
                chunks.append(self._create_chunk(
                    current_section,
                    current_section_title
                ))
                current_section = []

        # Adiciona última seção
        if current_section:
            chunks.append(self._create_chunk(
                current_section,
                current_section_title
            ))

        return chunks

    def _create_chunk(self, lines: list[str], title: str | None) -> ContentChunk:
        content = '\n'.join(lines)

        # Extrai códigos normativos do chunk
        codes = self. domain_processor.detect_codes(content)

        return ContentChunk(
            content=content,
            metadata={
                "section_title": title,
                "normative_codes": codes,
                "chunk_strategy": "dnit_specific",
                "has_technical_codes": bool(codes)
            }
        )
```

### 6.5 Deduplicação Inteligente

**Estratégia de Dedup:**

```python
class IntelligentDeduplicator:
    """Remove chunks duplicados ou muito similares."""

    def __init__(self):
        self.hash_threshold = 0.95  # 95% de overlap → duplicata exata
        self.semantic_threshold = 0.90  # 90% similaridade → duplicata semântica

    def deduplicate(self, chunks: list[ContentChunk]) -> list[ContentChunk]:
        unique_chunks = []
        seen_hashes = set()

        for chunk in chunks:
            # 1. Check hash-based dedup (rápido)
            content_hash = self._compute_hash(chunk.content)

            if content_hash in seen_hashes:
                continue  # Duplicata exata

            # 2. Check semantic similarity (mais lento)
            if self._is_semantically_duplicate(chunk, unique_chunks):
                continue  # Duplicata semântica

            # 3. Adiciona chunk único
            seen_hashes.add(content_hash)
            unique_chunks. append(chunk)

        return unique_chunks

    def _compute_hash(self, content: str) -> str:
        # Normaliza texto antes do hash
        normalized = re.sub(r'\s+', ' ', content.lower().strip())
        return hashlib. sha256(normalized.encode()).hexdigest()

    def _is_semantically_duplicate(self, chunk: ContentChunk, existing:  list[ContentChunk]) -> bool:
        chunk_embedding = self.embeddings.embed_query(chunk.content)

        for existing_chunk in existing:
            existing_embedding = self.embeddings. embed_query(existing_chunk. content)
            similarity = cosine_similarity([chunk_embedding], [existing_embedding])[0][0]

            if similarity > self.semantic_threshold:
                return True  # Muito similar a um chunk existente

        return False
```

---

## 7. Casos de Uso e Exemplos Práticos

### 7.1 Caso 1: Consulta de Norma Específica

#### Pergunta
```
"Quais os requisitos da norma DNIT-031/2006-ES para defensa metálica simples?"
```

#### Processamento Completo

**1. Query Analysis**
```yaml
QueryFeatures:
  query_type:  TECHNICAL_CODE
  has_exact_codes: true
  entities:  ["DNIT-031/2006-ES", "defensa metálica simples"]
  complexity: 0.3 (baixa - pergunta direta)
  needs_visual: false
```

**2. Query Expansion**
```yaml
Expanded Query (interno):
"requisitos norma DNIT-031/2006-ES^3.0 especificação
defensa barreira proteção metálica simples DMS"
```

**3. Routing Decision**
```yaml
Strategy: HYBRID (BM25 + Vector)
Reasoning:
  - Código exato detectado → BM25 crítico
  - Termo "requisitos" conceitual → Vector útil
  - Confiança:  0.95
```

**4. Hybrid Retrieval**

**BM25 Results:**
```yaml
1.  DNIT-031-2006-ES_Manual. pdf [Página 15]
   Score: 18.7
   Match: "DNIT-031/2006-ES" (3 ocorrências)

2. DNIT-031-2006-ES_Manual.pdf [Página 8]
   Score: 15.2
   Match: "defensa metálica simples" (2 ocorrências)

3. Especificacoes_Gerais_DNIT. pdf [Página 42]
   Score: 9.1
   Match: "DNIT-031/2006-ES" (1 ocorrência)
```

**Vector Results:**
```yaml
1. DNIT-031-2006-ES_Manual.pdf [Página 15]
   Score: 0.89
   Semântica: "requisitos, especificações, defensa"

2. Manual_Instalacao_Defensas.pdf [Página 23]
   Score: 0.85
   Semântica:  "procedimentos instalação defensa metálica"

3. DNIT-031-2006-ES_Manual.pdf [Página 8]
   Score: 0.82
   Semântica: "características técnicas defensa"
```

**RRF Fusion:**
```yaml
Final Ranking:
1. DNIT-031-2006-ES_Manual.pdf [Página 15] - RRF:  0.0328 ✅
2. DNIT-031-2006-ES_Manual.pdf [Página 8]  - RRF: 0.0321
3. Manual_Instalacao_Defensas.pdf [Pág 23] - RRF: 0.0163
4. Especificacoes_Gerais_DNIT.pdf [Pág 42] - RRF: 0.0159
5. Normas_Seguranca_Viaria.pdf [Pág 7]    - RRF: 0.0152
```

**5. Neural Reranking**
```yaml
Cross-Encoder Scores (query ↔ doc):
1. DNIT-031-2006-ES_Manual.pdf [Pág 15] → 0.94  MANTÉM TOP-1
2. DNIT-031-2006-ES_Manual.pdf [Pág 8]  → 0.91
3. Manual_Instalacao_Defensas.pdf [23]  → 0.78 (cai para 3º)
4. Especificacoes_Gerais_DNIT.pdf [42]  → 0.72
5. Normas_Seguranca_Viaria.pdf [7]      → 0.68
```

**6. Generation (GPT-4o)**

**Prompt Enviado:**
```
Você é um especialista em normas técnicas do DNIT. Baseando-se EXCLUSIVAMENTE
nos documentos fornecidos, responda a seguinte pergunta com precisão técnica:

PERGUNTA:  Quais os requisitos da norma DNIT-031/2006-ES para defensa metálica simples?

DOCUMENTOS DE REFERÊNCIA:

[1] DNIT-031-2006-ES_Manual.pdf - Página 15
A defensa metálica simples (DMS) especificada na norma DNIT-031/2006-ES deve
atender aos seguintes requisitos:
- Altura de instalação: 0,70m a partir do nível do pavimento
- Afastamento do bordo da pista: 0,30m mínimo
- Espaçamento entre postes: 4,00m (viga corrida) ou 2,00m (viga sobreposta)
- Material dos perfis: aço ASTM A-36 ou equivalente
- Espessura mínima da viga: 2,66mm
- Acabamento: galvanização a fogo (550g/m²)

[IMAGEM: Diagrama técnico mostrando vista frontal e lateral da instalação de DMS...]

...

INSTRUÇÕES:
- Cite SEMPRE a fonte (documento e página)
- Inclua valores numéricos exatos
- Mencione se há diagramas técnicos relevantes
- Seja conciso mas completo
```

**7. Resposta Final**

```markdown
## Requisitos da Norma DNIT-031/2006-ES para Defensa Metálica Simples

De acordo com a norma **DNIT-031/2006-ES**, a defensa metálica simples (DMS)
deve atender aos seguintes requisitos técnicos:

### Dimensões de Instalação
- **Altura**:  0,70m a partir do nível do pavimento
- **Afastamento**:  mínimo de 0,30m do bordo da pista
- **Espaçamento entre postes**:
  - 4,00m para sistema de viga corrida
  - 2,00m para sistema de viga sobreposta

### Especificações de Materiais
- **Material dos perfis**: aço ASTM A-36 ou equivalente
- **Espessura da viga**: mínimo 2,66mm
- **Acabamento**: galvanização a fogo com camada mínima de 550g/m²

### Informações Complementares
A norma inclui diagramas técnicos detalhando vistas frontal e lateral da
instalação (página 15 do manual DNIT-031/2006-ES).

---
**Fontes:**
[1] DNIT-031-2006-ES_Manual.pdf - Página 15 (confiança: 94%)
[2] DNIT-031-2006-ES_Manual.pdf - Página 8 (confiança:  91%)
```

**Metadata de Execução:**
```yaml
processing_metadata:
  strategy_used: hybrid
  documents_retrieved: 10
  documents_after_rerank: 5
  documents_used_in_context: 2
  confidence_score: 0.94

  query_analysis:
    complexity: 0.3
    query_type: technical_code
    has_exact_codes: true

  retrieval_breakdown:
    bm25_contribution: 0.45
    vector_contribution: 0.55
    fusion_algorithm: reciprocal_rank_fusion

  token_usage:
    input_tokens: 1847
    output_tokens: 289
    total_cost_usd: 0.0421

  timing:
    query_analysis: 0.12s
    retrieval: 0.87s
    reranking: 0.34s
    generation: 2.45s
    total: 3.78s
```

### 7.2 Caso 2: Consulta Multimodal (Diagrama Técnico)

#### Pergunta
```
"Como é o esquema de instalação da defensa metálica em curva?"
```

#### Processamento

**1. Query Analysis**
```yaml
QueryFeatures:
  query_type: PROCEDURAL
  needs_visual: true   CRÍTICO - "esquema" indica visual
  entities: ["defensa metálica", "curva", "instalação"]
  complexity:  0.6 (média - envolve procedimento)
```

**2. Retrieval com Filtro Multimodal**
```python
metadata_filters = {
    "has_visual_content": true,  # Apenas docs com imagens
    "content_type": "specification",
    "domain": "dnit_engineering"
}
```

**3. Documento Recuperado (Enriquecido)**

```yaml
Document_ID: "DNIT-031-2006_Page_18"

Content:
  "### 3.2 Instalação em Curvas

  A instalação de defensa metálica em trechos curvos exige atenção especial
  ao raio da curva e ao superelevação.

  [IMAGEM ID: img_page18_001]

  OCR Extraído da Imagem:
  - Raio mínimo:  R ≥ 50m
  - Superelevação máxima: 8%
  - Ângulo de inclinação: 10° (direção do tráfego)
  - Espaçamento reduzido: 3,50m (vs 4,00m em reta)

  Descrição AI da Imagem:
  Diagrama técnico mostrando vista superior de instalação de defensa metálica
  em curva.  Ilustra posicionamento dos postes com espaçamento reduzido de 3,50m,
  ângulo de inclinação de 10° na direção interna da curva, e marcações de raio
  de
