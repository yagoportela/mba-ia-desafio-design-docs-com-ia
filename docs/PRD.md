# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Marcos — Product Manager |
| **Status** | Em revisão |
| **Data** | 2026-11-21 |
| **Versão** | 1.0 |
| **Público** | Clientes B2B integradores, operadores e administradores do OMS |
| **Documentos relacionados** | [RFC](./RFC.md) · [FDD](./FDD.md) · [ADRs](./adrs/) |

> **Convenção de rastreabilidade:** citações no formato `[hh:mm] Nome` referem-se à `TRANSCRICAO.md`. Caminhos como `src/...` referem-se ao código existente no repositório. Quando o detalhe é técnico, o PRD referencia o [RFC](./RFC.md), o [FDD](./FDD.md) ou o ADR correspondente, sem reproduzir a implementação.

---

## 1. Resumo e Contexto da Feature

O OMS já controla o ciclo de vida dos pedidos, incluindo a máquina de estados, o histórico de mudanças e o controle de estoque dentro da transação de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). O que não existe hoje é um mecanismo de notificação externa: os clientes B2B precisam descobrir sozinhos que o status de um pedido mudou ([09:00] Marcos).

Esta feature entrega **webhooks outbound de mudança de status de pedidos** para clientes B2B do OMS. Cada cliente pode cadastrar endpoints HTTPS, escolher quais status deseja receber e receber eventos `order.status_changed` sempre que um pedido seu mudar de estado. A entrega é assíncrona e desacoplada da transação de pedidos: o evento é registrado de forma atômica e um processo separado realiza as chamadas HTTP ([09:06] Diego; [09:40] Bruno).

A entrega inclui os mecanismos necessários para que a integração seja confiável e segura: assinatura **HMAC-SHA256** com secret única por endpoint e rotação com grace period de 24 horas, entrega **at-least-once** com `X-Event-Id`, retry com backoff, DLQ com replay manual restrito a `ADMIN` e histórico dos últimos 100 envios por webhook ([09:20]–[09:26]; [09:17]–[09:19]; [09:34] Marcos; [09:36] Sofia).

**Em uma frase:** dar aos clientes B2B um canal push confiável e seguro para saber que o status de um pedido mudou, eliminando a necessidade de polling no `GET /orders`.

**O que não muda:** a fonte da verdade continua sendo o OMS. O webhook é uma notificação de mudança de status; se o cliente precisar de detalhes do pedido, como os itens, ele continua consultando `GET /orders/:id` ([09:43] Diego). Nesta fase a integração é feita por API, sem dashboard visual para o cliente ([09:39] Marcos; [09:40] Larissa).

---

## 2. Problema e Motivação

### 2.1 Situação atual

Atlas Comercial, MaxDistribuição e Nova Cargo são clientes B2B que hoje fazem **polling no `GET /orders`** para descobrir se o status dos pedidos mudou. Esse modelo deixa a integração lenta e cara para eles ([09:00] Marcos). A plataforma não oferece hoje nenhum mecanismo de notificação externa, fila ou webhook; o código atual não possui modelos ou módulos de webhook (`prisma/schema.prisma`; `src/modules/`).

Para esses clientes, qualquer latência abaixo de **10 segundos** já é considerada “tempo real”; o importante é não depender de atualização manual ou de polling constante ([09:02] Marcos).

### 2.2 Impacto comercial

Os três clientes fizeram um pedido formal pela capacidade ([09:00] Marcos). A Atlas sinalizou que, **se a entrega não ocorrer até o fim do trimestre, pode migrar para o concorrente** ([09:00] Marcos). O prazo comercial citado na reunião é o **fim de novembro** ([09:45] Marcos).

A motivação desta feature é, portanto, atender ao pedido dos três clientes B2B e reduzir o risco de churn sinalizado pela Atlas. O escopo desta fase é API-only, sem dashboard visual para o cliente.

### 2.3 Por que a notificação precisa ser assíncrona

A transação de mudança de status já é pesada: atualiza `orders`, insere em `order_status_history` e ajusta o `stockQuantity` dos produtos do pedido, debitando ou repondo conforme a transição ([09:04] Bruno). Um disparo HTTP síncrono dentro dessa transação faria **qualquer cliente lento travar a mudança de status de outros pedidos**; além disso, se o cliente estivesse fora do ar, não seria aceitável dar rollback na mudança de status ([09:04] Bruno; [09:05] Diego).

Por isso, a solução adotada é o padrão **outbox transacional no MySQL**: o evento é gravado na mesma transação da mudança de status e um worker separado dispara as chamadas HTTP ([09:06] Diego; [09:08] Larissa). O detalhamento da decisão está no [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md).

---

## 3. Público-Alvo e Cenários de Uso

