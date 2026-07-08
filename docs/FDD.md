# Feature Design Doc — Sistema de Webhooks de Notificação de Pedidos

**Autor:** Bruno (Eng. Pleno, time de Pedidos), com revisão de Diego (Eng. Sênior, Plataforma)
**Status:** Pronto para implementação
**Data:** reunião de definição em 09:00, quinta-feira
**Revisores técnicos:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança)
**Documentos relacionados:** [PRD](./PRD.md) · [RFC](./RFC.md) · [ADRs](./adrs/)

## 1. Contexto e motivação técnica

O OMS hoje não tem nenhum mecanismo de notificação externa. Toda mudança de status de pedido fica só no banco, e os três clientes B2B que pediram integração (Atlas Comercial, MaxDistribuição, Nova Cargo) descobrem a mudança fazendo polling em `GET /orders`. Este documento detalha como construir o módulo `webhooks`: como o evento nasce dentro da transação de `changeStatus`, como um worker separado processa a fila de saída, como a entrega é assinada e reentregue em caso de falha, e como cada peça se encaixa na estrutura de código já existente.

Este é o documento de implementação. As decisões arquiteturais fechadas (outbox no MySQL, retry com backoff, HMAC-SHA256, at-least-once, worker em processo separado, reuso de padrões) estão registradas com contexto e alternativas nos [ADRs](./adrs/); aqui elas são assumidas como decididas e detalhadas em nível de código, contrato e operação.

## 2. Objetivos técnicos

- Garantir que a inserção do evento de notificação seja atômica com a transação de mudança de status: se uma falhar, a outra também falha.
- Manter o disparo HTTP totalmente fora do caminho síncrono de `changeStatus`, para que a lentidão ou indisponibilidade de um cliente nunca trave a mudança de status de nenhum pedido.
- Entregar eventos dentro de 10 segundos da mudança de status, na ausência de falhas, através de um worker com polling de 2 segundos.
- Garantir at-least-once com identificador único por evento, permitindo que o cliente dedupe do lado dele.
- Seguir a estrutura de módulo, o padrão de erros, o logger e o middleware de erro já existentes no projeto, sem introduzir peças novas de infraestrutura.

## 3. Escopo e exclusões

Incluído:

- Módulo `src/modules/webhooks` com CRUD de configuração de webhook (criação, edição, remoção, listagem).
- Extensão do `OrderService.changeStatus` para publicar evento na outbox dentro da mesma transação.
- Worker (`src/worker.ts`) rodando como processo separado, em polling, processando a outbox e disparando as chamadas HTTP.
- Retry com backoff exponencial (5 tentativas) e persistência de falhas permanentes em tabela de dead-letter.
- Endpoint administrativo de reprocessamento manual de dead-letter.
- Consulta de histórico de entregas de um webhook.
- Assinatura HMAC-SHA256, geração e rotação de secret por endpoint.

Excluído (ver PRD e ADRs para detalhe e origem):

- Notificação por e-mail em falhas recorrentes.
- Painel visual para o cliente gerenciar webhooks.
- Rate limiting de disparo por cliente.
- Garantia de ordering global entre eventos de clientes ou pedidos diferentes (só há ordenação por `order_id` e apenas em regime single-worker).
- Arquivamento automático de eventos entregues na outbox.
- Múltiplos workers em paralelo (mencionado como evolução futura, não faz parte desta entrega).

## 4. Modelagem de dados

Três tabelas novas, todas com `id` UUID seguindo o padrão do restante do schema (`[09:51] Larissa`).

