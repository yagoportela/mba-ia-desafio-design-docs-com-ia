# RFC — Sistema de Webhooks de Notificação de Pedidos

## 1. Metadados

| Campo | Valor |
|---|---|
| **Título** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Diego — Engenheiro Sênior, time de Plataforma |
| **Status** | Em Revisão |
| **Data** | 2026-11-21 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Sofia (Engenheira de Segurança) e Bruno (Engenheiro Pleno — Pedidos) |

## 2. Resumo Executivo (TL;DR)

Propomos substituir o polling dos clientes B2B por webhooks outbound disparados a partir de um **outbox transacional no MySQL**. A mudança de status grava o evento na `webhook_outbox` dentro da mesma transação de `OrderService.changeStatus`; um **worker separado**, com novo entry point `src/worker.ts` (a criar) e script `npm run worker` (a criar), faz polling a cada 2 segundos e entrega os eventos. A segurança usa **HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h**. A confiabilidade usa **entrega at-least-once com `X-Event-Id`**, retry em **5 tentativas com backoff de 1m/5m/30m/2h/12h** e DLQ em `webhook_dead_letter`, com replay manual restrito a `ADMIN`.

## 3. Contexto e Problema

Atlas Comercial, MaxDistribuição e Nova Cargo são clientes B2B que hoje fazem polling no `GET /orders` para descobrir se o status dos pedidos mudou; a integração ficou lenta e cara para eles ([09:00] Marcos). Para esses clientes, qualquer latência abaixo de 10 segundos já é considerada “tempo real” ([09:02] Marcos). A Atlas sinalizou que pode migrar para o concorrente caso a entrega não ocorra até o fim do trimestre ([09:00] Marcos).

A alternativa de disparar o webhook sincronamente dentro do service de pedidos foi descartada: a transação de `changeStatus` já é pesada, pois atualiza `orders`, insere em `order_status_history` e ajusta o `stockQuantity` dos produtos, debitando ou repondo conforme a transição ([09:04] Bruno). Um HTTP call no meio dessa transação faria qualquer cliente lento travar a mudança de status de outros pedidos ([09:04] Bruno). Além disso, se o cliente estiver fora do ar, não é aceitável dar rollback na mudança de status ([09:04] Bruno; [09:05] Diego).

No código atual, `OrderService.changeStatus` em `src/modules/orders/order.service.ts` (L126–179) concentra a transação que precisa ser estendida. A API é inicializada por `src/server.ts`, e o projeto já possui o factory `createPrismaClient()` em `src/config/database.ts` (L4–8).

## 4. Proposta Técnica (Arquitetura)

### 4.1 Visão geral dos componentes

A solução tem dois componentes de runtime:

- **API HTTP**, inicializada por `src/server.ts`, que atende o fluxo normal de pedidos. A gravação do evento ocorre dentro da transação de `OrderService.changeStatus`, em `src/modules/orders/order.service.ts`, e não em uma chamada HTTP externa.
- **Worker separado**, com entry point `src/worker.ts` (a criar) e script `npm run worker` (a criar), que roda em outro processo Node, conecta-se ao mesmo MySQL e usa uma instância própria de `PrismaClient`, pois o cliente Prisma é por processo ([09:11] Diego; [09:30] Bruno). O worker faz polling a cada 2 segundos ([09:09] Diego; [09:10] Larissa).

O novo módulo `src/modules/webhooks` (a criar) segue o padrão existente de `src/modules`: controller, service, repository, routes e schemas ([09:27] Bruno). A lógica de processamento fica dentro do módulo, em `webhook.worker.ts` ou `webhook.processor.ts`; o nome final permanece em aberto e está registrado na seção 6 ([09:28] Bruno).

### 4.2 Fluxo de uma mudança de status

1. A API recebe a mudança de status e `OrderService.changeStatus` abre a transação, valida a transição, ajusta o estoque quando aplicável, atualiza `orders` e registra o histórico ([09:04] Bruno; `src/modules/orders/order.service.ts`, L126–179).
2. Ainda na mesma transação, uma função de enfileiramento recebe o `tx` e grava o evento na `webhook_outbox`. A função proposta é `publishWebhookEvent(tx, order, fromStatus, toStatus)` ([09:41] Bruno; [09:41] Diego). Se a inserção na outbox falhar, a transação inteira sofre rollback ([09:40] Bruno).
3. O evento só é inserido se houver algum webhook do customer interessado naquele status; caso nenhum queira, o evento nem é gravado, para economizar linhas ([09:34] Bruno; [09:34] Diego).
4. O evento guarda o payload renderizado como snapshot no momento da inserção, refletindo o estado do pedido quando o status mudou ([09:52] Larissa). IDs seguem o padrão UUID do projeto ([09:51] Larissa).
5. O worker busca os eventos pendentes mais antigos a cada 2 segundos, dispara a chamada HTTP e marca o resultado ([09:09] Diego; [09:10] Larissa).

