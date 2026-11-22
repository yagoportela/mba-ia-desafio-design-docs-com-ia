# Da Transcrição ao Pacote de Design Docs — Processo de Produção

## 1. Sobre o desafio

Este repositório é a minha entrega do desafio de Design Docs para o **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System (OMS) existente. Parti da transcrição de uma reunião técnica (`TRANSCRICAO.md`) e do código real da aplicação em `src/`, `prisma/` e `tests/`. A ideia era transformar uma conversa de aproximadamente 55 minutos em um pacote acionável e rastreável de **PRD, RFC, FDD, 7 ADRs e TRACKER**, sem mexer no código da aplicação.

Usei a IA como parceira de produção o tempo todo. Ela leu o repositório, extraiu fatos, propôs estruturas e escreveu os documentos; eu fiquei com a parte de maestro: separei decisões fechadas de lacunas, contive o escopo, revisei contradições e cobrei que cada afirmação tivesse origem na transcrição ou em um caminho real do código. No fim, ficou uma entrega puramente documental, com 210 itens auditáveis no `TRACKER`, 182 deles rastreados à transcrição e 28 a caminhos verificados do código.

## 2. Ferramentas de IA utilizadas

| Ferramenta / ambiente | Papel no processo |
|---|---|
| **VS Code** | Usei para escrever os prompts em Markdown, organizar os arquivos do pacote, revisar os documentos gerados e navegar pelo repositório. |
| **OpenCode (CLI/agente de código)** | Foi meu ambiente principal de execução. Usei para ler a transcrição inteira, explorar `src/`, `prisma/` e `tests/`, buscar símbolos e caminhos com `grep`/`glob`, gerar e editar os arquivos Markdown e rodar validações de consistência. |
| **DeepSeek V4.1 Flash 1017 (LLM)** | Foi o modelo por trás do agente. Me ajudou a estruturar o workflow, apoiar a engenharia, revisar os prompts, sintetizar a transcrição, redigir os documentos e revisar a arquitetura em ciclos sucessivos. |
| **Git e PowerShell** | Apoio mecânico, não IA. Usei para conferir contagens de linhas, proporções de fontes, existência de caminhos, formato dos timestamps, IDs duplicados e consistência das tabelas Markdown. |

Escrevi os prompts em Markdown no VS Code, organizei tudo por etapa e rodei na sessão do OpenCode. A revisão crítica aconteceu na própria sessão do agente, sem trocar para uma segunda ferramenta de IA. O código da aplicação foi só contexto, nunca alterei.

## 3. Workflow adotado

Organizei o trabalho em seis etapas. A ordem foi de propósito: primeiro os fatos, depois as decisões, a arquitetura, a implementação, o produto e, no fim, a auditoria cruzada.

### Etapa 1 — Extração factual e mapeamento de linhas reais do código

Comecei lendo a transcrição inteira e explorando `src/` e `prisma/`. Antes de escrever qualquer design doc, fiz um levantamento factual do que estava fechado, do que foi descartado ou adiado e dos caminhos reais que serviriam de gancho para o módulo de webhooks.

A regra dessa etapa era simples: **não deduzir, não inventar e não preencher lacuna sem fonte identificável**. O resultado foi um levantamento com timestamps `[hh:mm] Nome` e uma lista separada de lacunas, que usei como fonte de verdade nas etapas seguintes.

### Etapa 2 — Formalização das decisões em 7 ADRs

Transformei as decisões fechadas em sete ADRs no formato MADR, em `docs/adrs/ADR-001` a `ADR-007`:

- `ADR-001` — outbox transacional no MySQL.
- `ADR-002` — worker separado em polling de 2 segundos.
- `ADR-003` — política de retry, backoff e DLQ.
- `ADR-004` — HMAC-SHA256, secret por endpoint e rotação com grace period.
- `ADR-005` — entrega at-least-once e idempotência via `X-Event-Id`.
- `ADR-006` — reuso dos padrões arquiteturais existentes.
- `ADR-007` — snapshot de payload na inserção da outbox.

Cada ADR tem contexto, decisão, alternativas consideradas e consequências, sempre citando a fala ou o caminho de código que sustenta a decisão.