A feature é destinada a **sistemas**, não a consumidores finais. Os três públicos abaixo são os que interagem com a configuração, o consumo ou a operação dos webhooks.

| Persona | Objetivo | Cenários principais | Fonte |
|---|---|---|---|
| **Cliente B2B integrador** — sistemas externos da Atlas Comercial, MaxDistribuição e Nova Cargo | Receber notificações de mudança de status e manter seus sistemas atualizados sem polling. | Receber `order.status_changed`; validar a assinatura HMAC; deduplicar pelo `X-Event-Id`; consultar `GET /orders/:id` quando precisar de detalhes como `items`. | [09:00] Marcos; [09:20] Sofia; [09:25] Diego; [09:43] Diego |
| **Operador / usuário autenticado** | Gerenciar os endpoints de webhook do cliente. | Cadastrar endpoint HTTPS com a lista de status desejados; listar, editar e remover webhooks; rotacionar a secret do endpoint. Qualquer role autenticada pode operar o CRUD nesta fase. | [09:21] Sofia; [09:31] Marcos; [09:33] Bruno; [09:36] Marcos; [09:37] Sofia |
| **Administrador (`ADMIN`)** | Inspecionar entregas e recuperar eventos de falha permanente. | Consultar o histórico de entregas para investigar falhas; executar replay manual de um evento na DLQ. O replay é restrito a `ADMIN` e deve registrar quem o executou. | [09:34] Marcos; [09:35] Diego; [09:36] Sofia |

A autenticação é feita com JWT do sistema. Existem usuários que representam o cliente e usam a API diretamente; o `customerId` identifica o customer no body, na query ou no path, conforme o endpoint, e não vem do JWT ([09:32] Marcos; [09:32] Larissa).

### 3.1 Cenário ponta a ponta

1. Um operador autenticado cadastra um webhook para o customer, informando uma URL HTTPS e a lista de status que deseja receber ([09:31] Marcos; [09:33] Bruno).
2. A plataforma gera uma secret única para o endpoint e a devolve na criação ([09:31] Marcos; [09:21] Sofia).
3. Um pedido do customer muda de status no OMS ([09:00] Marcos).
4. Se ao menos um webhook ativo do customer estiver interessado no novo status, o evento `order.status_changed` é registrado e posteriormente entregue ao endpoint ([09:34] Bruno; [09:34] Diego).
5. O sistema do cliente valida a assinatura HMAC, deduplica pelo `X-Event-Id` e atualiza seu estado interno ([09:20] Sofia; [09:25] Diego).
6. Se o cliente precisar de dados detalhados do pedido, como os itens, consulta `GET /orders/:id` ([09:43] Diego).
7. Se a entrega falhar, a plataforma aplica retry com backoff; após o teto, o evento vai para a DLQ e pode ser reprocessado por um administrador ([09:15] Diego; [09:18] Diego; [09:36] Sofia).

---

## 4. Objetivos e Métricas de Sucesso

### 4.1 Objetivo de negócio

Entregar a capacidade de webhooks outbound para os três clientes B2B dentro da janela comercial acordada, reduzindo o risco de churn da Atlas. A estimativa é de **três sprints incluindo a revisão de segurança da Sofia** ([09:45] Marcos; [09:46] Larissa; [09:47] Larissa).

### 4.2 Métricas de produto

| ID | Objetivo | Métrica | Meta | Fonte |
|---|---|---|---|---|
| **PRD-OBJ-01** | Notificação percebida como “tempo real” | Tempo entre a mudança de status e o disparo da notificação | **< 10 segundos**. O worker faz polling a cada 2 segundos; esse é o piso operacional do worker e, no pior caso, adiciona até ~2 segundos antes do disparo. | [09:02] Marcos; [09:10] Larissa; [09:10] Marcos |
| **PRD-OBJ-02** | Resiliência a indisponibilidades temporárias do cliente | Janela de recuperação e número de tentativas | **~15 horas** de cobertura, com **5 tentativas** e backoff de **1m / 5m / 30m / 2h / 12h**. | [09:15] Diego; [09:17] Diego; [09:17] Larissa |
| **PRD-OBJ-03** | Visibilidade operacional das entregas recentes | Envios consultáveis por webhook | Consulta dos **últimos 100 envios** por webhook, com sucesso/falha, payload, response e tempo de resposta. | [09:34] Marcos |
| **PRD-OBJ-04** | Reduzir/eliminar o polling dos clientes B2B | Dependência de `GET /orders` para descoberta de mudança de status | Eliminar a necessidade de polling para descoberta de mudança de status. A call **não definiu percentual** de redução de tráfego. | [09:00] Marcos |
| **PRD-OBJ-05** | Cumprir a janela comercial da Atlas | Prazo de entrega | **3 sprints**, incluindo a revisão de segurança, até o **fim de novembro / fim do trimestre**. | [09:45] Marcos; [09:46] Larissa; [09:47] Larissa |