### 4.3 Confiabilidade e segurança em alto nível

- **Entrega at-least-once com idempotência:** o cliente pode receber o mesmo evento mais de uma vez e deduplica pelo `X-Event-Id`, um UUID gerado quando o evento entra na outbox ([09:24] Diego; [09:25] Diego; [09:26] Larissa). Exactly-once foi descartado por exigir coordenação dos dois lados ([09:25] Diego).
- **Assinatura HMAC-SHA256:** o payload é assinado com uma secret compartilhada; cada endpoint tem sua própria secret, e a rotação mantém a antiga válida por 24 horas em paralelo antes de desativá-la ([09:19] Sofia; [09:20] Sofia; [09:21] Sofia; [09:22] Sofia). URLs de webhook devem ser `https`, com recusa de `http` via validação Zod ([09:23] Sofia).
- **Retry e DLQ:** falhas são retentadas em 5 tentativas com backoff de 1m/5m/30m/2h/12h; depois do teto, o evento vai para a tabela separada `webhook_dead_letter` ([09:15] Diego; [09:17] Larissa; [09:18] Diego). Cada chamada HTTP tem timeout de 10 segundos e timeout é tratado como falha para retry ([09:42] Diego).
- **Replay administrativo:** o replay de DLQ recoloca o evento na outbox como pendente, exige role `ADMIN`, reutiliza o `requireRole` existente e registra quem executou a ação para auditoria ([09:18] Diego; [09:36] Sofia; [09:36] Larissa).
- **Limite de payload:** o payload máximo é de 64KB; se ultrapassar, a operação erra em vez de truncar ([09:23] Sofia; [09:24] Diego; [09:24] Larissa).
- **Observabilidade e erros:** reutilização do logger Pino e do error middleware centralizado, que já trata `AppError`, `Zod` e `Prisma` ([09:29] Bruno; [09:30] Larissa).

### 4.4 Limitações conhecidas aceitas nesta fase

- **Single-worker e ordering:** com um único worker, os eventos são processados na ordem de `created_at` da outbox, o que garante ordering apenas por `order_id`; não há garantia de ordering global ([09:12] Diego; [09:13] Larissa). Múltiplos workers, particionamento por `order_id` e lock pessimista ficam para o futuro ([09:13] Diego).
- **Sem arquivamento automatizado:** o arquivamento de linhas entregues após 30 dias está explicitamente fora do escopo desta feature ([09:08] Diego). A outbox cresce enquanto essa rotina não existir.
- **Itens fora desta fase:** email de alerta, dashboard visual e rate limiting de envio não entram nesta entrega; rate limiting fica como ponto em aberto ([09:37] Larissa; [09:38] Diego; [09:39] Larissa; [09:40] Larissa).

## 5. Alternativas Consideradas

- **Disparo HTTP síncrono no service de pedidos** ([09:04] Bruno; [09:05] Diego). Ofereceria notificação imediata, mas colocaria uma chamada externa dentro da transação pesada, faria um cliente lento bloquear outros pedidos e impediria rollback da mudança de status quando o cliente estivesse fora do ar. Descartado.
- **Redis Streams / Redis Cluster** ([09:07] Larissa; [09:07] Diego). Seria uma fila dedicada e mais reativa, mas exigiria subir mais infraestrutura; para um time pequeno, foi considerado overengineering. O outbox no MySQL existente resolve com menos operação. Descartado.
- **Trigger SQL no MySQL para notificar o worker** ([09:09] Bruno; [09:09] Diego). Aproveitaria o banco, mas o MySQL não tem listener nativo como `NOTIFY`/`LISTEN` do Postgres; o trigger só executa SQL e não notifica processo externo. Improvisar arquivo ou endpoint foi considerado esquisito, e o polling de 2 segundos atende ao requisito. Descartado.
- **Garantia exactly-once** ([09:25] Diego). Daria uma garantia mais forte, mas exigiria coordenação dos dois lados e seria muito mais complexo; at-least-once com `event_id` cobre 99% dos casos. Descartado.
- **Retry com 3 tentativas** ([09:16] Bruno; [09:16] Diego). Seria mais agressivo, mas é pouco para indisponibilidades reais; três tentativas em 30 minutos matariam um evento durante uma manutenção de duas horas. Descartado em favor de 5 tentativas.
- **Retry indefinido com backoff** ([09:15] Diego). Evitaria falha permanente, mas deixaria o evento pendurado para sempre se o cliente sumisse. Descartado em favor de um teto finito com DLQ.