### Etapa 3 — Consolidação da visão arquitetural no RFC

Com os ADRs prontos, escrevi o `docs/RFC.md` para propor a solução em nível de arquitetura: visão geral, fluxo de uma mudança de status, confiabilidade, segurança, alternativas descartadas e questões em aberto. Mantive o RFC curto, sem descer ao detalhe de DDL, schemas Prisma ou contratos completos do FDD.

Nessa etapa também deixei claro o que **não** estava decidido na call — rate limiting, endurecimento de autorização, contagem exata de tentativas, codificação do HMAC, tratamento de `4xx` versus `5xx` e nome do arquivo do worker — para não tratar exemplo ou hipótese como requisito fechado.

### Etapa 4 — Especificação detalhada no FDD

O `docs/FDD.md` ficou como o documento de implementação: fluxos de criação do evento na outbox, processamento pelo worker, retry/backoff, DLQ e replay; contratos dos endpoints; contrato do request outbound; matriz de erros `WEBHOOK_*`; modelagem de dados; observabilidade; dependências; critérios de aceite técnicos; e a seção obrigatória de integração com o sistema existente, com caminhos reais como `src/modules/orders/order.service.ts`, `src/middlewares/auth.middleware.ts`, `src/app.ts`, `src/routes/index.ts`, `src/config/database.ts`, `src/server.ts` e `prisma/schema.prisma`.

Quando a call não fechava um detalhe, o FDD marcava a premissa e a lacuna, em vez de apresentar aquilo como decisão fechada.

### Etapa 5 — Consolidação de negócio no PRD

Escrevi o `docs/PRD.md` na voz do Product Manager (Marcos), depois que as decisões e a implementação já estavam claras. Ele consolida problema, público-alvo, cenários, objetivos e métricas quantitativas, escopo incluído e excluído, requisitos funcionais e não funcionais, decisões e trade-offs, dependências, riscos, critérios de aceitação e estratégia de testes e validação.

Mantive o foco em produto e negócio: sem DDL, sem schema Prisma e sem detalhe interno do FDD. As metas vieram da call — latência abaixo de 10 segundos, polling de 2 segundos, janela de recuperação de ~15 horas com 5 tentativas e backoff `1m/5m/30m/2h/12h`, histórico dos últimos 100 envios e eliminação da dependência de polling.

### Etapa 6 — Auditoria cruzada e geração do TRACKER

Deixei o `docs/TRACKER.md` para o fim e varri PRD, RFC, FDD e os sete ADRs. A tabela liga cada item à origem na transcrição ou no código, com exatamente seis colunas: `ID`, `Documento`, `Tipo`, `Conteúdo (resumo)`, `Fonte` e `Localização`.

No fim, ficaram **210 itens**: **182 com `Fonte = TRANSCRICAO` (86,7%)** e **28 com `Fonte = CODIGO` (13,3%)**. A cobertura inclui 19/19 requisitos funcionais do PRD, 12/12 requisitos não funcionais, 7/7 itens fora de escopo, 8/8 decisões e trade-offs do PRD, 9/9 dependências do PRD, 7/7 riscos do PRD, 17/17 critérios de aceitação do PRD, 7/7 ADRs, 6/6 alternativas do RFC, 6/6 questões em aberto do RFC, 7/7 riscos do RFC, 7/7 contratos de endpoint, 7/7 erros `WEBHOOK_*`, 7/7 objetivos técnicos do FDD, 8 estratégias de resiliência mapeadas, 3 itens de observabilidade mapeados, 17/17 critérios de aceite técnicos, 9/9 riscos do FDD, dependências do FDD e 10/10 áreas de integração do FDD em 14 caminhos de código. Conferi mecanicamente que todos os timestamps e nomes existem na transcrição, que todos os caminhos `CODIGO` existem no repositório e que não há IDs duplicados.

## 4. Prompts customizados

Os prompts abaixo são versões reais do que usei em cada etapa. Escrevi no VS Code, ajustei ao longo dos ciclos para reforçar rastreabilidade, separar fato de hipótese e conter o escopo, e rodei no OpenCode para gerar os documentos.

