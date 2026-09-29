# Da Reunião ao Documento: Design Docs do Sistema de Webhooks de Pedidos

> O enunciado original do desafio está no repositório base: https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia

## Sobre o desafio

Uma empresa que opera um Order Management System (OMS) em produção decidiu, numa reunião de cerca de 55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos**. Três clientes B2B querem ser avisados quando o status de um pedido muda, em vez de fazer polling na API. A única memória dessa reunião é a transcrição literal (`TRANSCRICAO.md`). O trabalho foi transformá-la, junto com a leitura do código existente (Node.js + TypeScript, Express, Prisma/MySQL), num pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A regra que guiou tudo foi que **nada pode ser inventado**. Cada requisito, decisão ou restrição precisa apontar para uma fala da reunião (`[hh:mm] Nome`) ou para um arquivo real do repositório. Separar o que foi decidido do que foi **descartado**, **adiado** ou ficou **em aberto** foi tão importante quanto registrar as decisões. O código da aplicação não foi alterado: a entrega é só documental.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (em um projeto do Claude, com o repositório clonado) | Ferramenta principal. Leu a transcrição e o código, classificou as falas da reunião, redigiu todos os documentos, fez revisões críticas e executou os scripts de verificação |
| **Script Python de verificação** (gerado com a IA, executado localmente, fora do repositório) | Validação objetiva. Confere se cada `[hh:mm] Nome` citado existe na transcrição, se cada caminho de arquivo citado existe, a proporção TRANSCRICAO/CODIGO no tracker e se todo ID dos documentos tem linha no tracker |

## Workflow adotado

1. **Contextualização.** A IA leu `TRANSCRICAO.md` inteira e os arquivos centrais do código: `order.service.ts`, `order.status.ts`, `shared/errors/*`, os middlewares, o logger, `app.ts`, `server.ts`, `routes/index.ts`, `prisma/schema.prisma`, `package.json` e os testes.
2. **Filtragem dirigida da transcrição.** Cada fala relevante foi classificada em decisão, requisito, restrição, descartado, adiado, questão em aberto ou detalhe secundário, sempre com timestamp e falante (prompt 1, abaixo).
3. **ADRs primeiro.** Foram 7 ADRs: as 6 decisões principais e mais uma (snapshot do payload e filtro na inserção), porque essa decisão muda o modelo da outbox.
4. **RFC.** A proposta foi consolidada em nível de arquitetura, com links para os ADRs. As alternativas descartadas e as questões em aberto entraram aqui, sem o detalhe de implementação.
5. **FDD.** O desenho detalhado ficou aqui: modelo Prisma, fluxos, contratos, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção "Integração com o sistema existente" com 18 pontos de integração em arquivos reais. Escolhas de implementação que não foram faladas literalmente ficaram marcadas como **(desenho)**.
6. **PRD.** Ficou por último entre os grandes documentos, como consolidação de produto: problema, público, métricas, escopo e requisitos com IDs.
7. **Tracker.** Foi montado varrendo os documentos prontos, com uma linha por ID.
8. **Verificação automatizada e revisão adversarial** (prompt 3), repetidas até zerar os problemas.
9. **README**, escrito depois que o processo estava completo.

## Prompts customizados

**Prompt 1: filtragem dirigida da transcrição**, usado antes de escrever qualquer documento

```text
Leia TRANSCRICAO.md inteira. Monte uma tabela com UMA linha por fala relevante,
com as colunas: timestamp | falante | categoria | resumo em 1 linha.
Categorias permitidas (escolha uma):
  DECISAO (fechada explicitamente por alguém, ex.: "Decidido", "Anotado", "Tá decidido")
  REQUISITO_FUNCIONAL | REQUISITO_NAO_FUNCIONAL | RESTRICAO
  DESCARTADO (alternativa rejeitada: registre o motivo dito na reunião)
  ADIADO (explicitamente "próxima fase", "futuro", "observar e decidir depois")
  QUESTAO_EM_ABERTO (levantado e não decidido)
  GANCHO_CODIGO (menção a arquivo, classe, função ou padrão existente)
Regras:
- Não infira decisões: se ninguém fechou o ponto, NÃO é DECISAO.
- Separe o que foi dito na conversa lateral depois da saída de Marcos e Sofia (09:50+).
- Para cada GANCHO_CODIGO, abra o arquivo citado e confirme que existe; anote o caminho real.
- Liste ao final tudo que NÃO deve virar requisito (descartado + adiado), com timestamp.
```

**Prompt 2: geração de ADR com fronteira de altura**

```text
Escreva docs/adrs/ADR-00N-<kebab>.md no formato MADR para a decisão "<decisão>".
Seções obrigatórias: Status, Contexto, Decisão, Alternativas Consideradas, Consequências
(Positivas / Negativas / "Trade-off explícito" em uma frase).
Regras:
- Toda afirmação de contexto ou decisão termina com a fonte: ([hh:mm] Nome) ou o caminho
  real do arquivo em src/, prisma/ ou tests/.
- Alternativas: só as que foram faladas na reunião, cada uma com o motivo do descarte
  dito na reunião. Se não houver nenhuma falada, use uma plausível e marque como tal.
- Não descreva endpoints, payloads nem schema aqui (isso é FDD). Fale em decisão.
- Consequências negativas são obrigatórias e precisam ser concretas para ESTE código
  (ex.: impacto na transação de OrderService.changeStatus).
- Se a decisão tiver ambiguidade (ex.: contagem de tentativas), registre a interpretação
  adotada e aponte para "Questões em aberto" do RFC, em vez de escolher em silêncio.
```

**Prompt 3: revisão adversarial contra as fontes** (rodado depois de cada rodada de escrita)