```prisma
model WebhookEndpoint {
  id         String   @id @default(uuid()) @db.Char(36)
  customerId String   @db.Char(36)
  url        String   @db.VarChar(500)
  secret     String   @db.VarChar(255)
  oldSecret  String?  @db.VarChar(255)
  oldSecretExpiresAt DateTime?
  statuses   Json
  active     Boolean  @default(true)
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  customer   Customer @relation(fields: [customerId], references: [id])
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventType      String   @db.VarChar(64)
  payload        Json
  status         String   @db.VarChar(20) // pending | processing | delivered | failed
  attempts       Int      @default(0)
  nextAttemptAt  DateTime @default(now())
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  webhook    WebhookEndpoint   @relation(fields: [webhookId], references: [id])
  deliveries WebhookDelivery[]

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  webhookId      String   @db.Char(36)
  attempt        Int
  success        Boolean
  httpStatus     Int?
  responseBody   String?  @db.Text
  durationMs     Int?
  errorMessage   String?  @db.VarChar(500)
  attemptedAt    DateTime @default(now())

  outbox  WebhookOutbox   @relation(fields: [outboxId], references: [id])
  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([webhookId, attemptedAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @db.Char(36)
  webhookId    String   @db.Char(36)
  eventType    String   @db.VarChar(64)
  payload      Json
  reason       String   @db.VarChar(500)
  attempts     Int
  createdAt    DateTime @default(now())

  @@index([webhookId])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem, rastreadas à reunião:

- Índice em `status` + `nextAttemptAt` na outbox: o worker lê só os pendentes prontos para tentativa, em lotes pequenos (`[09:08] Diego`).
- DLQ em tabela separada, não como um status na própria outbox: mantém a leitura da outbox principal limpa e serve como evidência de debug e reprocessamento (`[09:18] Diego`).
- `payload` da outbox é o snapshot já renderizado no momento da inserção do evento, não uma referência ao pedido para renderização tardia. Se o pedido mudar depois, o evento entregue continua refletindo o estado no instante da mudança de status (`[09:51]-[09:52] Larissa/Diego/Bruno`).
- `WebhookEndpoint` guarda `secret` atual e `oldSecret` com expiração, suportando o grace period de rotação de 24h sem precisar de tabela extra (`[09:21] Sofia`).

## 5. Fluxos detalhados e diagramas

Sem diagramas nesta versão. Fluxos conceituais:

### 5.1 Criação do evento na outbox (dentro de `changeStatus`)

1. `OrderService.changeStatus` executa a transação normal: valida a transição, debita ou repõe estoque, atualiza `orders`, insere em `order_status_history`.
2. Antes do commit, dentro da mesma `tx`, chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
3. `publishWebhookEvent` busca, na mesma transação, os `WebhookEndpoint` ativos do `customerId` do pedido cujo array `statuses` contém o `toStatus`.
4. Se nenhum endpoint estiver inscrito naquele status, a função retorna sem inserir nada — o filtro é aplicado na inserção, não no envio, para não acumular linhas inúteis na outbox (`[09:33]-[09:34] Bruno/Diego`).
5. Para cada endpoint inscrito, monta o payload (ver seção de contratos) e insere uma linha em `webhook_outbox` com `status = 'pending'`, `attempts = 0`, `nextAttemptAt = now()`.
6. Se qualquer passo dentro da transação falhar — incluindo a inserção na outbox — toda a transação sofre rollback, e nem a mudança de status nem o evento existem (`[09:40]-[09:41] Bruno/Diego`).

### 5.2 Processamento pelo worker

1. `src/worker.ts` inicializa seu próprio `PrismaClient`, conectado à mesma `DATABASE_URL` da API, mas como processo Node independente (`[09:30] Bruno`).
2. Em loop, a cada 2 segundos, busca um lote de linhas de `webhook_outbox` com `status = 'pending'` e `nextAttemptAt <= now()`, ordenadas por `createdAt` ascendente.
3. Para cada linha, marca `status = 'processing'` antes de tentar o envio, evitando que um segundo ciclo do worker pegue a mesma linha (proteção suficiente em regime single-worker, conforme decidido).
4. Monta a requisição HTTP: URL do `WebhookEndpoint`, headers de assinatura (ver Contratos públicos), corpo é o `payload` já armazenado.
5. Executa o `POST` com timeout de 10 segundos.
6. Em caso de resposta HTTP 2xx: marca a linha como `status = 'delivered'`, grava uma linha em `webhook_deliveries` com `success = true`, `httpStatus`, `durationMs`, e segue para a próxima.
7. Em caso de erro (timeout, erro de rede, ou resposta não-2xx): grava a tentativa em `webhook_deliveries` com `success = false` e segue para o fluxo de retry.

### 5.3 Retry com backoff exponencial

1. Em cada falha, incrementa `attempts` na linha da outbox.
2. Se `attempts < 5`, calcula o próximo `nextAttemptAt` segundo a tabela de backoff e volta o `status` para `pending`:

   | Tentativa que falhou | Próxima tentativa após |
   | --- | --- |
   | 1ª | 1 minuto |
   | 2ª | 5 minutos |
   | 3ª | 30 minutos |
   | 4ª | 2 horas |
   | 5ª | 12 horas (última tentativa) |

   Progressão decidida para cobrir cerca de 15 horas de indisponibilidade do cliente, já observada em caso real de manutenção planejada (`[09:15]-[09:17] Diego/Bruno`).
3. Se `attempts` atinge 5 e a última tentativa também falhou, a linha sai da outbox: o worker cria uma linha em `webhook_dead_letter` com o payload, o motivo da última falha e o total de tentativas, e marca a linha da outbox como `status = 'failed'`.

### 5.4 Dead-letter e reprocessamento manual

1. Uma entrada em `webhook_dead_letter` fica visível apenas via consulta administrativa (fora do escopo desta entrega expor listagem completa; o endpoint de replay recebe o `id` diretamente).
2. Um administrador (role `ADMIN`) chama `POST /admin/webhooks/dead-letter/:id/replay`.
3. O serviço localiza a entrada em `webhook_dead_letter`, cria uma nova linha em `webhook_outbox` com `status = 'pending'`, `attempts = 0`, `nextAttemptAt = now()` e o mesmo payload, e remove (ou marca como reprocessada) a entrada de dead-letter.
4. A ação registra no log estruturado (Pino) o `userId` do administrador que executou o replay, para auditoria (`[09:36] Sofia`).

## 6. Contratos públicos (assinaturas, headers, exemplos)

Todos os endpoints de configuração e consulta seguem o prefixo `/api/webhooks`; o endpoint de replay fica em `/api/admin/webhooks`. Autenticação via `Authorization: Bearer <jwt>` em todos, seguindo o padrão já usado pelos demais módulos (`authenticate` + `validate` middlewares).

### 6.1 `POST /api/webhooks`

Cria um webhook para um customer. Requer usuário autenticado (qualquer role, por decisão explícita da reunião — `[09:37] Sofia`).

Request:

```json
{
  "customerId": "8f14e45f-ceea-467e-a2ac-b7d1c1b5b1a1",
  "url": "https://integra.atlascomercial.com/webhooks/orders",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "3a1e9f2a-1b2c-4d3e-9f0a-2b3c4d5e6f7a",
  "customerId": "8f14e45f-ceea-467e-a2ac-b7d1c1b5b1a1",
  "url": "https://integra.atlascomercial.com/webhooks/orders",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_5f1c9a2e7b3d4f6a8c1e2b3d4f5a6c7e",
  "active": true,
  "createdAt": "2026-07-08T12:00:00.000Z"
}
```

A `secret` só aparece completa nesta resposta de criação; não é possível recuperá-la depois (`[09:31] Marcos`).

### 6.2 `GET /api/webhooks?customerId=...`

Lista os webhooks cadastrados para um customer.

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "3a1e9f2a-1b2c-4d3e-9f0a-2b3c4d5e6f7a",
      "customerId": "8f14e45f-ceea-467e-a2ac-b7d1c1b5b1a1",
      "url": "https://integra.atlascomercial.com/webhooks/orders",
      "statuses": ["PAID", "SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-08T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

A `secret` nunca é retornada em listagem ou consulta individual, só na criação e na rotação.

### 6.3 `PATCH /api/webhooks/:id`

Edita URL, lista de status ou estado ativo de um webhook.

Request:

```json
{
  "statuses": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `200 OK`: mesmo formato do `GET` de um item.

### 6.4 `DELETE /api/webhooks/:id`

Remove um webhook cadastrado. Response `204 No Content`.

### 6.5 `POST /api/webhooks/:id/rotate-secret`

Gera uma nova secret para o endpoint. A secret anterior continua válida por 24 horas (`[09:21] Sofia`).

Response `200 OK`:

```json
{
  "id": "3a1e9f2a-1b2c-4d3e-9f0a-2b3c4d5e6f7a",
  "secret": "whsec_9c8b7a6d5e4f3a2b1c0d9e8f7a6b5c4d",
  "oldSecretExpiresAt": "2026-07-09T12:00:00.000Z"
}
```

### 6.6 `GET /api/webhooks/:id/deliveries`

Histórico das últimas 100 entregas de um webhook (`[09:34] Marcos`).

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "c1d2e3f4-5678-90ab-cdef-1234567890ab",
      "attempt": 1,
      "success": true,
      "httpStatus": 200,
      "durationMs": 184,
      "attemptedAt": "2026-07-08T12:00:03.000Z"
    },
    {
      "id": "d2e3f4a5-6789-01bc-def0-234567890abc",
      "attempt": 1,
      "success": false,
      "httpStatus": 503,
      "responseBody": "Service Unavailable",
      "durationMs": 10000,
      "attemptedAt": "2026-07-08T11:58:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

### 6.7 `POST /api/admin/webhooks/dead-letter/:id/replay`

Reprocessa manualmente uma entrada de dead-letter. Requer role `ADMIN` (`requireRole('ADMIN')`) (`[09:35]-[09:36] Larissa/Sofia`).

Response `202 Accepted`:

```json
{
  "outboxId": "e5f6a7b8-90cd-1e2f-3456-7890abcdef01",
  "status": "pending",
  "requeuedAt": "2026-07-08T13:00:00.000Z"
}
```

### 6.8 Chamada de saída: worker → endpoint do cliente

O worker envia `POST` para a `url` cadastrada no `WebhookEndpoint`.

Headers (`[09:44]-[09:45] Diego/Sofia`):

```
Content-Type: application/json
X-Event-Id: 5c9d8e7f-6a5b-4c3d-2e1f-0a9b8c7d6e5f
X-Webhook-Id: 3a1e9f2a-1b2c-4d3e-9f0a-2b3c4d5e6f7a
X-Signature: 5f1c9a2e7b3d4f6a8c1e2b3d4f5a6c7e...   (HMAC-SHA256 hex do corpo)
X-Timestamp: 2026-07-08T12:00:03.000Z
```

Body (`[09:43] Diego`):

```json
{
  "eventId": "5c9d8e7f-6a5b-4c3d-2e1f-0a9b8c7d6e5f",
  "eventType": "order.status_changed",
  "timestamp": "2026-07-08T12:00:03.000Z",
  "orderId": "9f0e1d2c-3b4a-5968-7a6c-5d4e3f2a1b0c",
  "orderNumber": "ORD-000128",
  "customerId": "8f14e45f-ceea-467e-a2ac-b7d1c1b5b1a1",
  "fromStatus": "PAID",
  "toStatus": "PROCESSING",
  "totalCents": 45900
}
```

O payload não inclui a lista de itens do pedido, para não inflar o corpo; se o cliente precisar de detalhe, consulta `GET /orders/:id` (`[09:43]-[09:44] Diego/Bruno`).

## 7. Configuração e opções

Linguagem e runtime:

- Node.js ≥ 20, TypeScript, mesma stack já usada pela API (`package.json`).

Armazenamento de estado:

- MySQL via Prisma (`5.22.0`), reaproveitando a instância já provisionada — sem Redis, sem broker de mensagens novo (`[09:07] Diego`).
- Outbox, deliveries e dead-letter vivem no mesmo banco da aplicação.

Ambiente de desenvolvimento:

- `docker-compose.yml` já existente sobe a API e o MySQL; o worker é um novo entry point Node que roda no mesmo ambiente, sem serviço de infraestrutura adicional.
- Novo script `npm run worker` em `package.json`, análogo ao padrão de `npm run dev` já existente, apontando para `src/worker.ts` (`[09:11] Larissa`).

Parâmetros do worker (configuráveis via variável de ambiente, seguindo o padrão de `src/config/env.ts`):

- `WEBHOOK_WORKER_POLL_INTERVAL_MS` (padrão 2000): intervalo do loop de polling (`[09:09] Diego`).
- `WEBHOOK_WORKER_BATCH_SIZE`: tamanho do lote lido por ciclo, para controlar throughput sem mudança de código.
- `WEBHOOK_DELIVERY_TIMEOUT_MS` (padrão 10000): timeout da chamada HTTP ao endpoint do cliente (`[09:42] Diego`).
- `WEBHOOK_MAX_PAYLOAD_BYTES` (padrão 65536): teto de 64KB para o payload do evento (`[09:23]-[09:24] Sofia/Diego`).
- `WEBHOOK_SECRET_ROTATION_GRACE_HOURS` (padrão 24): janela de validade da secret anterior após rotação (`[09:21] Sofia`).

Segurança e rede:

- URL do endpoint cadastrado precisa ser HTTPS; validação feita em nível de schema Zod na criação e edição, mesmo padrão do restante do projeto (`[09:23] Sofia`).
- Secret nunca é logada nem retornada fora da criação/rotação.

Validações recomendadas (schemas Zod do módulo `webhooks`):

- `url` exigida, string válida, protocolo `https://` obrigatório.
- `statuses` exigido, array não vazio de valores pertencentes ao enum `OrderStatus` do Prisma.
- `customerId` exigido, UUID, precisa existir em `customers`.

## 8. Erros, exceções e fallback

### 8.1 Consistência e atomicidade

Por que é crítico: a inserção do evento de webhook acontece dentro da mesma transação SQL que a mudança de status do pedido. Se as duas operações não forem atômicas, existe o risco de um pedido mudar de status sem gerar notificação, ou de um evento "fantasma" ser inserido sem a mudança de status correspondente ter sido persistida (`[09:40]-[09:41] Bruno/Diego`).

Garantias comportamentais (invariantes):

- Se a transação de `changeStatus` faz commit, e havia webhook ativo inscrito no `toStatus`, o evento existe na outbox.
- Se a transação sofre rollback por qualquer motivo — incluindo falha na própria inserção do evento — nem a mudança de status nem o evento existem.
- Em regime single-worker, uma linha da outbox nunca é processada duas vezes simultaneamente: o worker marca `status = 'processing'` antes de iniciar a tentativa de envio.

Atomicidade na inserção (outbox):

- Toda a lógica de `publishWebhookEvent` roda dentro do `Prisma.TransactionClient` (`tx`) já aberto por `changeStatus`, sem abrir transação própria — o mesmo padrão já usado por `debitStock` e `replenishStock` no `OrderService` (`[09:41] Diego`).
- Não há concorrência de escrita nesse ponto: a transação principal do pedido já serializa a operação.

Atomicidade no processamento (worker):

- Como a decisão da reunião foi manter um único worker em polling (`[09:12]-[09:13] Diego`), não há necessidade de lock distribuído ou lógica de particionamento nesta versão. A leitura em lote seguida da marcação `status = 'processing'` é suficiente para evitar duplicidade de processamento dentro do próprio processo.
- Escalar para múltiplos workers em paralelo exigiria lock pessimista ou particionamento por `order_id`, mas isso é explicitamente um problema futuro, fora do escopo desta entrega (`[09:13] Diego`).

### 8.2 Matriz de erros previstos

Todos seguem o padrão `AppError` já existente (`statusCode`, `errorCode`, `details` opcional), com prefixo `WEBHOOK_` reservado ao módulo (`[09:28]-[09:29] Bruno/Larissa`).

| Código | HTTP Status | Cenário |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `id` de webhook inexistente em GET, PATCH, DELETE, rotate-secret ou deliveries |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada não é HTTPS |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Tentativa de operação que depende de secret ainda não gerada (estado inconsistente) |
| `WEBHOOK_INVALID_STATUS` | 400 | Valor em `statuses` que não corresponde a um `OrderStatus` válido |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` informado na criação não existe |
| `WEBHOOK_DUPLICATE_ENDPOINT` | 409 | Mesmo `customerId` + `url` já cadastrado e ativo |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento ultrapassa 64KB no momento do envio pelo worker; evento não é enviado nem truncado, vai direto para o fluxo de falha (`[09:23]-[09:24] Sofia/Diego`) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `id` de dead-letter inexistente no replay |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno, não HTTP) | Timeout de 10s na chamada ao endpoint do cliente; registrado como falha e segue para retry (`[09:42] Sofia/Diego`) |
| `WEBHOOK_FORBIDDEN_ROLE` | 403 | Usuário sem role `ADMIN` tentando acessar o endpoint de replay (herdado do `ForbiddenError` via `requireRole`) |

Erros de validação de schema (Zod) e erros de infraestrutura (Prisma) continuam tratados pelo `errorMiddleware` central, sem necessidade de tratamento especial no módulo de webhooks — reuso direto do que já existe (`[09:29] Bruno`).

### 8.3 Fallback e falhas do worker

| Situação | Tratamento |
| --- | --- |
| Timeout na chamada HTTP ao endpoint do cliente | Registrado como falha em `webhook_deliveries`; entra no fluxo de retry com backoff (`[09:42] Diego`) |
| Resposta HTTP não-2xx do endpoint do cliente | Tratado como falha, mesmo fluxo de retry do timeout |
| Erro de rede (DNS, conexão recusada) | Tratado como falha, mesmo fluxo de retry |
| Payload maior que 64KB no momento do envio | Evento não é enviado; falha imediata sem consumir tentativa de rede, segue para o mesmo contador de `attempts` (`[09:23]-[09:24] Sofia/Diego`) |
| 5ª tentativa também falha | Evento sai da outbox e vai para `webhook_dead_letter`, com motivo da última falha registrado |
| Worker encerra (SIGINT/SIGTERM) no meio de um ciclo | Segue o mesmo padrão de shutdown gracioso de `src/server.ts`: finaliza o lote em processamento antes de desconectar do Prisma |

Não existe modo de fallback automático de canal (como envio por e-mail) em caso de indisponibilidade do cliente: essa possibilidade foi levantada e recusada nesta fase (`[09:37]-[09:38] Marcos/Larissa`). A única rede de segurança para falha permanente é a dead-letter com reprocessamento manual via endpoint admin.

## 9. Métricas, logs e tracing

Métricas:

- `webhook_outbox_published_total`: eventos inseridos na outbox, contador.
- `webhook_delivery_attempts_total{success}`: tentativas de entrega, com label de sucesso ou falha.
- `webhook_dead_letter_total`: eventos movidos para dead-letter, contador.
- `webhook_delivery_duration_ms`: histograma de duração da chamada HTTP ao endpoint do cliente.
- `webhook_publish_to_first_attempt_ms`: histograma de latência ponta a ponta, da mudança de status à primeira tentativa de envio — métrica direta do objetivo de latência abaixo de 10 segundos (`[09:02] Marcos`).

Logs estruturados (Pino):

- O worker loga, por tentativa de entrega, um evento estruturado com `outboxId`, `webhookId`, `customerId`, `attempt`, `httpStatus` (quando houver), `durationMs` e `success`.
- Falhas terminais (envio para dead-letter) geram log em nível `warn` com o motivo.
- O replay administrativo loga o `userId` do operador, `deadLetterId` e o novo `outboxId` gerado, em nível `info`, servindo como trilha de auditoria (`[09:36] Sofia`).
- Reuso direto da instância `logger` global (`src/shared/logger/index.ts`), sem transporte ou sink novo. A secret do webhook nunca deve aparecer em nenhum campo logado — não há redação automática por nome de campo customizado nos paths já configurados, então essa é uma regra de implementação a reforçar em revisão de código.

Tracing:

- Cada linha da outbox carrega o `eventId` (UUID) desde a criação; esse identificador é propagado no header `X-Event-Id` da chamada HTTP e em todas as linhas de log e de `webhook_deliveries` relacionadas, permitindo reconstruir o caminho completo de um evento — da inserção na transação até a entrega ou falha final — sem precisar de um sistema de tracing distribuído dedicado, que não existe hoje no projeto.

## 10. Dependências e compatibilidade

| Componente | Versão / observação |
| --- | --- |
| Node.js | ≥ 20, mesma versão já usada pela API |
| MySQL | via Prisma `5.22.0`, mesma instância já provisionada |
| Prisma Client | instância própria no worker (processo separado da API) |
| Pino | logger já existente, reaproveitado sem mudança |

Compatibilidade:

- Nenhuma dependência de infraestrutura nova é introduzida (sem Redis, sem broker de mensagens); outbox e DLQ vivem no MySQL já provisionado (`[09:07] Diego`).
- O worker precisa de sua própria instância de `PrismaClient`, pois `PrismaClient` é por processo; conecta na mesma `DATABASE_URL` da API, mas como processo Node independente (`[09:29]-[09:30] Diego/Bruno`).
- Migração de schema via Prisma Migrate, seguindo o padrão já usado (`prisma/migrations/`).

## 11. Integração com o sistema existente

- **[`src/modules/orders/order.service.ts`](../src/modules/orders/order.service.ts):** o método `changeStatus` (linhas 126-179) é estendido para chamar `publishWebhookEvent(tx, order, from, to)` dentro da mesma transação Prisma, logo após `tx.orderStatusHistory.create(...)` e antes do `refreshed`. A função recebe o `TxClient` da transação em andamento — o mesmo padrão já usado internamente por `debitStock` e `replenishStock`, que também recebem `tx` em vez de abrir transação própria (`[09:41] Bruno/Diego`). Se `publishWebhookEvent` lançar, a transação inteira sofre rollback automaticamente, do mesmo jeito que já acontece hoje com `InsufficientStockError` dentro de `debitStock`.

- **[`src/shared/errors/http-errors.ts`](../src/shared/errors/http-errors.ts):** as classes de erro do módulo de webhooks seguem o mesmo padrão de `InvalidStatusTransitionError` e `InsufficientStockError`: estendem `ConflictError`, `UnprocessableEntityError` ou `NotFoundError` conforme o caso, com códigos prefixados `WEBHOOK_*`. Nenhuma mudança é necessária nessas classes-base; o módulo apenas as reutiliza e estende, por exemplo `class WebhookNotFoundError extends NotFoundError` ou `class WebhookInvalidUrlError extends BadRequestError`.

- **[`src/middlewares/auth.middleware.ts`](../src/middlewares/auth.middleware.ts):** o endpoint `POST /api/admin/webhooks/dead-letter/:id/replay` usa `requireRole('ADMIN')` exatamente como já é exportado hoje, sem alteração no middleware. Os demais endpoints do módulo usam apenas `authenticate`, seguindo o mesmo padrão do router de customers (`buildCustomerRouter`).

- **[`src/middlewares/error.middleware.ts`](../src/middlewares/error.middleware.ts):** nenhuma mudança necessária. O middleware já trata qualquer instância de `AppError` (incluindo as novas `WEBHOOK_*`), `ZodError` de validação de schema e erros conhecidos do Prisma, então os erros do módulo de webhooks fluem pelo mesmo tratamento central.

- **[`src/shared/logger/index.ts`](../src/shared/logger/index.ts):** o worker importa a instância `logger` já configurada (Pino, com redação de campos sensíveis) em vez de instanciar um logger próprio. Os campos de log do worker (`outboxId`, `webhookId`, etc.) não colidem com os paths de redação existentes (`req.headers.authorization`, `*.password`, `*.token`), mas a secret do webhook nunca deve ser logada.

- **[`src/routes/index.ts`](../src/routes/index.ts):** o `buildApiRouter` ganha uma nova entrada `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e `router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks))`, seguindo exatamente o padrão de registro já usado para `customers`, `products` e `orders`.

- **[`prisma/schema.prisma`](../prisma/schema.prisma):** adiciona os quatro models novos (`WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter`) seguindo as convenções já usadas no schema: `id` como `String @id @default(uuid()) @db.Char(36)`, `@map` para nome de tabela em snake_case, índices explícitos nos campos usados em filtro (como já ocorre em `Order` e `Customer`).

- **[`src/server.ts`](../src/server.ts):** não é alterado. Serve de modelo direto para o novo `src/worker.ts`: mesmo padrão de bootstrap assíncrono, mesmo tratamento de `SIGINT`/`SIGTERM` para desligamento gracioso (o worker deve terminar o ciclo de polling atual antes de encerrar, e chamar `prisma.$disconnect()`).

## 12. Critérios de aceite técnicos

- Uma chamada a `changeStatus` que falhe em qualquer ponto — incluindo a inserção do evento de webhook — não deixa nem a mudança de status nem o evento persistidos.
- Um evento inserido na outbox é processado pelo worker em até 2 ciclos de polling (4 segundos) na ausência de backlog.
- Uma falha de entrega gera nova tentativa respeitando exatamente a progressão 1m/5m/30m/2h/12h.
- Após a 5ª tentativa falha, o evento aparece em `webhook_dead_letter` e não é mais processado pelo worker.
- Um replay administrativo recoloca o evento na outbox como `pending` e fica registrado em log com o `userId` de quem executou.
- Toda chamada HTTP de saída carrega `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` válidos.
- Uma URL não-HTTPS é recusada no cadastro com `WEBHOOK_INVALID_URL`, sem chegar a persistir.
- Um payload de evento acima de 64KB nunca é enviado truncado; a tentativa falha explicitamente.
- A secret completa só é retornada na criação e na rotação; nenhum outro endpoint a expõe.

## 13. Riscos e mitigação

| Risco | Impacto | Mitigação |
| --- | --- | --- |
| Worker cai no meio do processamento de um lote e deixa linhas em `status = 'processing'` indefinidamente | Evento fica "preso", nunca reentregue nem movido para dead-letter | Fora do escopo desta versão um mecanismo de recuperação automático; documentar como limitação conhecida e considerar um `processingTimeoutAt` em versão futura |
| Escala de clientes aumenta volume da outbox e o polling de 2s não dá conta do lote | Latência de entrega piora, objetivo de 10s deixa de ser cumprido | Índice em `(status, nextAttemptAt)` mantém a leitura eficiente; ajuste de tamanho de lote é parâmetro de configuração, não mudança de código |
| Secret armazenada em texto plano no banco | Vazamento de dados do banco expõe secrets de todos os clientes | Aceito nesta fase como consistente com o padrão já usado para outros segredos no projeto; revisão de segurança da Sofia antes do deploy deve validar essa decisão explicitamente (`[09:46] Sofia`) |
| Cliente recebe evento duplicado e não implementa deduplicação corretamente | Cliente processa a mesma mudança de status mais de uma vez do lado dele | Documentação clara do contrato (`X-Event-Id`) no portal do desenvolvedor, responsabilidade explícita do lado do cliente, alinhada ao padrão de mercado (`[09:26] Marcos`) |

## Anexos / Referências

- [PRD — Sistema de Webhooks de Notificação de Pedidos](./PRD.md)
- [RFC — Sistema de Webhooks de Notificação de Pedidos](./RFC.md)
- [ADRs](./adrs/)
- [TRANSCRICAO.md](../TRANSCRICAO.md) — origem de todas as decisões e requisitos citados neste documento