### Prompt 1 — Extração factual sem inferência

```markdown
Atue como Arquiteto de Software Sênior. Leia integralmente @TRANSCRICAO.md e explore
a estrutura de @src e @prisma.

NÃO escreva nenhum design doc ainda. Extraia um levantamento factual em 4 blocos:
1. Decisões técnicas FECHADAS, com o timestamp exato [hh:mm] Nome.
2. Itens explicitamente DESCARTADOS ou ADIADOS, com timestamp e motivo.
3. Requisitos funcionais, não funcionais e restrições citados na call, com timestamp.
4. Mapeamento do código existente: caminhos reais dos arquivos de pedidos, mudança de
   status, erros, autenticação e schema Prisma que servirão de gancho.

Regra absoluta: não deduza, não invente e não preencha lacunas sem fonte identificável.
Liste separadamente todas as lacunas e ambiguidades que a call não resolveu.
```

### Prompt 2 — ADRs no formato MADR

```markdown
Atue como o arquiteto responsável pelos ADRs da feature de webhooks.

Gere 7 ADRs em docs/adrs/ADR-NNN-titulo-em-kebab-case.md, no formato MADR:
Status, Contexto, Decisão, Alternativas Consideradas e Consequências (positivas,
negativas e trade-offs explícitos).

Cada afirmação deve citar a transcrição no formato [hh:mm] Nome ou um caminho real
do código. Separe decisões fechadas de detalhes ainda em aberto. Não transforme
exemplos citados na call em regras. Cubra: outbox MySQL, worker em polling, retry/
backoff/DLQ, HMAC e rotação de secret, at-least-once com X-Event-Id, reuso dos
padrões existentes e snapshot de payload.
```

### Prompt 3 — PRD de produto com rastreabilidade

```markdown
Atue como Marcos, Product Manager da reunião. Com base no levantamento factual, na
transcrição, no RFC, no FDD e nos ADRs, escreva docs/PRD.md para o Sistema de
Webhooks de Notificação de Pedidos.

Mantenha foco em produto, negócio, escopo e requisitos. Não duplique DDL SQL,
schemas Prisma ou detalhes internos do FDD.

Inclua: resumo e contexto; problema e motivação; público-alvo e cenários; objetivos
e métricas quantitativas; escopo incluído e fora de escopo com justificativa e
timestamp; pelo menos 12 requisitos funcionais numerados; requisitos não funcionais;
decisões e trade-offs com links para os ADRs; dependências; riscos com probabilidade,
impacto e mitigação; critérios de aceitação; e estratégia de testes.

Toda informação deve ser rastreável à transcrição ou ao código. Não invente métricas
nem requisitos sem origem na call.
```

### Prompt 4 — Varredura do TRACKER e detecção de alucinações

```markdown
Varra integralmente docs/PRD.md, docs/RFC.md, docs/FDD.md e todos os arquivos em
docs/adrs/*.md.

Gere docs/TRACKER.md com exatamente estas 6 colunas:
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |

Mapeie todos os PRD-FR, PRD-NFR, itens fora de escopo, ADR-001 a ADR-007,
alternativas descartadas e questões em aberto do RFC, contratos de endpoint,
erros WEBHOOK_* e ganchos de integração do FDD.

Regras: cobertura >= 80%; pelo menos 70% das linhas com Fonte = TRANSCRICAO e
Localização no formato [hh:mm] Nome; pelo menos 8 a 12 linhas com Fonte = CODIGO
apontando para caminhos reais. Valide contagens, IDs duplicados e existência dos
caminhos.

Ao final, liste no relatório de auditoria, fora do TRACKER, todo item dos
documentos que não tenha origem identificável na transcrição nem no código,
indicando arquivo e seção.
```

## 5. Iterações e ajustes

Foram **4 ciclos principais** até chegar ao resultado final. Cada um corrigiu um tipo de problema: primeiro excesso de inferência, depois excesso de escopo, depois excesso de detalhe e, no fim, excesso de confiança na rastreabilidade.

### Ajuste 1 — Separar decisões fechadas de lacunas

