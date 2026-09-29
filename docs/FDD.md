# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Pronto para implementação (pendente da revisão de segurança) |
| **Proposta de origem** | [RFC](RFC.md) |
| **Decisões** | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |
| **Rastreabilidade** | [TRACKER](TRACKER.md) (IDs `FDD-*`) |

> Convenção: caminhos seguidos de *(novo)* são arquivos **a criar** propostos na reunião ([09:11] Larissa; [09:27] e [09:28] Bruno); todos os demais caminhos citados existem hoje no repositório. Itens marcados com **(desenho)** são escolhas de implementação feitas neste FDD para tornar uma decisão da reunião executável. Eles não foram discutidos literalmente e devem ser validados no code review.

---

## 1. Contexto e motivação técnica

A aplicação é um OMS em Node.js 20 + TypeScript, com Express, Prisma 5 sobre MySQL 8, Zod e Pino (`package.json`). Ela **não tem nenhum mecanismo de eventos, filas ou notificação externa**. Clientes B2B fazem polling em `GET /api/v1/orders` para descobrir mudanças de status ([09:00] Marcos).

O ponto de origem dos eventos é `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). Ele roda em `prisma.$transaction` e, nessa ordem, valida a transição (`canTransition` em `src/modules/orders/order.status.ts`), debita ou repõe estoque, atualiza `orders.status` e insere em `order_status_history`. Qualquer notificação precisa ser **atômica** com essa transação sem **adicionar I/O externo** dentro dela ([09:04] Bruno; [09:06] Diego).

## 2. Objetivos técnicos

| ID | Objetivo | Origem |
| --- | --- | --- |
| FDD-OBJ-01 | Latência de ponta a ponta (commit da mudança até o POST no cliente) **< 10s** em operação normal. O polling contribui com até 2s | [09:02] Marcos; [09:10] Larissa |
| FDD-OBJ-02 | **Nenhuma mudança de status sem evento** e nenhum evento sem mudança de status (atomicidade via outbox) | [09:06] Diego; [09:40] Bruno |
| FDD-OBJ-03 | Tolerar indisponibilidade do cliente por até **cerca de 15h** sem perda (backoff 1m/5m/30m/2h/12h), com DLQ depois disso | [09:17] Diego |
| FDD-OBJ-04 | Autenticidade e integridade verificáveis pelo cliente (HMAC-SHA256, secret por endpoint) | [09:19] a [09:22] Sofia |
| FDD-OBJ-05 | Deduplicação possível do lado do cliente (`X-Event-Id`) | [09:25] Diego |
| FDD-OBJ-06 | Zero alteração em `error.middleware.ts`, zero dependência nova de runtime | [09:29] Bruno; [09:30] Larissa |

## 3. Escopo e exclusões

**Incluso**
- Módulo `src/modules/webhooks` *(novo)* com o CRUD de configuração, a rotação de secret, o histórico de entregas e o replay de DLQ ([09:27] Bruno; [09:31] a [09:36]).
- As tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.
- O enfileiramento em `changeStatus` via `publishWebhookEvent`.
- O worker `src/worker.ts` *(novo)* e o script `npm run worker` ([09:11] Larissa).
- Um único tipo de evento, `order.status_changed` ([09:43] Diego).

**Excluído**
| Item | Origem |
| --- | --- |
| Webhooks inbound (cliente enviando para nós) | [09:02] Marcos; [09:03] Sofia |
| Notificação por e-mail em falhas repetidas (próxima fase) | [09:37] Larissa |
| Rate limiting de saída ("observar e decidir depois") | [09:38] Diego; [09:39] Larissa |
| Dashboard visual (projeto do time de frontend) | [09:40] Larissa |
| Arquivamento de linhas entregues (cerca de 30 dias) | [09:08] Diego |
| Múltiplos workers e ordem global | [09:13] Diego e Larissa |
| Exactly-once | [09:25] Diego |
| Evento na **criação** do pedido. A reunião tratou só da mudança de status em `changeStatus`, e `create()` grava o status inicial `PENDING` fora desse método | [09:40] Bruno; `src/modules/orders/order.service.ts` |

## 4. Modelo de dados

Mudanças em `prisma/schema.prisma`. Todos os IDs são UUID `Char(36)`, no padrão do schema ([09:51] Larissa).

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  DELIVERED    // entregue
  FAILED       // falhou (movido para DLQ ou descartado)
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)   // válida até previousSecretExpiresAt
  previousSecretExpiresAt DateTime?
  events                  Json                          // ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id                String              @id @default(uuid()) @db.Char(36)
  eventId           String              @db.Char(36)       // X-Event-Id
  webhookEndpointId String              @db.Char(36)
  orderId           String              @db.Char(36)
  eventType         String              @db.VarChar(64)    // "order.status_changed"
  payload           Json                                   // snapshot renderizado na inserção
  status            WebhookOutboxStatus @default(PENDING)
  attempts          Int                 @default(0)
  nextAttemptAt     DateTime            @default(now())
  lastError         String?             @db.VarChar(1000)
  requestId         String?             @db.VarChar(64)    // correlação com o request da API
  deliveredAt       DateTime?
  createdAt         DateTime            @default(now())
  updatedAt         DateTime            @updatedAt

  webhookEndpoint WebhookEndpoint   @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)
  deliveries      WebhookDelivery[]

  @@unique([eventId, webhookEndpointId])
  @@index([status, nextAttemptAt, createdAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id                String   @id @default(uuid()) @db.Char(36)
  outboxId          String   @db.Char(36)
  webhookEndpointId String   @db.Char(36)
  eventId           String   @db.Char(36)
  attempt           Int
  success           Boolean
  responseStatus    Int?
  responseBody      String?  @db.Text   // truncado em 64KB
  durationMs        Int
  errorCode         String?  @db.VarChar(64)
  createdAt         DateTime @default(now())

  outbox          WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhookEndpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([webhookEndpointId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id                String    @id @default(uuid()) @db.Char(36)
  outboxId          String    @unique @db.Char(36)
  webhookEndpointId String    @db.Char(36)
  eventId           String    @db.Char(36)
  payload           Json
  failureReason     String    @db.VarChar(1000)
  attempts          Int
  failedAt          DateTime  @default(now())
  replayedAt        DateTime?
  replayedById      String?   @db.Char(36)

  webhookEndpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas:
- **Outbox com uma linha por (evento × webhook) (desenho).** Um mesmo evento pode ir para vários endpoints do customer. Cada endpoint tem o seu próprio ciclo de retry, então cada par vira uma linha. O `eventId` é o mesmo em todas as linhas do evento, o que o mantém "único por evento" ([09:25] Diego). O `X-Webhook-Id` distingue o endpoint ([09:44] Sofia).
- **Índices.** Os índices de `status` e `created_at` pedidos na reunião ([09:08] Diego) ficam compostos com `nextAttemptAt`, que é o campo que o worker filtra para respeitar o backoff.
- **DLQ separada** com payload, motivo e timestamp ([09:18] Diego). A linha da outbox fica como `FAILED`, e a leitura dos pendentes continua limpa.
- **`Customer` ganha** a relação `webhookEndpoints WebhookEndpoint[]`. Com `onDelete: Cascade`, o `DELETE /customers/:id` existente continua funcionando.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro de `changeStatus`)

```
PATCH /api/v1/orders/:id/status
 └─ OrderService.changeStatus  ── prisma.$transaction(tx) ─────────────────────────┐
     1. findUnique(order) / canTransition / debit|replenish stock (inalterado)      │
     2. tx.order.update(status = to)                                                │
     3. tx.orderStatusHistory.create(...)                                           │
     4. publishWebhookEvent(tx, order, from, to, { requestId })        ◄── NOVO     │
          a. endpoints = tx.webhookEndpoint.findMany({ customerId, active: true })  │
          b. alvo = endpoints.filter(e => e.events.includes(to))                    │
          c. se alvo vazio → return (nenhuma linha)                                 │
          d. eventId = uuidv4(); payload = renderPayload(order, from, to, eventId)  │
          e. tx.webhookOutbox.createMany(alvo.map(e => ({ eventId,                  │
                 webhookEndpointId: e.id, orderId, payload, status: PENDING,        │
                 nextAttemptAt: now, requestId })))                                 │
     5. findUnique(refreshed) e retorno (inalterado)                                │
 └─ qualquer exceção em 1..5 → ROLLBACK de tudo (status, histórico, estoque, outbox)┘
