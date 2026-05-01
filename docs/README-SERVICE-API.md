# Boundary HTTP da Plataforma

Este manual explica a API como fronteira operacional do produto. O objetivo não é repetir Swagger. O objetivo é mostrar o que o processo HTTP realmente faz antes, durante e depois de entregar uma resposta ao cliente.

## 1. Visão geral

O processo HTTP é a porta de entrada do sistema. Ele recebe a request, resolve identidade, aplica permissão, injeta correlation_id, escolhe se a resposta pode sair no mesmo request ou se precisa continuar de forma assíncrona e publica tanto a API JSON quanto a UI estática servida em /ui/static.

Em termos práticos, a API é o lugar onde o sistema deixa de ser intenção e vira operação observável. Se a borda está ruim, o resto do runtime herda contexto incompleto, autenticação inconsistente ou rota errada.

## 2. O problema que este boundary resolve

Sem uma borda HTTP organizada, a plataforma cairia em um padrão perigoso: cada domínio resolveria autenticação, YAML, status, stream e contratos de resposta do seu próprio jeito. Isso parece rápido no início, mas destrói rastreabilidade e faz o mesmo tipo de erro reaparecer em vários pontos.

O boundary existe para evitar isso. Ele centraliza o que toda request precisa saber antes de entrar em ingestão, RAG, workflows, assembly AST, canais, autenticação federada ou operação administrativa.

## 3. Conceitos necessários para entender o módulo

Boundary HTTP é a camada que protege o resto do sistema de detalhes de protocolo, cabeçalhos, sessão e formato de request.

Aceite assíncrono significa receber um pedido agora e continuar o trabalho depois, normalmente devolvendo task_id e uma superfície de acompanhamento.

SSE, ou Server-Sent Events, é um canal de streaming unidirecional usado aqui para acompanhar evolução de jobs. Um exemplo importante é /status/stream/{task_id}.

Mount de UI significa publicar arquivos estáticos dentro do mesmo processo HTTP. Aqui isso aparece no serving da UI administrativa em /ui/static.

## 4. Como o boundary funciona por dentro

A request entra no app FastAPI montado em service_api. Antes de qualquer regra de domínio, o processo aplica middlewares de sessão federada, rate limit, correlation_id, logging de request e handlers globais de erro. Depois disso, a request cai no router do domínio certo.

Se o fluxo exigir YAML, a API passa pelo resolvedor central de configuração. Se o fluxo exigir permissão, a borda verifica a capacidade declarada. Se o fluxo for longo, a API não finge conclusão. Ela aceita o pedido, devolve contrato de acompanhamento e deixa a continuidade para worker ou runtime assíncrono.

Esse mesmo processo HTTP também publica a UI administrativa. A interface que consome o assembly AST está em app/ui/static/js/admin-assembly-ast.js, mas ela não fala com um backend separado. Ela conversa com a mesma API que recebe as requests operacionais do produto.

## 5. Pipeline principal do boundary

1. Entrada da request.
2. Resolução de autenticação, permissão e correlation_id.
3. Enriquecimento de contexto, incluindo YAML quando necessário.
4. Encaminhamento ao router e service corretos.
5. Resposta curta no mesmo request ou aceite assíncrono com task_id.
6. Exposição de canais de acompanhamento, como /status/stream/{task_id}.

O valor desse pipeline é previsibilidade. Em vez de cada domínio improvisar sua própria superfície, todos passam pela mesma porta de entrada.

## 6. Decisões técnicas importantes

A primeira decisão importante é manter API JSON e UI administrativa no mesmo app. Isso reduz divergência entre contrato e tela, porque a interface que o operador usa bebe da mesma superfície HTTP que o backend já expõe.

