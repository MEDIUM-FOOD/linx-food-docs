# Manual técnico, executivo, comercial e estratégico: sistema de log, arquitetura de log, características e correlation_id

## 1. O que é esta feature

O sistema de log desta plataforma não é apenas um lugar onde mensagens são escritas. Ele é uma infraestrutura transversal de rastreabilidade que conecta quatro preocupações diferentes.

1. Identidade lógica da execução.
2. Emissão estruturada de eventos durante o runtime.
3. Persistência local e remota dos registros.
4. Recuperação administrativa dos logs para diagnóstico posterior.

O conceito central que amarra essas quatro partes é o correlation_id. No código atual, esse identificador não é um detalhe decorativo. Ele é a chave lógica da execução. Ele entra na borda HTTP, volta para o cliente, alimenta o logger do runtime, influencia nomes de arquivos de log dedicados e também guia a leitura administrativa posterior.

Isso significa que “logging” no projeto não é só escrita de linha em arquivo. É a capacidade de contar a história correta de uma execução específica, mesmo quando essa execução atravessa API, worker, scheduler e provider remoto.

## 2. Que problema ela resolve

Sem essa arquitetura, a plataforma sofreria com três problemas clássicos.

O primeiro é o problema de identidade. Um erro poderia aparecer no processo HTTP e continuar no worker, mas a operação não teria uma chave única para provar que aqueles eventos pertencem à mesma história.

O segundo é o problema de localização. Cada ambiente poderia guardar logs de forma diferente, e o time acabaria investigando por tentativa e erro, varrendo arquivos ou dashboards sem critério.

O terceiro é o problema de governança. Cada parte do sistema poderia criar seu próprio logger, seu próprio path ou sua própria regra de leitura, o que destruiria previsibilidade operacional.

O desenho atual resolve isso separando o que normalmente fica misturado.

1. A API define e devolve o correlation_id.
2. O runtime usa um logger canônico e estruturado.
3. Os destinos de escrita são configuráveis, mas governados.
4. A leitura administrativa passa por um provider canônico com contrato único.

## 3. Visão executiva

Para liderança, essa feature reduz risco operacional e melhora tempo de resposta a incidente.

O ganho executivo não está em “gerar mais logs”. O ganho está em conseguir responder perguntas operacionais com base em evidência.

- Qual execução falhou.
- Onde ela começou.
- Em qual processo ela continuou.
- Qual provider de logs precisa ser consultado.
- Como reconstruir a trilha sem varredura manual cega.

Isso melhora previsibilidade de suporte, auditoria técnica e governança de produção.

## 4. Visão comercial

Comercialmente, essa arquitetura sustenta uma capacidade importante para clientes corporativos: a plataforma não opera como caixa-preta. O sistema foi desenhado para devolver um identificador rastreável ao cliente e para permitir investigação posterior em diferentes ambientes, sem depender de acesso artesanal ao servidor.

Isso ajuda especialmente em cenários com exigência de auditoria, suporte de produção, integração corporativa e ambientes distribuídos.

O valor vendável real, confirmado no código, é este.

- o runtime devolve correlation_id ao cliente;
- o mesmo ID entra nos logs do processo;
- a leitura administrativa é resolvida por provider canônico;
- fora de development, não existe fallback implícito escondendo erro de configuração.

## 5. Visão estratégica

Estratégicamente, essa feature fortalece a plataforma porque transforma observabilidade em contrato arquitetural, não em convenção frágil.

O sistema reforça quatro princípios importantes.

1. O correlation_id é a identidade lógica da execução.
2. O logger canônico é a única porta correta para emissão estruturada.
3. A escrita de logs e a leitura administrativa são responsabilidades distintas.
4. O provider ativo fora de development precisa ser explícito e suportado.

Esse desenho prepara a plataforma para crescer sem criar múltiplos fluxos paralelos de observabilidade.

## 6. Conceitos necessários para entender

### Correlation ID

