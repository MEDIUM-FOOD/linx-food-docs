# Manual técnico e operacional: sistema de log, arquitetura de log, características e correlation_id

## 1. Escopo técnico deste manual

Este manual descreve o comportamento técnico real do sistema de logging no código atual. O foco é mostrar como o correlation_id é criado ou reaproveitado, como o logger canônico escreve eventos estruturados, quando surgem arquivo dedicado, sidecar e manifest, como o CloudWatch é ativado no worker e como a camada administrativa lê logs via provider canônico.

## 2. Entry points reais

Os pontos mais importantes do sistema confirmados no código são estes.

1. O middleware HTTP de request em src/api/service_api.py.
2. A fábrica create_logger_with_correlation em src/core/logging_system.py.
3. A base arquitetural BaseCorrelationComponent em src/core/base_correlation_component.py.
4. O resolvedor de provider administrativo em src/api/services/log_provider_service.py.
5. O leitor canônico em src/api/services/canonical_log_reader.py.
6. Os endpoints administrativos em src/api/routers/logs_router.py e o serviço compartilhado em src/api/services/logs_admin_service.py.

## 3. Contrato do correlation_id

### 3.1 Formato

CorrelationIdFactory.generate produz um identificador no formato YYYYMMDD_HHMMSS mais UUID. CorrelationIdFactory.is_valid valida exatamente esse contrato.

### 3.2 Normalização

normalize_correlation_id preserva o valor quando ele já é válido. Se não for, gera um novo ID no formato canônico. Isso evita que componentes internos operem com identificadores arbitrários ou inconsistentes.

### 3.3 Propagação em componentes

BaseCorrelationComponent impõe um contrato rígido.

- yaml_config é obrigatório;
- user_session é obrigatório;
- user_session.correlation_id é obrigatório;
- user_session.user_email é obrigatório;
- o logger do componente é sempre criado via create_logger_with_correlation.

Isso significa que, fora da borda HTTP, o ecossistema da aplicação espera receber correlation_id já embutido no yaml_config do fluxo.

## 4. Ciclo HTTP do correlation_id

O middleware log_requests segue a ordem abaixo.

1. Se a rota for de status assíncrono, usa um correlation_id compartilhado específico.
2. Caso contrário, tenta request.state.correlation_id.
3. Se não houver state, tenta o header x-correlation-id.
4. Se continuar vazio, gera um novo correlation_id.
5. O valor final é normalizado ou gerado no formato canônico.
6. request.state.correlation_id recebe o valor.
7. set_request_origin e set_request_correlation_id publicam o contexto do request.
8. O logger da requisição é criado com esse mesmo ID.
9. A resposta recebe X-Correlation-Id.
10. Se a resposta for JSON e ainda não trouxer correlationId ou correlation_id, o middleware injeta correlationId no body.

Esse ponto é central porque mostra que o correlation_id não é só metadado interno. Ele é parte do contrato HTTP.

## 5. Enriquecimento, sanitização e serialização do evento

O logging_system usa pipeline de structlog e formatter próprio do logging padrão.

### 5.1 Enriquecimento

_resolve_runtime_log_context combina campos do evento com contextvars do request para preencher, quando disponíveis:

- correlation_id
- run_id
- parent_run_id
- child_run_id

### 5.2 Flatten de extra

_flatten_logging_kwargs achata extra recebido no estilo logging padrão para que o payload final não fique com um dicionário aninhado desnecessário.

### 5.3 Sanitização

sanitize_structlog_event percorre a estrutura e mascara campos sensíveis, como tokens, segredos, credenciais e chaves, preservando o formato geral do evento.

### 5.4 Saída JSON

_StructuredLogFormatter converte LogRecord em payload JSON estruturado, inclusive quando o emissor usa logging padrão em vez do binding completo do structlog.

## 6. Destinos de escrita do runtime

### 6.1 Arquivo principal rotacionado

Quando enable_file_logging está ligado, _build_file_handler cria RotatingFileHandler usando:

- log_file_path
- log_output_directory, quando definido
- log_file_rotation_max_bytes
- log_file_rotation_backup_count

Esse arquivo é a trilha contínua principal do processo.

### 6.2 Console

Quando enable_console_logging está ligado, _build_console_handler cria StreamHandler para stdout com o mesmo formatter JSON.

### 6.3 Arquivo dedicado por correlação

create_logger_with_correlation cria arquivo dedicado apenas quando duas condições são verdadeiras.

1. enable_correlation_file_logging está habilitado.
2. O correlation_id é elegível, ou seja, válido no formato canônico.

