# ADR-006: Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Data da decisão:** reunião técnica de webhooks ([09:27] a [09:30])
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Status

Aceito

## Contexto

O OMS já tem convenções consolidadas, confirmadas no código:

| Padrão | Onde está |
| --- | --- |
| Módulo por domínio com `controller`, `service`, `repository`, `routes` e `schemas` (o módulo `auth` não tem `repository` e usa o de `users`) | `src/modules/{auth,users,customers,products,orders}/` |
| Classe base `AppError` (`statusCode`, `errorCode`, `details`) | `src/shared/errors/app-error.ts` |
| Erros específicos com código em SCREAMING_SNAKE_CASE (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) | `src/shared/errors/http-errors.ts` |
| Error middleware centralizado para `AppError`, `ZodError` e erros conhecidos do Prisma (`P2002`, `P2025`) | `src/middlewares/error.middleware.ts` |
| Validação com Zod via `validate({ body, query, params })` | `src/middlewares/validate.middleware.ts` |
| `authenticate` e `requireRole(...)` | `src/middlewares/auth.middleware.ts` |
| Logger Pino com redaction de campos sensíveis | `src/shared/logger/index.ts` |
| Composição manual de dependências e montagem das rotas em `/api/v1` | `src/app.ts`, `src/routes/index.ts` |
| IDs UUID em todas as entidades (a única exceção é o contador `order_number_sequence`, com `id` inteiro) | `prisma/schema.prisma` |

Uma feature nova é uma oportunidade para introduzir frameworks e padrões novos. O time decidiu explicitamente não fazer isso.

## Decisão

"Reuso máximo do que já existe" ([09:30] Larissa):

1. O **módulo fica em `src/modules/webhooks/`** (a criar), com a mesma estrutura dos demais: `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts` ([09:27] Bruno). A lógica do worker também fica no módulo ([09:28] Bruno).
2. **Os erros estendem as classes de `src/shared/errors/`**, com códigos de prefixo **`WEBHOOK_`** (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` etc.) ([09:28] Bruno; [09:29] Larissa).
3. **Nenhuma mudança no error middleware.** Como os novos erros são `AppError`, ele os trata sem alteração ([09:29] Bruno).
4. **O logger é o Pino existente**, sem biblioteca de log nova ([09:29] Bruno).
5. **A validação de entrada usa schemas Zod.** A exigência de URL `https` é uma regra de schema ([09:23] Sofia).
6. **A autorização usa `requireRole('ADMIN')`** no replay de DLQ ([09:36] Larissa).
7. **Os IDs são UUID**, como no resto do projeto ([09:51] Larissa).
8. **A integração com pedidos é feita por uma função** `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `tx` da transação atual. Não se injeta um repository inteiro no `OrderService` ([09:41] Bruno; [09:41] Diego).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Injetar um `WebhookRepository` no construtor do `OrderService`** | Acopla mais do que o necessário. Uma "função pura recebendo o tx" basta ([09:41] Diego). |
| **Tratamento de erro próprio no módulo (novo middleware ou formato de erro)** | Duplicaria o que `error.middleware.ts` já faz e quebraria a consistência do formato `{ error: { code, message, details } }` ([09:29] Bruno). |
| **Nova stack de log/observabilidade para o worker** | "Não vamos botar nada novo" ([09:29] Bruno). O Pino já está no projeto inteiro. |

## Consequências

**Positivas**
- A curva de aprendizado é zero para o time: o módulo tem a mesma cara dos outros.
- As respostas de erro são consistentes para os clientes da API.
- `changeStatus` sofre uma alteração mínima (uma chamada de função).

**Negativas**
- Herdamos as limitações atuais. Por exemplo, o projeto não tem biblioteca de métricas nem de tracing, o que limita a observabilidade à base de logs e consultas (ver [FDD](../FDD.md#observabilidade)).
- `OrderService` passa a depender de uma função do módulo de webhooks, uma dependência nova entre módulos.

**Trade-off explícito:** priorizamos consistência e velocidade de entrega sobre ferramentas potencialmente melhores (métricas nativas, tracing distribuído), que ficam para uma evolução futura.
