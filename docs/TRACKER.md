# Tracker de Rastreabilidade

Mapeia cada item registrado nos documentos de design à origem na transcrição (`TRANSCRICAO.md`) ou no código-fonte. Preenchido incrementalmente conforme os documentos (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/*.md`) são produzidos.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Restrição | Sistema hoje não tem notificação externa; clientes fazem polling em GET /orders | TRANSCRICAO | `[09:00] Marcos` |
| PRD-PROB-01 | docs/PRD.md | Requisito Não Funcional | Atlas Comercial ameaça migrar de plataforma se não houver notificação em tempo real até fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-01 | docs/PRD.md | Objetivo (métrica) | Latência de entrega abaixo de 10 segundos definida como "tempo real" pelos clientes | TRANSCRICAO | `[09:02] Marcos` |
| PRD-ESCOPO-01 | docs/PRD.md | Restrição (fora de escopo) | Webhooks são só outbound; cliente não envia webhook para a plataforma | TRANSCRICAO | `[09:02] Marcos` |
| PRD-ESCOPO-02 | docs/PRD.md | Restrição (fora de escopo) | Notificação por e-mail em falhas recorrentes fica para fase futura | TRANSCRICAO | `[09:37] Marcos` / `[09:38] Larissa` |
| PRD-ESCOPO-03 | docs/PRD.md | Restrição (fora de escopo) | Dashboard visual para o cliente é projeto separado do time de frontend | TRANSCRICAO | `[09:39] Marcos` / `[09:40] Larissa` |
| PRD-ESCOPO-04 | docs/PRD.md | Restrição (fora de escopo) | Rate limiting de disparo ao cliente adiado para observação em produção | TRANSCRICAO | `[09:38] Diego` / `[09:39] Larissa` |
| PRD-ESCOPO-05 | docs/PRD.md | Restrição (fora de escopo) | Sem garantia de ordering global entre eventos, só por order_id em single-worker | TRANSCRICAO | `[09:12]-[09:14] Diego/Larissa` |
| PRD-ESCOPO-06 | docs/PRD.md | Restrição (fora de escopo) | Arquivamento de eventos entregues após ~30 dias fica fora desta feature | TRANSCRICAO | `[09:08] Diego` |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cliente cadastra webhook (URL, status de interesse); secret gerada pela plataforma | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição (PATCH) de webhook cadastrado | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção (DELETE) de webhook cadastrado | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem (GET) dos webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de status aplicado na inserção do evento na outbox, não no envio | TRANSCRICAO | `[09:33]-[09:34] Marcos/Bruno/Diego` |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de últimas 100 entregas de um webhook (GET /webhooks/:id/deliveries) | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Endpoint admin de replay manual de dead-letter (POST /admin/webhooks/dead-letter/:id/replay) | TRANSCRICAO | `[09:18] Diego` / `[09:35] Larissa` |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | HMAC-SHA256 assinando o payload enviado ao cliente | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry com backoff exponencial, 5 tentativas, antes de mover para dead-letter | TRANSCRICAO | `[09:15]-[09:17] Diego` |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | X-Event-Id único por evento para deduplicação no cliente (at-least-once) | TRANSCRICAO | `[09:24]-[09:25] Diego` |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Recusa de cadastro de webhook com URL sem HTTPS | TRANSCRICAO | `[09:23] Sofia` |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Worker com polling de 2 segundos garante latência dentro do limite de 10s | TRANSCRICAO | `[09:09]-[09:10] Diego/Marcos` |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Inserção do evento na mesma transação SQL da mudança de status | TRANSCRICAO | `[09:40]-[09:41] Bruno/Diego` |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Worker roda como processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB de payload, rejeitado (não truncado) se ultrapassar | TRANSCRICAO | `[09:23]-[09:24] Sofia/Diego` |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP ao endpoint do cliente | TRANSCRICAO | `[09:42] Diego` |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Replay de dead-letter deve ficar registrado para auditoria | TRANSCRICAO | `[09:36] Sofia` |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Módulo reaproveita padrão de módulos, AppError, Pino e error middleware existentes | TRANSCRICAO | `[09:27]-[09:30] Bruno/Larissa` |
| PRD-TRADE-01 | docs/PRD.md | Trade-off | Outbox em MySQL em vez de Redis Streams, evita subir infraestrutura nova | TRANSCRICAO | `[09:06]-[09:07] Diego/Larissa` |
| PRD-TRADE-02 | docs/PRD.md | Trade-off | Polling de 2s em vez de trigger reativo, pela ausência de LISTEN/NOTIFY no MySQL | TRANSCRICAO | `[09:09] Diego` |
| PRD-TRADE-03 | docs/PRD.md | Trade-off | 5 tentativas de retry em vez de 3, para cobrir indisponibilidades reais já observadas | TRANSCRICAO | `[09:15]-[09:17] Diego/Bruno` |
| PRD-TRADE-04 | docs/PRD.md | Trade-off | At-least-once com dedup no cliente em vez de exactly-once | TRANSCRICAO | `[09:24]-[09:26] Diego/Sofia` |
| PRD-TRADE-05 | docs/PRD.md | Trade-off | Snapshot do payload na inserção do evento, não renderização tardia | TRANSCRICAO | `[09:51]-[09:52] Larissa/Diego/Bruno` |
| PRD-DEP-01 | docs/PRD.md | Dependência | Transação `changeStatus` do serviço de pedidos precisa ser estendida | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-02 | docs/PRD.md | Dependência | Middleware `requireRole` reaproveitado para restringir replay de DLQ a ADMIN | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-DEP-03 | docs/PRD.md | Dependência | Classe `AppError` e padrão de erros reaproveitados para os erros do módulo webhook | CODIGO | `src/shared/errors/app-error.ts` |
| PRD-DEP-04 | docs/PRD.md | Dependência | Revisão de segurança da Sofia sobre HMAC e geração de secret antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente indisponível por longos períodos causa acúmulo de retries ou perda de notificação | TRANSCRICAO | `[09:15]-[09:18] Diego` |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret em log do cliente (já ocorreu antes) | TRANSCRICAO | `[09:22] Diego` |
| PRD-RISK-03 | docs/PRD.md | Risco | Chamada HTTP síncrona travando transação de mudança de status | TRANSCRICAO | `[09:04] Bruno` |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajada de eventos satura cliente com chamadas simultâneas | TRANSCRICAO | `[09:38]-[09:39] Diego/Larissa` |
| PRD-TEST-01 | docs/PRD.md | Restrição | Reserva de pelo menos 2 dias úteis para revisão de segurança da Sofia antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| FDD-CTX-01 | docs/FDD.md | Restrição | Nenhuma chamada HTTP pode acontecer dentro da transação síncrona de changeStatus | TRANSCRICAO | `[09:03]-[09:06] Bruno/Larissa/Diego` |
| FDD-MODELO-01 | docs/FDD.md | Decisão | Outbox e DLQ em tabelas separadas, DLQ mantém a leitura da outbox principal limpa | TRANSCRICAO | `[09:18] Diego` |
| FDD-MODELO-02 | docs/FDD.md | Decisão | Índice em (status, nextAttemptAt) na outbox para leitura eficiente pelo worker em lotes pequenos | TRANSCRICAO | `[09:08] Diego` |
| FDD-MODELO-03 | docs/FDD.md | Decisão | IDs das novas tabelas em UUID, seguindo padrão do restante do schema | TRANSCRICAO | `[09:51] Larissa` |
| FDD-MODELO-04 | docs/FDD.md | Decisão | Payload armazenado como snapshot renderizado na inserção, não referência para renderização tardia | TRANSCRICAO | `[09:51]-[09:52] Larissa/Diego/Bruno` |
| FDD-MODELO-05 | docs/FDD.md | Restrição | Secret atual e anterior (com expiração) armazenadas no próprio WebhookEndpoint para suportar rotação com grace period | TRANSCRICAO | `[09:21] Sofia` |
| FDD-FLUXO-01 | docs/FDD.md | Requisito Funcional | publishWebhookEvent busca endpoints ativos inscritos no status e insere na outbox dentro da mesma tx | TRANSCRICAO | `[09:33]-[09:34] Bruno/Diego` |
| FDD-FLUXO-02 | docs/FDD.md | Requisito Funcional | Worker roda loop de polling a cada 2s buscando eventos pendentes prontos para tentativa | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLUXO-03 | docs/FDD.md | Requisito Funcional | Worker abre PrismaClient próprio, processo separado da API | TRANSCRICAO | `[09:29]-[09:30] Diego/Bruno` |
| FDD-FLUXO-04 | docs/FDD.md | Requisito Funcional | Backoff exponencial 1m/5m/30m/2h/12h antes de mover para dead-letter | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLUXO-05 | docs/FDD.md | Requisito Funcional | Reprocessamento manual recria linha na outbox como pending e remove/marca entrada de dead-letter | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLUXO-06 | docs/FDD.md | Requisito Não Funcional | Replay administrativo registra userId de quem executou, para auditoria | TRANSCRICAO | `[09:36] Sofia` |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/webhooks cria webhook, secret retornada só na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/webhooks lista webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/webhooks/:id edita URL, status de interesse e estado ativo | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/webhooks/:id remove webhook cadastrado | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/webhooks/:id/rotate-secret gera nova secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/webhooks/:id/deliveries retorna últimas 100 entregas | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/admin/webhooks/dead-letter/:id/replay exige role ADMIN | TRANSCRICAO | `[09:35]-[09:36] Larissa/Sofia` |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers de saída: X-Event-Id, X-Webhook-Id, X-Signature, X-Timestamp | TRANSCRICAO | `[09:44]-[09:45] Diego/Sofia` |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Payload de evento não inclui items do pedido, mantém corpo enxuto | TRANSCRICAO | `[09:43]-[09:44] Diego/Bruno` |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Prefixo WEBHOOK_ para todos os códigos de erro do módulo | TRANSCRICAO | `[09:28]-[09:29] Bruno/Larissa` |
| FDD-ERRO-02 | docs/FDD.md | Requisito Funcional | WEBHOOK_INVALID_URL recusa cadastro de URL sem HTTPS | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-03 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_PAYLOAD_TOO_LARGE: evento acima de 64KB é rejeitado, nunca truncado | TRANSCRICAO | `[09:23]-[09:24] Sofia/Diego` |
| FDD-ERRO-04 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_DELIVERY_TIMEOUT: timeout de 10s tratado como falha, segue para retry | TRANSCRICAO | `[09:42] Diego` |
| FDD-RESIL-01 | docs/FDD.md | Trade-off | At-least-once, cliente responsável por dedup via X-Event-Id | TRANSCRICAO | `[09:24]-[09:26] Diego/Sofia` |
| FDD-INTEG-01 | docs/FDD.md | Dependência | changeStatus estendido para chamar publishWebhookEvent(tx, order, from, to) dentro da mesma transação | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INTEG-02 | docs/FDD.md | Dependência | Classes de erro do módulo estendem AppError/ConflictError/NotFoundError já existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INTEG-03 | docs/FDD.md | Dependência | Endpoint de replay usa requireRole('ADMIN') já exportado pelo middleware de auth | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INTEG-04 | docs/FDD.md | Dependência | Erros do módulo fluem pelo errorMiddleware central sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INTEG-05 | docs/FDD.md | Dependência | Worker reaproveita a instância de logger Pino já configurada | CODIGO | `src/shared/logger/index.ts` |
| FDD-INTEG-06 | docs/FDD.md | Dependência | Novo router de webhooks registrado em buildApiRouter seguindo padrão dos demais módulos | CODIGO | `src/routes/index.ts` |
| FDD-INTEG-07 | docs/FDD.md | Dependência | Novos models Prisma seguem convenção de id UUID, @map e índices explícitos | CODIGO | `prisma/schema.prisma` |
| FDD-INTEG-08 | docs/FDD.md | Dependência | src/worker.ts segue o mesmo padrão de bootstrap e shutdown gracioso de server.ts | CODIGO | `src/server.ts` |

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | docs/RFC.md | Restrição | Chamada síncrona no changeStatus travaria mudança de status de outros pedidos se um cliente estivesse lento/fora do ar | TRANSCRICAO | `[09:04]-[09:06] Bruno/Larissa/Diego` |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox transacional: evento gravado na mesma transação SQL do changeStatus | TRANSCRICAO | `[09:06] Diego` |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker dedicado em polling de 2s, processo separado da API | TRANSCRICAO | `[09:09]-[09:11] Diego` |
| RFC-PROP-03 | docs/RFC.md | Decisão | Entrega assinada (HMAC-SHA256), at-least-once com X-Event-Id, retry com backoff e dead-letter | TRANSCRICAO | `[09:15]-[09:26] Diego` |
| RFC-PROP-04 | docs/RFC.md | Requisito Funcional | Cliente configura endpoint por customer, com secret exclusiva gerada na criação | TRANSCRICAO | `[09:21] Sofia` |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Redis Streams descartado por exigir infraestrutura nova para um time pequeno | TRANSCRICAO | `[09:07] Diego/Larissa` |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Trigger de banco reativo descartado pela ausência de LISTEN/NOTIFY no MySQL | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-03 | docs/RFC.md | Trade-off | 3 tentativas de retry descartadas por não cobrir indisponibilidade real já observada (2h) | TRANSCRICAO | `[09:15]-[09:16] Diego/Bruno` |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Exactly-once descartado por complexidade de coordenação, a favor de at-least-once + dedup no cliente | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de disparo por cliente, adiado para observação em produção | TRANSCRICAO | `[09:38]-[09:39] Diego/Larissa` |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Notificação de fallback por e-mail em falhas recorrentes, adiada para fase futura | TRANSCRICAO | `[09:37]-[09:38] Marcos/Larissa` |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Recuperação de eventos presos em status "processing" caso o worker caia no meio do ciclo | ANÁLISE RFC | Não discutido na reunião; lacuna identificada ao desenhar o worker na proposta técnica (não rastreável a TRANSCRICAO ou CODIGO) |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Estratégia de escala para múltiplos workers (particionamento por order_id ou lock pessimista) | TRANSCRICAO | `[09:12]-[09:13] Diego` |
| RFC-RISK-01 | docs/RFC.md | Risco | Ausência de alerta automático para worker parado; entrega para silenciosamente até alguém notar | ANÁLISE RFC | Não discutido na reunião; lacuna identificada na proposta técnica (não rastreável a TRANSCRICAO ou CODIGO) |
| RFC-RISK-02 | docs/RFC.md | Risco | Superfície nova de exposição de dados de pedido para fora da infraestrutura da empresa | TRANSCRICAO | `[09:19] Sofia` |
| RFC-INTEG-01 | docs/RFC.md | Dependência | publishWebhookEvent chamado dentro da transação de changeStatus, recebendo o tx client em andamento | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-EST-01 | docs/RFC.md | Restrição | Estimativa de três sprints, incluindo revisão de segurança da Sofia ao final | TRANSCRICAO | `[09:45]-[09:46] Larissa` |

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox transacional no MySQL para desacoplar notificação da transação de changeStatus | TRANSCRICAO | `[09:06]-[09:08] Diego` |
| ADR-001-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Redis Streams descartado por overengineering para o tamanho do time | TRANSCRICAO | `[09:07] Diego/Larissa` |
| ADR-001-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Dependência | Reaproveita o padrão de tx já usado por debitStock/replenishStock | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002-01 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | Worker dedicado (src/worker.ts) em processo separado, polling de 2s | TRANSCRICAO | `[09:09]-[09:11] Diego` |
| ADR-002-02 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Trade-off | Trigger de banco reativo descartado pela ausência de LISTEN/NOTIFY no MySQL | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-03 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Restrição | Ordenação garantida só por order_id, single-worker nesta fase | TRANSCRICAO | `[09:12]-[09:14] Diego/Marcos` |
| ADR-002-04 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Dependência | src/worker.ts espelha o padrão de bootstrap e shutdown gracioso de server.ts | CODIGO | `src/server.ts` |
| ADR-003-01 | docs/adrs/ADR-003-retry-backoff-dead-letter.md | Decisão | Retry com 5 tentativas em backoff exponencial 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:15]-[09:17] Diego` |
| ADR-003-02 | docs/adrs/ADR-003-retry-backoff-dead-letter.md | Decisão | Falhas permanentes persistidas em tabela webhook_dead_letter separada | TRANSCRICAO | `[09:18] Diego` |
| ADR-003-03 | docs/adrs/ADR-003-retry-backoff-dead-letter.md | Trade-off | 3 tentativas descartadas por não cobrir indisponibilidade real de ~2h já observada | TRANSCRICAO | `[09:15]-[09:16] Diego/Bruno` |
| ADR-004-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret exclusiva por endpoint, sem secret global | TRANSCRICAO | `[09:19]-[09:21] Sofia` |
| ADR-004-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Secret global única descartada pelo precedente real de vazamento em log de cliente | TRANSCRICAO | `[09:21]-[09:22] Sofia/Diego` |
| ADR-005-01 | docs/adrs/ADR-005-garantia-at-least-once-com-id-de-evento.md | Decisão | Garantia at-least-once com X-Event-Id (UUID) para dedup no cliente | TRANSCRICAO | `[09:24]-[09:25] Diego` |
| ADR-005-02 | docs/adrs/ADR-005-garantia-at-least-once-com-id-de-evento.md | Trade-off | Exactly-once descartado por complexidade de coordenação distribuída | TRANSCRICAO | `[09:24]-[09:25] Diego` |
| ADR-006-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Módulo de webhooks segue estrutura, erros, logger e auth já existentes, sem convenção nova | TRANSCRICAO | `[09:27]-[09:30] Bruno/Diego/Larissa` |
| ADR-006-02 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Dependência | Classes de erro do módulo estendem as classes-base já existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-03 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Dependência | Logger Pino global reaproveitado sem sink novo | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-04 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Dependência | Middleware de erro central já trata AppError, Zod e Prisma sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-05 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Dependência | requireRole('ADMIN') reaproveitado sem modificação para o endpoint de replay | CODIGO | `src/middlewares/auth.middleware.ts` |