```

- O filtro por status assinado acontece **na inserção**: sem assinante, não entra linha ([09:34] Bruno; [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)).
- O payload é um **snapshot**. Ele é renderizado aqui e nunca mais recalculado ([09:52] Larissa).
- Uma falha no insert propaga a exceção. O `$transaction` faz rollback e o cliente da API recebe 500 pelo `error.middleware.ts` ([09:40] Bruno).
- O `uuidv4` vem da dependência `uuid` que já existe (usada em `request-logger.middleware.ts`).

### 5.2 Processamento pelo worker

```
src/worker.ts (novo, bootstrap) → WebhookWorker.start()
 on start: recoverStuck(): PROCESSING com updatedAt < now - 60s → PENDING   (desenho)
 loop a cada WEBHOOK_POLL_INTERVAL_MS (2000):
   1. batch = findMany({ status: PENDING, nextAttemptAt <= now },
                       orderBy: createdAt asc, take: WEBHOOK_BATCH_SIZE (10))
   2. updateMany(ids, status: PROCESSING)
   3. para cada evento, em ordem de createdAt (sequencial):
        a. endpoint = findUnique(webhookEndpointId)
           - inexistente → linha já removida por cascade (nada a fazer)
           - active=false → status FAILED, lastError=WEBHOOK_INACTIVE, sem DLQ   (desenho)
        b. body = JSON.stringify(payload)
           - Buffer.byteLength(body) > 65536 → vai direto à DLQ, WEBHOOK_PAYLOAD_TOO_LARGE
        c. headers = buildHeaders(endpoint, eventId, body)   (seção 6.8)
        d. POST endpoint.url com fetch nativo, AbortSignal.timeout(10_000), redirect: 'manual'
        e. grava webhook_deliveries (attempt, success, status, body truncado, durationMs)
        f. 2xx → status DELIVERED, deliveredAt=now, attempts+1
           senão → fluxo de retry (5.3)
   4. aguarda o intervalo; em SIGTERM/SIGINT termina o evento corrente e desconecta o Prisma
