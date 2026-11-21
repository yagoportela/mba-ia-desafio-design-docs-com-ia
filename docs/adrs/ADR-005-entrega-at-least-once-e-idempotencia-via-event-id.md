# ADR-005 — Entrega at-least-once e idempotência via event-id

## Status

Aceito.

## Contexto

O worker pode reenviar eventos em caso de falha, timeout ou retry. Diego definiu que a entrega será at-least-once: pode acontecer de o cliente receber o mesmo evento duas vezes, e ele precisa estar preparado ([09:24] Diego). Para permitir a deduplicação do lado do cliente, a plataforma envia um `event_id` no header `X-Event-Id`, com um UUID gerado quando o evento entra na outbox; o id é único por evento ([09:25] Diego). Larissa formalizou: “At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão” ([09:26] Larissa).

Diego comparou com o padrão de mercado: Stripe e GitHub fazem assim; garantir exactly-once exigiria coordenação dos dois lados e seria muito mais complexo; at-least-once com `event_id` resolve 99% dos casos ([09:25] Diego). O payload também inclui `event_id` ([09:43] Diego), além do header `X-Event-Id` ([09:25] Diego).

No código, o projeto já usa UUID como padrão de identificação no `prisma/schema.prisma` (IDs `@default(uuid()) @db.Char(36)`, por exemplo L26, L41, L75 e L117) e tem a dependência `uuid` 11.0.3 no `package.json` (L25–34). A outbox ainda não existe no schema; o evento será inserido no fluxo de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, L126–179).

## Decisão

Adotar entrega at-least-once, assumindo que o cliente pode receber o mesmo evento mais de uma vez ([09:24] Diego). O cliente deduplica pelo `X-Event-Id` ([09:25] Diego; [09:26] Larissa). O `event_id` é um UUID gerado quando o evento entra na outbox e é único por evento ([09:25] Diego). O payload também contém `event_id` ([09:43] Diego), e o header `X-Event-Id` é enviado em cada request ([09:25] Diego).

**Detalhes a fechar no FDD/RFC:** a transcrição não afirma explicitamente que o `event_id` do payload e o `X-Event-Id` do header são o mesmo valor ([09:25] Diego; [09:43] Diego). Também não define se retries e replay preservam o mesmo `event_id` ou geram um novo evento, nem a versão do UUID a ser usada. O tempo de retenção dos ids de deduplicação do lado do cliente não foi especificado. Esses pontos precisam ser fechados no FDD/RFC.

## Alternativas Consideradas

### Garantia exactly-once

Diego descartou exactly-once porque exigiria coordenação dos dois lados e seria muito mais complexo; at-least-once com `event_id` resolveria 99% dos casos ([09:25] Diego). A alternativa foi rejeitada por custo de coordenação e complexidade.

## Consequências

### Positivas

- O modelo é mais simples que exactly-once e não exige coordenação entre plataforma e cliente ([09:25] Diego).
- O cliente tem um identificador estável para deduplicar eventos repetidos ([09:25] Diego; [09:26] Larissa).
- O `event_id` é gerado na entrada da outbox ([09:25] Diego).
- O padrão é compatível com o fluxo de retry e DLQ (ADR-003).

### Negativas

- A responsabilidade pela deduplicação é transferida ao cliente ([09:25] Sofia; [09:25] Diego reconhece que “joga responsabilidade pro cliente”).
- O cliente pode receber o mesmo evento mais de uma vez e precisa lidar com efeitos colaterais duplicados ([09:24] Diego).
- O cliente precisa persistir os `event_id` recebidos pelo tempo necessário para deduplicar; esse tempo não foi definido ([09:25] Diego).
- Se o cliente não deduplicar, a garantia at-least-once pode gerar processamento duplicado.

### Trade-offs

- **Simplicidade versus garantia exactly-once:** at-least-once evita coordenação dos dois lados, mas não elimina duplicatas ([09:25] Diego).
- **Responsabilidade no cliente versus complexidade na plataforma:** deduplicar no cliente é mais simples para a plataforma, mas exige implementação e persistência do lado do cliente ([09:25] Diego).
- **Cobertura de 99% versus casos extremos:** a solução cobre a maioria dos casos, mas não oferece exactly-once ([09:25] Diego).