> **Nota:** a reunião não definiu metas de volume de eventos, receita, disponibilidade, taxa de sucesso ou percentual de redução de polling. Este PRD registra apenas as metas quantitativas acordadas; não cria números sem origem.

---

## 5. Escopo (Incluso e Fora de Escopo)

### 5.1 Incluso nesta entrega

- **Webhooks outbound** de mudança de status de pedidos para clientes B2B, com `event_type = order.status_changed` ([09:00] Marcos; [09:43] Diego).
- **CRUD autenticado de endpoints**: cadastro via `POST`, listagem via `GET`, edição via `PATCH` e remoção via `DELETE` ([09:31] Marcos; [09:33] Bruno).
- **Filtro de status por webhook**: cada endpoint escolhe quais status deseja receber; o filtro é aplicado na inserção do evento na outbox ([09:33] Marcos; [09:34] Bruno).
- **Estado ativo** do webhook e associação ao `customerId` informado no body, na query ou no path, não derivado do JWT ([09:21] Bruno; [09:21] Sofia; [09:32] Larissa).
- **Secret única por endpoint**, gerada pela plataforma e devolvida na criação ([09:21] Sofia; [09:31] Marcos).
- **Rotação de secret** via API, com a secret anterior válida por **24 horas** em paralelo e depois expirada ([09:21] Sofia).
- **Assinatura HMAC-SHA256** sobre o corpo do request, enviada no header `X-Signature` ([09:20] Sofia; [09:22] Sofia).
- **Headers de entrega**: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` ([09:25] Diego; [09:44] Diego; [09:44] Sofia).
- **Entrega at-least-once** com `X-Event-Id` único por evento para deduplicação no cliente ([09:24] Diego; [09:25] Diego; [09:26] Larissa).
- **Snapshot do payload** no momento da inserção na outbox, sem `items`, refletindo o estado do pedido naquela transição ([09:43] Diego; [09:52] Larissa).
- **Outbox transacional no MySQL**, com a inserção do evento dentro da mesma transação de `OrderService.changeStatus` ([09:06] Diego; [09:40] Bruno; [09:41] Diego).
- **Worker separado** com polling de 2 segundos, single-worker nesta fase ([09:09] Diego; [09:10] Larissa; [09:11] Diego; [09:12] Diego).
- **Retry com backoff** de 1m/5m/30m/2h/12h, **timeout HTTP de 10 segundos** e **DLQ** para falha permanente ([09:15] Diego; [09:17] Larissa; [09:18] Diego; [09:42] Diego).
- **Histórico dos últimos 100 envios** por webhook ([09:34] Marcos).
- **Replay manual de DLQ** por endpoint administrativo, restrito a `ADMIN` e com log de auditoria de quem executou ([09:18] Diego; [09:35] Diego; [09:36] Sofia).
- **Limite de payload de 64KB**, com erro caso ultrapasse e **sem truncamento** ([09:23] Sofia; [09:24] Diego; [09:24] Larissa).
- **Documentação de integração no portal do desenvolvedor**, incluindo HMAC, deduplicação e consulta a `GET /orders/:id`, sob responsabilidade do PM ([09:26] Marcos; [09:40] Marcos).

### 5.2 Fora de escopo

| Item | Status | Justificativa | Fonte |
|---|---|---|---|
| **Webhooks inbound** — cliente enviando eventos para a plataforma | **Descartado** | Os clientes querem receber, não mandar. O escopo é somente outbound. | [09:02] Marcos; [09:03] Sofia |
| **Dashboard/painel visual para o cliente** | **Fora de escopo** | Nesta fase, somente endpoints. O painel é um projeto separado do time de frontend. | [09:39] Marcos; [09:40] Larissa |
| **Notificação por e-mail em caso de falha de webhook** | **Adiado para fase futura** | E-mail está fora do escopo desta fase; a decisão é medir o impacto antes de considerar a próxima fase. A sugestão de “3 falhas seguidas” não foi fechada como limiar. | [09:37] Marcos; [09:37] Larissa; [09:38] Marcos |
| **Arquivamento/limpeza automática de eventos entregues após 30 dias** | **Fora de escopo** | O arquivamento de linhas entregues após 30 dias foi explicitamente deixado fora do escopo desta feature. A outbox cresce enquanto essa rotina não existir. | [09:08] Diego |
| **Rate limiting de envio para o cliente** | **Adiado / ponto em aberto** | Não faz parte do escopo; a decisão é observar e implementar se virar problema. | [09:38] Diego; [09:39] Larissa |
| **Garantia de ordering global entre pedidos diferentes** | **Não exigida / fora de escopo** | A garantia é apenas por `order_id` e enquanto houver single-worker. Os clientes nunca pediram ordering global; querem saber se cada pedido mudou. | [09:13] Larissa; [09:14] Marcos |
| **Endurecimento da autorização do CRUD de webhooks** | **Adiado** | Por enquanto qualquer role autenticada pode operar o CRUD; a possibilidade de endurecer fica para mais adiante. | [09:36] Marcos; [09:37] Sofia |

> **Limites aceitos nesta fase:** single-worker, ordering apenas por `order_id`, outbox sem rotina de expurgo e ausência de rate limiting. Esses pontos são decisões de escopo, não defeitos da entrega.

---

## 6. Requisitos Funcionais

| ID | Requisito | Fonte |
|---|---|---|
| **PRD-FR-01** | Notificar clientes B2B, por webhook outbound, quando o status de um pedido mudar, usando `event_type = order.status_changed`. | [09:00] Marcos; [09:43] Diego |
| **PRD-FR-02** | Exigir autenticação JWT em todas as operações de configuração. O CRUD de webhooks pode ser usado por qualquer role autenticada nesta fase. O `customerId` deve ser informado no body, na query ou no path, conforme o endpoint; ele não é derivado do JWT, pois o JWT atual representa o operador e não o cliente. | [09:31] Marcos; [09:32] Larissa; [09:36] Marcos; [09:37] Sofia |
| **PRD-FR-03** | Permitir o cadastro de webhook via `POST`, com URL, lista de status de interesse e estado ativo (default `true`). A plataforma gera a secret e a devolve na criação. | [09:31] Marcos; [09:21] Sofia |
| **PRD-FR-04** | Permitir a listagem dos webhooks de um customer via `GET`. | [09:33] Bruno |
| **PRD-FR-05** | Permitir a edição de um webhook via `PATCH`, incluindo URL, lista de status e estado ativo. | [09:33] Bruno; [09:21] Bruno |
| **PRD-FR-06** | Permitir a remoção de um webhook via `DELETE`. | [09:33] Bruno |
| **PRD-FR-07** | Aplicar o filtro de status na inserção do evento na outbox: se nenhum webhook ativo do customer quiser aquele status, o evento não é inserido. | [09:33] Marcos; [09:34] Bruno; [09:34] Diego |
| **PRD-FR-08** | Permitir a rotação da secret por endpoint por meio de endpoint autenticado, gerando uma nova secret. A secret anterior permanece válida por 24 horas em paralelo e depois expira. | [09:21] Sofia |
| **PRD-FR-09** | Assinar o corpo do request com HMAC-SHA256 e enviar a assinatura no header `X-Signature`. | [09:20] Sofia; [09:22] Sofia |
| **PRD-FR-10** | Enviar os headers `X-Event-Id` com UUID único do evento, `X-Timestamp` com o timestamp do envio (que o cliente pode usar para detectar replay attack, se quiser), `X-Webhook-Id` com o id do endpoint (para o cliente saber qual cadastro originou o envio) e `Content-Type: application/json`. | [09:25] Diego; [09:44] Diego; [09:44] Sofia; [09:45] Diego |
| **PRD-FR-11** | Gravar o payload como snapshot no momento da inserção na outbox, refletindo o estado do pedido naquela transição. O payload contém `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order, como `total_cents`; não inclui `items`. O cliente consulta `GET /orders/:id` se precisar de detalhes. | [09:43] Diego; [09:52] Larissa |
| **PRD-FR-12** | Garantir entrega at-least-once. O cliente pode receber o mesmo evento mais de uma vez e deve deduplicar pelo `X-Event-Id`, que é único por evento e gerado na entrada da outbox. | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| **PRD-FR-13** | Retentar falhas de entrega em 5 tentativas, com backoff de 1m/5m/30m/2h/12h. Após o teto, considerar falha permanente e mover o evento para a DLQ. | [09:15] Diego; [09:17] Larissa |
| **PRD-FR-14** | Registrar a DLQ em estrutura separada, com payload, motivo da falha e timestamp, para debug e reprocessamento. | [09:18] Diego |
| **PRD-FR-15** | Disponibilizar `GET /webhooks/:id/deliveries` com os últimos 100 envios do webhook, incluindo sucesso/falha, payload, response e tempo de resposta. | [09:34] Marcos |
| **PRD-FR-16** | Permitir replay manual de DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, recolocando o evento na outbox como pendente. O replay exige role `ADMIN` e deve registrar quem executou a ação para auditoria. | [09:18] Diego; [09:35] Diego; [09:36] Sofia; [09:36] Larissa |
| **PRD-FR-17** | Inserir o evento na outbox dentro da mesma transação da mudança de status. Se a inserção na outbox falhar, a transação inteira deve sofrer rollback; não pode existir status alterado sem evento. | [09:06] Diego; [09:40] Bruno; [09:41] Diego |
| **PRD-FR-18** | Recusar URLs `http`; apenas URLs `https` são aceitas no cadastro e na edição. | [09:23] Sofia |
| **PRD-FR-19** | Recusar payload acima de 64KB com erro, sem truncar e sem enviar o evento. | [09:23] Sofia; [09:24] Diego; [09:24] Larissa |