```text
Aja como revisor hostil. Para os documentos em docs/:
1. Liste toda afirmação que NÃO tem [hh:mm] Nome nem caminho de arquivo como fonte.
2. Liste toda afirmação que CONTRADIZ a transcrição (datas, números, quem decidiu o quê,
   o que foi adiado vs. descartado) ou o código (nomes de classe, códigos de erro,
   comportamento de middlewares).
3. Liste itens descartados/adiados na reunião que aparecem como requisito.
4. Liste caminhos de arquivo citados que não existem (os que são "a criar" precisam
   estar marcados como novos).
5. Verifique se o RFC repete o detalhe do FDD (endpoints, payloads, schema): isso é erro
   de altura.
Para cada item: arquivo, trecho, problema, correção proposta. Depois rode o script de
verificação e confira se o tracker cobre todos os IDs dos documentos.
```

## Iterações e ajustes

Foram **5 ciclos principais** de geração, revisão e correção. Os ajustes concretos mais relevantes:

1. **Plano de sprints contradizia a reunião.** Na primeira versão do FDD, a integração de `publishWebhookEvent` em `changeStatus` estava na sprint 1. Na revisão contra a transcrição ficou claro que Larissa ([09:46]) colocou "integração no order.service e testes ponta a ponta" junto com o CRUD, no terceiro bloco. O plano foi corrigido para seguir a estimativa dela.
2. **Endpoint de listagem contradizia uma decisão.** O `GET` de webhooks tinha saído com `customerId` na query string. A reunião definiu que o customer vai "no body ou no path" ([09:32] Larissa). A rota virou `GET /api/v1/customers/:customerId/webhooks`, e o FDD passou a documentar como montá-la em `src/routes/index.ts`.
3. **Código de erro incompatível com o código real.** A IA listou `WEBHOOK_INVALID_URL` como código HTTP devolvido diretamente. Lendo `src/middlewares/validate.middleware.ts`, todo `ZodError` vira `ValidationError` com o código `VALIDATION_ERROR`. Como a regra de https é de schema Zod ([09:23] Sofia) e o error middleware não deve mudar ([09:29] Bruno), a matriz passou a dizer onde o código `WEBHOOK_INVALID_URL` aparece de fato (em `details`). Pelo mesmo motivo, ficou registrado que `NotFoundError` tem o código fixo `NOT_FOUND` e que são precisas subclasses próprias para `WEBHOOK_NOT_FOUND`.
4. **Caminhos de arquivo inexistentes.** O script de verificação acusou `src/worker.ts`, `src/modules/webhooks/...` e `tests/webhooks.test.ts`. São arquivos propostos na reunião que ainda não existem. Todos passaram a ser marcados explicitamente como *(novo)* / "a criar", e o FDD ganhou uma convenção no topo dizendo que os demais caminhos existem.
5. **Tracker com cobertura incompleta.** A primeira versão tinha 199 linhas e deixava de fora cenários do PRD, parte dos critérios de aceite técnicos do FDD e itens da proposta do RFC. Um cruzamento automático entre os IDs dos documentos e o tracker levou a 229 linhas, com 100% dos IDs cobertos, 85% das linhas com fonte na transcrição e 35 com fonte no código.
6. **Ambiguidades registradas, não escondidas.** "5 tentativas" contra 5 intervalos de backoff (1m/5m/30m/2h/12h, "quase 15 horas entre primeira falha e última tentativa"), a ordem por pedido quebrada durante um retry e a secret que precisa ser recuperável para assinar viraram questões em aberto no RFC. Não foram resolvidas em silêncio.
7. **Limite de 64KB no lugar errado.** Um primeiro desenho validava o tamanho dentro de `changeStatus`, o que faria uma mudança de status legítima sofrer rollback. O limite foi movido para o worker: o evento vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, e o status do pedido não é afetado.
8. **Revisão final contra a checklist de aceite.** Uma última passada, item por item, achou três pontos: só o `POST` de cadastro tinha exemplo explícito de request **e** response (os demais contratos do FDD ganharam o request HTTP completo, e o `PATCH` ganhou o JSON de resposta); o `Status` dos ADRs estava como metadado e virou seção própria; e o `ADR-006` dizia "IDs UUID em todas as tabelas", quando `order_number_sequence` usa `id` inteiro, e que todo módulo tem `repository`, quando `auth` reaproveita o de `users`. Também foram marcados como *(novo)* os últimos trechos que citavam `src/worker.ts` sem a marcação.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md): por que e o quê (problema, métricas, escopo e fora de escopo, requisitos)
2. [`docs/RFC.md`](docs/RFC.md): a proposta técnica, as alternativas descartadas e as questões em aberto
3. [`docs/adrs/`](docs/adrs/README.md): as 7 decisões, uma por arquivo
   - [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md) Outbox no MySQL
   - [ADR-002](docs/adrs/ADR-002-worker-separado-em-polling.md) Worker separado em polling
   - [ADR-003](docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) Retry com backoff e DLQ
   - [ADR-004](docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) HMAC-SHA256 com secret por endpoint
   - [ADR-005](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) At-least-once com `X-Event-Id`
   - [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md) Reuso dos padrões existentes
   - [ADR-007](docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) Snapshot do payload e filtro na inserção
4. [`docs/FDD.md`](docs/FDD.md): como construir (modelo, fluxos, contratos, erros, observabilidade, integração com o código)
5. [`docs/TRACKER.md`](docs/TRACKER.md): a origem de cada item, na transcrição ou no código

Arquivos de referência (não alterados): [`TRANSCRICAO.md`](TRANSCRICAO.md) e o código em `src/`, `prisma/` e `tests/`.
