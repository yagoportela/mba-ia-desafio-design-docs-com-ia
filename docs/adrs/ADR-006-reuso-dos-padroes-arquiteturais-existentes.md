# ADR-006 — Reuso dos padrões arquiteturais existentes

## Status

Aceito.

## Contexto

A codebase já tem um padrão claro: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas ([09:27] Bruno). Bruno propôs que o webhook seguisse o mesmo padrão, em `src/modules/webhooks`, e Diego concordou ([09:27] Bruno; [09:28] Diego). Larissa formalizou o reuso máximo: `AppError`, Pino, error middleware, padrão de módulos, padrão de schemas Zod e padrão de códigos de erro; o webhook fica como um módulo igual aos outros ([09:30] Larissa).

O projeto já tem classes de erro específicas, como `InsufficientStockError` e `InvalidStatusTransitionError`, com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`; Bruno propôs seguir o mesmo padrão para webhook, com códigos como `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno). Larissa determinou o prefixo `WEBHOOK_` para tudo do módulo ([09:29] Larissa). O logger Pino já está no projeto inteiro e não será trocado ([09:29] Bruno). O middleware de erro centralizado já trata `AppError`, `Zod` e `Prisma`, e vai capturar os erros do módulo sem mudança ([09:29] Bruno).

O `requireRole` já existe e será reutilizado para o replay de DLQ, que exige `ADMIN` ([09:36] Larissa). O padrão de IDs do projeto é UUID; Larissa confirmou “UUID, segue o padrão do resto do projeto. Tudo é uuid” ([09:51] Larissa). A lógica do worker fica dentro do módulo, em `webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).

Os caminhos reais que materializam esses padrões são:

- `src/shared/errors/app-error.ts` — classe base `AppError` com `statusCode`, `errorCode` e `details` (L3–16).
- `src/shared/errors/http-errors.ts` — classes específicas como `InvalidStatusTransitionError` (L45–53) e `InsufficientStockError` (L55–63).
- `src/shared/errors/index.ts` — barrel de exportação dos erros (L1–13).
- `src/middlewares/error.middleware.ts` — tratamento centralizado de `AppError`, `ZodError` e erros conhecidos do Prisma (L14–65).
- `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole` (L27–61).
- `src/middlewares/validate.middleware.ts` — validação Zod de body, query e params (L11–37).
- `src/shared/logger/index.ts` — logger Pino compartilhado (L13–30).
- `src/modules/orders/order.service.ts` — transação de `changeStatus` (L126–179) e tipo `TxClient` (L24).
- `src/modules/orders/order.status.ts` — máquina de estados (L3–37).
- `src/modules/orders/order.schemas.ts` — padrão de schemas Zod (L4–30).
- `src/modules/orders/order.routes.ts`, `order.controller.ts` e `order.repository.ts` — padrão de rotas, controller e repository.
- `prisma/schema.prisma` — enums `UserRole` e `OrderStatus` (L11–23), modelos com IDs UUID (`@default(uuid()) @db.Char(36)`) e relações com `Customer` e `Order` (L25–131).

## Decisão

Criar o módulo `src/modules/webhooks` com a mesma estrutura dos demais domínios: controller, service, repository, routes e schemas ([09:27] Bruno; [09:30] Larissa). A lógica de processamento do worker fica dentro desse módulo, em `webhook.worker.ts` ou `webhook.processor.ts`; o nome final permanece em aberto e está registrado no RFC/FDD ([09:28] Bruno).

Reutilizar os padrões existentes:

- **Erros:** novas classes específicas para o domínio de webhooks, seguindo o padrão de `AppError` e das classes de `src/shared/errors/http-errors.ts`, com códigos prefixados por `WEBHOOK_` ([09:28] Bruno; [09:29] Larissa). Os erros devem ser exportados por `src/shared/errors/index.ts` e tratados pelo `src/middlewares/error.middleware.ts` sem mudança ([09:29] Bruno).
- **Logger:** usar o Pino já existente em `src/shared/logger/index.ts`; não adicionar um novo logger ([09:29] Bruno).
- **Autenticação:** reutilizar `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts`; o replay de DLQ exige `ADMIN` ([09:36] Larissa).
- **Validação:** reutilizar `validate` de `src/middlewares/validate.middleware.ts` e o padrão de schemas Zod de `src/modules/orders/order.schemas.ts` ([09:30] Larissa).
- **IDs:** usar UUID, seguindo o padrão de `prisma/schema.prisma` ([09:51] Larissa).
- **Integração com pedidos:** usar a função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` da transação de `OrderService.changeStatus`, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno; [09:41] Diego).