```

- **Um único worker nesta fase.** Com o processamento sequencial por `createdAt`, a ordem por `order_id` fica preservada nos casos sem retry ([09:12] Diego). Ver a limitação na seção 8.4.
- **Recuperação de linhas presas em `PROCESSING` (desenho).** Se o worker morrer depois do POST e antes do update, a linha volta a `PENDING` e é reenviada. Isso é coberto pelo at-least-once ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
- **`redirect: 'manual'` (desenho).** Um 3xx conta como falha. Seguir redirecionamento poderia levar a uma URL `http`, contornando a regra de TLS obrigatório ([09:23] Sofia).

### 5.3 Retry com backoff

| Tentativa | Momento | Se falhar |
| --- | --- | --- |
| 1 (inicial) | ≤ 2s após o commit | agenda +1 min |
| 2 | +1 min | agenda +5 min |
| 3 | +5 min | agenda +30 min |
| 4 | +30 min | agenda +2 h |
| 5 | +2 h | agenda +12 h |
| 6 | +12 h (cerca de 14h36 após a 1ª falha) | **DLQ** |

- `RETRY_DELAYS_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]` ([09:17] Diego).
- Na falha da tentativa `n` (com `n ≤ 5`): `attempts = n`, `status = PENDING`, `nextAttemptAt = now + RETRY_DELAYS_MS[n-1]` e `lastError = <código>`.
- Na falha da tentativa 6, a linha segue para a DLQ (5.4).
- Contam como falha: timeout de 10s ([09:42] Diego), status HTTP fora de 2xx, erro de rede/TLS e 3xx.
- *Interpretação:* "5 tentativas" corresponde às 5 retentativas agendadas depois do envio inicial, que é o que fecha com "quase 15 horas entre primeira falha e última tentativa". Está registrado como [RFC-OQ-07](RFC.md#5-questões-em-aberto).

### 5.4 DLQ

Em uma única transação:
1. `webhook_dead_letter.create({ outboxId, webhookEndpointId, eventId, payload, failureReason, attempts })`
2. `webhook_outbox.update({ status: FAILED, lastError })`
3. log `warn` `webhook.dead_lettered` com `eventId`, `webhookId`, `customerId` e `failureReason`

Os motivos possíveis são `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` (último erro anexado) e `WEBHOOK_PAYLOAD_TOO_LARGE`.

### 5.5 Replay manual da DLQ

```
POST /api/v1/admin/webhooks/dead-letter/:id/replay   (authenticate + requireRole('ADMIN'))
 └─ $transaction:
     1. dl = findUnique(id)                 → 404 WEBHOOK_DEAD_LETTER_NOT_FOUND
     2. dl.replayedAt != null               → 409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (desenho)
     3. outbox.update(dl.outboxId, { status: PENDING, attempts: 0, nextAttemptAt: now, lastError: null })
     4. dl.update({ replayedAt: now, replayedById: req.user.id })
 └─ logger.info({ adminUserId, deadLetterId, eventId, webhookId }, 'webhook.dead_letter.replayed')
```

O evento volta **com o mesmo `eventId`**. Para o cliente é o mesmo evento, e a deduplicação por `X-Event-Id` continua valendo ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). O registro de quem fez o replay atende à auditoria pedida em [09:36] por Sofia.

### 5.6 Rotação de secret

```
POST /api/v1/webhooks/:id/rotate-secret
 1. newSecret = crypto.randomBytes(32).toString('hex')
 2. update: previousSecret = secret atual, previousSecretExpiresAt = now + 24h, secret = newSecret
 3. responde a nova secret (única vez em que ela aparece, além da criação)
