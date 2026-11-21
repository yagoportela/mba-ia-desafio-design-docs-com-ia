# ADR-007 — Snapshot de payload na inserção da outbox

## Status

Aceito.

## Contexto

Ao modelar a outbox, Bruno perguntou se o evento guardaria o payload renderizado ou apenas o `order_id`, renderizando no momento do envio ([09:51] Bruno). Larissa preferiu o payload renderizado já na inserção: se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou; caso contrário, haveria “caso esquisito” ([09:52] Larissa). Diego concordou com o snapshot na inserção ([09:52] Diego), e Bruno fechou: “Beleza, snapshot. Decidido” ([09:52] Bruno).

O payload definido para o evento é JSON com `event_id`, `event_type` igual a `order.status_changed`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`; `items` não é enviado, para não inflar o payload ([09:43] Diego). Se o cliente quiser detalhes, consulta `GET /orders/:id` depois ([09:43] Diego).

Há também um limite de tamanho: Sofia defendeu que, se o evento tiver 500KB, a plataforma não deve enviar; ela prefere errar a truncar, porque “se chegou nesse tamanho, tem algo errado” ([09:23] Sofia). Diego sugeriu 64KB como teto generoso, e Larissa registrou 64KB de limite com erro caso ultrapasse, classificando como requisito não funcional ([09:24] Diego; [09:24] Larissa).

No código, a inserção ocorrerá dentro da transação de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, L126–179). O `prisma/schema.prisma` atual não possui modelo de outbox; a coluna que armazenará o payload ainda precisa ser definida.

## Decisão

O evento da outbox guarda o payload já renderizado no momento da inserção, e não apenas o `order_id` ([09:52] Larissa). O snapshot deve refletir o estado do pedido no momento da mudança de status; se o pedido mudar depois, o evento permanece com o estado original ([09:52] Larissa; [09:52] Diego).

O payload segue o contrato definido em [09:43] Diego: `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`; `items` não é incluído ([09:43] Diego). O limite de payload é de 64KB; se ultrapassar, a operação deve errar, sem truncar ([09:23] Sofia; [09:24] Diego; [09:24] Larissa).

**Detalhes a fechar no FDD/RFC:** a transcrição não enumera a lista completa de “campos básicos da order” além do exemplo `total_cents` ([09:43] Diego), nem define o tipo de coluna que armazenará o payload no MySQL. Também não está definido se a validação de 64KB ocorre na inserção ou no envio, qual código/status de erro é usado quando o limite é ultrapassado, nem se o limite considera apenas o corpo do payload ou também headers/assinatura ([09:23] Sofia; [09:24] Larissa). Esses pontos precisam ser fechados no FDD/RFC.

## Alternativas Consideradas

### Guardar apenas `order_id` e renderizar no momento do envio

Bruno levantou essa alternativa ([09:51] Bruno); Larissa descartou porque, se o pedido mudar depois, o evento não refletiria o estado de quando o status mudou; ela preferiu o snapshot já renderizado na inserção ([09:52] Larissa). A alternativa foi rejeitada por perder a fidelidade temporal do evento.

### Truncar o payload quando ultrapassar o limite

Sofia considerou a possibilidade de truncar, mas defendeu errar: se o evento chegou a 500KB, há algo errado ([09:23] Sofia). Larissa registrou 64KB de limite com erro caso ultrapasse ([09:24] Larissa). A alternativa de truncamento foi descartada porque produziria um payload incompleto e potencialmente inválido para o cliente.

## Consequências

### Positivas

- O evento preserva o estado do pedido no momento da mudança de status, mesmo que o pedido seja alterado depois ([09:52] Larissa).
- O cliente recebe um payload autocontido, sem precisar consultar o estado atual do pedido para entender a transição ([09:43] Diego).
- O snapshot serve como evidência do que foi notificado em cada evento.
- O limite de 64KB evita payloads excessivamente grandes e privilegia a integridade do evento ([09:23] Sofia; [09:24] Diego).

### Negativas

- O armazenamento do payload renderizado duplica dados que já existem no pedido.
- O payload pode ficar desatualizado em relação ao pedido atual, por design; o cliente precisa consultar `GET /orders/:id` se quiser detalhes atuais ([09:43] Diego).
- Eventos que ultrapassem 64KB falham em vez de serem truncados; o tratamento operacional posterior não foi definido ([09:23] Sofia; [09:24] Larissa).
- A lista completa de campos do snapshot ainda não está fechada ([09:43] Diego).

### Trade-offs

- **Fidelidade temporal versus armazenamento:** o snapshot preserva o estado no momento da mudança, mas duplica dados ([09:52] Larissa).
- **Erro versus truncamento:** errar preserva a integridade do payload, mas pode impedir a entrega de um evento grande; truncar permitiria enviar algo, mas com dados incompletos ([09:23] Sofia; [09:24] Larissa).
- **Payload autocontido versus payload enxuto:** incluir campos básicos evita consultas adicionais, mas aumenta o tamanho; `items` foi excluído para não inflar ([09:43] Diego).
- **Snapshot versus renderização tardia:** o snapshot é mais fiel ao momento da transição, mas não reflete mudanças posteriores do pedido ([09:52] Larissa).