> **Nota sobre retry:** a call fechou “5 tentativas” e os cinco intervalos de backoff, mas não explicitou se o envio inicial conta como uma das 5. O [FDD](./FDD.md) registra a premissa de 1 envio inicial + 5 retries para preservar os cinco intervalos; a contagem exata segue como ponto de implementação a confirmar.

---

## 7. Requisitos Não Funcionais

| ID | Requisito | Meta / Restrição | Fonte |
|---|---|---|---|
| **PRD-NFR-01** | Latência de notificação percebida como “tempo real” | **< 10 segundos** entre a mudança de status e o disparo. O polling do worker ocorre a cada **2 segundos**; esse é o piso operacional do worker e, no pior caso, adiciona até ~2 segundos. | [09:02] Marcos; [09:10] Larissa; [09:10] Marcos |
| **PRD-NFR-02** | Atomicidade transacional no MySQL | A mudança de status e a inserção do evento na outbox devem commitar ou sofrer rollback juntas. Se a outbox falhar, a mudança de status não pode ser persistida. | [09:06] Diego; [09:40] Bruno; [09:41] Diego |
| **PRD-NFR-03** | Isolamento de performance | Um cliente lento ou fora do ar não pode bloquear a mudança de status de outros pedidos nem a API de pedidos. | [09:04] Bruno; [09:04] Larissa |
| **PRD-NFR-04** | Timeout HTTP do worker | Cada chamada HTTP tem timeout de **10 segundos**; timeout é tratado como falha e entra no retry. | [09:42] Diego; [09:42] Sofia |
| **PRD-NFR-05** | Limite de payload | Teto de **64KB**; se ultrapassar, a operação deve **errar, sem truncar**. | [09:23] Sofia; [09:24] Diego; [09:24] Larissa |
| **PRD-NFR-06** | HTTPS obrigatório | A URL do webhook deve ser `https`; `http` é recusado com erro de validação. | [09:23] Sofia |
| **PRD-NFR-07** | Garantia de entrega | **At-least-once**; duplicatas são possíveis e o cliente deve deduplicar pelo `X-Event-Id`. | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| **PRD-NFR-08** | Ordering | Garantia apenas por `order_id` e enquanto houver **single-worker**. Não há garantia de ordering global entre pedidos diferentes nem com múltiplos workers. | [09:12] Diego; [09:13] Larissa |
| **PRD-NFR-09** | Segurança da assinatura | HMAC-SHA256 sobre o corpo do request, secret única por endpoint e rotação com grace period de 24 horas. | [09:20] Sofia; [09:21] Sofia; [09:22] Sofia |
| **PRD-NFR-10** | Observabilidade e auditoria | Reutilizar o logger Pino e registrar a auditoria do replay com o usuário administrador que executou a ação. | [09:29] Bruno; [09:36] Sofia |
| **PRD-NFR-11** | Independência do worker | O processamento de entregas deve rodar em processo separado da API, para não ser perdido quando a API reiniciar. | [09:11] Diego; [09:11] Larissa |
| **PRD-NFR-12** | Simplicidade operacional | Usar o MySQL existente; não introduzir Redis Cluster ou nova infraestrutura de fila nesta fase. | [09:07] Diego; [09:07] Larissa |