```

Durante o grace de 24h, o worker assina com as duas secrets (6.8). Depois de `previousSecretExpiresAt`, só a nova é usada ([09:21] Sofia). Se houver uma nova rotação durante o grace, a secret "anterior" passa a ser a atual, e a mais antiga perde validade na hora **(desenho)**.

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (montadas em `src/routes/index.ts`) e exigem `Authorization: Bearer <JWT>` (`authenticate`). O CRUD aceita qualquer role autenticada ([09:37] Sofia). O replay exige `ADMIN` ([09:36] Sofia). O `customerId` vem no body ou no path, **nunca do JWT**, porque o JWT é do usuário operador ([09:32] Larissa). Os erros seguem o formato existente `{ "error": { "code", "message", "details?" } }`.

### 6.1 `POST /api/v1/webhooks`: cadastrar webhook

Request:
```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "5b1f0c1e-8a55-4c43-9a51-2f0c3b7d9e10",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30",
  "customerId": "5b1f0c1e-8a55-4c43-9a51-2f0c3b7d9e10",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2c4e7a1b8d3f6e0a5c7b9d2e4f6a8c1b3d5e7f9a0c2e4b6d8f0a1c3e5b7d9f",
  "createdAt": "2026-10-05T14:03:11.402Z",
  "updatedAt": "2026-10-05T14:03:11.402Z"
}
```
- A `secret` é gerada pela plataforma e devolvida **somente aqui e na rotação** ([09:31] Marcos).
- `events` precisa ser uma lista não vazia e sem repetição, com valores entre `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` e `CANCELLED`. `PENDING` é recusado porque nenhuma transição de `src/modules/orders/order.status.ts` tem `PENDING` como destino.

| Status | Quando |
| --- | --- |
| 201 | Criado |
| 400 | `VALIDATION_ERROR` (URL não-https com `details[].message` = `WEBHOOK_INVALID_URL`, `events` inválido etc.) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

### 6.2 `GET /api/v1/customers/:customerId/webhooks?page=1&pageSize=20`: listar webhooks de um customer

Request (sem body):
```http
GET /api/v1/customers/5b1f0c1e-8a55-4c43-9a51-2f0c3b7d9e10/webhooks?page=1&pageSize=20
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK` (formato `paginated()` de `src/shared/http/response.ts`):
```json
{
  "data": [
    {
      "id": "0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30",
      "customerId": "5b1f0c1e-8a55-4c43-9a51-2f0c3b7d9e10",
      "url": "https://hooks.atlascomercial.com.br/oms/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotationPendingUntil": null,
      "createdAt": "2026-10-05T14:03:11.402Z",
      "updatedAt": "2026-10-05T14:03:11.402Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
A `secret` **nunca** é listada. O `customerId` vai **no path**, uma das duas opções definidas em [09:32] por Larissa (no POST ele vai no body).

| Status | Quando |
| --- | --- |
| 200 | OK (lista possivelmente vazia) |
| 400 | `VALIDATION_ERROR` (`customerId` não-UUID, paginação inválida) |
| 401 | `UNAUTHORIZED` |

### 6.3 `PATCH /api/v1/webhooks/:id`: editar webhook

Request:
```http
PATCH /api/v1/webhooks/0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "url": "https://hooks.atlascomercial.com.br/v2/orders", "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Todos os campos são opcionais, mas é preciso ao menos um. `secret` e `customerId` não são editáveis por aqui.

Response `200 OK` (mesmo formato do item da listagem, sem a `secret`):
```json
{
  "id": "0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30",
  "customerId": "5b1f0c1e-8a55-4c43-9a51-2f0c3b7d9e10",
  "url": "https://hooks.atlascomercial.com.br/v2/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotationPendingUntil": null,
  "createdAt": "2026-10-05T14:03:11.402Z",
  "updatedAt": "2026-10-05T16:40:02.915Z"
}
```

| Status | Quando |
| --- | --- |
| 200 | Atualizado |
| 400 | `VALIDATION_ERROR` (inclui `WEBHOOK_INVALID_URL` em `details`) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

A mudança em `events` vale **para eventos futuros**. Os eventos já enfileirados seguem como snapshot ([ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)).

### 6.4 `DELETE /api/v1/webhooks/:id`: remover webhook

Request (sem body):
```http
DELETE /api/v1/webhooks/0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30
Authorization: Bearer eyJhbGciOi...
```

Response `204 No Content` (sem body).

| Status | Quando |
| --- | --- |
| 204 | Removido. Outbox, entregas e DLQ do endpoint são removidos por cascade |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret`: rotacionar secret

Request (sem body):
```http
POST /api/v1/webhooks/0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30/rotate-secret
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK`:
```json
{
  "id": "0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30",
  "secret": "4a6c8e0b2d4f6a8c0e2b4d6f8a0c2e4b6d8f0a2c4e6b8d0f2a4c6e8b0d2f4a6c",
  "previousSecretExpiresAt": "2026-10-06T14:10:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| 200 | Rotacionada. A antiga vale até `previousSecretExpiresAt` (24h) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.6 `GET /api/v1/webhooks/:id/deliveries`: histórico de entregas

Devolve as **últimas 100** tentativas, da mais recente para a mais antiga ([09:34] Marcos). Aceita `?limit=` entre 1 e 100, com padrão 100.

Request (sem body):
```http
GET /api/v1/webhooks/0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30/deliveries?limit=100
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "c1d2e3f4-0a1b-4c2d-8e3f-5a6b7c8d9e0f",
      "eventId": "7d3e9a10-2b4c-4e6f-8a1b-3c5d7e9f1a2b",
      "eventType": "order.status_changed",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "payload": { "event_id": "7d3e9a10-2b4c-4e6f-8a1b-3c5d7e9f1a2b", "event_type": "order.status_changed", "...": "..." },
      "createdAt": "2026-10-05T15:21:07.118Z"
    },
    {
      "id": "b0c1d2e3-9f0a-4b1c-8d2e-4f5a6b7c8d9e",
      "eventId": "7d3e9a10-2b4c-4e6f-8a1b-3c5d7e9f1a2b",
      "eventType": "order.status_changed",
      "attempt": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10001,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "payload": { "...": "..." },
      "createdAt": "2026-10-05T15:20:05.004Z"
    }
  ]
}
```

| Status | Quando |
| --- | --- |
| 200 | OK |
| 400 | `VALIDATION_ERROR` (`limit` fora de 1..100) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: reprocessar DLQ

Exige `requireRole('ADMIN')`. Request (sem body):
```http
POST /api/v1/admin/webhooks/dead-letter/e5f6a7b8-1c2d-4e3f-9a0b-6c7d8e9f0a1b/replay
Authorization: Bearer <JWT de um usuário ADMIN>
```

Response `202 Accepted`:
```json
{
  "deadLetterId": "e5f6a7b8-1c2d-4e3f-9a0b-6c7d8e9f0a1b",
  "outboxId": "a9b8c7d6-5e4f-4a3b-8c2d-1e0f9a8b7c6d",
  "eventId": "7d3e9a10-2b4c-4e6f-8a1b-3c5d7e9f1a2b",
  "status": "PENDING",
  "replayedAt": "2026-10-06T09:12:44.530Z",
  "replayedById": "3c4d5e6f-7a8b-4c9d-0e1f-2a3b4c5d6e7f"
}
```

| Status | Quando |
| --- | --- |
| 202 | Reenfileirado. A entrega acontece no próximo ciclo do worker |
| 401 | `UNAUTHORIZED` |
| 403 | `FORBIDDEN` (role diferente de `ADMIN`, via `requireRole`) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` |

