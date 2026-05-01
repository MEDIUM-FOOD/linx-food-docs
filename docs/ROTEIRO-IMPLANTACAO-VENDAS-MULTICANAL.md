# Roteiro de Implantação Comercial – WhatsApp, Instagram e Multicanal

## Objetivo
Guiar pré-venda, descoberta, implantação e pós-go-live dos pacotes
WhatsApp, Instagram ou Multicanal. Dividimos em duas partes claras:

- O que o cliente precisa nos fornecer (checklist didático)
- O que a nossa equipe define e prepara internamente

### Para quem esta começando (for dummies)
Pense neste roteiro como um checklist de viagem. Ele diz o que o cliente
precisa entregar e o que a equipe Plataforma de Agentes de IA prepara para o go-live.

### Impacto para o usuario
- Implantacao mais rapida e previsivel.
- Menos retrabalho por falta de dados ou credenciais.
- Go-live com menos riscos operacionais.

## O que o cliente precisa fornecer
- Responsáveis: patrocinador, dono do processo, operação e contato técnico.
- Metas simples: tempo de resposta desejado, volume diário de mensagens,
  principais indicadores a acompanhar (ex.: conversão, satisfação).
- Fontes de conhecimento: FAQ, catálogo, políticas, tabelas de preço,
  scripts de atendimento, listas de produtos/serviços e documentos de
  suporte (PDF, planilhas, links internos permitidos).
- Catálogo rico de produtos/serviços: quanto mais detalhada for a descrição,
  melhor a IA responde. Incluir atributos e tags que ajudam o cliente a
  escolher (ex.: cor, tamanho, fragrância, “refrescante”, “confortável”,
  “ideal para verão”, material, validade, combinações sugeridas). Catálogo
  pobre gera respostas pobres; catálogo rico gera conversas ricas e
  precisas.
- Canais contratados e ativos (explicado de forma simples):
  - WhatsApp: número que será usado no atendimento; quem fornece a conta
    comercial (provedor oficial do WhatsApp Business, chamado de BSP, que é
    a empresa por onde o número é registrado e por onde as mensagens saem);
    se o número já está verificado como conta comercial; quais modelos de
    mensagem já foram aprovados (textos prontos que o WhatsApp aprova para
    disparos iniciados pela empresa, como avisos de pedido ou cobrança).
  - Instagram: Page ID (id da página Facebook ligada à conta IG), Instagram
    Business ID (id da conta IG), App ID e App Secret (da aplicação Meta),
    versão da Graph API em uso, token da página (com permissões ativas), URL
    pública para receber o webhook (callback) e verify token escolhido pelo
    cliente para validar o webhook.
- Volumetria e janelas: horários de maior movimento, campanhas ou datas
  especiais previstas, e como a marca prefere responder em público
  (comentários) e em privado (DM) — por exemplo, resposta curta no
  comentário e convite para DM.
- Operação humana: em quais situações deve chamar um atendente (ex.: pedido
  explícito, tema sensível, cliente recorrente); quem é o time que assume;
  como marcar um atendimento como prioritário/“VIP” (por exemplo, palavra
  chave ou lista de clientes importantes).
- Integrações desejadas (quando existirem): nome do sistema (ERP/CRM/
  e-commerce/pagamento), tipo de acesso (API ou arquivo), contato técnico
  para alinhar formato de dados. Não é necessário o contrato técnico neste
  momento; basta saber o sistema e o ponto de contato.
- Telemetria: se a empresa já possui banco para armazenar interações, indicar
  o schema/tabela ou confirmar uso do padrão interaction_runs que fornecemos.

## O que a nossa equipe define internamente
- Planejamento de workflows por canal (saudação, qualificação, orçamento,
  follow-up, handoff para humano).
- Configuração de YAMLs e roteamento dos canais no tenant.
- Regras de segurança e privacidade: como mascarar dados sensíveis, onde
  armazenar chaves, como registrar logs e métricas.
- Integrações técnicas: formato dos conectores, limites de uso, testes de
  sandbox quando disponível.
- Telemetria e dashboards: como gravar interaction_runs, colunas exigidas e
  visualizações básicas para operação.

## Discovery e levantamento (Semana 0)
- Kickoff: alinhar objetivos e prioridades do cliente.
- Coleta do checklist do cliente (fontes, tokens, números, IDs, metas).
- Confirmação do pacote escolhido (WA, IG ou Multicanal).
- Planejamento conjunto de testes de aceitação (roteiros simples por canal).

## Cronograma recomendado
### Pacote único (WhatsApp ou Instagram): 2–4 semanas
- Semana 1: provisionar canal, validar credenciais e webhook, publicar YAML
  base e ingestão inicial de conhecimento.