Correlation ID é o identificador lógico único que permite responder “qual execução estou investigando?”. No projeto, ele é gerado pela fábrica central quando não vem da borda HTTP e é normalizado para um formato padrão com timestamp mais UUID.

### Logger canônico

É a função central que cria loggers estruturados para toda a aplicação. O projeto trata isso como regra arquitetural, não como conveniência. A base de componentes correlacionados proíbe recorrer a logging.getLogger local como atalho.

### Log estruturado

Log estruturado significa que o evento é serializado como JSON com campos como timestamp, level, logger e contexto adicional, em vez de depender apenas de texto livre.

### Arquivo dedicado por correlação

Quando a configuração permite e o correlation_id é elegível, cada execução ganha um arquivo específico. Isso facilita diagnóstico por caso e reduz dependência do arquivo compartilhado principal.

### Sidecar e manifest

O sistema grava um sidecar por correlação e um manifest append-only com o vínculo entre correlation_id, metadata_filename e log_filename. Isso existe para localizar a família de logs sem varredura cega do diretório.

### Provider canônico de leitura

É a camada administrativa que sabe de onde os logs devem ser lidos. O runtime que escreve não precisa saber se a consulta posterior virá do filesystem, do AWS CloudWatch, da Northflank ou do Azure Log Analytics.

## 7. Como a feature funciona por dentro

O fluxo começa na camada HTTP. O middleware de request resolve o correlation_id usando esta ordem.

1. request.state.correlation_id, se já existir.
2. header X-Correlation-Id.
3. geração de novo ID pela fábrica central.

Depois disso, o valor efetivo é normalizado, gravado no state do request, registrado no contexto local do request e usado para criar o logger da requisição.

Na resposta, o sistema devolve X-Correlation-Id e, quando a resposta é JSON compatível, injeta correlationId no corpo sem sobrescrever um campo já existente.

Durante o runtime, o logger escreve eventos estruturados no arquivo principal, no console, no arquivo dedicado por correlação e, quando habilitado no fluxo correto, no CloudWatch. Em paralelo, sidecar e manifest registram metadados que permitem reencontrar o artefato físico daquela correlação.

Quando a operação precisa ler logs administrativamente, outra metade da arquitetura entra em ação. A camada de provider resolve a origem ativa do ambiente. Em development, o provider é forçado para filesystem. Fora de development, o tipo precisa ser explicitamente configurado como filesystem, aws_cloudwatch, northflank ou azure. Se isso não acontecer, o código falha fechado.

## 8. Divisão em etapas ou submódulos

### 8.1 Identidade lógica da execução

Essa etapa existe para garantir que toda execução importante tenha uma identidade rastreável. Ela resolve, normaliza e propaga o correlation_id.

### 8.2 Enriquecimento e sanitização do evento

Essa etapa mistura contexto do request com o payload do evento, achata campos adicionais e mascara informações sensíveis antes da serialização.

### 8.3 Escrita operacional do runtime

Essa etapa grava a trilha real da execução em destinos configurados, como arquivo principal, console, arquivo por correlação e CloudWatch.

### 8.4 Metadados de lookup

Essa etapa grava sidecar e manifest para ligar identidade lógica e localização física do log.

### 8.5 Resolução administrativa do provider

Essa etapa decide de onde os logs serão lidos para investigação posterior, sem bypass local fora do fluxo oficial.

### 8.6 Materialização e análise

Quando o provider é remoto, essa etapa materializa o conteúdo em artefato local temporário. Depois disso, o leitor canônico trata filesystem e origem remota com o mesmo contrato downstream.

## 9. Pipeline ou fluxo principal

1. Uma requisição entra na API.
2. O middleware resolve ou gera o correlation_id.
3. O request recebe logger correlacionado.
4. Os eventos são emitidos como JSON estruturado.
5. O runtime grava arquivo compartilhado, console e, quando elegível, arquivo dedicado por correlação.
6. Sidecar e manifest registram o vínculo entre correlação e arquivo físico.
7. Quando alguém consulta /logs, o sistema resolve o provider administrativo ativo.
8. Se a origem for remota, os eventos são materializados localmente.
9. O leitor canônico organiza a família de logs e entrega o material ao pipeline de análise.

