# ADR-003 — Política de retry, backoff e DLQ

## Status

Aceito.

## Contexto

O worker pode encontrar clientes offline, lentos ou com falhas temporárias. Diego propôs backoff exponencial: tentar de novo depois de algum tempo, aumentar o intervalo e, depois de um teto de tentativas, considerar falha permanente e mover para uma DLQ ([09:15] Diego). A política fechada foi de 5 tentativas com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas ([09:17] Larissa). O total é de quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego).

Para a DLQ, Diego propôs uma tabela `webhook_dead_letter` separada, com payload, motivo da falha e timestamp; a leitura da outbox principal fica mais limpa e a DLQ serve como evidência para debug e reprocessamento ([09:18] Diego). O replay manual seria feito por um endpoint admin, `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego; [09:35] Diego). Sofia exigiu role `ADMIN` e log de quem fez o replay, para auditoria ([09:36] Sofia); Larissa decidiu que o replay usa o `requireRole` existente ([09:36] Larissa).

O timeout do HTTP call do worker foi definido em 10 segundos: cliente lento que não responde em 10 segundos é tratado como falha e marcado para retry ([09:42] Diego; [09:42] Sofia). No código, o guard de autorização já existe em `src/middlewares/auth.middleware.ts` (`requireRole`, L49–61), e há um exemplo de uso com `requireRole('ADMIN')` em `src/modules/users/user.routes.ts` (L12–18). O `prisma/schema.prisma` ainda não possui modelo de DLQ; a tabela `webhook_dead_letter` precisará ser criada.

## Decisão

Adotar 5 tentativas com backoff de 1m/5m/30m/2h/12h; após o teto, considerar falha permanente e mover o evento para a DLQ ([09:15] Diego; [09:17] Larissa). Cada HTTP call tem timeout de 10 segundos; timeout é tratado como falha e marca para retry ([09:42] Diego; [09:42] Sofia).

A DLQ fica em uma tabela separada `webhook_dead_letter`, com payload, motivo da falha e timestamp ([09:18] Diego). O replay manual é feito por `POST /admin/webhooks/dead-letter/:id/replay`, recolocando o evento na outbox como pendente ([09:18] Diego; [09:35] Diego). O endpoint exige role `ADMIN` e reutiliza o `requireRole` existente ([09:36] Larissa), e deve logar quem fez o replay para auditoria ([09:36] Sofia).

**Detalhes a fechar no FDD/RFC:** a transcrição diz “5 tentativas” e lista cinco intervalos de backoff, mas não explicita se o primeiro envio conta como uma das 5 tentativas ou se são 5 retries após o primeiro envio ([09:15] Diego; [09:17] Larissa). Também não há diferenciação entre respostas 4xx e 5xx; timeout e falha são tratados genericamente como falha para retry ([09:15] Diego; [09:42] Diego). As colunas exatas da DLQ, a semântica do `:id` no replay (id da DLQ ou id do evento/outbox) e se o replay preserva o mesmo `event_id`/payload não estão definidos ([09:18] Diego; [09:19] Larissa). O destino do log de auditoria (Pino ou tabela) também não foi especificado.

## Alternativas Consideradas

### 3 tentativas de retry

Bruno defendeu 3 tentativas, por ser mais agressivo ([09:16] Bruno); Diego descartou porque é pouco: se o cliente teve indisponibilidade de manhã, seriam três tentativas em 30 minutos e o evento morreria; já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). A alternativa foi descartada por não cobrir indisponibilidades reais.

### Retry indefinido com backoff

Diego mencionou que algumas pessoas defendem retry indefinido com backoff, mas descartou porque o evento ficaria pendurado para sempre se o cliente sumisse ([09:15] Diego). A alternativa foi descartada em favor de um teto finito de tentativas.

### Registrar falha permanente na própria outbox, em vez de tabela separada

Larissa perguntou se a DLQ deveria ser uma tabela separada ou apenas marcar como “failed” na própria outbox ([09:17] Larissa). Diego preferiu uma tabela `webhook_dead_letter` separada, com payload, motivo da falha e timestamp, para deixar a leitura da outbox principal mais limpa e servir como evidência para debug e reprocessamento ([09:18] Diego). A alternativa de marcar “failed” na outbox foi descartada por misturar o fluxo principal com o histórico de falhas permanentes.

## Consequências

### Positivas

- A política de retry cobre falhas temporárias e indisponibilidades de até 12 ou 24 horas ([09:15] Diego).
- O teto de tentativas evita eventos pendurados para sempre ([09:15] Diego).
- A DLQ separada mantém a outbox principal mais limpa e preserva evidências para debug e reprocessamento ([09:18] Diego).
- O replay administrativo permite recuperar eventos após falha permanente ([09:18] Diego; [09:19] Larissa).
- O timeout de 10 segundos impede que um cliente lento prenda o worker indefinidamente ([09:42] Diego).

### Negativas

- Um evento pode levar quase 15 horas entre a primeira falha e a última tentativa antes de ir para a DLQ ([09:17] Diego).
- A DLQ exige intervenção manual para reprocessamento; não há replay automático ([09:18] Diego).
- A tabela de DLQ adiciona mais uma estrutura de persistência e operação.
- A auditoria do replay depende de log; o destino desse log não foi definido ([09:36] Sofia).

### Trade-offs

- **5 tentativas versus 3:** 5 cobrem uma janela maior de indisponibilidade, ao custo de manter o evento pendente por mais tempo ([09:16] Diego; [09:17] Larissa).
- **Teto finito versus retry indefinido:** o teto evita eventos eternos, mas exige DLQ e intervenção manual ([09:15] Diego).
- **DLQ separada versus “failed” na outbox:** a separação melhora leitura e evidência, mas adiciona uma tabela e um fluxo de replay ([09:18] Diego).
- **Timeout de 10 segundos versus clientes lentos:** o timeout protege o worker, mas classifica clientes lentos como falha e os coloca no ciclo de retry ([09:42] Diego).
