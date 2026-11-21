# ADR-001 — Outbox transacional no MySQL

## Status

Aceito.

## Contexto

Clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — precisam ser notificados quando o status dos pedidos deles muda; hoje eles fazem polling no `GET /orders`, o que torna a integração lenta e cara para eles ([09:00] Marcos). Para esses clientes, qualquer latência abaixo de 10 segundos já é considerada “tempo real” ([09:02] Marcos). A solução precisa, portanto, notificar de forma assíncrona sem introduzir uma chamada HTTP no meio da transação de mudança de status.

A transação atual de `changeStatus` já é pesada: atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` dos produtos do pedido ([09:04] Bruno). Um HTTP call síncrono no meio dessa transação faria um cliente lento travar a mudança de status de outros pedidos ([09:04] Bruno), e não é aceitável dar rollback na mudança de status caso o cliente esteja fora do ar ([09:04] Bruno; [09:05] Diego). Diego propôs o padrão outbox: inserir o evento na mesma transação SQL que atualiza `orders` e `order_status_history`, deixando um worker separado para disparar as chamadas HTTP ([09:06] Diego). Larissa formalizou a decisão: “Tá decidido então: outbox em MySQL” ([09:08] Larissa).

No código atual, o ponto de integração é `OrderService.changeStatus` em `src/modules/orders/order.service.ts` (L126–179). O método abre `this.prisma.$transaction` (L131), valida a transição, debita ou recompõe estoque (L151–156), atualiza `order.status` (L158) e cria o registro em `OrderStatusHistory` (L159–167). O tipo `TxClient = Prisma.TransactionClient` já existe no arquivo (L24). A garantia central é: se a transação principal commitar, o evento foi registrado; se ela der rollback, o evento some junto ([09:06] Diego; [09:40] Bruno; [09:41] Diego).

## Decisão

Adotar o padrão outbox no MySQL existente, com uma tabela `webhook_outbox`. A inserção do evento deve ocorrer dentro da mesma transação de `OrderService.changeStatus`, por meio de uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação atual ([09:41] Bruno). Diego concordou explicitamente com a função pura recebendo o `tx`, sem injetar um repository inteiro no `OrderService` ([09:41] Diego).

A decisão inclui o filtro na inserção: se nenhum webhook do customer quer aquele status, o evento nem é inserido na outbox, para economizar linhas na tabela ([09:34] Bruno; [09:34] Diego). Esse é um detalhe de implementação acordado entre Bruno e Diego; ele não aparece no resumo final de [09:48] Larissa.

O evento deve seguir as decisões complementares: identificador UUID (ADR-006), payload renderizado como snapshot no momento da inserção (ADR-007) e processamento por um worker separado (ADR-002).

**Detalhes a fechar no FDD/RFC:** a transcrição não define as colunas exatas de `webhook_outbox`, a posição exata da inserção dentro da transação (antes ou depois de `order.update` e do insert em `order_status_history`), nem a máquina de estados formal da outbox. Os estados `pendente`, `processando`, `falhou` e `entregue` são citados ([09:08] Diego), mas não formalizados como enum. O requisito explícito é apenas que a inserção ocorra na mesma transação.

## Alternativas Consideradas

### Disparo síncrono dentro do service de orders

Bruno argumentou que “síncrono não rola” porque a transação já é pesada e qualquer cliente lento travaria a mudança de status para outros pedidos ([09:04] Bruno); Larissa concordou ([09:04] Larissa); Diego afirmou que síncrono estava “fora de questão” ([09:05] Diego). A alternativa foi descartada porque não há como dar rollback na mudança de status quando o cliente está fora do ar ([09:04] Bruno).

### Redis Streams ou Redis Cluster

Larissa levantou Redis Streams como alternativa ([09:07] Larissa); Diego descartou porque exigiria subir mais infraestrutura e, para um time pequeno, “subir Redis Cluster pra isso é overengineering”; o outbox no MySQL existente resolveria ([09:07] Diego). O trade-off é menos infraestrutura e simplicidade operacional versus uma solução de fila dedicada.

### Trigger SQL no banco

Bruno perguntou se um trigger do banco não seria mais reativo ([09:09] Bruno); Diego explicou que o MySQL não tem listener nativo como `NOTIFY`/`LISTEN` do Postgres, que o trigger só executa SQL e não notifica processo externo, e que improvisar arquivo ou endpoint seria “esquisito”; o polling de 2 segundos atende ao requisito ([09:09] Diego). A alternativa foi descartada por não resolver a notificação ao worker.

### Filtrar no momento do envio, em vez de na inserção

Diego perguntou se o filtro deveria ocorrer “na inserção do outbox ou na hora de mandar” ([09:34] Diego); Bruno respondeu que seria na inserção, para economizar linhas na tabela, e Diego concordou ([09:34] Bruno; [09:34] Diego). A alternativa de filtrar no envio foi descartada porque armazenaria eventos que nenhum webhook do customer quer receber.

## Consequências

### Positivas

- A atomicidade entre mudança de status e registro do evento elimina a possibilidade de status mudar sem evento ou de evento existir sem commit da transação ([09:06] Diego; [09:40] Bruno).
- Não é necessário subir Redis, Cluster ou outra infraestrutura; o MySQL existente resolve ([09:07] Diego).
- A mudança de status não fica refém da latência ou da indisponibilidade do cliente, porque o HTTP call sai do caminho transacional ([09:04] Bruno).
- O filtro na inserção evita linhas desnecessárias na outbox ([09:34] Bruno).

### Negativas

- A transação de mudança de status, já descrita como pesada ([09:04] Bruno), passa a incluir mais uma escrita.
- Se a inserção na outbox falhar, a mudança de status também falha por rollback; não pode haver status mudado e evento não registrado ([09:40] Bruno; [09:41] Diego).
- A outbox cresce com o tempo; o arquivamento de linhas entregues após 30 dias foi explicitamente deixado fora do escopo desta feature ([09:08] Diego).
- A entrega depende do worker e do polling; a latência de notificação não é imediata (ADR-002).

### Trade-offs

- **Consistência versus disponibilidade da mudança de status:** optou-se por garantir que não exista status mudado sem evento, aceitando que uma falha na outbox derrube a transação ([09:40] Bruno; [09:41] Diego).
- **Simplicidade operacional versus fila dedicada:** o outbox no MySQL evita nova infraestrutura, mas troca uma solução de fila especializada por polling e processamento em lote ([09:07] Diego; [09:09] Diego).
- **Economia de linhas versus acoplamento ao fluxo transacional:** filtrar na inserção reduz o volume da outbox, mas concentra a decisão de elegibilidade do evento no caminho da mudança de status ([09:34] Bruno; [09:34] Diego).
- **Snapshot versus renderização tardia:** guardar o payload renderizado aumenta o armazenamento, mas preserva o estado do pedido no momento da mudança (ADR-007; [09:52] Larissa).
