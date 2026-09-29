# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião técnica, em detalhamento |
| **Prazo alvo** | Fim de novembro (pedido da Atlas Comercial), estimado em 3 sprints |
| **Documentos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O Order Management System (OMS) passará a **avisar automaticamente os clientes B2B sempre que o status de um pedido deles mudar**, por meio de **webhooks**: uma chamada HTTPS para uma URL cadastrada pelo cliente. Hoje a plataforma não tem nenhum mecanismo de notificação externa, e os clientes precisam perguntar repetidamente pela API se algo mudou.

A feature nasceu de um pedido formal de três clientes B2B: **Atlas Comercial, MaxDistribuição e Nova Cargo** ([09:00] Marcos).

## 2. Problema e motivação

- **Integração lenta e cara para o cliente.** Os clientes fazem polling periódico em `GET /orders` para descobrir mudanças de status ([09:00] Marcos).
- **Risco comercial concreto.** A Atlas sinalizou que pode migrar para o concorrente se a solução não for entregue até o fim do trimestre ([09:00] Marcos). O prazo combinado é o **fim de novembro** ([09:45] Marcos).
- **Lacuna do produto.** Não existe hoje nenhum canal proativo de notificação sobre pedidos.

## 3. Público-alvo e cenários de uso

**Público**
- **Primário:** as equipes de integração dos clientes B2B (inicialmente Atlas, MaxDistribuição e Nova Cargo), que consomem a API do OMS com usuários que os representam ([09:32] Marcos).
- **Secundário:** administradores da plataforma (role `ADMIN`), que reprocessam notificações com falha ([09:36] Sofia).

**Cenários**
1. **Acompanhar a expedição.** A Nova Cargo cadastra um webhook para `SHIPPED` e `DELIVERED` e passa a receber só essas mudanças, sem polling ([09:33] Marcos).
2. **Cliente em manutenção.** O endpoint da MaxDistribuição fica fora do ar por 2h numa manutenção planejada. As notificações são retentadas e entregues quando ele volta ([09:16] Diego).
3. **Rotação de credencial.** A Atlas suspeita que vazou a secret. Ela pede uma nova pela API e tem 24h para atualizar os sistemas sem perder notificações ([09:21] Sofia; [09:22] Diego).
4. **Auditoria de entregas.** O cliente consulta os últimos 100 envios, com sucesso ou falha, payload, resposta e tempo, para investigar um pedido ([09:34] Marcos).
5. **Recuperação manual.** Um admin reprocessa uma notificação que esgotou as tentativas depois que o cliente corrigiu o problema ([09:18] Diego).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| PRD-OBJ-01 | Notificar em "tempo real" na percepção do cliente | Tempo entre a mudança de status e a entrega da notificação (1ª tentativa bem-sucedida) | **p95 < 10 segundos** ([09:02] Marcos). O desenho adiciona no máximo 2s de espera ([09:10] Larissa) |
| PRD-OBJ-02 | Atender os clientes que pediram a feature | Clientes solicitantes com webhook ativo recebendo eventos | **3 de 3** (Atlas, MaxDistribuição e Nova Cargo) até o **fim de novembro** ([09:00], [09:45] Marcos) |
| PRD-OBJ-03 | Não perder notificações por indisponibilidade temporária do cliente | Notificações entregues após indisponibilidade de até ~15h | **100%** entregues por retry, sem intervenção manual ([09:17] Diego e Marcos) |
| PRD-OBJ-04 | Entregar no prazo estimado | Sprints até o deploy, incluindo a revisão de segurança | **≤ 3 sprints** ([09:46], [09:47] Larissa) |

## 5. Escopo

### 5.1 Incluso
- Notificações **de saída** (da plataforma para o cliente) quando um pedido **muda de status**.
- Cadastro, edição, remoção e listagem de webhooks por cliente, com a escolha de quais status notificar.
- Assinatura de segurança de cada notificação, com rotação de credencial.
- Retentativas automáticas, fila de falhas definitivas (DLQ) e reprocessamento manual por admin.
- Histórico de entregas consultável pelo cliente.