Quando isso acontece, o runtime:

- calcula nome determinístico com create_correlation_log_filename;
- resolve o diretório a partir de log_correlation_directory e log_output_directory;
- cria FileHandler específico;
- aplica _CorrelationFileIsolationFilter para bloquear eventos de outra correlação;
- cria logger específico sem propagar para o root em ambiente normal.

### 6.4 Logger compartilhado

Se o ID não for elegível para arquivo dedicado, o sistema cria logger compartilhado com nome lógico derivado do identificador informado. Isso é usado para componentes de infraestrutura e identificadores estáticos.

## 7. Sidecar, manifest e lookup físico

Quando create_logger_with_correlation gera arquivo dedicado, ele chama write_correlation_origin_sidecar e registra metadados da correlação.

Os artefatos relevantes são estes.

- arquivo sidecar por correlação em _meta;
- correlation_manifest.jsonl append-only com correlation_id, metadata_filename e log_filename.

resolve_log_filename_from_manifest permite reconstruir o nome do log físico a partir do correlation_id, evitando lookup por listagem cega do diretório.

## 8. CloudWatch na escrita do runtime

### 8.1 Regra de adiamento

O runner HTTP chama defer_cloudwatch_until_worker antes de entregar o controle ao processo da API. Isso significa que o handler CloudWatch não deve ser tratado como anexado imediatamente no bootstrap genérico.

### 8.2 Ativação efetiva

No lifespan da aplicação, o código chama activate_cloudwatch_for_worker. Essa função:

- verifica se enable_cloudwatch_logging está ativo;
- libera o adiamento;
- garante setup_logging se necessário;
- obtém o handler CloudWatch;
- anexa o handler ao root se ainda não estiver presente;
- registra o stream ativo no estado da aplicação.

O próprio lifespan grava em application.state.cloudwatch_stream_name e application.state.cloudwatch_handler_ativo se o CloudWatch ficou ativo naquele worker.

### 8.3 Implicação prática

Documentar isso corretamente é importante porque “CloudWatch habilitado” e “CloudWatch efetivamente ativo neste worker” não são a mesma coisa.

## 9. Faulthandler

enable_canonical_faulthandler_logging ativa dumps de faulthandler no diretório canônico de logs. O arquivo segue o padrão faulthandler_pid.log. Isso serve para diagnóstico de falhas de baixo nível e não substitui o log de negócio.

## 10. Provider canônico de leitura

### 10.1 Regra de resolução

resolve_active_log_provider_type segue esta regra exata.

- Se environment for development, o provider é sempre filesystem.
- Fora de development, LOG_PROVIDER_TYPE precisa ser explicitamente um dos valores suportados.
- Os valores suportados no código lido são filesystem, northflank, aws_cloudwatch e azure.
- Valor ausente ou inválido fora de development gera RuntimeError.

Isso confirma que não existe fallback implícito de provider fora de development.

### 10.2 Providers concretos

build_log_provider instancia uma destas classes.

- FileSystemProvider
- AWSCloudWatchProvider
- NorthflankProvider
- AzureLogProvider

## 11. Comportamento dos providers

### 11.1 FileSystemProvider

O provider local usa CanonicalLogReader e exige pelo menos um seletor:

- correlation_id
- log_name
- all_logs igual a true

Sem isso, a análise local falha com erro explícito.

### 11.2 AWSCloudWatchProvider

Exige log_group e pelo menos um seletor útil entre correlation_id, filter_pattern, stream_prefix ou janela temporal. O resultado é materializado em arquivo local temporário para consumo downstream.

### 11.3 NorthflankProvider

Exige project_id, token e ao menos um workload alvo. A resposta remota é convertida em linhas prefixadas pelo workload antes da materialização local.

### 11.4 AzureLogProvider

Exige workspace_id, query e credenciais de acesso para gerar token. O retorno remoto também é convertido e materializado localmente.

## 12. Leitor canônico e família de logs

CanonicalLogReader é a abstração única para leitura e lookup de logs locais materializados. Suas responsabilidades técnicas incluem:

- resolver diretório real de logs;
- validar existência de arquivo ou diretório via provider;
- listar caminhos por padrão controlado;
- detectar rotação;
- inferir papel operacional de cada arquivo como api, worker, scheduler ou correlacionado;
- ordenar a família de arquivos de forma estável.

Esse leitor é importante porque a investigação deixa de tratar log como arquivo isolado e passa a tratar log como família operacional da mesma correlação.