## 10. Decisões técnicas e trade-offs

### Separar escrita de leitura administrativa

Ganho: reduz acoplamento e permite trocar a origem de leitura sem reescrever o runtime que emite logs.

Custo: exige duas camadas e uma etapa de materialização quando a origem é remota.

### Falhar fechado fora de development

Ganho: impede que produção esconda erro de configuração atrás de fallback implícito.

Custo: configuração errada quebra cedo.

### Adiar CloudWatch até o ciclo do worker

Ganho: evita anexação remota prematura no bootstrap genérico da API e torna explícito quando o handler realmente ficou ativo naquele processo.

Custo: exige que a equipe entenda que “CloudWatch habilitado” não significa “handler imediatamente anexado em qualquer ponto do processo”.

### Usar sidecar e manifest

Ganho: lookup confiável por correlação sem depender de listar cegamente o diretório inteiro.

Custo: aumenta o número de artefatos operacionais sob gestão.

## 11. Configurações que mudam o comportamento

As configurações mais importantes confirmadas no código são estas.

- enable_file_logging
- enable_console_logging
- log_file_path
- log_output_directory
- log_file_rotation_max_bytes
- log_file_rotation_backup_count
- enable_correlation_file_logging
- log_correlation_directory
- enable_cloudwatch_logging
- cloudwatch_log_group
- cloudwatch_log_stream_prefix
- cloudwatch_region_name
- environment
- LOG_PROVIDER_TYPE

## 12. O que acontece em caso de sucesso

No caminho feliz, a API devolve o correlation_id ao cliente, o runtime emite eventos estruturados, os arquivos correlacionados e seus metadados são criados quando aplicável, e a camada administrativa consegue reencontrar a família certa de logs usando o provider canônico.

## 13. O que acontece em caso de erro

Os erros mais importantes confirmados no código são estes.

- Provider de logs inválido fora de development gera erro explícito.
- Filesystem provider sem selector válido é rejeitado.
- CloudWatch provider sem log_group ou sem filtros mínimos falha explicitamente.
- Northflank sem token, project_id ou workload alvo falha explicitamente.
- Azure sem workspace_id, query ou credenciais falha explicitamente.

O ponto importante é que a arquitetura não tenta “adivinhar outro provider” para mascarar a configuração errada.

## 14. Observabilidade e diagnóstico

Para investigar corretamente, o primeiro passo não é procurar um arquivo qualquer. O primeiro passo é identificar o correlation_id da execução.

Depois disso, a ordem correta é esta.

1. Confirmar se a resposta devolveu X-Correlation-Id.
2. Confirmar se o ambiente estava em development ou não.
3. Confirmar qual provider administrativo estava ativo.
4. Confirmar se a correlação gerou arquivo dedicado ou só trilha no arquivo principal.
5. Confirmar se CloudWatch foi de fato ativado no worker daquele processo.

## 15. Impacto técnico

Tecnicamente, essa feature reforça a arquitetura hexagonal e o isolamento do domínio operacional. O runtime escreve logs por uma abstração canônica e a leitura administrativa usa outra abstração canônica, o que reduz acoplamento e torna o comportamento mais observável.

## 16. Impacto executivo

Executivamente, a feature melhora suporte, auditoria e previsibilidade de operação. Ela reduz tempo perdido com busca manual e diminui risco de investigar o log errado.

## 17. Impacto comercial

Comercialmente, a arquitetura permite afirmar com segurança que a plataforma entrega rastreabilidade operacional séria, com identidade de execução, persistência estruturada e investigação administrativa governada.

## 18. Impacto estratégico

Estrategicamente, essa base torna a observabilidade um contrato de produto. Isso sustenta futuras capacidades de automação de suporte, análise automática de incidentes e governança multiambiente.

