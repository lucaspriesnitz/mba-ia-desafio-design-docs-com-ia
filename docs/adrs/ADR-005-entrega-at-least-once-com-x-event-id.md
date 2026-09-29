# ADR-005: Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

- **Data da decisão:** reunião técnica de webhooks ([09:24] a [09:26])
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)

## Status

Aceito

## Contexto

Com a outbox, o worker e os retries, existem cenários em que o mesmo evento é enviado mais de uma vez. Por exemplo: o cliente processou o request mas respondeu depois do timeout de 10s, ou o worker caiu depois do envio e antes de marcar a linha como entregue. Precisamos definir a semântica de entrega e como o cliente lida com duplicatas.

## Decisão

1. A plataforma garante entrega **at-least-once**. O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso ([09:24] Diego).
2. Cada evento recebe um **UUID (`event_id`) gerado no momento em que entra na outbox**, único por evento, e enviado no header **`X-Event-Id`** ([09:25] Diego).
3. **O cliente deduplica** do lado dele pelo `event_id` ([09:25] Diego; decisão [09:26] Larissa).
4. O comportamento será documentado **em destaque no portal de desenvolvedor** ([09:26] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Exactly-once** | Exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com `event_id` "resolve 99% dos casos" e é o que Stripe e GitHub fazem ([09:25] Diego). |
| **At-most-once (sem retry)** | É incompatível com a política de retry do [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md). Uma indisponibilidade temporária faria o cliente perder eventos. |

## Consequências

**Positivas**
- O worker fica simples: na dúvida, reenvia.
- O `event_id` também serve de chave de correlação em logs e no histórico de entregas.

**Negativas**
- **A responsabilidade de deduplicar passa para o cliente** ([09:25] Sofia: "Isso joga responsabilidade pro cliente").
- Um cliente que ignore o `X-Event-Id` pode processar o mesmo evento duas vezes.

**Trade-off explícito:** transferimos parte da complexidade (idempotência) para o consumidor em troca de uma implementação muito mais simples e alinhada ao mercado.
