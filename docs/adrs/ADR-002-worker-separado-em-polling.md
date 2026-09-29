# ADR-002: Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Data da decisão:** reunião técnica de webhooks ([09:10] a [09:13])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Status

Aceito

## Contexto

Com a outbox decidida ([ADR-001](ADR-001-outbox-no-mysql.md)), falta definir **quem** lê a tabela e **como** descobre eventos novos.

Restrições levantadas na reunião:

- O cliente considera "tempo real" qualquer coisa **abaixo de 10 segundos** ([09:02] Marcos).
- O MySQL não tem um mecanismo nativo de notificação a processos externos, como o `LISTEN/NOTIFY` do Postgres. Triggers só executam SQL ([09:09] Diego).
- Se o worker rodar dentro da API, um restart da API derruba o worker junto ([09:11] Diego).
- Hoje o único entry-point do projeto é `src/server.ts`, que sobe o Express e o `PrismaClient` de `src/config/database.ts`.

## Decisão

1. **Polling em loop a cada 2 segundos.** O worker busca os eventos pendentes mais antigos em lotes pequenos, processa e marca o resultado ([09:08], [09:09] Diego; [09:10] Larissa). A latência de 2 segundos no pior caso foi explicitamente aceita ([09:10] Larissa).
2. **Processo separado da API.** Um novo entry-point `src/worker.ts` e um script `npm run worker`, no mesmo molde de `src/server.ts` ([09:11] Larissa; [09:28] Bruno).
3. **Mesma stack e mesmo banco, instância própria de `PrismaClient`.** O `PrismaClient` é por processo, então o worker cria o seu com a mesma `DATABASE_URL` ([09:11] Bruno; [09:30] Bruno).
4. **A lógica fica dentro do módulo.** O processamento vive em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`), arquivo a criar, e `src/worker.ts` só faz o bootstrap ([09:28] Bruno).
5. **Uma única instância do worker nesta fase.** A ordem de entrega por pedido é garantida apenas implicitamente, pelo processamento em ordem de `created_at` com um único worker ([09:12] Diego; [09:13] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Trigger de banco para acionar o worker de forma reativa** | O MySQL não notifica processos externos. Seria preciso improvisar (escrever em arquivo, chamar um endpoint), o que "fica esquisito". Polling de 2s já atende a meta de menos de 10s ([09:09] Bruno e Diego). |
| **Worker rodando dentro do processo da API** | Um restart ou deploy da API derrubaria o worker ([09:11] Diego: "Só não pode ser o mesmo processo"). |
| **Múltiplos workers em paralelo** | Perde a ordem por pedido. Particionar por `order_id` ou usar lock pessimista fica para quando houver necessidade de escala ([09:13] Diego: "problema do futuro"). |

## Consequências

**Positivas**
- A latência fica em até cerca de 2s mais o tempo de envio, bem dentro da meta de 10s.
- É simples de implementar e de operar, sem dependência nova.
- API e worker têm ciclos de vida independentes.

**Negativas**
- Há consultas constantes ao banco (uma a cada 2s), mesmo sem eventos.
- Nesta fase existe um ponto único de processamento: com o worker parado, os eventos se acumulam (sem perda) até ele voltar.
- **Limitação conhecida:** a ordem é garantida **por `order_id`, enquanto houver um único worker**. Não existe ordem global ([09:13] Larissa). Os clientes não pediram ordem global ([09:14] Marcos).
- O deploy passa a ter dois processos para operar.

**Trade-off explícito:** trocamos reatividade (sub-segundo) e escala horizontal por simplicidade operacional e ordem implícita por pedido.