## 13. Boundary administrativo

O boundary HTTP de logs delega para serviços compartilhados, não para leitura ad hoc de filesystem.

analyze_logs_via_admin_provider faz o fluxo principal.

1. Resolve user_email efetivo.
2. Constrói o provider canônico.
3. Executa prepare_logs no provider concreto.
4. Recebe logs_dir e cleanup_paths do provider.
5. Monta yaml mínimo para análise in-process.
6. Executa AnalyzeLogsCommand com o contrato local unificado.
7. Limpa artefatos temporários no bloco final.

Isso confirma que a consulta administrativa sempre passa pelo provider oficial e pelo mesmo pipeline de análise downstream, independentemente da origem.

## 14. Configurações que mais mudam comportamento

Estas são as configurações que mais alteram o comportamento observado.

- log_level
- log_format
- enable_console_logging
- enable_file_logging
- log_output_directory
- log_file_path
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
- log_provider_http_timeout_seconds
- log_provider_retry_attempts
- log_provider_retry_wait_seconds

## 15. Pipeline técnico de ponta a ponta

1. Request entra na API.
2. Middleware resolve ou gera correlation_id.
3. request.state e contextvars recebem esse valor.
4. create_logger_with_correlation monta o logger apropriado.
5. Eventos são serializados em JSON com contexto e sanitização.
6. Arquivo principal, console e arquivo dedicado recebem a saída conforme a configuração.
7. Sidecar e manifest registram o vínculo da correlação com o arquivo físico.
8. Lifecycle da aplicação ativa CloudWatch no worker quando aplicável.
9. Operações administrativas constroem provider canônico.
10. O provider prepara logs locais ou materializa logs remotos.
11. CanonicalLogReader organiza a família de arquivos.
12. AnalyzeLogsCommand produz o diagnóstico.

## 16. Caminho feliz

Um cenário feliz típico é este.

1. A API recebe request sem correlation_id e gera um novo valor.
2. A resposta devolve X-Correlation-Id e correlationId no corpo JSON quando aplicável.
3. O runtime registra eventos estruturados sob esse mesmo ID.
4. O arquivo dedicado por correlação é criado e indexado no manifest.
5. O operador chama /logs usando o mesmo ID.
6. O provider correto é resolvido e o leitor canônico reencontra a família de logs.

## 17. Cenários de erro confirmados

### Provider inválido fora de development

Sintoma: build_log_provider falha cedo.

Causa confirmada: LOG_PROVIDER_TYPE ausente ou inválido fora de development.

### Filesystem sem selector

Sintoma: consulta administrativa retorna erro 400.

Causa confirmada: ausência de correlation_id, log_name e all_logs na requisição local.

### CloudWatch sem filtros mínimos

Sintoma: consulta administrativa retorna erro 400.

Causa confirmada: ausência de log_group ou de seletor útil.

### Materialização remota falhou

Sintoma: consulta administrativa retorna erro 502 ou 500.

Causa confirmada: falha de comunicação, payload inválido ou ausência de eventos na origem remota.

## 18. Observabilidade e diagnóstico

Para investigar esse subsistema, a sequência correta é esta.

1. Confirmar qual correlation_id a execução recebeu.
2. Confirmar se a resposta HTTP devolveu X-Correlation-Id.
3. Confirmar se o ambiente está em development ou fora dele.
4. Confirmar o provider ativo em application.state.active_log_provider_type.
5. Confirmar se o worker ativou CloudWatch em application.state.cloudwatch_handler_ativo.
6. Confirmar se a correlação gerou sidecar e manifest quando arquivo dedicado era esperado.

## 19. Como colocar para funcionar

### 19.1 Development

O caminho mínimo confirmado no código é este.

1. environment em development.
2. Provider administrativo automaticamente em filesystem.
3. enable_file_logging e ou enable_correlation_file_logging ativos.
4. API devolvendo X-Correlation-Id.

### 19.2 CloudWatch como destino de escrita

O mínimo confirmado no código é este.

1. enable_cloudwatch_logging verdadeiro.
2. group e parâmetros AWS coerentes.
3. ativação efetiva do worker no lifespan.

### 19.3 Provider remoto de leitura

Fora de development, o mínimo confirmado é este.

1. LOG_PROVIDER_TYPE explícito.
2. parâmetros específicos do provider ativo preenchidos.
3. request administrativa contendo seletores compatíveis.

## 20. Exemplos práticos guiados

### Exemplo 1: request HTTP com correlation_id reaproveitado do header