---

## 8. Decisões e Trade-offs Principais

As decisões abaixo foram fechadas na reunião e detalhadas nos ADRs. O PRD registra o impacto de produto de cada escolha.

| Decisão | Escolha | Trade-off principal | ADR |
|---|---|---|---|
| **Mecanismo de fila** | Outbox transacional no MySQL existente, com inserção na mesma transação da mudança de status. | Consistência transacional e menos infraestrutura; em troca, a outbox cresce e depende de polling. Redis Streams/Cluster foi descartado por exigir mais infraestrutura para um time pequeno. | [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md) |
| **Processo de entrega** | Worker separado em polling de 2 segundos. | Isola o worker de reinícios da API e evita HTTP síncrono; em troca, adiciona até ~2 segundos de latência e mais um processo para operar. Trigger SQL foi descartado por não notificar processo externo. | [ADR-002](./adrs/ADR-002-worker-separado-em-polling.md) |
| **Escala nesta fase** | Single-worker, com ordering implícita por `order_id`. | Preserva a ordem por pedido e simplifica a operação; em troca, limita a vazão. Múltiplos workers, particionamento por `order_id` e lock pessimista ficam para o futuro. | [ADR-002](./adrs/ADR-002-worker-separado-em-polling.md) |
| **Garantia de entrega** | At-least-once com `X-Event-Id` para deduplicação no cliente. | Simplicidade e padrão de mercado; em troca, o cliente pode receber duplicatas e precisa implementar deduplicação. Exactly-once exigiria coordenação dos dois lados. | [ADR-005](./adrs/ADR-005-entrega-at-least-once-e-idempotencia-via-event-id.md) |
| **Conteúdo do evento** | Snapshot do payload renderizado no momento da inserção na outbox. | Preserva o estado do pedido naquela transição, mesmo que ele mude depois; em troca, duplica dados e aumenta o armazenamento. Renderização tardia foi descartada por perder a fidelidade temporal. | [ADR-007](./adrs/ADR-007-snapshot-de-payload-na-insercao-da-outbox.md) |
| **Política de falha** | 5 tentativas com backoff de 1m/5m/30m/2h/12h e DLQ após o teto. | Cobre uma janela de quase 15 horas e evita evento pendurado para sempre; em troca, a entrega pode demorar e exige replay manual após a DLQ. Três tentativas e retry indefinido foram descartados. | [ADR-003](./adrs/ADR-003-politica-de-retry-backoff-e-dlq.md) |
| **Autenticação do webhook** | HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h. | Limita o impacto de um vazamento e permite migração sem interrupção; em troca, adiciona gestão de credenciais e mantém a secret antiga válida por 24h. Secret global foi descartada; a alternativa de secret fixa sem rotação também foi rejeitada. | [ADR-004](./adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md) |
| **Padrões de implementação** | Reuso de `AppError`, Pino, error middleware, módulos, schemas Zod e códigos `WEBHOOK_*`. | Consistência com a codebase e menor custo de revisão; em troca, o módulo herda as limitações e o estilo dos padrões existentes. | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md) |

