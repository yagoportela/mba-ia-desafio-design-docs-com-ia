# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

**Escopo varrido:** `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-001` a `ADR-007`.

**Resumo:** 210 linhas mapeadas; **182 `TRANSCRICAO` (86,7%)** e **28 `CODIGO` (13,3%)**. Cobertura: 19/19 `PRD-FR`, 12/12 `PRD-NFR`, 7/7 itens fora de escopo, 8/8 decisões e trade-offs do PRD, 9/9 dependências do PRD, 7/7 riscos do PRD, 17/17 critérios de aceitação do PRD, 7/7 ADRs, 6/6 alternativas do RFC, 6/6 questões em aberto do RFC, 7/7 riscos do RFC, 7/7 contratos de endpoint, 7/7 erros `WEBHOOK_*`, 7/7 objetivos técnicos do FDD, 8 estratégias de resiliência mapeadas, 3 itens de observabilidade mapeados, 17/17 critérios de aceite técnicos, 9/9 riscos do FDD, dependências do FDD e 10/10 áreas de integração do FDD (14 caminhos).

> **Convenção:** `Fonte = TRANSCRICAO` usa a localização no formato `[hh:mm] Nome`; `Fonte = CODIGO` usa caminho real verificado no repositório. IDs como `PRD-OBJ-*`, `PRD-OOS-*`, `PRD-R-*`, `PRD-AC-*`, `RFC-*` e `FDD-*` foram atribuídos neste tracker para granularidade; `PRD-FR-*`, `PRD-NFR-*` e `ADR-*` preservam os IDs dos documentos. O PRD usa `PRD-NFR-*` para requisitos não funcionais; o tracker preserva esse ID. Itens sem origem identificável não foram forçados na tabela; as premissas técnicas estão declaradas no FDD e as questões não decididas estão no RFC.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Notificação em menos de 10 s; polling de 2 s adiciona até ~2 s | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Janela de recuperação de ~15 h com 5 tentativas e backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Histórico dos últimos 100 envios por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Eliminar a dependência de polling no GET /orders; sem percentual definido | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Entrega em 3 sprints, incluindo revisão de segurança, até fim de novembro/trimestre | TRANSCRICAO | [09:45] Marcos; [09:46] Larissa; [09:47] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Webhook outbound de mudança de status com event_type order.status_changed | TRANSCRICAO | [09:00] Marcos; [09:43] Diego |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | CRUD autenticado por JWT; qualquer role autenticada; customerId no body/query/path, não no JWT | TRANSCRICAO | [09:32] Larissa; [09:36] Marcos; [09:37] Sofia |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | POST cria webhook com URL, lista de status e estado ativo; secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos; [09:21] Sofia |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | GET lista webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | PATCH edita URL, lista de status e estado ativo | TRANSCRICAO | [09:33] Bruno; [09:21] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | DELETE remove webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Filtro de status na inserção da outbox; sem webhook interessado, sem evento | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno; [09:34] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret por endpoint, com secret anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do corpo, header X-Signature | TRANSCRICAO | [09:20] Sofia; [09:22] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Timestamp, X-Webhook-Id e Content-Type application/json | TRANSCRICAO | [09:25] Diego; [09:44] Diego; [09:44] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Snapshot do payload na inserção, campos acordados, sem items; detalhes via GET /orders/:id | TRANSCRICAO | [09:43] Diego; [09:52] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Entrega at-least-once; deduplicação do cliente por X-Event-Id | TRANSCRICAO | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | 5 tentativas com backoff 1m/5m/30m/2h/12h; depois DLQ | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | DLQ separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | GET /webhooks/:id/deliveries com os últimos 100 envios | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Replay manual em POST /admin/webhooks/dead-letter/:id/replay, restrito a ADMIN e auditado | TRANSCRICAO | [09:18] Diego; [09:35] Diego; [09:36] Sofia |
| PRD-FR-17 | docs/PRD.md | Requisito Funcional | Inserção na outbox na mesma transação da mudança de status; rollback em falha | TRANSCRICAO | [09:06] Diego; [09:40] Bruno; [09:41] Diego |
| PRD-FR-18 | docs/PRD.md | Requisito Funcional | Recusar URL HTTP; aceitar apenas HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-19 | docs/PRD.md | Requisito Funcional | Payload acima de 64 KB gera erro, sem truncar e sem enviar | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência <10 s; polling de 2 s é o piso operacional e adiciona até ~2 s | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade transacional no MySQL entre status e outbox | TRANSCRICAO | [09:06] Diego; [09:40] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento/offline não bloqueia changeStatus nem a API de pedidos | TRANSCRICAO | [09:04] Bruno; [09:04] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP de 10 s; timeout é falha e entra no retry | TRANSCRICAO | [09:42] Diego; [09:42] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64 KB; erro sem truncamento | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | HTTPS obrigatório; HTTP recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once; duplicatas possíveis e dedup pelo X-Event-Id | TRANSCRICAO | [09:24] Diego; [09:26] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordering apenas por order_id enquanto single-worker; sem ordering global | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256, secret única por endpoint e rotação com grace period de 24 h | TRANSCRICAO | [09:20] Sofia; [09:22] Sofia |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Reutilizar Pino e auditar o replay com o usuário administrador | TRANSCRICAO | [09:29] Bruno; [09:36] Sofia |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado para não ser perdido no restart da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Usar MySQL existente; sem Redis Cluster ou nova infraestrutura de fila | TRANSCRICAO | [09:07] Diego; [09:07] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos; [09:03] Sofia |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Dashboard/painel visual | TRANSCRICAO | [09:39] Marcos; [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | E-mail em falha de webhook, adiado | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Arquivamento/limpeza após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de Escopo | Rate limiting de envio, adiado/observar | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| PRD-OOS-06 | docs/PRD.md | Fora de Escopo | Ordering global | TRANSCRICAO | [09:13] Larissa; [09:14] Marcos |
| PRD-OOS-07 | docs/PRD.md | Fora de Escopo | Endurecimento da autorização do CRUD | TRANSCRICAO | [09:36] Marcos; [09:37] Sofia |
| PRD-R-01 | docs/PRD.md | Risco | Crescimento da webhook_outbox sem expurgo de 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-R-02 | docs/PRD.md | Risco | Cliente sem dedup por X-Event-Id | TRANSCRICAO | [09:24] Diego; [09:26] Marcos |
| PRD-R-03 | docs/PRD.md | Risco | Rajadas sem rate limiting sobrecarregam endpoint do cliente | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| PRD-R-04 | docs/PRD.md | Risco | Atraso de 3 sprints frente ao prazo da Atlas | TRANSCRICAO | [09:45] Marcos; [09:46] Larissa; [09:47] Marcos |
| PRD-R-05 | docs/PRD.md | Risco | Worker parado/reiniciado atrasa notificações | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| PRD-R-06 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:21] Sofia; [09:22] Diego; [09:46] Sofia |
| PRD-R-07 | docs/PRD.md | Risco | Payload >64 KB causa rollback da mudança de status | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa; [09:43] Diego |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | Evento entregue em menos de 10 s | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-AC-02 | docs/PRD.md | Critério de Aceitação | Sem evento se nenhum webhook ativo quer o status | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | CRUD autenticado com customerId no body/query/path | TRANSCRICAO | [09:32] Larissa; [09:36] Marcos |
| PRD-AC-04 | docs/PRD.md | Critério de Aceitação | Secret devolvida na criação; rotação mantém a anterior por 24 h | TRANSCRICAO | [09:21] Sofia; [09:31] Marcos |
| PRD-AC-05 | docs/PRD.md | Critério de Aceitação | Headers e HMAC válidos no request outbound | TRANSCRICAO | [09:20] Sofia; [09:44] Diego |
| PRD-AC-06 | docs/PRD.md | Critério de Aceitação | Snapshot sem items; GET /orders/:id para detalhes | TRANSCRICAO | [09:43] Diego; [09:52] Larissa |
| PRD-AC-07 | docs/PRD.md | Critério de Aceitação | Retry em 5 tentativas e DLQ após o teto | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| PRD-AC-08 | docs/PRD.md | Critério de Aceitação | Replay ADMIN com auditoria; não-ADMIN recusado | TRANSCRICAO | [09:36] Sofia; [09:18] Diego |
| PRD-AC-09 | docs/PRD.md | Critério de Aceitação | GET deliveries retorna os últimos 100 envios | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-10 | docs/PRD.md | Critério de Aceitação | HTTP recusado; HTTPS aceito | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-11 | docs/PRD.md | Critério de Aceitação | Payload >64 KB falha sem truncar | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| PRD-AC-12 | docs/PRD.md | Critério de Aceitação | Cliente lento não bloqueia changeStatus | TRANSCRICAO | [09:04] Bruno |
| PRD-AC-13 | docs/PRD.md | Critério de Aceitação | Status e outbox commitam/rollback juntos | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| PRD-AC-14 | docs/PRD.md | Critério de Aceitação | At-least-once documentado; ordering por order_id | TRANSCRICAO | [09:13] Larissa; [09:24] Diego; [09:26] Larissa |
| PRD-AC-15 | docs/PRD.md | Critério de Aceitação | Itens fora de escopo não implementados/prometidos | TRANSCRICAO | [09:02] Marcos; [09:08] Diego; [09:13] Larissa; [09:14] Marcos; [09:37] Larissa; [09:39] Larissa; [09:40] Larissa |
| PRD-AC-16 | docs/PRD.md | Critério de Aceitação | Documentação publicada no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-AC-17 | docs/PRD.md | Critério de Aceitação | Revisão de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEC-01 | docs/PRD.md | Decisão/Trade-off | Outbox transacional no MySQL, com inserção na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego; [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão/Trade-off | Worker separado em polling de 2 s, com single-worker nesta fase | TRANSCRICAO | [09:09] Diego; [09:10] Larissa; [09:11] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão/Trade-off | Single-worker e ordering apenas por order_id; múltiplos workers adiados | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| PRD-DEC-04 | docs/PRD.md | Decisão/Trade-off | Entrega at-least-once com deduplicação pelo X-Event-Id | TRANSCRICAO | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| PRD-DEC-05 | docs/PRD.md | Decisão/Trade-off | Snapshot do payload na inserção da outbox | TRANSCRICAO | [09:52] Larissa; [09:52] Diego |
| PRD-DEC-06 | docs/PRD.md | Decisão/Trade-off | Retry em 5 tentativas com backoff, timeout de 10 s e DLQ | TRANSCRICAO | [09:15] Diego; [09:17] Larissa; [09:18] Diego; [09:42] Diego |
| PRD-DEC-07 | docs/PRD.md | Decisão/Trade-off | HMAC-SHA256, secret por endpoint e rotação com grace period de 24 h | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia; [09:22] Sofia |
| PRD-DEC-08 | docs/PRD.md | Decisão/Trade-off | Reuso de AppError, Pino, error middleware, módulos, Zod e códigos WEBHOOK_* | TRANSCRICAO | [09:27] Bruno; [09:29] Bruno; [09:30] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | OrderService.changeStatus como ponto de integração transacional | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Banco MySQL existente via Prisma | CODIGO | src/config/database.ts |
| PRD-DEP-03 | docs/PRD.md | Dependência | Autenticação JWT e middleware requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-04 | docs/PRD.md | Dependência | Logger Pino e error middleware centralizado | CODIGO | src/shared/logger/index.ts |
| PRD-DEP-05 | docs/PRD.md | Dependência | Validação Zod de body, query e params | CODIGO | src/middlewares/validate.middleware.ts |
| PRD-DEP-06 | docs/PRD.md | Dependência | Novo worker src/worker.ts e script npm run worker, a criar | TRANSCRICAO | [09:11] Diego; [09:11] Larissa |
| PRD-DEP-07 | docs/PRD.md | Dependência | Revisão de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-08 | docs/PRD.md | Dependência | Prontidão dos clientes B2B para HMAC e deduplicação | TRANSCRICAO | [09:20] Sofia; [09:24] Diego; [09:25] Diego |
| PRD-DEP-09 | docs/PRD.md | Dependência | Documentação de integração no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos; [09:43] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo HTTP síncrono no service de pedidos | TRANSCRICAO | [09:04] Bruno; [09:05] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams/Cluster | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger SQL para notificar o worker | TRANSCRICAO | [09:09] Bruno; [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Retry com 3 tentativas | TRANSCRICAO | [09:16] Bruno; [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Retry indefinido com backoff | TRANSCRICAO | [09:15] Diego |
| RFC-Q-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio: observar e decidir depois | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-Q-02 | docs/RFC.md | Questão em Aberto | Endurecimento da autorização do CRUD | TRANSCRICAO | [09:36] Marcos; [09:37] Sofia |
| RFC-Q-03 | docs/RFC.md | Questão em Aberto | Contagem exata de tentativas: 5 totais ou 1 envio + 5 retries | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| RFC-Q-04 | docs/RFC.md | Questão em Aberto | Codificação HMAC e inclusão do X-Timestamp | TRANSCRICAO | [09:20] Sofia; [09:44] Diego |
| RFC-Q-05 | docs/RFC.md | Questão em Aberto | Tratamento de respostas 4xx versus 5xx | TRANSCRICAO | [09:15] Diego; [09:42] Diego |
| RFC-Q-06 | docs/RFC.md | Questão em Aberto | Nome do arquivo do worker: webhook.worker.ts ou webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| RFC-RISK-01 | docs/RFC.md | Risco | Impacto na transação de orders e rollback se a outbox falhar | TRANSCRICAO | [09:04] Bruno; [09:40] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Operação de um segundo processo Node para o worker | TRANSCRICAO | [09:11] Diego; [09:30] Bruno |
| RFC-RISK-03 | docs/RFC.md | Risco | Revisão de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| RFC-RISK-04 | docs/RFC.md | Risco | Crescimento da outbox sem rotina de arquivamento de 30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-RISK-05 | docs/RFC.md | Risco | Vazão e ordering limitadas pelo single-worker | TRANSCRICAO | [09:12] Diego; [09:13] Diego |
| RFC-RISK-06 | docs/RFC.md | Risco | Duplicatas no cliente por entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| RFC-RISK-07 | docs/RFC.md | Risco | Prazo de 3 sprints frente à expectativa da Atlas | TRANSCRICAO | [09:45] Marcos; [09:46] Larissa; [09:47] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Outbox transacional no MySQL, com inserção na mesma transação | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker separado em polling de 2 s | TRANSCRICAO | [09:10] Larissa; [09:11] Diego |
| ADR-003 | docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md | Decisão | Retry 5 tentativas, backoff, DLQ e replay admin | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Decisão | HMAC-SHA256, secret por endpoint, rotação de 24 h e HTTPS | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-e-idempotencia-via-event-id.md | Decisão | Entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md | Decisão | Reuso de AppError, Pino, error middleware, módulos, Zod e WEBHOOK_* | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-de-payload-na-insercao-da-outbox.md | Decisão | Snapshot do payload na inserção da outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Alternativa Descartada | Filtrar no envio em vez de filtrar na inserção | TRANSCRICAO | [09:34] Diego; [09:34] Bruno |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Worker dentro da mesma instância da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Escala horizontal com múltiplos workers nesta fase | TRANSCRICAO | [09:12] Diego; [09:13] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-politica-de-retry-backoff-e-dlq.md | Alternativa Descartada | Marcar falha na própria outbox em vez de DLQ separada | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| ADR-004-ALT-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Alternativa Descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Alternativa Descartada | Aceitar URLs HTTP | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT-03 | docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md | Alternativa Descartada | Secret fixa sem rotação | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md | Alternativa Descartada | IDs auto incrementais em vez de UUID | TRANSCRICAO | [09:51] Diego; [09:51] Larissa |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md | Alternativa Descartada | Injetar repository de webhook no OrderService | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| ADR-006-ALT-03 | docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md | Alternativa Descartada | Novo logger ou tratamento de erro separado | TRANSCRICAO | [09:29] Bruno |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-de-payload-na-insercao-da-outbox.md | Alternativa Descartada | Guardar apenas order_id e renderizar o payload no envio | TRANSCRICAO | [09:51] Bruno; [09:52] Larissa |
| ADR-007-ALT-02 | docs/adrs/ADR-007-snapshot-de-payload-na-insercao-da-outbox.md | Alternativa Descartada | Truncar payload acima do limite | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| FDD-API-01 | docs/FDD.md | Contrato de Endpoint | POST /api/v1/webhooks; cadastro; 201/400/401/404/422; path e status são premissa do FDD | TRANSCRICAO | [09:31] Marcos; [09:21] Sofia |
| FDD-API-02 | docs/FDD.md | Contrato de Endpoint | GET /api/v1/webhooks; listagem; 200/400/401/404; path e status são premissa do FDD | TRANSCRICAO | [09:33] Bruno |
| FDD-API-03 | docs/FDD.md | Contrato de Endpoint | PATCH /api/v1/webhooks/:id; edição; 200/400/401/404; path e status são premissa do FDD | TRANSCRICAO | [09:33] Bruno |
| FDD-API-04 | docs/FDD.md | Contrato de Endpoint | DELETE /api/v1/webhooks/:id; remoção; 204/401/404; path e status são premissa do FDD | TRANSCRICAO | [09:33] Bruno |
| FDD-API-05 | docs/FDD.md | Contrato de Endpoint | POST /api/v1/webhooks/:id/rotate-secret; rotação; 200/401/404; path e status são premissa do FDD | TRANSCRICAO | [09:21] Sofia |
| FDD-API-06 | docs/FDD.md | Contrato de Endpoint | GET /api/v1/webhooks/:id/deliveries; últimos 100 envios; 200/401/404; path e status são premissa do FDD | TRANSCRICAO | [09:34] Marcos |
| FDD-API-07 | docs/FDD.md | Contrato de Endpoint | POST /api/v1/admin/webhooks/dead-letter/:id/replay; replay; 200/401/403/404; path e status são premissa do FDD | TRANSCRICAO | [09:18] Diego; [09:35] Diego |
| FDD-OUT-01 | docs/FDD.md | Contrato Outbound | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type | TRANSCRICAO | [09:25] Diego; [09:44] Diego; [09:44] Sofia |
| FDD-OUT-02 | docs/FDD.md | Contrato Outbound | Payload com event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id e total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro de Domínio | Código citado como exemplo: WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro de Domínio | Código citado como exemplo: WEBHOOK_INVALID_URL (400) | TRANSCRICAO | [09:28] Bruno; [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro de Domínio | Código citado como exemplo: WEBHOOK_SECRET_REQUIRED (422) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro de Domínio | Código proposto no FDD: WEBHOOK_PAYLOAD_TOO_LARGE (422) | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro de Domínio | Código proposto no FDD: WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:18] Diego; [09:35] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro de Domínio | Código proposto no FDD: WEBHOOK_CUSTOMER_NOT_FOUND (404) | TRANSCRICAO | [09:31] Marcos; [09:32] Larissa |
| FDD-ERR-07 | docs/FDD.md | Erro de Domínio | Código proposto no FDD: WEBHOOK_INVALID_STATUS (400) | TRANSCRICAO | [09:31] Marcos; [09:33] Marcos |
| FDD-INT-01 | docs/FDD.md | Gancho de Integração | OrderService.changeStatus e transação; ponto de chamada do futuro publishWebhookEvent | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Gancho de Integração | Máquina de estados e status válidos do pedido | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Gancho de Integração | AppError como base dos erros WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Gancho de Integração | Classes de erro específicas e padrão de códigos | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Gancho de Integração | authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Gancho de Integração | Error middleware centralizado para AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Gancho de Integração | buildControllers/buildApp e mount em /api/v1 | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Gancho de Integração | Registro de routers e tipo Controllers | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Gancho de Integração | createPrismaClient para o worker em processo separado | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Gancho de Integração | Padrão de bootstrap/shutdown de src/server.ts a replicar no worker | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Gancho de Integração | Modelos existentes, UUID e ausência de modelos de webhook | CODIGO | prisma/schema.prisma |
| FDD-INT-12 | docs/FDD.md | Gancho de Integração | validate para body/query/params e URL HTTPS | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-13 | docs/FDD.md | Gancho de Integração | Logger Pino compartilhado e lista de redaction | CODIGO | src/shared/logger/index.ts |
| FDD-INT-14 | docs/FDD.md | Gancho de Integração | Scripts e dependências; npm run worker a adicionar | CODIGO | package.json |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da transação de pedidos | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Worker separado busca pendentes, entrega e marca o resultado | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Retry com backoff e timeout HTTP de 10 s | TRANSCRICAO | [09:15] Diego; [09:17] Larissa; [09:42] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Movimentação para DLQ e replay manual por ADMIN | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-MODEL-01 | docs/FDD.md | Modelagem | Tabela webhook_outbox, status e índices de leitura | TRANSCRICAO | [09:06] Diego; [09:08] Diego |
| FDD-MODEL-02 | docs/FDD.md | Modelagem | Tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| FDD-MODEL-03 | docs/FDD.md | Modelagem | Configuração com url, secret, customerId e estado ativo | TRANSCRICAO | [09:21] Bruno; [09:21] Sofia |
| FDD-DEP-01 | docs/FDD.md | Dependência | Entry point src/worker.ts e script npm run worker, a criar | TRANSCRICAO | [09:11] Larissa |
| FDD-OBJ-01 | docs/FDD.md | Objetivo Técnico | Garantir consistência transacional entre status e outbox | TRANSCRICAO | [09:06] Diego; [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo Técnico | Atingir latência compatível com polling de 2 s e "tempo real" <10 s | TRANSCRICAO | [09:09] Diego; [09:10] Larissa |
| FDD-OBJ-03 | docs/FDD.md | Objetivo Técnico | Isolar falhas do cliente do caminho transacional de pedidos | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-04 | docs/FDD.md | Objetivo Técnico | Garantir at-least-once, retry com backoff e DLQ | TRANSCRICAO | [09:15] Diego; [09:17] Larissa; [09:24] Diego |
| FDD-OBJ-05 | docs/FDD.md | Objetivo Técnico | Aplicar HMAC-SHA256, secret por endpoint e HTTPS | TRANSCRICAO | [09:19] Sofia; [09:22] Sofia; [09:23] Sofia |
| FDD-OBJ-06 | docs/FDD.md | Objetivo Técnico | Reutilizar Pino e auditar o replay de DLQ | TRANSCRICAO | [09:29] Bruno; [09:36] Sofia |
| FDD-OBJ-07 | docs/FDD.md | Objetivo Técnico | Reutilizar os padrões existentes de módulos, erros, validação e schema | TRANSCRICAO | [09:30] Larissa |
| FDD-RES-01 | docs/FDD.md | Estratégia de Resiliência | Timeout HTTP de 10 s tratado como falha | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Estratégia de Resiliência | Backoff de 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Estratégia de Resiliência | Teto de tentativas e movimentação para DLQ | TRANSCRICAO | [09:15] Diego; [09:18] Diego |
| FDD-RES-04 | docs/FDD.md | Estratégia de Resiliência | Idempotência at-least-once com X-Event-Id | TRANSCRICAO | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| FDD-RES-05 | docs/FDD.md | Estratégia de Resiliência | Rotação de secret com grace period de 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-RES-06 | docs/FDD.md | Estratégia de Resiliência | Inserção atômica na outbox com rollback da transação | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-RES-07 | docs/FDD.md | Estratégia de Resiliência | Worker em processo separado com ciclo de vida próprio | TRANSCRICAO | [09:11] Diego |
| FDD-RES-08 | docs/FDD.md | Estratégia de Resiliência | Limite de payload de 64 KB sem truncamento | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Reutilização do logger Pino existente | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Auditoria do replay com o usuário administrador | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação de logs por X-Request-Id e ids do evento | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DEP-02 | docs/FDD.md | Dependência | uuid 11.0.3 para geração de identificadores | CODIGO | package.json |
| FDD-DEP-03 | docs/FDD.md | Dependência | zod 3.23.8 para schemas e validação | CODIGO | package.json |
| FDD-DEP-04 | docs/FDD.md | Dependência | pino 9.5.0 para logs estruturados | CODIGO | package.json |
| FDD-DEP-05 | docs/FDD.md | Dependência | @prisma/client/prisma 5.22.0 para persistência | CODIGO | prisma/schema.prisma |
| FDD-DEP-06 | docs/FDD.md | Dependência | express 4.21.1 e jsonwebtoken 9.0.2 para API e auth | CODIGO | package.json |
| FDD-DEP-07 | docs/FDD.md | Dependência | fetch global e AbortController no Node >=20 | CODIGO | package.json |
| FDD-AC-01 | docs/FDD.md | Critério de Aceite Técnico | changeStatus grava a outbox na mesma transação e sofre rollback em falha | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-AC-02 | docs/FDD.md | Critério de Aceite Técnico | Nenhum evento quando nenhum webhook ativo quer o toStatus | TRANSCRICAO | [09:34] Bruno; [09:34] Diego |
| FDD-AC-03 | docs/FDD.md | Critério de Aceite Técnico | Snapshot do evento na inserção reflete o estado da transição | TRANSCRICAO | [09:52] Larissa |
| FDD-AC-04 | docs/FDD.md | Critério de Aceite Técnico | Payload >64 KB gera WEBHOOK_PAYLOAD_TOO_LARGE e rollback, sem truncar | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| FDD-AC-05 | docs/FDD.md | Critério de Aceite Técnico | Índice status+createdAt e leitura de PENDING em lote | TRANSCRICAO | [09:08] Diego |
| FDD-AC-06 | docs/FDD.md | Critério de Aceite Técnico | Worker em src/worker.ts com PrismaClient próprio e shutdown gracioso | TRANSCRICAO | [09:11] Diego; [09:11] Larissa; [09:30] Bruno |
| FDD-AC-07 | docs/FDD.md | Critério de Aceite Técnico | Polling de 2 s e single-worker nesta fase | TRANSCRICAO | [09:09] Diego; [09:10] Larissa; [09:12] Diego |
| FDD-AC-08 | docs/FDD.md | Critério de Aceite Técnico | Retry 1m/5m/30m/2h/12h e timeout de 10 s | TRANSCRICAO | [09:17] Larissa; [09:42] Diego |
| FDD-AC-09 | docs/FDD.md | Critério de Aceite Técnico | Após a última falha, DLQ e outbox FAILED | TRANSCRICAO | [09:18] Diego |
| FDD-AC-10 | docs/FDD.md | Critério de Aceite Técnico | Replay preserva eventId/payload, reativa PENDING e exige ADMIN | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-AC-11 | docs/FDD.md | Critério de Aceite Técnico | At-least-once com X-Event-Id para deduplicação | TRANSCRICAO | [09:24] Diego; [09:26] Larissa |
| FDD-AC-12 | docs/FDD.md | Critério de Aceite Técnico | HMAC-SHA256, secret única, rotação de 24 h e HTTPS | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia; [09:23] Sofia |
| FDD-AC-13 | docs/FDD.md | Critério de Aceite Técnico | Todos os endpoints usam authenticate; replay usa requireRole ADMIN | TRANSCRICAO | [09:32] Larissa; [09:36] Larissa |
| FDD-AC-14 | docs/FDD.md | Critério de Aceite Técnico | Erros WEBHOOK_* tratados pelo error middleware existente | TRANSCRICAO | [09:28] Bruno; [09:29] Bruno |
| FDD-AC-15 | docs/FDD.md | Critério de Aceite Técnico | Logs Pino sem secrets e auditoria do replay | TRANSCRICAO | [09:29] Bruno; [09:36] Sofia |
| FDD-AC-16 | docs/FDD.md | Critério de Aceite Técnico | Nenhuma funcionalidade descartada/adiada implementada | TRANSCRICAO | [09:08] Diego; [09:37] Larissa; [09:39] Larissa; [09:40] Larissa; [09:13] Diego |
| FDD-AC-17 | docs/FDD.md | Critério de Aceite Técnico | Reaproveitamento do harness de testes de pedidos para a cobertura técnica | CODIGO | tests/orders.test.ts |
| FDD-RISK-01 | docs/FDD.md | Risco | Falha na inserção da outbox provoca rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Crescimento da outbox sem rotina de arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Single-worker limita vazão e ordering | TRANSCRICAO | [09:12] Diego; [09:13] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Duplicatas no cliente por at-least-once | TRANSCRICAO | [09:24] Diego; [09:26] Marcos |
| FDD-RISK-05 | docs/FDD.md | Risco | Vazamento de secret compromete a assinatura | TRANSCRICAO | [09:21] Sofia; [09:46] Sofia |
| FDD-RISK-06 | docs/FDD.md | Risco | Payload acima de 64 KB causa rollback | TRANSCRICAO | [09:43] Diego; [09:23] Sofia |
| FDD-RISK-07 | docs/FDD.md | Risco | Worker parado ou reiniciado atrasa a entrega | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-08 | docs/FDD.md | Risco | Tratamento indefinido de 4xx versus 5xx | TRANSCRICAO | [09:15] Diego; [09:42] Diego |
| FDD-RISK-09 | docs/FDD.md | Risco | Prazo e revisão de segurança podem atrasar a entrega | TRANSCRICAO | [09:45] Marcos; [09:46] Larissa; [09:46] Sofia |