- Semana 2: configurar workflows, testes guiados, treinamento rápido da
  operação e handoff humano.
- Semana 3: piloto controlado, coleta de métricas e ajustes finos.
- Semana 4 (opcional): go-live amplo e suporte assistido.

### Pacote Multicanal: 4–6 semanas
- Semana 1: discovery e provisionamento de WhatsApp e Instagram em ambiente
  controlado; ingestão inicial de conhecimento.
- Semana 2: workflows por canal (WhatsApp para vendas, Instagram comentário
  → DM) e telemetria integrada.
- Semana 3: testes ponta a ponta por canal, treinamento da operação e handoff
  humano (HIL).
- Semana 4: piloto WhatsApp + Instagram com métricas de resposta e conversão.
- Semana 5: ajustes cruzados (handoff Instagram → WhatsApp, histórico
  unificado), dashboards e alertas.
- Semana 6: go-live multicanal, hiper-care e transição para operação.

## Checklists práticos
### Documentos e dados que o cliente pode enviar
- FAQ oficial (PDF ou doc) e políticas (troca, devolução, prazos).
- Catálogo ou lista de produtos/serviços com descrição, preço e estoque.
- Scripts de atendimento já usados pela equipe.
- Templates de mensagem aprovados (WhatsApp) ou modelos desejados.
- Links de site institucional ou base de conhecimento pública permitida.

### Acessos e credenciais
- WhatsApp: número que será usado, e o BSP (fornecedor oficial do WhatsApp
  Business, ou seja, a empresa por onde o número é registrado e de onde as
  mensagens são enviadas). Informar também se existem modelos de mensagem
  já aprovados para disparos iniciados pela empresa.
- Instagram: Page ID, Instagram Business ID, App ID/secret, token de página,
  URL de callback pública, verify token.
- Telemetria (opcional): dados de conexão se o cliente quiser usar seu
  próprio banco para interaction_runs.

### Checklist detalhado por canal (campos do onboarding)
#### WhatsApp (tela de onboarding)
- Access Key (X-API-Key): chave do tenant para autorizar o provisionamento.
- Canal WhatsApp: escolha do canal/cliente já cadastrado no tenant.
- Telefone (DDD + número) ou formato +E.164 para importação.
- phone_number_id (quando o número já existe na Meta): aparece no
  Gerenciador Meta e no BSP; identifica o número na Graph API.
- Código SMS de verificação (6 dígitos) enviado ao número na ativação.
- BSP/fornecedor: quem opera o número (Twilio, Infobip, Gupshup, Zenvia,
  etc.), pois é quem libera tokens/credenciais e mostra status dos modelos.
- Modelos aprovados: lista de textos prontos aprovados pelo WhatsApp para
  envios iniciados pela empresa (nome, idioma, categoria), úteis para
  notificações.
- Webhook atual (se houver) e callback desejado: usado para assumir ou
  remover callbacks antigos; precisa da URL pública e do verify token.

#### Instagram (tela de onboarding)
- X-API-Key: chave do tenant para autorizar o provisionamento.
- Instagram Business Account ID e Facebook Page ID.
- Nome comercial e e-mail de contato.
- (Opcional) Channel ID para referência interna.
- Page Access Token com scopes instagram_manage_messages,
  instagram_manage_comments, pages_manage_metadata.
- App ID e App Secret da aplicação Meta.
- Callback URL pública (HTTPS) para receber os webhooks.
- Verify Token (palavra-chave escolhida para validar o webhook).
- Graph API Version (ex.: v20.0).

## Go-live e pós-go-live
- Revisar tokens e validade antes do go-live.
- Treinar a operação para handoff humano e escalonamento.
- Hiper-care nas 1–2 primeiras semanas: acompanhar métricas simples
  (resposta, conversão, erros) e ajustar YAML/prompts.

## Entregáveis esperados
- YAMLs de canal/workflow validados e versionados.
- Telemetria ativa (interaction_runs) e acesso a dashboards básicos.
- Checklist de go-live assinado e runbook simples de operação (como acionar
  humano, como registrar issues, como rotacionar chaves).

## Riscos e mitigação
- Credenciais ou escopos faltando: uso do checklist e teste guiado de canal.
- Dados insuficientes: começar com FAQ e políticas; depois catálogo.
- Janela de DM expirada (Instagram): responder no comentário e pedir nova
  interação.
- Volume alto inesperado: ativar fila/backoff e revisar limites de envio.

## Próximos passos sugeridos
1. Enviar o checklist de cliente e agendar discovery.
2. Confirmar pacote (WA, IG ou Multicanal) e ativos disponíveis.
3. Receber documentos base (FAQ, políticas, catálogo) e credenciais de canal.
4. Publicar YAML inicial, configurar telemetria e marcar teste guiado.