## 19. Exemplos práticos guiados

### Exemplo 1: incidente em requisição HTTP

Cenário: o cliente recebe erro 500.

Entrada: a resposta devolve X-Correlation-Id.

Processamento: o time usa esse ID para localizar a trilha da execução no provider ativo.

Saída: a investigação não depende de adivinhar qual arquivo ou stream contém o caso certo.

### Exemplo 2: análise administrativa em provider remoto

Cenário: o ambiente usa Northflank ou CloudWatch.

Processamento: o provider canônico consulta a origem remota, materializa um artefato local temporário e entrega esse material ao mesmo pipeline de análise usado para filesystem.

Saída: a camada de diagnóstico não precisa conhecer cada API remota em detalhes.

## 20. Explicação 101

Pense no sistema de logs como um prontuário por execução. O correlation_id é o número do prontuário. O logger canônico é a recepção que registra tudo no formato certo. Os arquivos e providers são os lugares onde esse prontuário fica guardado. E a camada administrativa é o balcão que sabe exatamente onde buscar esse prontuário depois, sem depender de adivinhação.

## 21. Limites e pegadinhas

- Logging habilitado não significa que todo ambiente usa o mesmo provider de leitura.
- CloudWatch habilitado não significa handler anexado imediatamente em qualquer etapa do processo.
- Arquivo dedicado por correlação só existe quando o ID é elegível e a configuração permite.
- Fora de development, não existe fallback implícito de provider.

## 22. Troubleshooting

### Sintoma: a API respondeu, mas ninguém acha o log certo

Causa provável: o time não está usando o correlation_id devolvido na resposta ou está consultando o provider errado para o ambiente.

### Sintoma: o provider administrativo quebra em produção

Causa provável: LOG_PROVIDER_TYPE ausente ou inválido fora de development.

### Sintoma: CloudWatch não mostra eventos esperados

Causa provável: o handler não foi ativado naquele worker, ou o ambiente não está com os parâmetros mínimos do CloudWatch coerentes.

## 23. Checklist de entendimento

- Entendi o papel do correlation_id.
- Entendi a diferença entre escrita de runtime e leitura administrativa.
- Entendi por que existe provider canônico.
- Entendi por que CloudWatch é adiado até o worker.
- Entendi por que fora de development o provider precisa ser explícito.

## 24. Evidências no código

- src/core/logging_system.py
  - Motivo da leitura: confirmar logger canônico, handlers, CloudWatch e arquivo por correlação.
  - Símbolos relevantes: create_logger_with_correlation, defer_cloudwatch_until_worker, activate_cloudwatch_for_worker.
  - Comportamento confirmado: logging estruturado, arquivo dedicado por correlação e ativação tardia do CloudWatch.

- src/core/correlation_id_factory.py
  - Motivo da leitura: confirmar formato do correlation_id.
  - Símbolos relevantes: CorrelationIdFactory.generate, CorrelationIdFactory.is_valid.
  - Comportamento confirmado: formato YYYYMMDD_HHMMSS mais UUID.

- src/core/log_origin_metadata.py
  - Motivo da leitura: confirmar sidecar, manifest e contexto local do request.
  - Símbolos relevantes: write_correlation_origin_sidecar, resolve_log_filename_from_manifest, set_request_correlation_id.
  - Comportamento confirmado: metadados de correlação persistidos e contexto local de request.

- src/api/service_api.py
  - Motivo da leitura: confirmar borda HTTP, header X-Correlation-Id e injeção em JSON.
  - Símbolos relevantes: log_requests, \_inject_correlation_in_json_body.
  - Comportamento confirmado: correlation_id nasce ou é reaproveitado na API e volta na resposta.

- src/api/services/log_provider_service.py
  - Motivo da leitura: confirmar provider canônico e regra de resolução por ambiente.
  - Símbolos relevantes: resolve_active_log_provider_type, build_log_provider.
  - Comportamento confirmado: filesystem forçado em development e provider explícito fora de development.
