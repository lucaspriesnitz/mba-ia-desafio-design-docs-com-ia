# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autora** | Larissa (Tech Lead), consolidando a reunião técnica ([09:50] Larissa: "Eu vou abrir o doc de design da feature") |
| **Status** | Em revisão |
| **Data** | 2026-09-28 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenheira de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por **webhooks outbound** sempre que um pedido muda de status. O evento é gravado numa **tabela outbox no MySQL, na mesma transação** da mudança de status. Um **worker em processo separado** lê essa tabela **a cada 2 segundos** e faz o POST HTTPS para o endpoint do cliente. O POST vai **assinado com HMAC-SHA256** (uma secret por endpoint) e carrega **`X-Event-Id`** para deduplicação, com garantia **at-least-once**. Falhas são retentadas com **backoff de 1m/5m/30m/2h/12h**. Esgotadas as tentativas, o evento vai para uma **DLQ** com replay manual restrito a `ADMIN`. Tudo é construído como mais um módulo, `src/modules/webhooks` (novo), reaproveitando os padrões existentes. A estimativa é de **3 sprints**, com a revisão de segurança incluída.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para ser notificados quando o status dos pedidos muda. Hoje eles fazem polling em `GET /orders`, o que deixa a integração "lenta e cara" ([09:00] Marcos). A Atlas sinalizou que pode migrar para o concorrente se não houver entrega até o fim do trimestre. O prazo pedido é o **fim de novembro** ([09:45] Marcos).

Para o cliente, "tempo real" é **menos de 10 segundos** ([09:02] Marcos). Hoje a aplicação não tem nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status acontece numa transação já pesada em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que atualiza `orders`, insere em `order_status_history` e mexe no estoque ([09:04] Bruno).

O desafio central é **notificar sem acoplar a latência e a disponibilidade do cliente a essa transação**, e sem perder eventos.

## 3. Proposta técnica

```
 API (src/server.ts)                          Worker (src/worker.ts, novo)
 ┌──────────────────────────────┐             ┌──────────────────────────────────┐
 │ PATCH /orders/:id/status     │             │ loop a cada 2s                   │
 │ changeStatus() $transaction: │             │  1. lê lote de PENDING vencidos  │
 │   update orders              │  MySQL      │  2. assina (HMAC-SHA256)         │
 │   insert order_status_history│ ┌─────────┐ │  3. POST https (timeout 10s)     │──► endpoint do cliente
 │   publishWebhookEvent(tx,...)├►│ outbox  │◄┤  4. 2xx → DELIVERED              │
 │   (só se algum webhook       │ └─────────┘ │     falha → agenda retry / DLQ   │
 │    assina o to_status)       │ ┌─────────┐ │  5. grava histórico de entregas  │
 └──────────────────────────────┘ │  DLQ    │◄┤                                  │
   CRUD /webhooks, replay (ADMIN) └─────────┘ └──────────────────────────────────┘
```

Os componentes da proposta, cada um com a decisão formal no ADR correspondente:

1. **Captura transacional (Outbox).** `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação. Se o insert falhar, tudo sofre rollback ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)). O evento é gravado como **snapshot enxuto**, e só é inserido se algum webhook ativo do customer assinar o `to_status` ([ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)).
2. **Despacho assíncrono.** Um worker em **processo próprio** (`npm run worker`) usa o mesmo banco e um `PrismaClient` próprio, e faz polling de 2s. Nesta fase há **uma única instância** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).
3. **Resiliência.** O timeout é de 10s, com 5 retentativas em backoff exponencial até cerca de 15h. Depois delas, o evento vai para a DLQ em tabela separada, com replay manual auditado ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
4. **Segurança.** O corpo é assinado com HMAC-SHA256, com uma secret por endpoint gerada pela plataforma. A rotação mantém a secret antiga válida por 24h em paralelo, e as URLs têm de ser `https` ([ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)).
5. **Semântica de entrega.** A entrega é at-least-once, e o `X-Event-Id` (UUID gerado na outbox) permite a deduplicação pelo cliente ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
6. **Encaixe no código.** O novo módulo `src/modules/webhooks` segue o padrão do projeto: `AppError` com códigos `WEBHOOK_*`, o error middleware sem alteração, o Pino, o Zod e o `requireRole` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).
7. **Superfície de API.** O cliente ganha um CRUD de webhooks (autenticado, qualquer role), a rotação de secret e o histórico de entregas. O replay de DLQ exige `ADMIN` ([09:31] a [09:36]). Os contratos, o modelo de dados e a matriz de erros estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| ID | Alternativa | Trade-off que motivou o descarte |
| --- | --- | --- |
| RFC-ALT-01 | **Disparo síncrono** do HTTP dentro de `changeStatus` | Um cliente lento travaria a mudança de status de outros pedidos. Um cliente fora do ar forçaria rollback de uma mudança de negócio válida ([09:04] Bruno; [09:06] Diego). |
| RFC-ALT-02 | **Redis Streams** (ou outra fila externa) | Exige subir e operar infraestrutura nova. Para um time pequeno é overengineering, e o MySQL já resolve ([09:07] Larissa; [09:07] Diego). |
| RFC-ALT-03 | **Trigger de banco** para acordar o worker | O MySQL não tem `LISTEN/NOTIFY`. Notificar um processo externo exigiria gambiarras (arquivo, endpoint). Polling de 2s já cumpre a meta de menos de 10s ([09:09] Diego). |
| RFC-ALT-04 | **Worker dentro do processo da API** | Um restart da API mataria o worker ([09:11] Diego). |
| RFC-ALT-05 | **Retry indefinido**, ou apenas **3 tentativas** | O indefinido deixa eventos pendurados para sempre. Com 3, uma janela de uns 30 min não cobre indisponibilidades reais de cerca de 2h ([09:15] a [09:16] Diego). |
| RFC-ALT-06 | **Secret global** da plataforma | Um vazamento comprometeria todos os clientes ([09:21] Sofia). |
| RFC-ALT-07 | **Exactly-once** | Exigiria coordenação dos dois lados, com complexidade desproporcional ao ganho ([09:25] Diego). |

## 5. Questões em aberto

| ID | Questão | Origem | Encaminhamento proposto |
| --- | --- | --- | --- |
| RFC-OQ-01 | **Rate limiting de saída por cliente.** 50 mudanças de status por minuto significam 50 chamadas ao cliente. | [09:38] Diego; [09:39] Larissa | Fora do escopo. Observar o volume por cliente em produção (métrica no FDD) e decidir depois. |
| RFC-OQ-02 | **Escala para múltiplos workers** sem perder a ordem por pedido | [09:13] Diego e Bruno | Hoje é single-worker. As opções futuras são particionar por `order_id` ou usar lock pessimista. |
| RFC-OQ-03 | **Notificar o cliente (e-mail) quando o webhook falha repetidamente** | [09:37] Marcos; [09:37] Larissa | Adiado para a próxima fase, "depois que a gente medir o impacto". |
| RFC-OQ-04 | **Arquivamento de linhas entregues** da outbox (algo como 30 dias) | [09:08] Diego | Fora do escopo desta feature. Precisa de um dono antes que a tabela cresça. |
| RFC-OQ-05 | **Endurecer as permissões do CRUD** de webhooks (hoje qualquer role autenticada) | [09:37] Sofia | "Mais pra frente a gente pode endurecer." |
| RFC-OQ-06 | **Ordem durante retries.** Com backoff, um evento posterior do mesmo pedido pode ser entregue antes de um anterior que está aguardando retry. A reunião só tratou da ordem com um único worker. | Inferido de [09:12] Diego + [09:17] Diego | Proposta: aceitar e documentar para o cliente (o `timestamp` e o par `from_status`/`to_status` permitem reconciliar). A alternativa é bloquear eventos de um pedido enquanto houver um anterior pendente. Decidir na revisão. |
| RFC-OQ-07 | **Contagem de tentativas.** "5 tentativas" com 5 intervalos de backoff dá 1 envio inicial + 5 retentativas (cerca de 15h após a primeira falha)? | [09:17] Diego; [09:48] Larissa | O FDD adota 1 + 5, coerente com "quase 15 horas entre primeira falha e última tentativa". Confirmar. |
| RFC-OQ-08 | **Pontos para a revisão de segurança:** criptografia em repouso da secret (ela precisa ser recuperável para assinar) e `X-Timestamp` fora da assinatura | Inferido de [09:20] Sofia + [09:44] Diego | Levar para a revisão da Sofia (pelo menos 2 dias úteis antes do deploy, [09:46] Sofia). |
| RFC-OQ-09 | **Como o admin descobre o `id` da DLQ para o replay.** A reunião definiu só o endpoint de replay. | Inferido de [09:18] Diego | Por ora, consulta operacional ao banco ou aos logs `webhook.dead_lettered`. Um endpoint de listagem da DLQ pode ser proposto em revisão. |

## 6. Impacto e riscos

**Impacto no sistema existente**
- **`changeStatus`:** ganha uma consulta aos webhooks ativos do customer e, no máximo, alguns inserts na mesma transação. Uma falha nesse ponto passa a bloquear a mudança de status, o que é intencional ([09:40] Bruno).
- **Banco:** ganha as tabelas de configuração, outbox, DLQ e histórico de entregas, além de uma leitura periódica a cada 2s.
- **Operação:** passa a haver um segundo processo (`npm run worker`) para deploy e monitoramento.
- **Clientes:** precisam validar HMAC e deduplicar por `X-Event-Id`. Marcos documenta isso no portal ([09:26], [09:40] Marcos).

**Riscos principais** (a matriz completa, com probabilidade e impacto, está no [PRD](PRD.md#riscos-e-mitigação))
- **Prazo:** 3 sprints contra o prazo de fim de novembro da Atlas ([09:46] Larissa).
- **Worker parado:** os eventos se acumulam sem perda, mas a latência passa de 10s. Precisa de alerta sobre a idade do evento pendente mais antigo.
- **Vazamento de secret no cliente:** mitigado pela secret por endpoint e pela rotação ([09:22] Diego).
- **Crescimento da outbox:** o arquivamento está fora do escopo (RFC-OQ-04).

## 7. Decisões relacionadas

- [ADR-001: Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker separado em polling de 2s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005: At-least-once com X-Event-Id](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007: Snapshot do payload e filtro na inserção](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)