Entrada: cliente envia X-Correlation-Id válido.

Processamento: o middleware reaproveita esse valor, normaliza se necessário e o usa como logger da requisição.

Saída: a resposta devolve o mesmo X-Correlation-Id e o runtime escreve sob a mesma identidade lógica.

### Exemplo 2: análise local de família de logs

Entrada: operador informa correlation_id para o endpoint administrativo.

Processamento: FileSystemProvider prepara o diretório local e CanonicalLogReader organiza os arquivos api, worker e scheduler relacionados.

Saída: AnalyzeLogsCommand recebe um conjunto coerente de artefatos da mesma execução.

### Exemplo 3: análise remota via provider

Entrada: operador usa provider remoto como CloudWatch ou Northflank.

Processamento: o provider consulta a origem, materializa um artefato local temporário e devolve PreparedLogSource.

Saída: o pipeline administrativo trabalha do mesmo jeito que no caso local.

## 21. Explicação 101

O sistema funciona como uma cadeia de custódia dos eventos. A API dá um número para a execução. O logger usa esse número para registrar o que aconteceu. Os metadados guardam onde esse rastro foi salvo. E a camada administrativa sabe recuperar esse rastro depois, seja ele local ou remoto.

## 22. Limites e pegadinhas

- correlation_id inválido não é preservado; ele é substituído por um formato canônico.
- Identificador estático de infraestrutura não gera arquivo dedicado por correlação.
- CloudWatch habilitado não significa handler anexado antes do ciclo correto do worker.
- Fora de development, provider ausente não cai silenciosamente para filesystem.

## 23. Checklist de entendimento

- Entendi como o correlation_id nasce ou é reaproveitado.
- Entendi a diferença entre logger compartilhado e logger com arquivo dedicado.
- Entendi o papel de sidecar e manifest.
- Entendi a ativação tardia do CloudWatch.
- Entendi a resolução do provider administrativo.
- Entendi o papel do CanonicalLogReader.

## 24. Evidências no código

- src/core/correlation_id_factory.py
  - Motivo da leitura: confirmar geração e validação do correlation_id.
  - Símbolos relevantes: CorrelationIdFactory.generate, CorrelationIdFactory.is_valid.
  - Comportamento confirmado: formato canônico com timestamp e UUID.

- src/core/base_correlation_component.py
  - Motivo da leitura: confirmar contrato arquitetural de propagação.
  - Símbolos relevantes: BaseCorrelationComponent, _setup_isolated_logger.
  - Comportamento confirmado: yaml_config, user_session, correlation_id e user_email são obrigatórios.

- src/core/logging_system.py
  - Motivo da leitura: confirmar handlers, sanitização, CloudWatch e logger canônico.
  - Símbolos relevantes: normalize_correlation_id, create_logger_with_correlation, defer_cloudwatch_until_worker, activate_cloudwatch_for_worker.
  - Comportamento confirmado: logger estruturado, arquivo dedicado por correlação e ativação tardia do CloudWatch.

- src/core/log_origin_metadata.py
  - Motivo da leitura: confirmar manifest, sidecar e contexto local de request.
  - Símbolos relevantes: create_correlation_metadata_filename, resolve_log_filename_from_manifest, set_request_correlation_id.
  - Comportamento confirmado: vínculo persistido entre correlation_id e arquivo físico.

- src/api/service_api.py
  - Motivo da leitura: confirmar borda HTTP e resposta com correlation_id.
  - Símbolos relevantes: log_requests, \_inject_correlation_in_json_body, lifespan.
  - Comportamento confirmado: X-Correlation-Id na resposta, correlationId no JSON quando aplicável e ativação do CloudWatch no worker.

- src/api/services/log_provider_service.py
  - Motivo da leitura: confirmar provider canônico e regra de ambiente.
  - Símbolos relevantes: resolve_active_log_provider_type, build_log_provider.
  - Comportamento confirmado: filesystem forçado em development e provider explícito fora dele.

- src/api/services/canonical_log_reader.py
  - Motivo da leitura: confirmar organização da família de logs.
  - Símbolos relevantes: CanonicalLogReader, infer_log_role, sort_log_family_paths.
  - Comportamento confirmado: ordenação estável e leitura por família operacional.

- src/api/services/logs_admin_service.py
  - Motivo da leitura: confirmar fluxo administrativo oficial.
  - Símbolos relevantes: analyze_logs_via_admin_provider, build_logs_admin_provider.
  - Comportamento confirmado: toda análise administrativa passa pelo provider canônico e limpa artefatos temporários.
