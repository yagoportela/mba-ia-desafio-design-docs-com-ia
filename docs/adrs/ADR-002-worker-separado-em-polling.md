# ADR-002 — Worker separado em polling

## Status

Aceito.

## Contexto

Com o padrão outbox definido no ADR-001, é necessário um processo que leia os eventos pendentes e dispare as chamadas HTTP. Diego propôs polling em loop: a cada 2 segundos, buscar os eventos pendentes mais antigos, processar e marcar ([09:09] Diego). Larissa registrou a decisão: “Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos” ([09:10] Larissa). Marcos confirmou que 2 segundos atende ao requisito de tempo real ([09:10] Marcos).

O worker não pode rodar dentro da mesma instância da API; se a API reiniciar, o worker seria perdido ([09:11] Diego). Larissa indicou uma entry point nova, `src/worker.ts`, e um script `npm run worker`, seguindo o modelo de `src/server.ts` ([09:11] Larissa). O worker usa o mesmo banco e a mesma stack, mas não o mesmo processo ([09:11] Diego). Bruno esclareceu que `PrismaClient` é por processo: o worker abre uma instância nova, com o mesmo banco e a mesma `DATABASE_URL` ([09:30] Bruno).

Nesta fase, a operação é single-worker. Com um único worker, os eventos são processados na ordem de `created_at` da outbox, o que dá ordering implícita por `order_id`; múltiplos workers em paralelo perderiam essa garantia ([09:12] Diego). Larissa documentou a limitação: não há garantia de ordering global, apenas por `order_id` e enquanto for single-worker ([09:13] Larissa).

No código atual, `src/server.ts` concentra o bootstrap da API e o shutdown gracioso com `SIGINT`/`SIGTERM` (L6–22), e `src/config/database.ts` expõe `createPrismaClient()` (L4–8) e o singleton `prisma` (L10). O `package.json` não possui script `worker`; os scripts existentes estão em L10–21. A lógica de processamento deve ficar dentro do módulo de webhooks, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).

## Decisão

Criar uma entry point separada `src/worker.ts` e um script `npm run worker` ([09:11] Larissa). O worker roda como processo separado da API ([09:11] Diego), conecta-se ao mesmo banco e usa a mesma stack, com uma instância própria de `PrismaClient` por processo ([09:11] Diego; [09:30] Bruno). O factory existente `createPrismaClient()` em `src/config/database.ts` (L4–8) é o ponto de apoio para essa instância; o singleton `prisma` (L10) é o cliente do processo da API.

O processamento é feito por polling a cada 2 segundos: buscar os eventos pendentes mais antigos, processar e marcar como entregue ([09:09] Diego; [09:10] Larissa). A operação é single-worker nesta fase, com ordering implícita por `created_at` e por `order_id`; não há garantia de ordering global ([09:12] Diego; [09:13] Larissa). Múltiplos workers, particionamento por `order_id` e lock pessimista ficam para o futuro ([09:13] Diego).

A lógica de processamento fica dentro do módulo `src/modules/webhooks`, em `webhook.worker.ts` ou `webhook.processor.ts`; a escolha do nome final não foi fechada ([09:28] Bruno).

**Detalhes a fechar no FDD/RFC:** a transcrição não define a query exata de polling, o tamanho quantificado do “batch pequeno” ([09:08] Diego), a máquina de estados do worker (`processando`, `falhou`, `entregue`) nem o comportamento exato de shutdown do worker. O nome final do arquivo de processamento também permanece em aberto: `webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).

## Alternativas Consideradas

### Manter o worker dentro do processo da API

Diego descartou essa alternativa porque, se a API reiniciar, o worker é perdido ([09:11] Diego). A separação de processo isola o ciclo de vida do worker do ciclo de vida HTTP, ao custo de operar mais um processo.

### Usar trigger do banco para acordar o worker

Bruno perguntou se um trigger do banco não seria mais reativo ([09:09] Bruno); Diego explicou que o MySQL não tem listener nativo tipo `NOTIFY`/`LISTEN`, que o trigger só executa SQL e não notifica processo externo, e que improvisar arquivo ou endpoint ficaria esquisito; o polling de 2 segundos atende ao requisito ([09:09] Diego). A alternativa foi descartada por não notificar um processo externo de forma nativa.

### Escalar para múltiplos workers já nesta fase

Diego afirmou que múltiplos workers em paralelo perderiam a garantia de ordering; por enquanto seria single-worker e ordering implícita por `order_id` ([09:12] Diego). A escalabilidade futura poderia usar particionamento por `order_id` ou lock pessimista, mas isso é “problema do futuro, não agora” ([09:13] Diego). A alternativa foi adiada para preservar a ordering nesta fase.

## Consequências

### Positivas

- Reiniciar a API não interrompe o worker, porque ele é um processo separado ([09:11] Diego).
- O polling de 2 segundos atende ao requisito de latência abaixo de 10 segundos ([09:10] Larissa; [09:10] Marcos).
- O worker usa o mesmo banco e a mesma stack, sem introduzir nova infraestrutura ([09:11] Diego).
- O single-worker simplifica a ordering nesta fase: os eventos são processados na ordem de `created_at`, com ordering implícita por `order_id` ([09:12] Diego).

### Negativas

- A latência mínima de entrega é de 2 segundos no pior caso, por causa do polling ([09:10] Larissa).
- O worker depende de polling contínuo; a transcrição não quantifica o custo de polling nem o volume de consultas.
- O single-worker limita a capacidade de processamento e impede escala horizontal nesta fase ([09:12] Diego; [09:13] Diego).
- O processo separado precisa de sua própria instância de `PrismaClient` e de seu próprio ciclo de vida, incluindo conexão e shutdown ([09:11] Diego; [09:30] Bruno).

### Trade-offs

- **Simplicidade versus latência:** o polling evita infraestrutura de fila e triggers, mas impõe um piso de 2 segundos ([09:09] Diego; [09:10] Larissa).
- **Ordering versus escala:** manter um único worker preserva a ordem por `order_id`, mas limita a vazão; a escala futura exigirá particionamento ou lock ([09:12] Diego; [09:13] Diego).
- **Processo separado versus operação:** a separação protege o worker de reinícios da API, mas adiciona mais um processo para operar ([09:11] Diego).
- **Polling versus reatividade:** o polling é simples e previsível, mas não é orientado a eventos como um trigger ou uma fila dedicada ([09:09] Diego).