A segunda decisão é publicar o slice agentic por feature flag. Quando FEATURE_AGENTIC_AST_ENABLED está ativa, o app monta endpoints como /config/assembly/draft, /config/assembly/objective-to-yaml, /config/assembly/validate, /config/assembly/confirm, /config/assembly/catalog e /config/assembly/schema. Quando a flag está desligada, o app não deveria fingir suporte parcial.

A terceira decisão é tratar acompanhamento assíncrono como parte do boundary, não como apêndice. Por isso status, SSE e WebSocket vivem na borda pública, e não escondidos em outro processo.

## 7. O que acontece em caso de sucesso

No caminho feliz, a API recebe a request, valida contexto, chama o domínio certo e devolve uma resposta coerente com o tipo de trabalho. Para operações curtas, isso significa retorno imediato. Para operações longas, significa aceite com task_id, endpoints de status e stream aberto para o operador acompanhar.

O sucesso operacional não é só HTTP 200 ou 202. É conseguir explicar qual rota respondeu, qual contexto entrou, qual correlation_id costurou a execução e qual canal oficial de acompanhamento o cliente deve usar.

## 8. O que acontece em caso de erro

Quando o erro acontece antes do aceite, normalmente ele está em autenticação, permissão, payload, rate limit ou resolução de YAML. Quando o erro acontece depois do aceite, a suspeita principal sai do boundary e entra no worker, na fila ou no domínio assíncrono.

No slice de assembly AST, um 404 pode significar ausência de feature e não ausência de código, porque a publicação depende da flag. Em situações assim, diagnosticar pela rota apenas não basta; é preciso conferir configuração de ambiente.

## 9. Observabilidade e diagnóstico

A melhor forma de investigar a borda HTTP é seguir esta ordem.

1. Confirmar se a request entrou com correlation_id válido.
2. Confirmar se a autenticação e a permissão foram resolvidas.
3. Confirmar se o fluxo exigia YAML e se ele foi resolvido.
4. Confirmar se a resposta era curta ou assíncrona.
5. Em fluxo assíncrono, seguir por task_id e /status/stream/{task_id}.

Em linguagem simples: descubra primeiro se a falha é de entrada. Só depois discuta domínio, fila ou worker.

## 10. Exemplo prático guiado

Imagine uma operação agentic assistida pela UI administrativa. O operador usa a tela publicada em /ui/static. Essa interface, apoiada por app/ui/static/js/admin-assembly-ast.js, chama endpoints como /config/assembly/draft e /config/assembly/objective-to-yaml. A API recebe a intenção, passa pelo fluxo de autenticação, aplica a feature flag do assembly, resolve o contrato e devolve a resposta governada. Se a operação for só de autoria, a resposta sai ali mesmo. Se a operação seguinte envolver execução longa, o boundary muda de papel e entrega task_id para acompanhamento.

## 11. Explicação 101

Pense no boundary HTTP como a recepção de um hospital. A recepção não faz a cirurgia, mas decide se o paciente pode entrar, coleta identificação, abre a ficha, encaminha para a área certa e diz como acompanhar o caso. Se a recepção estiver desorganizada, até uma equipe clínica excelente vai trabalhar com informação errada.

## 12. Evidências no código

- [src/api/service_api.py](../src/api/service_api.py): montagem do app FastAPI, middlewares, routers e publicação da UI.
- [src/api/routers/config_resolution.py](../src/api/routers/config_resolution.py): resolução central do YAML para fluxos HTTP.
- [src/api/routers/config_assembly_router.py](../src/api/routers/config_assembly_router.py): publicação dos endpoints agentic governados, como /config/assembly/draft e /config/assembly/objective-to-yaml.
- [src/api/routers/streaming_router.py](../src/api/routers/streaming_router.py): canais de acompanhamento como /status/stream/{task_id}.
- [app/ui/static/js/admin-assembly-ast.js](../app/ui/static/js/admin-assembly-ast.js): UI real que consome a API de assembly AST.
- [app/runners/api_runner.py](../app/runners/api_runner.py): bootstrap operacional do processo HTTP.