> **Pontos de detalhe em aberto:** codificação da assinatura (hex ou base64), se o `X-Timestamp` entra no conteúdo assinado, path exato do endpoint de rotação, contagem exata de tentativas e tratamento de `4xx` versus `5xx` permanecem registrados no [RFC](./RFC.md) e recebem premissas no [FDD](./FDD.md). Esses pontos não alteram o escopo de produto.

---

## 9. Dependências

| Dependência | Uso no produto | Fonte |
|---|---|---|
| **Módulo de pedidos — `OrderService.changeStatus`** (`src/modules/orders/order.service.ts`) | Ponto de integração transacional para publicar o evento na outbox. A mudança de status já atualiza `orders`, insere em `order_status_history` e ajusta estoque. | [09:04] Bruno; [09:40] Bruno; [09:41] Bruno |
| **Banco MySQL existente** (`src/config/database.ts`; `prisma/schema.prisma`) | Persistência da configuração, da outbox, da DLQ e do histórico de entregas. A decisão é não introduzir nova infraestrutura. | [09:06] Diego; [09:07] Diego |
| **Autenticação JWT e middleware `requireRole`** (`src/middlewares/auth.middleware.ts`; exemplo de uso em `src/modules/users/user.routes.ts`) | Autenticar o CRUD de webhooks e restringir o replay de DLQ a `ADMIN`. O JWT atual não contém `customerId`; por isso o `customerId` vai no body, na query ou no path, conforme o endpoint. | [09:32] Larissa; [09:36] Larissa; [09:36] Sofia |
| **Logger Pino e error middleware** (`src/shared/logger/index.ts`; `src/middlewares/error.middleware.ts`; `src/shared/errors/app-error.ts`) | Logs estruturados da API e do worker, tratamento centralizado de erros e auditoria do replay. Não será introduzido um novo logger. | [09:29] Bruno; [09:30] Larissa; [09:36] Sofia |
| **Validação Zod** (`src/middlewares/validate.middleware.ts`) | Validar URL HTTPS, lista de status, parâmetros e body dos endpoints, seguindo o padrão existente. | [09:23] Sofia; [09:30] Larissa |
| **Novo processo do worker** (`src/worker.ts` e script `npm run worker`, a criar; padrão de bootstrap e shutdown de `src/server.ts`) | Executar o polling e as entregas em processo separado da API, com ciclo de vida próprio. | [09:11] Diego; [09:11] Larissa; [09:30] Bruno |
| **Revisão de segurança da Sofia** | Portão obrigatório antes do deploy; pelo menos **2 dias úteis** para revisar HMAC e geração de secret. | [09:46] Sofia |
| **Prontidão dos clientes B2B** | Os clientes precisam implementar a verificação HMAC e a deduplicação por `X-Event-Id` para consumir o webhook corretamente. | [09:20] Sofia; [09:24] Diego; [09:25] Diego |
| **Documentação no portal do desenvolvedor** | O PM deve documentar o contrato de integração, o comportamento at-least-once, a deduplicação e o uso de `GET /orders/:id` para detalhes. | [09:26] Marcos; [09:40] Marcos; [09:43] Diego |