### 5.2 Fora de escopo

| ID | Item | Situação | Origem |
| --- | --- | --- | --- |
| PRD-OUT-01 | **Aviso por e-mail** ao cliente quando o webhook falha repetidamente | **Adiado** para a próxima fase, "depois que a gente medir o impacto" | [09:37] Marcos; [09:37] Larissa |
| PRD-OUT-02 | **Dashboard visual** para o cliente ver os webhooks | **Descartado** nesta feature. É um projeto separado do time de frontend, e aqui entram só endpoints | [09:39] Marcos; [09:40] Larissa |
| PRD-OUT-03 | **Rate limiting** de envio por cliente | **Adiado**: "observar e decidir depois" | [09:38] Diego; [09:39] Larissa |
| PRD-OUT-04 | **Webhooks de entrada** (cliente enviando para nós) | **Descartado**. Os clientes só querem receber | [09:02] Marcos; [09:03] Sofia |
| PRD-OUT-05 | **Garantia de ordem global** entre pedidos e múltiplos workers | **Descartado nesta fase.** A ordem vale só por pedido | [09:13] Larissa; [09:14] Marcos |
| PRD-OUT-06 | **Arquivamento** de notificações entregues (cerca de 30 dias) | **Fora do escopo** desta feature | [09:08] Diego |
| PRD-OUT-07 | **Entrega exatamente uma vez** (exactly-once) | **Descartado**. O cliente deduplica | [09:25] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-FR-01 | Notificar os webhooks do cliente quando um pedido dele **muda de status** | [09:00] Marcos; [09:40] Bruno |
| PRD-FR-02 | Permitir ao cliente **cadastrar** um webhook informando a URL e a lista de status desejados. A plataforma **gera a secret e a devolve na criação** | [09:31] Marcos |
| PRD-FR-03 | O cadastro é feito pela **API da plataforma, com o JWT do sistema**. O cliente (`customer_id`) é informado no body ou no path, e **não é extraído do JWT** | [09:32] Marcos; [09:32] Larissa |
| PRD-FR-04 | Permitir **editar** (PATCH), **remover** (DELETE) e **listar** (GET) os webhooks de um cliente | [09:33] Bruno |
| PRD-FR-05 | Cada webhook escolhe **quais status** quer receber. Status não assinados não geram notificação | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-06 | Permitir ao cliente **consultar o histórico das últimas 100 entregas** de um webhook, com sucesso ou falha, payload, resposta e tempo de resposta (`GET /webhooks/:id/deliveries`) | [09:34] Marcos |
| PRD-FR-07 | Permitir ao cliente **rotacionar a secret** via API. A anterior continua válida **por 24h** | [09:21] Sofia |
| PRD-FR-08 | **Retentar automaticamente** as notificações com falha: 5 retentativas em 1m, 5m, 30m, 2h e 12h | [09:17] Diego; [09:17] Larissa |
| PRD-FR-09 | Mover para uma **fila de falhas definitivas (DLQ)**, com payload, motivo e data, as notificações que esgotarem as tentativas | [09:18] Diego |
| PRD-FR-10 | Permitir a um **ADMIN reprocessar** um item da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), **registrando quem fez** | [09:18] Diego; [09:36] Sofia |
| PRD-FR-11 | Cada notificação leva um **identificador único do evento** (`X-Event-Id`), para o cliente descartar duplicatas | [09:25] Diego |
| PRD-FR-12 | Cada notificação é **assinada** (`X-Signature`), para o cliente verificar a origem e a integridade | [09:19], [09:20] Sofia |
| PRD-FR-13 | Cada notificação informa **qual webhook** a originou (`X-Webhook-Id`) e o **momento do envio** (`X-Timestamp`) | [09:44] Sofia; [09:44] Diego |
| PRD-FR-14 | O conteúdo da notificação é **enxuto**: identificação do evento e do pedido, status anterior e novo, cliente, valor total e data/hora da mudança. **Os itens não vão** | [09:43] Diego; [09:44] Bruno |
| PRD-FR-15 | A notificação reflete o pedido **no momento da mudança de status**, mesmo que ele mude depois | [09:52] Larissa |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-NFR-01 | **Latência** abaixo de 10s entre a mudança de status e a notificação. A espera máxima de 2s introduzida pelo desenho é aceita | [09:02] Marcos; [09:10] Larissa |
| PRD-NFR-02 | **Nenhuma mudança de status sem notificação correspondente registrada.** Se a notificação não puder ser registrada, a mudança de status não acontece | [09:40] Bruno; [09:41] Diego |
| PRD-NFR-03 | **Entrega pelo menos uma vez** (at-least-once). Duplicatas são possíveis e devem estar documentadas para o cliente | [09:24] Diego; [09:26] Marcos |
| PRD-NFR-04 | **Somente HTTPS.** URLs `http` são recusadas no cadastro | [09:23] Sofia |
| PRD-NFR-05 | **Credencial exclusiva por webhook**, sem credencial global | [09:21] Sofia |
| PRD-NFR-06 | **Tamanho máximo de 64KB** por notificação. Acima disso, ela não é enviada e é tratada como erro, sem truncar | [09:23] Sofia; [09:24] Diego e Larissa |
| PRD-NFR-07 | **Timeout de 10s** por tentativa de envio | [09:42] Diego |
| PRD-NFR-08 | **Isolamento:** um cliente lento ou fora do ar não pode atrasar a mudança de status de nenhum pedido | [09:04] Bruno |
| PRD-NFR-09 | **Ordem por pedido** preservada nesta fase, sem garantia de ordem global | [09:12] Diego; [09:13] Larissa |
| PRD-NFR-10 | **Auditoria:** o reprocessamento manual registra o autor | [09:36] Sofia |
| PRD-NFR-11 | **Sem infraestrutura nova.** A solução usa o banco e a stack existentes | [09:07] Diego; [09:30] Larissa |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Registrar a notificação no próprio banco, junto com a mudança de status (outbox) | Consistência total com zero infraestrutura nova, ao custo de o banco também servir de fila | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Processo dedicado que verifica novas notificações a cada 2s | Simplicidade em troca de até 2s de atraso e de escala horizontal adiada | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em cerca de 15h, depois DLQ com reprocessamento manual | Tolera manutenções longas do cliente, com possível atraso de horas | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| Assinatura HMAC-SHA256 com uma credencial por webhook e rotação de 24h | Mais gestão de credenciais em troca de contenção de vazamentos | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| Entrega pelo menos uma vez, com deduplicação pelo cliente | Implementação simples e padrão de mercado, com a responsabilidade passada ao cliente | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Construir como mais um módulo, com os padrões do projeto | Velocidade e consistência, herdando as limitações atuais de observabilidade | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Notificação enxuta, congelada no momento da mudança | Fidelidade histórica, com o cliente precisando consultar os itens à parte | [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Origem |
| --- | --- | --- |
| PRD-DEP-01 | **Revisão de segurança da Sofia** (pelo menos 2 dias úteis) antes do deploy, focada em HMAC e geração de secret | [09:46] Sofia; [09:49] Sofia |
| PRD-DEP-02 | **Documentação no portal do desenvolvedor**, a cargo do Marcos, com destaque para duplicatas e deduplicação | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-03 | **Confirmação do prazo com a Atlas** (Marcos) | [09:47] Marcos |
| PRD-DEP-04 | **Clientes preparados** para validar a assinatura HMAC-SHA256 e deduplicar por `X-Event-Id` | [09:20] Sofia; [09:25] Diego |
| PRD-DEP-05 | **Usuários da plataforma que representam cada cliente**, usados para autenticar o cadastro | [09:32] Marcos |
| PRD-DEP-06 | **Sessão de revisão do design** com Bruno e Diego antes de começar a codar | [09:50] Larissa |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| PRD-RSK-01 | **Não entregar até o fim de novembro** e a Atlas migrar para o concorrente | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limiting fora), estimativa de 3 sprints com a revisão de segurança já incluída, e prazo confirmado com a Atlas ([09:00], [09:46], [09:47]) |
| PRD-RSK-02 | **Cliente fora do ar por muito tempo** perde notificações | Média | Alto | Retentativas por cerca de 15h, DLQ com evidência e reprocessamento manual por admin ([09:16] a [09:18] Diego) |
| PRD-RSK-03 | **Vazamento de credencial** do lado do cliente (já aconteceu antes) | Média | Alto | Credencial por webhook e rotação com convivência de 24h ([09:21] Sofia; [09:22] Diego) |
| PRD-RSK-04 | **Cliente processa a mesma notificação duas vezes** | Alta | Médio | `X-Event-Id` e documentação destacada no portal ([09:25] Diego; [09:26] Marcos) |
| PRD-RSK-05 | **Volume alto de notificações** sobrecarrega o cliente | Baixa | Médio | Monitorar o volume por cliente e decidir sobre rate limiting depois ([09:38] Diego; [09:39] Larissa) |
| PRD-RSK-06 | **Crescimento contínuo** do registro de notificações | Alta (no longo prazo) | Médio | Índices adequados. O arquivamento precisa ser tratado em uma fase seguinte ([09:08] Diego) |