### 6.8 Contrato do webhook enviado ao cliente (outbound)

```http
POST https://hooks.atlascomercial.com.br/oms/orders
Content-Type: application/json
X-Event-Id: 7d3e9a10-2b4c-4e6f-8a1b-3c5d7e9f1a2b
X-Webhook-Id: 0e7c2b9a-3f1d-4a8e-9c55-7d2e1f6a4b30
X-Timestamp: 2026-10-05T15:20:05.001Z
X-Signature: sha256=5d41402abc4b2a76b9719d911017c592ae2f...e3b0

{
  "event_id": "7d3e9a10-2b4c-4e6f-8a1b-3c5d7e9f1a2b",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-05T15:20:03.877Z",
  "order_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "order_number": "ORD-000123",
  "customer_id": "5b1f0c1e-8a55-4c43-9a51-2f0c3b7d9e10",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "total_cents": 18500
}
```

| Header | Conteúdo | Origem |
| --- | --- | --- |
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento, gerado na inserção da outbox | [09:25] Diego |
| `X-Webhook-Id` | `id` do endpoint cadastrado | [09:44] Sofia |
| `X-Timestamp` | Momento do envio (ISO 8601), para o cliente detectar replay se quiser | [09:44] Diego |
| `X-Signature` | `sha256=` + hex de `HMAC-SHA256(secret, rawBody)` | [09:20] Sofia |

- **Assinatura:** calculada sobre os **bytes exatos do corpo enviado** ([09:22] Sofia). O cliente deve comparar em tempo constante.
- **Durante o grace de rotação (desenho):** `X-Signature: sha256=<hmac com a nova>,sha256=<hmac com a anterior>`. O cliente aceita se **qualquer uma** bater. Assim a secret antiga "fica válida por 24h em paralelo" num fluxo outbound ([09:21] Sofia).
- **Payload:** os campos de [09:43] Diego. O `timestamp` é o instante da mudança de status (o snapshot). Os itens **não** vão; para eles, `GET /api/v1/orders/:id`.
- **Resposta esperada:** qualquer `2xx` é sucesso. O corpo da resposta é gravado (truncado em 64KB) no histórico.

## 7. Matriz de erros

As classes novas vão em `src/shared/errors/http-errors.ts`, estendendo `AppError` e suas subclasses, e são exportadas em `src/shared/errors/index.ts`, como `InsufficientStockError`. Uma subclasse própria é necessária porque `NotFoundError` tem o código fixo `NOT_FOUND`.

**Erros de API (síncronos, tratados pelo `error.middleware.ts`)**

