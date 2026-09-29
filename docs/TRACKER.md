# Tracker de Rastreabilidade

Este tracker liga cada item registrado nos documentos à sua origem: a transcrição da reunião (`TRANSCRICAO.md`) ou o código-fonte do repositório.

- **Fonte `TRANSCRICAO`:** a Localização aponta o primeiro turno de fala que sustenta o item, no formato `[hh:mm] Nome`. Quando o mesmo ponto é reforçado por outras falas, elas aparecem citadas no próprio documento.
- **Fonte `CODIGO`:** a Localização é o caminho real do arquivo no repositório.
- Itens marcados **(inferido)** ou **(desenho)** nos documentos são interpretações ou escolhas de implementação feitas a partir da fala indicada. A linha aponta a fala que motivou a inferência.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo por notificação de status | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-01 | `docs/PRD.md` | Problema | Clientes fazem polling em `GET /orders`, o que torna a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-02 | `docs/PRD.md` | Risco de negócio | Atlas pode migrar para o concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-03 | `docs/PRD.md` | Restrição | Prazo pedido pela Atlas: fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-PRB-04 | `docs/PRD.md` | Contexto | Aplicação sem nenhum mecanismo de notificação externa | CODIGO | `src/app.ts` |
| PRD-PUB-01 | `docs/PRD.md` | Público | Cadastro feito por usuários da plataforma que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | `docs/PRD.md` | Público | Admins reprocessam falhas | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-01 | `docs/PRD.md` | Cenário | Cliente assina só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | `docs/PRD.md` | Cenário | Cliente com manutenção planejada de 2h | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | `docs/PRD.md` | Cenário | Cliente que vazou a secret em log | TRANSCRICAO | [09:22] Diego |
| PRD-CEN-04 | `docs/PRD.md` | Cenário | Auditoria das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-05 | `docs/PRD.md` | Cenário | Admin reprocessa item da DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-01 | `docs/PRD.md` | Métrica | Latência p95 < 10s ("tempo real" para o cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Métrica | 3 de 3 clientes solicitantes ativos até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Métrica | 100% das notificações entregues após indisponibilidade de até ~15h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | `docs/PRD.md` | Métrica | Entrega em até 3 sprints, com a revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OUT-01 | `docs/PRD.md` | Fora de escopo | Aviso por e-mail em falhas repetidas (adiado para a próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual (projeto do time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | `docs/PRD.md` | Fora de escopo | Rate limiting de saída ("observar e decidir depois") | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | `docs/PRD.md` | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | `docs/PRD.md` | Fora de escopo | Ordem global e múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-OUT-06 | `docs/PRD.md` | Fora de escopo | Arquivamento de linhas entregues depois de ~30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-07 | `docs/PRD.md` | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Notificar quando o pedido muda de status | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Cadastro com URL e status desejados, com secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Cadastro via API com JWT, sem extrair `customer_id` do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | PATCH, DELETE e GET (listar por customer) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com validade paralela de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Retry 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | DLQ com payload, motivo e data | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Replay de DLQ por ADMIN, registrando o autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | `X-Event-Id` para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Notificação assinada (`X-Signature`) | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Headers `X-Webhook-Id` e `X-Timestamp` | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Payload enxuto, sem os itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-15 | `docs/PRD.md` | Requisito Funcional | Notificação reflete o estado no momento da mudança | TRANSCRICAO | [09:52] Larissa |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência < 10s, com espera de até 2s aceita | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Nenhuma mudança de status sem notificação registrada | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | At-least-once, documentado para o cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Somente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Credencial exclusiva por webhook | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Limite de 64KB, com erro e sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10s por tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Cliente lento não afeta a mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Ordem por pedido, sem ordem global | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-11 | `docs/PRD.md` | Restrição | Sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança da Sofia (≥ 2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Confirmação do prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Clientes preparados para HMAC e deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Usuários que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-DEP-06 | `docs/PRD.md` | Dependência | Sessão de revisão do design com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| PRD-RSK-01 | `docs/PRD.md` | Risco | Atraso do prazo e churn da Atlas | TRANSCRICAO | [09:00] Marcos |
| PRD-RSK-02 | `docs/PRD.md` | Risco | Cliente fora do ar por muito tempo | TRANSCRICAO | [09:16] Diego |
| PRD-RSK-03 | `docs/PRD.md` | Risco | Vazamento de credencial do cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-04 | `docs/PRD.md` | Risco | Processamento duplicado pelo cliente | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-05 | `docs/PRD.md` | Risco | Volume alto sobrecarrega o cliente | TRANSCRICAO | [09:38] Diego |
| PRD-RSK-06 | `docs/PRD.md` | Risco | Crescimento contínuo da tabela | TRANSCRICAO | [09:07] Bruno |
| PRD-AC-01 | `docs/PRD.md` | Critério de Aceitação | Notificação assinada em < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | `docs/PRD.md` | Critério de Aceitação | Status não assinado não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-03 | `docs/PRD.md` | Critério de Aceitação | URL `http` recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-04 | `docs/PRD.md` | Critério de Aceitação | Secret devolvida na criação e nunca listada | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-05 | `docs/PRD.md` | Critério de Aceitação | Retry completo e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-06 | `docs/PRD.md` | Critério de Aceitação | Replay: 403 para OPERATOR e sucesso auditado para ADMIN | TRANSCRICAO | [09:36] Larissa |
| PRD-AC-07 | `docs/PRD.md` | Critério de Aceitação | Secret antiga válida por 24h após a rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-08 | `docs/PRD.md` | Critério de Aceitação | Histórico de até 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-09 | `docs/PRD.md` | Critério de Aceitação | Falha no registro impede a mudança de status | TRANSCRICAO | [09:41] Diego |
| PRD-TST-01 | `docs/PRD.md` | Estratégia de Teste | Testes de integração com Vitest e Supertest no padrão existente | CODIGO | `tests/orders.test.ts` |
| RFC-META-01 | `docs/RFC.md` | Metadado | Larissa abre o doc de design, e os participantes revisam | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Transação de status já pesada (orders, history, estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-02 | `docs/RFC.md` | Contexto | `changeStatus` em `$transaction` com estoque e histórico | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Captura transacional via outbox e `publishWebhookEvent` | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Worker em processo próprio, polling de 2s e instância única | TRANSCRICAO | [09:11] Diego |
| RFC-PROP-03 | `docs/RFC.md` | Decisão | Timeout de 10s, 5 retentativas e DLQ com replay | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | HMAC-SHA256, secret por endpoint, rotação de 24h, https | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Novo módulo nos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| RFC-PROP-07 | `docs/RFC.md` | Decisão | CRUD para qualquer role autenticada, replay só ADMIN | TRANSCRICAO | [09:37] Sofia |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | Disparo síncrono no `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Redis Streams / infraestrutura nova | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Trigger de banco reativo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Worker no processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-07 | `docs/RFC.md` | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-OQ-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:38] Diego |
| RFC-OQ-02 | `docs/RFC.md` | Questão em aberto | Escala para múltiplos workers (particionar ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-OQ-03 | `docs/RFC.md` | Questão em aberto | E-mail em falhas repetidas | TRANSCRICAO | [09:37] Marcos |
| RFC-OQ-04 | `docs/RFC.md` | Questão em aberto | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-OQ-05 | `docs/RFC.md` | Questão em aberto | Endurecer as permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-OQ-06 | `docs/RFC.md` | Questão em aberto (inferido) | Retry pode inverter a ordem de eventos do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| RFC-OQ-07 | `docs/RFC.md` | Questão em aberto (inferido) | Contagem: envio inicial + 5 retentativas? | TRANSCRICAO | [09:17] Diego |
| RFC-OQ-08 | `docs/RFC.md` | Questão em aberto (inferido) | Criptografia em repouso da secret e timestamp fora da assinatura | TRANSCRICAO | [09:44] Diego |
| RFC-OQ-09 | `docs/RFC.md` | Questão em aberto (inferido) | Como o admin descobre o id da DLQ | TRANSCRICAO | [09:18] Diego |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | Falha de enfileiramento bloqueia a mudança de status (intencional) | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | `docs/RFC.md` | Impacto | Segundo processo para operar | TRANSCRICAO | [09:11] Larissa |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Outbox em MySQL, na mesma transação do status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Síncrono fora de questão | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Redis Cluster é overengineering para um time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-CTX | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto (código) | `changeStatus`, `canTransition`, `debitStock`/`replenishStock` | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-IDX | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Status (pendente/processando/falhou/entregue) e índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| ADR-001-UUID | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | IDs UUID como no resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em polling de 2s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa descartada | Trigger/NOTIFY: MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-PROC | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | `src/worker.ts` *(novo)* e `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-PRISMA | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | `PrismaClient` próprio, mesma `DATABASE_URL` | TRANSCRICAO | [09:30] Bruno |
| ADR-002-LIM | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Limitação | Ordem só por `order_id` com single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-002-COD | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Contexto (código) | Único entry-point atual e cliente Prisma | CODIGO | `src/server.ts` |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Decisão | 5 tentativas, 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Alternativa descartada | 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT-03 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Alternativa descartada | `failed` na própria outbox em vez de tabela de DLQ | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DLQ | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Decisão | Tabela `webhook_dead_letter` | TRANSCRICAO | [09:18] Diego |
| ADR-003-REPLAY | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Decisão | Replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-003-COD | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Reuso (código) | `requireRole('ADMIN')` | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-003-TRD | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Trade-off | Atraso de ~15h aceitável | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALG | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Decisão | SHA-256 como padrão de mercado | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CFG | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Configuração com URL, secret, customer_id e ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-TLS | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Restrição | HTTPS obrigatório via schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-NEG | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Trade-off | Secret recuperável, diferente de `passwordHash` | CODIGO | `prisma/schema.prisma` |
| ADR-005 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão | At-least-once com `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Alternativa descartada | Exactly-once exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-TRD | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Trade-off | Responsabilidade de deduplicar vai para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Módulo em `src/modules/webhooks` *(novo)* | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Classes de erro com prefixo `WEBHOOK_` | TRANSCRICAO | [09:28] Bruno |
| ADR-006-LOG | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Pino e error middleware sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-FN | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | `publishWebhookEvent(tx, …)` em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-006-COD-01 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Padrão existente | `AppError` (statusCode, errorCode, details) | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-COD-02 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Padrão existente | `InsufficientStockError`, `InvalidStatusTransitionError` | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-COD-03 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Padrão existente | Tratamento centralizado de AppError, Zod e Prisma | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-COD-04 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Padrão existente | Logger Pino com redaction | CODIGO | `src/shared/logger/index.ts` |
| ADR-007 | `docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md` | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-FLT | `docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md` | Decisão | Filtro de status na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-PAY | `docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md` | Decisão | Campos do payload, sem os itens | TRANSCRICAO | [09:43] Diego |
| ADR-007-ALT-01 | `docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md` | Alternativa descartada | Renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT-02 | `docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md` | Alternativa descartada | Truncar payloads grandes | TRANSCRICAO | [09:23] Sofia |
| FDD-OBJ-01 | `docs/FDD.md` | Objetivo técnico | Latência < 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-02 | `docs/FDD.md` | Objetivo técnico | Atomicidade entre status e evento | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-03 | `docs/FDD.md` | Objetivo técnico | Tolerância de ~15h | TRANSCRICAO | [09:17] Diego |
| FDD-OBJ-04 | `docs/FDD.md` | Objetivo técnico | Autenticidade verificável (HMAC, secret por endpoint) | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-05 | `docs/FDD.md` | Objetivo técnico | Deduplicação pelo cliente | TRANSCRICAO | [09:25] Diego |
| FDD-OBJ-06 | `docs/FDD.md` | Restrição | Nenhuma dependência nova e error middleware inalterado | TRANSCRICAO | [09:29] Bruno |
| FDD-ESC-02 | `docs/FDD.md` | Exclusão | Webhooks inbound | TRANSCRICAO | [09:03] Sofia |
| FDD-ESC-03 | `docs/FDD.md` | Exclusão | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-ESC-01 | `docs/FDD.md` | Exclusão | Evento na criação do pedido fora de escopo (`create()` grava PENDING) | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-MOD-01 | `docs/FDD.md` | Modelo de dados | Novos modelos Prisma com UUID `Char(36)` | CODIGO | `prisma/schema.prisma` |
| FDD-MOD-02 | `docs/FDD.md` | Modelo de dados (desenho) | Uma linha por evento × webhook, com o mesmo `eventId` | TRANSCRICAO | [09:44] Sofia |
| FDD-MOD-03 | `docs/FDD.md` | Modelo de dados | Configuração com url, secret, customer e ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-FLX-01 | `docs/FDD.md` | Fluxo | Inserção na outbox dentro da transação, com rollback em falha | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-02 | `docs/FDD.md` | Fluxo | Worker lê pendentes em lote pequeno por `created_at` | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-03 | `docs/FDD.md` | Fluxo (desenho) | Recuperação de linhas presas em PROCESSING | TRANSCRICAO | [09:24] Diego |
| FDD-FLX-04 | `docs/FDD.md` | Fluxo | Retry agendado por `nextAttemptAt` | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-05 | `docs/FDD.md` | Fluxo | DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-06 | `docs/FDD.md` | Fluxo | Replay volta à outbox como pendente, com log do autor | TRANSCRICAO | [09:36] Sofia |
| FDD-FLX-07 | `docs/FDD.md` | Fluxo | Rotação com grace de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLX-08 | `docs/FDD.md` | Reuso (código) | `uuid` já é dependência (usado no request logger) | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | `POST /webhooks` devolve a secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | `GET /customers/:customerId/webhooks` (customer no path) | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | `PATCH /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | `DELETE /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | `POST /webhooks/:id/rotate-secret` | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | `GET /webhooks/:id/deliveries` (últimas 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Headers outbound (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type`) | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | Payload `order.status_changed` | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-10 | `docs/FDD.md` | Contrato | Prefixo `/api/v1`, formato de erro e `paginated()` | CODIGO | `src/routes/index.ts` |
| FDD-CONTRATO-11 | `docs/FDD.md` | Restrição (código) | `PENDING` não é destino de transição, então é recusado em `events` | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-CONTRATO-12 | `docs/FDD.md` | Contrato (desenho) | Assinatura dupla durante o grace de rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-01 | `docs/FDD.md` | Erro | `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | `docs/FDD.md` | Erro | Prefixo `WEBHOOK_` em todo o módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | `docs/FDD.md` | Erro (código) | `validate` converte ZodError em `VALIDATION_ERROR` | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-ERR-04 | `docs/FDD.md` | Erro (código) | `NotFoundError` tem código fixo `NOT_FOUND`, então é preciso subclasse própria | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERR-07 | `docs/FDD.md` | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-08 | `docs/FDD.md` | Erro | `FORBIDDEN` via `requireRole` no replay | TRANSCRICAO | [09:36] Larissa |
| FDD-ERR-09 | `docs/FDD.md` | Erro | `WEBHOOK_INVALID_EVENTS` (lista de status assinados) | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-05 | `docs/FDD.md` | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` (64KB) | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-06 | `docs/FDD.md` | Erro | `WEBHOOK_DELIVERY_TIMEOUT` (10s) | TRANSCRICAO | [09:42] Diego |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Sem fallback por outro canal (e-mail adiado) | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-03 | `docs/FDD.md` | Resiliência (desenho) | Não seguir redirect para preservar o TLS obrigatório | TRANSCRICAO | [09:23] Sofia |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logs estruturados com o Pino existente | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade (código) | Redaction de `*.secret` em `redactPaths` | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Alerta de idade do pendente > 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-04 | `docs/FDD.md` | Observabilidade | Métrica de volume por cliente para decidir rate limiting | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-05 | `docs/FDD.md` | Observabilidade (código) | Tracing por `requestId` (`X-Request-Id`) | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-OBS-06 | `docs/FDD.md` | Observabilidade (código) | Sem libs de métricas ou tracing no projeto | CODIGO | `package.json` |
| FDD-INT-01 | `docs/FDD.md` | Integração | `changeStatus` chama `publishWebhookEvent` no `tx` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Controller passa `req.id` | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | Novas classes de erro e export | CODIGO | `src/shared/errors/index.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | `requireRole('ADMIN')` na rota de replay | CODIGO | `src/modules/users/user.routes.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Novas variáveis no `envSchema` | CODIGO | `src/config/env.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Worker obtém o seu `PrismaClient` | CODIGO | `src/config/database.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | `src/worker.ts` *(novo)* no molde do bootstrap/shutdown | CODIGO | `src/server.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | DI manual em `buildControllers` | CODIGO | `src/app.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Listagem com `paginated()` | CODIGO | `src/shared/http/response.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Limpeza das novas tabelas nos testes | CODIGO | `tests/setup.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Factory `createTestWebhook` | CODIGO | `tests/helpers/factories.ts` |
| FDD-INT-12 | `docs/FDD.md` | Integração | Chaves novas no exemplo de ambiente | CODIGO | `.env.example` |
| FDD-INT-13 | `docs/FDD.md` | Integração | Worker na mesma stack e no mesmo banco | TRANSCRICAO | [09:11] Diego |
| FDD-DEP-01 | `docs/FDD.md` | Dependência (código) | MySQL 8.0 | CODIGO | `docker-compose.yml` |
| FDD-DEP-02 | `docs/FDD.md` | Dependência | Documentação no portal | TRANSCRICAO | [09:40] Marcos |
| FDD-CA-01 | `docs/FDD.md` | Critério de aceite técnico | Rollback quando o insert na outbox falha | TRANSCRICAO | [09:41] Diego |
| FDD-CA-02 | `docs/FDD.md` | Critério de aceite técnico | Sem assinante, sem linha na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-03 | `docs/FDD.md` | Critério de aceite técnico | Commit até POST < 10s | TRANSCRICAO | [09:10] Larissa |
| FDD-CA-04 | `docs/FDD.md` | Critério de aceite técnico | Headers e assinatura conferem | TRANSCRICAO | [09:44] Diego |
| FDD-CA-05 | `docs/FDD.md` | Critério de aceite técnico | Backoff completo e DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-CA-06 | `docs/FDD.md` | Critério de aceite técnico | Timeout vira retry | TRANSCRICAO | [09:42] Diego |
| FDD-CA-07 | `docs/FDD.md` | Critério de aceite técnico | Payload > 64KB vai à DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-08 | `docs/FDD.md` | Critério de aceite técnico | URL http recusada e secret só na criação | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-09 | `docs/FDD.md` | Critério de aceite técnico | Duas assinaturas durante 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-10 | `docs/FDD.md` | Critério de aceite técnico | Replay: 403 OPERATOR, 202 ADMIN e log do autor | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-11 | `docs/FDD.md` | Critério de aceite técnico | Deliveries limitadas a 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-12 | `docs/FDD.md` | Critério de aceite técnico | Reenvio após crash com o mesmo X-Event-Id | TRANSCRICAO | [09:24] Diego |
| FDD-CA-13 | `docs/FDD.md` | Critério de aceite técnico | Testes existentes seguem verdes | CODIGO | `tests/orders.test.ts` |
| FDD-RSK-01 | `docs/FDD.md` | Risco | Transação de `changeStatus` mais lenta | TRANSCRICAO | [09:04] Bruno |
| FDD-RSK-02 | `docs/FDD.md` | Risco | Vazamento de secret no cliente | TRANSCRICAO | [09:22] Diego |
| FDD-RSK-03 | `docs/FDD.md` | Risco | Worker parado sem alerta | TRANSCRICAO | [09:11] Diego |
| FDD-RSK-04 | `docs/FDD.md` | Risco | Envio duplicado | TRANSCRICAO | [09:24] Diego |
| FDD-RSK-05 | `docs/FDD.md` | Risco (inferido) | Secret legível no banco | TRANSCRICAO | [09:20] Sofia |
| FDD-RSK-06 | `docs/FDD.md` | Risco | Crescimento da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-RSK-07 | `docs/FDD.md` | Risco | Bombardeio do cliente | TRANSCRICAO | [09:38] Diego |
| FDD-RSK-08 | `docs/FDD.md` | Risco | Ordem invertida durante retry | TRANSCRICAO | [09:13] Larissa |
| FDD-PLAN-01 | `docs/FDD.md` | Plano | Três sprints (modelagem, worker/retry, CRUD/integração/HMAC) | TRANSCRICAO | [09:46] Larissa |
| FDD-PLAN-02 | `docs/FDD.md` | Plano | Revisão de segurança antes do deploy | TRANSCRICAO | [09:49] Sofia |