## 11. Critérios de aceitação

| ID | Critério |
| --- | --- |
| PRD-AC-01 | Dado um webhook ativo assinando `SHIPPED`, quando um pedido do cliente vai para `SHIPPED`, então o endpoint recebe uma notificação assinada em menos de 10s. |
| PRD-AC-02 | Dado um webhook que assina só `DELIVERED`, quando o pedido vai para `SHIPPED`, então nenhuma notificação é enviada a ele. |
| PRD-AC-03 | Dado um cadastro com URL `http://`, então a API recusa com erro de validação. |
| PRD-AC-04 | Dado um cadastro válido, então a resposta contém a secret gerada pela plataforma, e a listagem posterior não a exibe. |
| PRD-AC-05 | Dado um endpoint do cliente fora do ar, então a notificação é retentada em 1m, 5m, 30m, 2h e 12h e, se todas falharem, aparece na DLQ com o motivo. |
| PRD-AC-06 | Dado um item na DLQ, quando um OPERATOR tenta reprocessar, então recebe 403. Quando um ADMIN reprocessa, a notificação é reenviada, com o mesmo `X-Event-Id`, e o autor fica registrado em log. |
| PRD-AC-07 | Dada uma rotação de secret, então por 24h as notificações podem ser validadas com a secret antiga ou com a nova, e depois disso só com a nova. |
| PRD-AC-08 | Dado um webhook com envios, então `GET /webhooks/:id/deliveries` mostra até as últimas 100 entregas, com sucesso ou falha, payload, resposta e tempo. |
| PRD-AC-09 | Dada uma falha ao registrar a notificação, então a mudança de status do pedido também não acontece. |

## 12. Estratégia de testes e validação

- **Testes de integração (Vitest + Supertest, como em `tests/orders.test.ts`)** para os endpoints de cadastro, edição, remoção, listagem, rotação, deliveries e replay, incluindo as permissões (OPERATOR x ADMIN).
- **Testes da transação de mudança de status**: verificar que a notificação é registrada junto com a mudança, que não é registrada quando não há assinante, e que há rollback quando o registro falha (PRD-AC-01, PRD-AC-02 e PRD-AC-09).
- **Testes do processo de envio** com um endpoint falso controlável (200, 500, lentidão acima de 10s, payload acima de 64KB) e relógio controlado, para verificar o backoff completo e a DLQ.
- **Teste de assinatura**: o cliente de teste recalcula o HMAC-SHA256 e compara, inclusive durante o período de rotação.
- **Revisão de segurança** da Sofia antes do deploy (PRD-DEP-01).
- **Validação com os clientes**: onboarding assistido de Atlas, MaxDistribuição e Nova Cargo, acompanhando a latência p95 (PRD-OBJ-01) e a taxa de sucesso por cliente nas primeiras semanas.