| Código | HTTP | Classe | Quando | Origem |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError extends AppError` | `:id` de webhook inexistente | [09:28] Bruno |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError extends AppError` | `customerId` inexistente no cadastro | [09:29] Larissa (prefixo) |
| `WEBHOOK_INVALID_URL` | 400 | regra Zod em `webhook.schemas.ts` | URL não-https ou malformada. Chega como `VALIDATION_ERROR` com `details: [{ path: "url", message: "WEBHOOK_INVALID_URL" }]`, porque `validate.middleware.ts` converte todo `ZodError` em `ValidationError` | [09:23] Sofia; [09:28] Bruno |
| `WEBHOOK_INVALID_EVENTS` | 400 | regra Zod | `events` vazio, com repetição ou com status inválido/`PENDING` (mesma mecânica em `details`) | [09:33] Marcos |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError extends AppError` | `:id` de DLQ inexistente | [09:18] Diego |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `ConflictError(…, 'WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED')` | replay de item já reprocessado **(desenho)** | [09:36] Sofia (auditoria) |
| `FORBIDDEN` | 403 | `ForbiddenError` (existente, via `requireRole`) | replay sem role `ADMIN` | [09:36] Larissa |
| `UNAUTHORIZED` | 401 | `UnauthorizedError` (existente) | sem JWT ou JWT inválido | `src/middlewares/auth.middleware.ts` |

**Erros assíncronos (worker; gravados em `webhook_deliveries.errorCode`, `webhook_outbox.lastError` e nos logs, sem resposta HTTP)**

| Código | Quando | Tratamento |
| --- | --- | --- |
| `WEBHOOK_DELIVERY_TIMEOUT` | sem resposta em 10s ([09:42] Diego) | retry |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | resposta fora de 2xx (inclui 3xx) | retry |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | DNS, conexão recusada, falha de TLS | retry |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | 6ª tentativa falhou | DLQ |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | corpo > 64KB ([09:24] Larissa) | DLQ direto, sem envio e sem truncar |
| `WEBHOOK_SECRET_REQUIRED` | endpoint sem secret no momento de assinar (estado inconsistente) ([09:28] Bruno) | DLQ direto |
| `WEBHOOK_INACTIVE` | endpoint desativado depois do enfileiramento | `FAILED`, sem envio e sem DLQ **(desenho)** |

**Falha no enfileiramento** (dentro de `changeStatus`): o erro do Prisma se propaga, a transação faz rollback e a API responde `500 INTERNAL_SERVER_ERROR` pelo fluxo que já existe no `error.middleware.ts`. O log `Unhandled error in request` leva o `requestId`. Esse comportamento é intencional ([09:40] Bruno).

## 8. Estratégias de resiliência

1. **Timeout:** 10s por chamada, com `AbortSignal.timeout` ([09:42] Diego).
2. **Retries:** 5, com backoff fixo exponencial de 1m/5m/30m/2h/12h (5.3), agendados por `nextAttemptAt`. Não há sleep em memória, então o agendamento sobrevive a um restart do worker.
3. **DLQ:** tabela separada, com replay manual auditado (5.4, 5.5).
4. **Fallback e degradação:**
   - *Cliente fora:* os eventos se acumulam na outbox dele sem afetar os outros clientes nem a API.
   - *Worker fora:* os eventos ficam `PENDING` sem perda e são processados quando ele volta. As linhas `PROCESSING` órfãs são recuperadas no start.
   - *Banco fora:* a API já falha em `changeStatus`, e o worker loga o erro e tenta de novo no próximo ciclo sem derrubar o processo.
   - Não há fallback por outro canal nesta fase (e-mail adiado, [09:37] Larissa).
5. **Limitação conhecida de ordem:** a ordem é garantida por `order_id` apenas com single-worker e sem retry pendente. Um evento em backoff pode ser ultrapassado por um evento posterior do mesmo pedido. O cliente reconcilia por `timestamp` e por `from_status`/`to_status` ([RFC-OQ-06](RFC.md#5-questões-em-aberto)).
6. **Proteção do processo:** o limite de 64KB no envio e na gravação do corpo de resposta evita consumo excessivo de memória e de disco.

## 9. Observabilidade

O projeto não tem biblioteca de métricas nem de tracing (`package.json`), e a decisão foi não adicionar stack nova ([09:29] Bruno). Por isso a observabilidade se apoia no **Pino** e em **consultas às tabelas do módulo**.

### 9.1 Logs (Pino, `src/shared/logger/index.ts`)

Os logs são estruturados. O `msg` segue o `snake_case` já usado no projeto (`server_started`, `http_request`), com o prefixo `webhook.` para agrupar os eventos do módulo. O worker usa o mesmo `logger`, com `service` sobrescrito para `order-management-worker` via `logger.child`.

| Evento (`msg`) | Nível | Campos |
| --- | --- | --- |
| `webhook.event_enqueued` | info | `requestId`, `orderId`, `eventId`, `webhookIds`, `fromStatus`, `toStatus` |
| `webhook.delivery_succeeded` | info | `eventId`, `webhookId`, `customerId`, `attempt`, `responseStatus`, `durationMs`, `latencyMs` (commit → entrega) |
| `webhook.delivery_failed` | warn | os mesmos campos, mais `errorCode` e `nextAttemptAt` |
| `webhook.dead_lettered` | warn | `eventId`, `webhookId`, `customerId`, `failureReason`, `attempts` |
| `webhook.dead_letter.replayed` | info | `adminUserId`, `deadLetterId`, `eventId`, `webhookId` ([09:36] Sofia) |
| `webhook.secret_rotated` | info | `webhookId`, `customerId`, `previousSecretExpiresAt`, `userId` |
| `webhook.worker.started` / `stopped` / `recovered_stuck` | info | `pollIntervalMs`, `batchSize`, `recovered` |
| `webhook.worker.cycle_error` | error | `err` |

**Segredos:** `'*.secret'` e `'*.previousSecret'` entram em `redactPaths`. A URL é logada, e o corpo de resposta do cliente não.

### 9.2 Métricas

Sem um backend de métricas, o worker emite a cada 30 ciclos (cerca de 1 min) um log `webhook.worker.stats`, calculado com consultas indexadas **(desenho)**. Essas métricas podem ser extraídas dos logs pela plataforma de logs em uso.

| Métrica | Fonte | Uso / alerta sugerido |
| --- | --- | --- |
| `outbox_pending_count` | `count(status=PENDING)` | tendência de backlog |
| `outbox_oldest_due_age_seconds` | `now - min(nextAttemptAt)` em `PENDING` vencidos | **alerta > 10s**, pois viola a meta de latência ([09:02] Marcos) |
| `delivery_latency_ms` p50/p95 | `latencyMs` dos logs de sucesso | meta p95 < 10s na 1ª tentativa |
| `delivery_success_rate` | sucesso / tentativas, por `customerId` | saúde por cliente |
| `dead_letter_count_24h` | `count(failedAt > now-24h)` | alerta em crescimento |
| `deliveries_per_minute` por `customerId` | `webhook_deliveries` | insumo para decidir rate limiting ([RFC-OQ-01](RFC.md#5-questões-em-aberto)) |
| heartbeat do worker | ausência de `webhook.worker.stats` por mais de 2 min | alerta de worker parado |

### 9.3 Tracing

Não há tracing distribuído no projeto. A correlação é feita por IDs:
- O `requestId` gerado em `src/middlewares/request-logger.middleware.ts` (header `X-Request-Id`) é gravado em `webhook_outbox.requestId` e repetido nos logs do worker. Isso liga o `PATCH /orders/:id/status` a cada tentativa de entrega.
- O `eventId` (`X-Event-Id`) liga os nossos logs aos logs do cliente em uma investigação conjunta.
- O `webhook_deliveries` guarda a linha do tempo completa por evento.
- OpenTelemetry fica como evolução futura, fora deste escopo ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

## 10. Integração com o sistema existente

| # | Arquivo existente | Integração |
| --- | --- | --- |
| 1 | `src/modules/orders/order.service.ts` | Em `changeStatus`, logo após `tx.orderStatusHistory.create(...)`, entra `await publishWebhookEvent(tx, order, from, to, { requestId })`, **dentro do mesmo `this.prisma.$transaction`** ([09:40] Bruno). O `order` vem do `findUnique` já feito no método e tem `id`, `orderNumber`, `customerId` e `totalCents`. `create()`, `debitStock` e `replenishStock` não mudam. `changeStatus` ganha um 4º parâmetro opcional, `context?: { requestId?: string }`. |
| 2 | `src/modules/orders/order.controller.ts` | `changeStatus` passa `{ requestId: req.id }` ao service. É a única mudança. |
| 3 | `src/modules/orders/order.status.ts` | É a fonte dos status válidos para assinatura. O schema de `events` é derivado de `OrderStatus` menos `PENDING`, que não é destino de nenhuma transição em `transitions`. Não é alterado. |
| 4 | `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` | Recebem `WebhookNotFoundError`, `WebhookCustomerNotFoundError` e `WebhookDeadLetterNotFoundError` (estendendo `AppError`, status 404). O conflito de replay usa `ConflictError` com o código `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`, como `InvalidStatusTransitionError` faz com `INVALID_STATUS_TRANSITION` ([09:28] Bruno). |
| 5 | `src/middlewares/error.middleware.ts` | **Sem alteração.** Todo erro novo é `AppError`, e erros Zod e Prisma (`P2002`/`P2025`) já são tratados ([09:29] Bruno). |
| 6 | `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas do módulo, com `router.use(authenticate)`, como em `order.routes.ts`. `requireRole('ADMIN')` na rota de replay, como `user.routes.ts` já faz ([09:36] Larissa). |
| 7 | `src/middlewares/validate.middleware.ts` | Todos os endpoints usam `validate({ body, query, params })` com os schemas de `webhook.schemas.ts`. As regras de URL https e de `events` ficam no Zod ([09:23] Sofia). |
| 8 | `src/middlewares/request-logger.middleware.ts` | É a fonte do `req.id` propagado para a outbox (tracing, 9.3). Não é alterado. |
| 9 | `src/shared/logger/index.ts` | O worker e o módulo usam o `logger` existente. `'*.secret'` e `'*.previousSecret'` entram em `redactPaths` ([09:29] Bruno). |
| 10 | `src/config/env.ts` | O `envSchema` ganha `WEBHOOK_POLL_INTERVAL_MS` (padrão 2000), `WEBHOOK_HTTP_TIMEOUT_MS` (padrão 10000) e `WEBHOOK_BATCH_SIZE` (padrão 10), todos com default. O `.env.example` recebe as três chaves. |
| 11 | `src/config/database.ts` | O worker importa `prisma`, e como `createPrismaClient()` executa por processo, ele obtém a **sua própria instância** com a mesma `DATABASE_URL` ([09:30] Bruno). |
| 12 | `src/server.ts` | Serve de molde para o novo `src/worker.ts`: `bootstrap()`, handlers de `SIGINT`/`SIGTERM` com `prisma.$disconnect()` e log `bootstrap_failed` em caso de erro ([09:11] Larissa). |
| 13 | `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController` (e o controller admin), com o mesmo padrão manual de DI. O tipo `Controllers` ganha `webhooks` e `adminWebhooks`. |
| 14 | `src/routes/index.ts` | `router.use('/webhooks', buildWebhookRouter(...))`, `router.use('/customers/:customerId/webhooks', buildCustomerWebhookRouter(...))` (com `Router({ mergeParams: true })`, registrado antes de `/customers`) e `router.use('/admin/webhooks', buildAdminWebhookRouter(...))`. O router existente de customers não muda. |
| 15 | `src/shared/http/response.ts` | `paginated()` na listagem de webhooks. |
| 16 | `prisma/schema.prisma` | Os modelos da seção 4, mais a relação `webhookEndpoints` em `Customer`. É gerada uma migração nova em `prisma/migrations/`. |
| 17 | `package.json` | Os scripts `"worker": "tsx --env-file=.env src/worker.ts"` (entry-point *(novo)*) e `"worker:start": "node --env-file=.env dist/worker.js"`, no molde de `dev`/`start`. **Nenhuma dependência nova:** `fetch` e `crypto` são nativos do Node 20, e `uuid` já existe. |
| 18 | `tests/setup.ts` e `tests/helpers/factories.ts` | O `beforeEach` limpa `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhookEndpoint` antes de `order*` e `customer`. Entra a nova factory `createTestWebhook`. |

