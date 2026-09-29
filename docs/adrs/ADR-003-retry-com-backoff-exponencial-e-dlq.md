# ADR-003: Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada

- **Data da decisão:** reunião técnica de webhooks ([09:14] a [09:19])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Status

Aceito

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou devolvendo erro. Precisamos decidir quantas vezes retentar, com qual espaçamento, e o que fazer quando desistirmos.

Dados da reunião:

- Já houve cliente com indisponibilidade de 2 horas em manutenção planejada ([09:16] Diego).
- Retry indefinido deixa eventos pendurados para sempre se o cliente sumir ([09:15] Diego).
- Uma chamada que não responde em 10 segundos é tratada como falha ([09:42] Diego).

## Decisão

1. **Backoff exponencial com teto de 5 tentativas**, nos intervalos **1 min, 5 min, 30 min, 2 h e 12 h**. São quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego; decisão [09:17] Larissa).
   - *Interpretação registrada:* os 5 intervalos se contam **a partir da primeira falha**. São portanto uma tentativa inicial mais 5 retentativas agendadas, o que fecha com as "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). Essa leitura deve ser confirmada na revisão (ver [RFC](../RFC.md), questões em aberto).
2. **Timeout de 10 segundos** por chamada HTTP. Um estouro conta como falha e agenda o próximo retry ([09:42] Diego).
3. **DLQ em tabela separada, `webhook_dead_letter`**, com o payload, o motivo da falha e o timestamp. Ela serve de evidência para debug e reprocessamento ([09:18] Diego).
4. **Reprocessamento manual** por `POST /admin/webhooks/dead-letter/:id/replay`, que devolve o evento à outbox como pendente ([09:18] Diego; [09:35] Diego).
5. O replay exige a **role `ADMIN`**, reaproveitando o `requireRole` existente (`src/middlewares/auth.middleware.ts`), e **registra em log quem fez o replay**, para auditoria ([09:36] Sofia; [09:36] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Retry indefinido com backoff** | Um evento fica pendurado para sempre se o cliente sumir ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Cobre só uns 30 minutos. Uma indisponibilidade de manhã, ou uma manutenção planejada de 2h, mataria o evento ([09:16] Diego). |
| **Marcar como `failed` na própria outbox, sem tabela de DLQ** | Suja a leitura da outbox principal. Uma tabela separada deixa a outbox limpa e concentra a evidência de falha ([09:17] Larissa; [09:18] Diego). |

## Consequências

**Positivas**
- Cobre indisponibilidades de até cerca de 15h sem intervenção humana ([09:17] Marcos: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele").
- Toda falha permanente fica rastreável e pode ser reprocessada.
- O replay é auditável e restrito a administradores.

**Negativas**
- Um evento pode chegar ao cliente com até cerca de 15h de atraso. Nesse intervalo o cliente não é avisado proativamente (o aviso por e-mail está **fora de escopo** nesta fase, [09:37] Larissa).
- O reprocessamento é manual e depende de alguém agir.
- **Efeito colateral no ordering:** enquanto um evento de um pedido espera o retry, um evento posterior do mesmo pedido pode ser entregue antes dele. Esse ponto não foi discutido explicitamente e está listado como questão em aberto no [RFC](../RFC.md).

**Trade-off explícito:** priorizamos uma janela longa de tolerância a indisponibilidade do cliente, aceitando atraso potencial e reprocessamento manual, em vez de descartar rápido ou tentar para sempre.