## 6. Questões em Aberto

1. **Rate limiting de envio:** se um cliente tiver muitos pedidos mudando em pouco tempo, a plataforma pode bombardear o endpoint. A decisão foi observar e implementar se virar problema ([09:38] Diego; [09:39] Larissa).
2. **Endurecimento da autorização do CRUD:** por enquanto qualquer role autenticada pode operar o CRUD; a possibilidade de endurecer ficou para mais adiante ([09:36] Marcos; [09:37] Sofia).
3. **Contagem exata de tentativas:** a call diz “5 tentativas” e lista cinco intervalos de backoff, mas não explicita se o primeiro envio conta como uma das 5 ou se são 5 retries após o envio inicial ([09:15] Diego; [09:17] Larissa).
4. **Detalhes do HMAC:** falta definir codificação hex ou base64 da assinatura e se o `X-Timestamp` entra no conteúdo assinado; a call fala apenas em HMAC-SHA256 sobre o corpo do request ([09:20] Sofia; [09:44] Diego).
5. **Retry para `4xx` versus `5xx`:** a transcrição não diferencia respostas do cliente; timeout e falha são tratados genericamente como falha para retry ([09:15] Diego; [09:42] Diego).
6. **Nome do arquivo do worker:** `webhook.worker.ts` ou `webhook.processor.ts`; ambos foram propostos, sem escolha final ([09:28] Bruno).

Detalhes de modelagem — colunas exatas da outbox e da DLQ, nomes dos modelos de configuração e de histórico de entregas e semântica do `:id` no replay — ficam para o FDD.

## 7. Impacto e Riscos

- **Impacto na transação de `orders`:** a mudança de status passa a incluir mais uma escrita; se a outbox falhar, a transação inteira sofre rollback, então a disponibilidade da mudança de status fica acoplada à inserção do evento ([09:04] Bruno; [09:40] Bruno).
- **Operação de um segundo processo Node:** o worker exige `npm run worker`, uma instância própria de `PrismaClient` e seu próprio ciclo de vida, além do processo da API ([09:11] Diego; [09:30] Bruno).
- **Revisão de segurança:** Sofia reservou pelo menos 2 dias úteis para revisar HMAC e geração de secret antes do deploy ([09:46] Sofia).
- **Crescimento da outbox:** sem a rotina de arquivamento de 30 dias, a tabela tende a crescer; essa rotina está fora do escopo desta fase ([09:08] Diego).
- **Vazão e ordering:** a operação é single-worker nesta fase e garante ordering apenas por `order_id`; escala horizontal exigirá particionamento por `order_id` ou lock pessimista no futuro ([09:12] Diego; [09:13] Diego).
- **Duplicatas:** como a entrega é at-least-once, o cliente precisa deduplicar; se não fizer, pode processar o mesmo evento mais de uma vez ([09:24] Diego).
- **Prazo:** a Atlas quer a entrega até o fim de novembro; a estimativa é de 3 sprints incluindo a revisão da Sofia ([09:45] Marcos; [09:46] Larissa; [09:47] Larissa).

## 8. Decisões Relacionadas

| ADR | Decisão |
|---|---|
| [ADR-001 — Outbox transacional no MySQL](./adrs/ADR-001-outbox-transacional-no-mysql.md) | Gravação do evento na mesma transação da mudança de status. |
| [ADR-002 — Worker separado em polling](./adrs/ADR-002-worker-separado-em-polling.md) | Processo separado, polling de 2s, single-worker e ordering por `order_id`. |
| [ADR-003 — Política de retry, backoff e DLQ](./adrs/ADR-003-politica-de-retry-backoff-e-dlq.md) | 5 tentativas, backoff, timeout de 10s e replay administrativo. |
| [ADR-004 — Autenticação HMAC-SHA256 e rotação de secret](./adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md) | Assinatura por endpoint, rotação com grace period e HTTPS. |
| [ADR-005 — Entrega at-least-once e idempotência via event-id](./adrs/ADR-005-entrega-at-least-once-e-idempotencia-via-event-id.md) | Duplicatas possíveis e deduplicação pelo `X-Event-Id`. |
| [ADR-006 — Reuso dos padrões arquiteturais existentes](./adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md) | Módulo `src/modules/webhooks`, erros `WEBHOOK_*`, Pino, middleware e UUID. |
| [ADR-007 — Snapshot de payload na inserção da outbox](./adrs/ADR-007-snapshot-de-payload-na-insercao-da-outbox.md) | Payload renderizado no momento da inserção e limite de 64KB sem truncamento. |