**Arquivos novos:** `src/worker.ts`, `src/modules/webhooks/webhook.{controller,service,repository,routes,schemas}.ts`, `webhook.worker.ts` (loop), `webhook.publisher.ts` (`publishWebhookEvent`), `webhook.signer.ts` (HMAC), `webhook.admin.{controller,routes}.ts` e `tests/webhooks.test.ts`.

## 11. Dependências e compatibilidade

- **Runtime:** Node ≥ 20 (`engines`), com `fetch`, `AbortSignal.timeout` e `crypto.createHmac` nativos.
- **Banco:** MySQL 8.0 (`docker-compose.yml`), com Prisma 5.22. A migração é **aditiva** (tabelas novas e uma relação), sem alteração de colunas existentes.
- **Compatibilidade da API:** nenhum endpoint existente muda de contrato. O `PATCH /orders/:id/status` mantém request e response. O único efeito visível é que uma falha de enfileiramento passa a resultar em 500 com rollback.
- **Deploy:** um novo processo, `npm run worker:start`, em instância única. A ordem é migrar o banco, subir a API e depois o worker.
- **Pessoas e processos:** revisão de segurança da Sofia (pelo menos 2 dias úteis) antes do deploy ([09:46] Sofia). Documentação para os clientes no portal, a cargo do Marcos ([09:26], [09:40] Marcos).