---

## 10. Riscos e Mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação | Fonte |
|---|---|---|---|---|---|
| **PRD-R-01** | Crescimento contínuo da `webhook_outbox` sem rotina de expurgo de 30 dias. | **Alta** | **Médio** | Monitorar volume e idade dos eventos pendentes; tratar a rotina de expurgo em fase futura, pois está fora do escopo desta feature. | [09:08] Diego |
| **PRD-R-02** | Cliente B2B não implementa deduplicação por `X-Event-Id` e processa o mesmo evento mais de uma vez. | **Média** | **Alto** | Destacar o comportamento at-least-once e a deduplicação no portal do desenvolvedor; incluir o tema na documentação e no suporte aos três clientes. | [09:24] Diego; [09:25] Diego; [09:26] Marcos |
| **PRD-R-03** | Rajadas de pedidos mudando de status sobrecarregam o endpoint do cliente, pois não há rate limiting nesta fase. | **Média** | **Médio** | Observar o volume e decidir depois; manter o rate limiting como ponto em aberto e orientar os clientes sobre a característica de envio. | [09:38] Diego; [09:39] Larissa |
| **PRD-R-04** | Atraso no cronograma de 3 sprints frente ao prazo da Atlas até o fim do trimestre / fim de novembro. | **Alta** | **Alto** | Proteger o escopo acordado, acompanhar o cronograma semanalmente, agendar os 2 dias úteis da revisão da Sofia e alinhar a expectativa com a Atlas. | [09:45] Marcos; [09:46] Larissa; [09:46] Sofia; [09:47] Marcos |
| **PRD-R-05** | Worker parado ou reiniciado atrasa as notificações e aumenta a fila pendente. | **Média** | **Alto** | Manter o worker em processo separado, com shutdown gracioso; monitorar a fila e as entregas; planejar escala futura se necessário. | [09:09] Diego; [09:11] Diego |
| **PRD-R-06** | Vazamento de secret de um endpoint compromete a autenticidade dos webhooks. | **Baixa** | **Alto** | Secret única por endpoint, rotação com grace period de 24h, redaction de secrets nos logs e revisão de segurança da Sofia antes do deploy. | [09:21] Sofia; [09:22] Diego; [09:46] Sofia |
| **PRD-R-07** | Evento acima de 64KB bloqueia a mudança de status por causar rollback da transação. | **Baixa** | **Médio** | Manter o payload enxuto, sem `items`; falhar com erro explícito, sem truncar; monitorar a ocorrência. | [09:23] Sofia; [09:24] Larissa; [09:43] Diego |

> **Nota:** probabilidade e impacto são avaliações qualitativas do PM para priorização; não são medições feitas na reunião.

---

## 11. Critérios de Aceitação

Os critérios abaixo derivam dos requisitos funcionais, dos requisitos não funcionais e das decisões registradas nas seções anteriores.