**Detalhes a fechar no FDD/RFC:** a transcrição cita apenas exemplos de códigos `WEBHOOK_*`, não a lista completa ([09:28] Bruno). A hierarquia exata das novas classes de erro, os nomes finais dos arquivos do módulo, o nome final do arquivo do worker (`webhook.worker.ts` ou `webhook.processor.ts`) e os modelos/tabelas de configuração de webhook e de deliveries não foram definidos ([09:27]–[09:28] Bruno; [09:34] Marcos). O `prisma/schema.prisma` atual não contém modelos de webhook, outbox, DLQ ou deliveries.

## Alternativas Consideradas

### IDs auto incrementais em vez de UUID

Diego perguntou se a outbox deveria usar id auto incremental ou UUID ([09:51] Diego); Larissa respondeu UUID, seguindo o padrão do resto do projeto, no qual “tudo é uuid” ([09:51] Larissa). A alternativa de auto incremental foi descartada para manter consistência com o padrão existente.

### Injetar um repository de webhook no `OrderService`

Bruno considerou que a integração o obrigaria a passar um repository de webhook para o `OrderService` ou usar uma função de “enqueue event” ([09:41] Bruno). Ele propôs `publishWebhookEvent(tx, order, fromStatus, toStatus)`, e Diego preferiu a função pura recebendo o `tx`, sem injetar o repository inteiro ([09:41] Diego). A alternativa de injetar o repository foi descartada por acoplar mais o serviço de pedidos ao módulo de webhooks.

### Adicionar um novo logger

Bruno afirmou que o logger Pino já está no projeto inteiro e que “não vamos botar nada novo” ([09:29] Bruno). A alternativa de introduzir um logger próprio para webhooks foi descartada para evitar duplicação de infraestrutura de log.

### Alterar o middleware de erro centralizado

Bruno observou que o middleware já trata `AppError`, `Zod` e `Prisma` e que vai capturar os erros do módulo sem precisar mudar nada ([09:29] Bruno). A alternativa de criar um tratamento de erro separado foi descartada para manter o comportamento e o formato de resposta existentes.

## Consequências

### Positivas

- O módulo de webhooks nasce consistente com os demais domínios, reduzindo curva de aprendizado e revisão ([09:27] Bruno; [09:30] Larissa).
- O tratamento de erros, logging e autenticação já existe e é reutilizado, evitando duplicação ([09:29] Bruno; [09:36] Larissa).
- O prefixo `WEBHOOK_` mantém a rastreabilidade dos erros do módulo ([09:29] Larissa).
- A função `publishWebhookEvent(tx, ...)` mantém o `OrderService` menos acoplado ao repository de webhooks ([09:41] Bruno; [09:41] Diego).
- O padrão UUID é consistente com o schema Prisma existente ([09:51] Larissa; `prisma/schema.prisma`).

### Negativas

- O módulo herda as limitações e o estilo dos padrões existentes; não há um tratamento específico para particularidades de webhooks além do previsto ([09:30] Larissa).
- A lista de códigos `WEBHOOK_*` e a hierarquia de erros ainda precisam ser detalhadas ([09:28] Bruno).
- O nome final do arquivo do worker e dos modelos de dados permanece em aberto ([09:28] Bruno; [09:34] Marcos).
- O `prisma/schema.prisma` não tem modelos de webhook; a integração de persistência exigirá novas tabelas e relações ([09:21] Sofia; [09:34] Marcos).

### Trade-offs

- **Consistência versus especialização:** reutilizar os padrões existentes reduz complexidade, mas pode não acomodar necessidades específicas de webhooks sem extensões futuras ([09:30] Larissa).
- **Função pura versus repository injetado:** a função `publishWebhookEvent(tx, ...)` reduz acoplamento, mas concentra a lógica de enfileiramento em uma função em vez de um repository ([09:41] Bruno; [09:41] Diego).
- **UUID versus auto incremental:** UUID mantém o padrão e permite identificadores globais, mas não é sequencial; a call não detalha impactos de índice ou armazenamento ([09:51] Larissa).
- **Reuso do middleware versus tratamento dedicado:** o middleware centralizado evita mudanças, mas o módulo fica dependente do formato de erro existente ([09:29] Bruno).