## 12. Critérios de aceite técnicos

| ID | Critério |
| --- | --- |
| FDD-CA-01 | Uma mudança de status com webhook assinante gera exatamente 1 linha `PENDING` por endpoint assinante, na mesma transação. Com falha forçada no insert, `orders.status` e `order_status_history` **não** mudam. |
| FDD-CA-02 | Uma mudança de status sem webhook assinante do `to_status` **não** gera linha na outbox. |
| FDD-CA-03 | Com o worker rodando e o cliente respondendo 200, o intervalo entre o commit e o POST é < 10s (p95 em teste de integração local). |
| FDD-CA-04 | O POST contém `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` e `Content-Type: application/json`, e `X-Signature` confere com `HMAC-SHA256(secret, rawBody)`. |
| FDD-CA-05 | Com o cliente respondendo 500 sempre, as tentativas seguem +1m/+5m/+30m/+2h/+12h (verificado por `nextAttemptAt` com relógio controlado), e depois da 6ª falha existe uma linha em `webhook_dead_letter` e a outbox fica `FAILED`. |
| FDD-CA-06 | Um cliente que demora mais de 10s é registrado como `WEBHOOK_DELIVERY_TIMEOUT` e reagendado. |
| FDD-CA-07 | Um payload > 64KB vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem nenhum POST. |
| FDD-CA-08 | O cadastro com URL `http://` retorna 400. A criação retorna `secret`, e o GET de listagem nunca a retorna. |
| FDD-CA-09 | Depois de `rotate-secret`, durante 24h o `X-Signature` carrega as duas assinaturas. Depois de 24h, só a nova. |
| FDD-CA-10 | O replay com OPERATOR retorna 403. Com ADMIN, retorna 202, a outbox volta a `PENDING` com o mesmo `eventId`, e o log `webhook.dead_letter.replayed` contém `adminUserId`. |
| FDD-CA-11 | `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, ordenados do mais recente, com sucesso/falha, payload, resposta e `durationMs`. |
| FDD-CA-12 | O worker morto no meio do processamento e reiniciado reenvia os eventos presos em `PROCESSING` (com o mesmo `X-Event-Id`). |
| FDD-CA-13 | `npm test` e `npm run lint` passam, e os testes existentes de `tests/orders.test.ts` seguem verdes. |

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| A transação de `changeStatus` fica mais lenta com a consulta de webhooks e os inserts | Média | Médio | Índice `(customerId, active)`, `createMany` num único round-trip e nenhuma chamada externa dentro da transação |
| O worker para e ninguém percebe | Média | Alto | Heartbeat `webhook.worker.stats`, alerta sobre `outbox_oldest_due_age_seconds` > 10s e restart automático pelo orquestrador |
| Envio duplicado (timeout depois do processamento, crash do worker) | Alta | Baixo | Esperado pelo at-least-once. O `X-Event-Id` é estável entre retries e replays |
| Vazamento de secret no lado do cliente | Baixa | Alto | Secret por endpoint, rotação com grace de 24h e redaction no Pino ([09:22] Diego) |
| Secret legível no banco (necessária para assinar) | Baixa | Alto | Acesso restrito ao banco. A criptografia em repouso vai para a revisão da Sofia ([RFC-OQ-08](RFC.md#5-questões-em-aberto)) |
| Crescimento da outbox e do histórico | Alta (no longo prazo) | Médio | Índices por `status`/`createdAt`. O arquivamento está fora do escopo, mas é necessário ([RFC-OQ-04](RFC.md#5-questões-em-aberto)) |
| Cliente com alto volume sofre "bombardeio" | Média | Médio | Métrica `deliveries_per_minute` por customer e decisão posterior de rate limiting ([09:39] Larissa) |
| Ordem invertida durante retry | Baixa | Baixo | Documentar no portal. O payload traz `from_status`, `to_status` e `timestamp` ([RFC-OQ-06](RFC.md#5-questões-em-aberto)) |

## 14. Plano de entrega

Segundo a estimativa de [09:46] Larissa, são 3 sprints:

| Sprint | Entregas |
| --- | --- |
| 1 | Modelagem (seção 4) e migração da outbox e da DLQ |
| 2 | Worker (`src/worker.ts` *(novo)*, polling, envio) e retry/DLQ |
| 3 | CRUD de configuração e deliveries (meia sprint), integração com `order.service.ts` e testes ponta a ponta (meia sprint), HMAC, schemas e validações ("mais um pouco"), e a revisão de segurança da Sofia (pelo menos 2 dias úteis) antes do deploy |
