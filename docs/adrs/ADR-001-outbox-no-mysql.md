# ADR-001: Padrão Outbox no MySQL existente para eventos de webhook

- **Data da decisão:** reunião técnica de webhooks (quinta-feira, [09:08])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)

## Status

Aceito

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser notificados quando o status dos seus pedidos muda. Hoje eles fazem polling em `GET /orders` ([09:00] Marcos).

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que:

1. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
2. debita ou repõe estoque (`debitStock` / `replenishStock`);
3. atualiza `orders.status`;
4. insere uma linha em `order_status_history`.

Bruno descreveu essa transação como "já pesada" ([09:04]). Um HTTP call dentro dela faria qualquer cliente lento travar a mudança de status de outros pedidos. E se o cliente estiver fora do ar, não é aceitável dar rollback na mudança de status ([09:04] Bruno).

Precisamos de um mecanismo que garanta duas coisas: **se o status mudou, o evento existe; se a transação falhou, o evento não existe** ([09:06] Diego). E isso sem acoplar a latência do cliente à transação.

## Decisão

Adotar o **padrão Transactional Outbox** sobre o **MySQL já usado pela aplicação** ([09:08] Larissa: "Tá decidido então: outbox em MySQL").

- Uma nova tabela `webhook_outbox` recebe o evento **dentro da mesma transação** de `changeStatus`, junto com `orders` e `order_status_history` ([09:06] Diego; [09:40] Bruno).
- Se o insert na outbox falhar, a transação inteira faz rollback. Não pode existir status alterado sem evento ([09:40] Bruno; [09:41] Diego: "Se ficar fora da transação, perde a garantia toda").
- A tabela tem um campo de status (pendente, processando, falhou, entregue) e índices em `status` e `created_at` ([09:08] Diego).
- As chaves primárias são UUID, seguindo o padrão do schema Prisma (`@default(uuid()) @db.Char(36)`) ([09:51] Larissa).
- Um worker separado consome a tabela (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Disparo HTTP síncrono dentro de `changeStatus`** | Um cliente lento trava a mudança de status de outros pedidos, e uma falha do cliente forçaria rollback de uma mudança de negócio legítima ([09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão"). |
| **Redis Streams (ou broker equivalente)** | Exigiria subir e operar infraestrutura nova (Redis Cluster). Para um time pequeno isso é overengineering, e o MySQL existente resolve ([09:07] Larissa; [09:07] Diego). Também perderia a atomicidade com a transação do MySQL sem um mecanismo adicional. |

## Consequências

**Positivas**
- A atomicidade vem de graça: o evento é gravado ou descartado junto com a mudança de status.
- Não entra infraestrutura nova. É o mesmo banco, o mesmo Prisma e o mesmo backup.
- A latência e a disponibilidade do cliente ficam isoladas da API de pedidos.

**Negativas**
- A transação de `changeStatus` ganha mais um insert (e uma consulta aos webhooks ativos do cliente), ficando um pouco mais longa.
- A tabela cresce continuamente. O arquivamento de linhas entregues (algo como 30 dias) foi explicitamente deixado **fora do escopo** desta feature ([09:08] Diego).
- O banco transacional passa a ser também uma fila. Carga de leitura do worker compete com a carga da API.

**Trade-off explícito:** aceitamos acoplar a fila ao banco transacional (e sua carga) em troca de consistência forte entre status e evento e de zero infraestrutura nova.
