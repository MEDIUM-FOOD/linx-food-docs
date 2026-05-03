# Manual técnico por etapa: runtime compartilhado do frontend AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o runtime JavaScript puro compartilhado entre as páginas AG-UI do repositório. Ele reúne o cliente SSE via POST, o store de estado local e o sidecar reutilizável de chat e HIL.

## 2. Cliente SSE via POST

O cliente compartilhado abre AG-UI com fetch POST, não com EventSource tradicional. Isso existe porque o contrato do run precisa enviar corpo JSON tipado.

Comportamentos confirmados:

- resolve endpoint absoluto a partir do contexto atual
- monta headers Content-Type e Accept para text event stream
- injeta X-API-Key quando a tela fornece essa credencial
- lê X-Correlation-Id do header de resposta
- faz parse incremental de blocos SSE em texto
- suporta retentativa explícita controlada, mas não cria correlation id local por conta própria

## 3. Store de estado compartilhado

O store local mantém um retrato completo da sessão AG-UI no browser:

- run
- messages
- tools
- state
- activities
- steps
- interrupts
- rawEvents
- customEvents
- lastEvent

Ele também aplica JSON Patch de forma compatível com o backend, inclusive move, copy e test. Isso permite que páginas estáticas complexas acompanhem snapshots e deltas sem framework reativo pesado.

## 4. Sidecar reutilizável

O sidecar encapsula um padrão pronto de interação:

- mostra status e correlation id
- renderiza mensagens e timeline de tools
- exibe contexto serializado da tela atual
- integra interrupções HIL usando o painel compartilhado
- monta payload de resume AG-UI no mesmo endpoint de run

Na prática, ele é a ponte entre o protocolo AG-UI e uma experiência administrativa reutilizável em páginas HTML estáticas.

## 5. Onde esta etapa costuma falhar

Os problemas mais comuns são:

- falha HTTP na abertura do stream
- parsing SSE quebrado por bloco malformado
- aplicação inválida de JSON Patch no state store
- ausência de HilContract para montar resume
- UI sem apiKeyProvider quando a superfície precisa de X-API-Key explícito

## 6. Diagnóstico recomendado

Para investigar o runtime frontend:

1. confirme se o cliente recebeu X-Correlation-Id
2. cheque se os blocos SSE fecham corretamente com linha em branco
3. valide se os eventos emitidos batem com o modelo AG-UI
4. confira se o store local conseguiu aplicar snapshots e deltas
5. em HIL, garanta que o contrato de resume exista antes de clicar em aprovar ou rejeitar

## 7. Evidências no código

- app/ui/static/js/shared/ag-ui-client.js
  - Motivo: cliente SSE compartilhado.
  - Comportamento confirmado: AG-UI usa POST com corpo JSON e parse incremental de SSE.
- app/ui/static/js/shared/ag-ui-state-store.js
  - Motivo: reconstrução do estado local.
  - Comportamento confirmado: o store guarda mensagens, tools, interrupts, snapshots e deltas compatíveis com o backend.
- app/ui/static/js/shared/ag-ui-sidecar-chat.js
  - Motivo: UI compartilhada de chat e HIL.
  - Comportamento confirmado: o sidecar reutiliza o runtime comum e monta resume AG-UI no mesmo endpoint.
