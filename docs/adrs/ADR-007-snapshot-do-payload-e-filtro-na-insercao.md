# ADR-007: Payload enxuto renderizado como snapshot e filtro de eventos na inserção da outbox

- **Data da decisão:** reunião técnica de webhooks ([09:33] a [09:34], [09:43] e [09:51] a [09:52])
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Status

Aceito

## Contexto

Duas decisões secundárias afetam diretamente o modelo da outbox:

1. **O que a outbox guarda:** o payload já renderizado ou só o `order_id`, para renderizar no envio ([09:51] Bruno)? Um pedido pode mudar de novo entre a inserção do evento e o envio, principalmente com retries de até cerca de 15h ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
2. **Onde filtrar:** cada webhook escolhe os status que quer receber ([09:33] Marcos). O filtro pode acontecer na inserção da outbox ou no envio ([09:34] Diego).

## Decisão

1. **Snapshot na inserção.** O payload é renderizado no momento em que o evento entra na outbox, dentro da transação de `changeStatus`. Assim o evento reflete o estado do pedido **no instante da mudança de status** ([09:52] Larissa; [09:52] Diego; [09:52] Bruno).
2. **Payload enxuto:** `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Os itens não vão.** Quem quiser detalhes consulta `GET /orders/:id` ([09:43] Diego; [09:44] Bruno).
3. **Filtro na inserção.** Se nenhum webhook ativo do customer assina o `to_status`, **nenhuma linha é inserida** ([09:34] Bruno; [09:34] Diego).
4. **Teto de 64KB por payload.** Acima disso o evento **não é enviado e gera erro**, sem truncar ([09:23] Sofia; [09:24] Diego; [09:24] Larissa). Com o payload enxuto, na prática esse teto nunca deve ser atingido.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Guardar só o `order_id` e renderizar no envio** | O evento passaria a refletir o estado atual, e não o estado da transição, o que cria "caso esquisito" ([09:52] Larissa). |
| **Filtrar na hora do envio** | Grava linhas que nunca serão enviadas e ocupa a tabela à toa ([09:34] Bruno: "Economiza linha na tabela"). |
| **Enviar o pedido completo, com itens** | Infla o payload sem necessidade. O cliente pode buscar o detalhe sob demanda ([09:43] Diego). |
| **Truncar payloads grandes** | Esconde um sintoma de erro. "Se chegou nesse tamanho, tem algo errado" ([09:23] Sofia). |

## Consequências

**Positivas**
- O evento é imutável e historicamente correto, e um replay da DLQ envia exatamente o que foi gerado.
- A outbox só guarda o que será enviado, e o worker não precisa consultar `orders`.

**Negativas**
- Uma mudança nas assinaturas de um webhook **não afeta eventos já enfileirados**. Um webhook que deixa de assinar um status ainda recebe o que já estava na outbox.
- O cliente precisa de uma chamada extra para obter os itens do pedido.
- O filtro adiciona uma consulta aos webhooks ativos do customer dentro da transação de `changeStatus`.

**Trade-off explícito:** trocamos atualidade (dados "frescos" no envio) por fidelidade histórica e simplicidade no worker.
