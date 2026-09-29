# ADR-004: Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Data da decisão:** reunião técnica de webhooks ([09:19] a [09:23])
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Status

Aceito

## Contexto

Os webhooks são **outbound**: saem da plataforma para os clientes, e nunca no sentido contrário ([09:02] Marcos; [09:03] Sofia). Eles levam dados de pedidos para uma URL fora da nossa infraestrutura. O cliente precisa conseguir verificar duas coisas: que a requisição veio de nós, e que o payload não foi adulterado no caminho ([09:19] Sofia).

Já tivemos um cliente que vazou a própria secret no log da aplicação dele ([09:22] Diego).

## Decisão

1. **HMAC-SHA256 sobre o corpo do request**, com a assinatura enviada no header `X-Signature` ([09:20] Sofia; decisão [09:22] Sofia).
2. **Uma secret única por endpoint de webhook**, e não uma secret global da plataforma ([09:21] Sofia).
3. **A secret é gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
4. A configuração guarda **URL, secret, `customer_id` e estado ativo** ([09:21] Bruno; [09:21] Sofia).
5. **Rotação sob demanda por endpoint da API.** Depois de rotacionar, a secret antiga continua válida **por 24h em paralelo** com a nova e então expira ([09:21] Sofia).
6. **HTTPS obrigatório.** Uma URL `http://` é recusada com erro de validação no schema Zod. Isso é uma validação, não uma decisão arquitetural à parte ([09:23] Sofia).
7. O código de HMAC e de geração de secret passa por **revisão de segurança da Sofia, com pelo menos 2 dias úteis, antes do deploy** ([09:46] Sofia; [09:49] Sofia).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Secret global da plataforma** | Se vaza uma, vaza tudo ([09:21] Sofia). |
| **Rotação com troca imediata (sem grace period)** | O cliente não teria tempo de migrar os sistemas dele. Os 24h em paralelo existem exatamente para isso ([09:21] Sofia). |
| **Permitir URLs `http`** | Os dados de pedido trafegariam sem criptografia em trânsito. TLS é obrigatório ([09:23] Sofia). |

## Consequências

**Positivas**
- HMAC-SHA256 é padrão de mercado, e todo cliente sério já tem biblioteca para isso ([09:20] Sofia).
- Um vazamento fica contido a um único endpoint, e a rotação permite a recuperação sem downtime.

**Negativas**
- A secret precisa estar **recuperável em claro** no servidor para assinar. Não dá para guardar só o hash, como fazemos com `passwordHash` em `users`. A criptografia em repouso não foi discutida e fica como ponto para a revisão de segurança (ver [RFC](../RFC.md)).
- Durante o grace period, o envio precisa carregar assinaturas das duas secrets (o formato está detalhado no [FDD](../FDD.md)), o que aumenta um pouco a complexidade para o cliente.
- A assinatura cobre só o corpo. O `X-Timestamp` ([09:44] Diego) vai fora da assinatura, então a proteção contra replay fica a cargo do cliente, "se quiser".

**Trade-off explícito:** aceitamos o custo de guardar e rotacionar uma secret por endpoint em troca de contenção de vazamentos e autenticidade verificável pelo cliente.
