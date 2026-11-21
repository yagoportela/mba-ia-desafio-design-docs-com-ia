# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

**Escopo varrido:** `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-001` a `ADR-007`.

**Resumo:** 136 linhas mapeadas; **122 `TRANSCRICAO` (89,7%)** e **14 `CODIGO` (10,3%)**. Cobertura: 19/19 `PRD-FR`, 12/12 `PRD-NFR`, 7/7 itens fora de escopo, 7/7 ADRs, 6/6 alternativas do RFC, 6/6 questões em aberto do RFC, 7/7 contratos de endpoint, 7/7 erros `WEBHOOK_*` e 10/10 áreas de integração do FDD (14 caminhos).

> **Convenção:** `Fonte = TRANSCRICAO` usa a localização no formato `[hh:mm] Nome`; `Fonte = CODIGO` usa caminho real verificado no repositório. IDs como `PRD-OBJ-*`, `PRD-OOS-*`, `PRD-R-*`, `PRD-AC-*`, `RFC-*` e `FDD-*` foram atribuídos neste tracker para granularidade; `PRD-FR-*`, `PRD-NFR-*` e `ADR-*` preservam os IDs dos documentos. O PRD usa `PRD-NFR-*` para requisitos não funcionais; o tracker preserva esse ID. Itens sem origem identificável não foram forçados na tabela e estão listados separadamente no chat.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Notificação em menos de 10 s; polling de 2 s adiciona até ~2 s | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Janela de recuperação de ~15 h com 5 tentativas e backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Histórico dos últimos 100 envios por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Eliminar a dependência de polling no GET /orders; sem percentual definido | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Entrega em 3 sprints, incluindo revisão de segurança, até fim de novembro/trimestre | TRANSCRICAO | [09:45] Marcos; [09:46] Larissa; [09:47] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Webhook outbound de mudança de status com event_type order.status_changed | TRANSCRICAO | [09:00] Marcos; [09:43] Diego |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | CRUD autenticado por JWT; qualquer role autenticada; customer_id no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa; [09:36] Marcos; [09:37] Sofia |
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
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | CRUD autenticado com customer_id no body/path | TRANSCRICAO | [09:32] Larissa; [09:36] Marcos |
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
| FDD-MODEL-03 | docs/FDD.md | Modelagem | Configuração com url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno; [09:21] Sofia |
| FDD-DEP-01 | docs/FDD.md | Dependência | Entry point src/worker.ts e script npm run worker, a criar | TRANSCRICAO | [09:11] Larissa |