| ID | Critério de aceitação | Como verificar |
|---|---|---|
| **PRD-AC-01** | Um cliente com webhook ativo recebe o evento `order.status_changed` em menos de 10 segundos após a mudança de status. | Teste ponta a ponta com mudança de status e endpoint de teste. |
| **PRD-AC-02** | Nenhum evento é gerado quando nenhum webhook ativo do customer quer o novo status. | Teste com filtro de status que exclui o status alterado. |
| **PRD-AC-03** | Um usuário autenticado consegue cadastrar, listar, editar e remover webhooks de um customer, informando o `customerId` no body, na query ou no path, conforme o endpoint. | Testes de API com JWT válido e sem JWT. |
| **PRD-AC-04** | A secret é gerada pela plataforma e devolvida na criação; a rotação mantém a secret anterior válida por 24 horas em paralelo. | Teste de criação e rotação com verificação das duas secrets. |
| **PRD-AC-05** | O request outbound contém `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`; a assinatura HMAC-SHA256 é válida com a secret do endpoint. | Teste de contrato do request outbound. |
| **PRD-AC-06** | O payload é um snapshot da transição, não inclui `items` e contém os campos acordados; `GET /orders/:id` continua disponível para detalhes. | Inspeção do payload recebido e consulta ao pedido. |
| **PRD-AC-07** | Falhas temporárias são retentadas em 5 tentativas com backoff de 1m/5m/30m/2h/12h; após o teto, o evento é movido para a DLQ. | Simulação de endpoint indisponível e verificação da sequência e da DLQ. |
| **PRD-AC-08** | O replay de DLQ recoloca o evento como pendente, exige role `ADMIN` e registra quem executou a ação; usuário não `ADMIN` é recusado. | Teste com usuário `ADMIN` e com usuário de outra role. |
| **PRD-AC-09** | `GET /webhooks/:id/deliveries` retorna os últimos 100 envios, com sucesso/falha, payload, response e tempo de resposta. | Teste de histórico com mais de 100 registros. |
| **PRD-AC-10** | URLs `http` são recusadas com erro de validação; apenas `https` é aceito. | Teste de cadastro e edição com URL HTTP e HTTPS. |
| **PRD-AC-11** | Payload acima de 64KB gera erro, sem truncamento e sem envio do evento. | Teste com payload acima do limite. |
| **PRD-AC-12** | Um cliente lento ou fora do ar não bloqueia a mudança de status de outros pedidos. | Teste com endpoint lento e verificação da transação de pedidos. |
| **PRD-AC-13** | A mudança de status e a inserção do evento commitam ou sofrem rollback juntas. | Teste de falha na inserção da outbox e verificação do rollback. |
| **PRD-AC-14** | A entrega at-least-once e a deduplicação por `X-Event-Id` estão documentadas; a garantia de ordering é apenas por `order_id` enquanto single-worker. | Revisão da documentação e teste de eventos do mesmo pedido. |
| **PRD-AC-15** | Os itens fora de escopo — inbound, dashboard, e-mail, arquivamento de 30 dias, rate limiting e garantia de ordering global — não foram implementados ou prometidos nesta entrega. | Revisão de escopo e demonstração da feature. |
| **PRD-AC-16** | A documentação de integração está publicada no portal do desenvolvedor, incluindo HMAC, deduplicação, at-least-once e consulta a `GET /orders/:id`. | Revisão da documentação publicada. |
| **PRD-AC-17** | A revisão de segurança da Sofia foi concluída antes do deploy, com pelo menos 2 dias úteis reservados. | Registro de conclusão da revisão. |

---

## 12. Estratégia de Testes e Validação

A validação combina testes automatizados, testes de contrato e aceitação de produto. O código existente já possui um harness de testes de pedidos (`tests/orders.test.ts` e `tests/helpers/factories.ts`) que pode ser reaproveitado para o fluxo de mudança de status.

| Frente | O que validar | Critérios cobertos |
|---|---|---|
| **API e autorização** | CRUD autenticado, `customerId` no body/query/path, listagem, edição, remoção, estado ativo e recusa de acesso sem JWT. | PRD-AC-03 |
| **Fluxo ponta a ponta** | Cadastro do webhook, mudança de status no OMS, registro do evento e entrega ao endpoint em menos de 10 segundos. | PRD-AC-01 |
| **Contrato outbound** | Headers, assinatura HMAC-SHA256, `Content-Type`, payload snapshot, ausência de `items` e consulta a `GET /orders/:id`. | PRD-AC-05; PRD-AC-06 |
| **Filtro de status** | Evento não é gerado quando nenhum webhook ativo do customer quer o status; evento é gerado quando ao menos um quer. | PRD-AC-02 |
| **Resiliência** | Timeout de 10 segundos, retry com backoff, teto de 5 tentativas e movimentação para DLQ. | PRD-AC-07 |
| **Replay e auditoria** | Replay por `ADMIN`, retorno do evento a pendente e log de auditoria com o usuário que executou; recusa de não `ADMIN`. | PRD-AC-08 |
| **Histórico de entregas** | Retorno dos últimos 100 envios com sucesso/falha, payload, response e tempo de resposta. | PRD-AC-09 |
| **Validações de segurança** | Recusa de URL HTTP, limite de 64KB sem truncamento, geração e rotação de secret. | PRD-AC-04; PRD-AC-10; PRD-AC-11 |
| **Isolamento e atomicidade** | Cliente lento não bloqueia `changeStatus`; falha na outbox provoca rollback da mudança de status. | PRD-AC-12; PRD-AC-13 |
| **Regressão** | O fluxo existente de pedidos, estoque, histórico e autenticação continua funcionando. | PRD-AC-03; PRD-AC-13 |
| **Segurança e documentação** | Revisão da Sofia antes do deploy e documentação de integração publicada para os clientes. | PRD-AC-16; PRD-AC-17 |

### 12.1 Portões de lançamento

- Todos os critérios de aceitação `PRD-AC-01` a `PRD-AC-17` verificados.
- Revisão de segurança da Sofia concluída, com pelo menos 2 dias úteis antes do deploy.
- Documentação de integração publicada no portal do desenvolvedor.
- Confirmação de que os itens fora de escopo não foram incluídos.
- Alinhamento de prazo e contrato de integração comunicado aos três clientes B2B.

> **Nota sobre capacidade:** a reunião não definiu metas de volume, carga ou throughput. Testes de capacidade devem ser tratados como observação operacional, não como critério de aceite quantitativo deste PRD.