O primeiro problema era a IA tratar exemplo ou hipótese como requisito. A correção foi separar explicitamente as **decisões fechadas** das **17 lacunas não decididas** na call. Isso evitou que o FDD ou o PRD fixassem regra inventada para retry de `4xx` versus `5xx`, codificação hexadecimal ou base64 do HMAC, inclusão do `X-Timestamp` no conteúdo assinado, semântica do `:id` no replay e contagem exata entre 5 tentativas totais ou 1 envio + 5 retries. Esses pontos ficaram como questões em aberto no RFC e como premissas declaradas no FDD.

### Ajuste 2 — Conter o RFC em 2–4 páginas

Na primeira versão, o RFC começou a virar um FDD: entrou DDL, schema, payload completo e matriz de erro. Movi esse material para o FDD e reescrevi o RFC em nível de arquitetura: visão geral, alternativas, questões em aberto e impactos. O RFC ficou curto e sem duplicar o detalhamento de implementação, preservando a fronteira entre “o que propomos e por quê” (RFC) e “como construir em detalhe” (FDD).

### Ajuste 3 — Auditoria cruzada e detecção de alucinações

Na última revisão, montei o `TRACKER` e usei ele para uma auditoria de alucinações. A auditoria separou três coisas que estavam misturadas: o que veio literalmente da transcrição, o que veio de caminhos reais do código e o que era **premissa técnica declarada** do FDD — por exemplo, nomes de modelos e colunas, paths do CRUD, status codes, formato da assinatura, métricas de observabilidade e granularidade do histórico por tentativa. O que não tinha origem identificável ficou fora do `TRACKER` para não passar como decisão da call; as premissas técnicas ficaram no FDD e as questões não decididas no RFC. Também corrigi uma atribuição errada no RFC, que colocava Diego como Tech Lead quando a transcrição define Larissa nesse papel.

### Ciclos e refinamentos

- **Ciclo 1:** extração factual, mapa do código e identificação das lacunas.
- **Ciclo 2:** sete ADRs e primeira versão do RFC; corte de detalhes de implementação.
- **Ciclo 3:** FDD e PRD; separação entre especificação técnica e produto/negócio.
- **Ciclo 4:** auditoria cruzada, `TRACKER` com 210 itens e auditoria de alucinações.

No fim, a documentação não só descreve a feature: ela mostra de onde cada requisito, decisão e restrição veio — e onde a call foi omissa.

## 6. Como navegar a entrega

### Árvore de arquivos

```text
.
├── README.md
├── TRANSCRICAO.md
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── README.md
│       ├── ADR-001-outbox-transacional-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-politica-de-retry-backoff-e-dlq.md
│       ├── ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md
│       ├── ADR-005-entrega-at-least-once-e-idempotencia-via-event-id.md
│       ├── ADR-006-reuso-dos-padroes-arquiteturais-existentes.md
│       └── ADR-007-snapshot-de-payload-na-insercao-da-outbox.md
├── src/          # contexto do OMS, não alterado
├── prisma/       # contexto do OMS, não alterado
└── tests/        # contexto do OMS, não alterado
```

### Ordem sugerida de leitura

1. **`README.md`** — este documento, com o processo, as ferramentas e os prompts.
2. **`docs/PRD.md`** — por que e o quê: problema, público, escopo, métricas e critérios de aceite.
3. **`docs/RFC.md`** — como pretendemos resolver, alternativas descartadas e questões em aberto.
4. **`docs/adrs/`** — por que cada decisão arquitetural foi tomada, com contexto e consequências.
5. **`docs/FDD.md`** — como construir: fluxos, contratos, erros, modelagem e integração com o código.
6. **`docs/TRACKER.md`** — de onde veio cada item: transcrição, código, cobertura e lacunas.

Para uma leitura orientada a negócio, comece pelo PRD e pelo RFC. Para uma leitura orientada a implementação, siga dos ADRs para o FDD. Para auditar a consistência da entrega, use o `TRACKER` como referência cruzada e confira as linhas de `Fonte = CODIGO` nos caminhos reais do repositório.
