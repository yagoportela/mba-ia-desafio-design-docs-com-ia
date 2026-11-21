# ADR-004 — Autenticação HMAC-SHA256 e rotação de secret

## Status

Aceito.

## Contexto

A plataforma vai expor eventos com dados de pedidos para endpoints fora da infraestrutura. O cliente precisa conseguir validar que a requisição veio realmente da plataforma e que ninguém adulterou o payload no meio ([09:19] Sofia). Sofia propôs o padrão HMAC: assinar o payload com uma secret compartilhada entre a plataforma e o cliente e enviar a assinatura em um header do tipo `X-Signature` ([09:20] Sofia). Bruno perguntou qual algoritmo; Sofia definiu HMAC-SHA256 como padrão de mercado ([09:20] Sofia).

Cada endpoint de webhook do cliente deve ter uma secret única, não uma secret global da plataforma; se uma vazar, não vaza todas ([09:21] Sofia). A secret também precisa ser rotacionável: endpoint para o cliente pedir uma nova secret; quando ele rotaciona, a antiga permanece válida por 24 horas em paralelo, para dar tempo de migrar os sistemas dele, e depois morre ([09:21] Sofia). Sofia formalizou: “HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h” ([09:22] Sofia).

Além disso, TLS é obrigatório: a URL do webhook tem que ser `https`; se o cliente cadastrar `http`, a plataforma recusa com erro de validação ([09:23] Sofia). Sofia classificou isso como validação no schema Zod, não como decisão arquitetural separada ([09:23] Sofia).

No código, o padrão de validação Zod está em `src/middlewares/validate.middleware.ts` (L11–37): o middleware converte `ZodError` em `ValidationError` (L25–32). A classe `ValidationError` está em `src/shared/errors/http-errors.ts` (L9–13). Um exemplo de schema Zod do domínio está em `src/modules/orders/order.schemas.ts` (L4–30). O `prisma/schema.prisma` ainda não tem modelo de configuração de webhook; a tabela precisará armazenar `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno; [09:21] Sofia).

## Decisão

Adotar HMAC-SHA256 sobre o corpo do request, com assinatura enviada no header `X-Signature` ([09:20] Sofia; [09:22] Sofia). Cada endpoint de webhook tem uma secret única, gerada pela plataforma e devolvida na criação ([09:21] Sofia; [09:31] Marcos). A secret é rotacionável por endpoint: o cliente pode pedir uma nova secret pela API; a antiga permanece válida por 24 horas em paralelo e depois morre ([09:21] Sofia). A tabela de configuração armazena `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno; [09:21] Sofia).

A URL do webhook deve ser `https`; URLs `http` são recusadas com erro de validação no schema Zod ([09:23] Sofia). A validação deve reutilizar o middleware `validate` existente e o padrão de schemas Zod do projeto (`src/middlewares/validate.middleware.ts`, L11–37; `src/modules/orders/order.schemas.ts`, L4–30).

**Detalhes a fechar no FDD/RFC:** a transcrição não define a codificação da assinatura (hex ou base64), nem se o `X-Timestamp` entra no conteúdo assinado; ela diz apenas “HMAC-SHA256 sobre o corpo do request” ([09:20]–[09:22] Sofia; [09:44] Diego). Também não estão definidos: o algoritmo de geração da secret, se a secret é armazenada em texto puro ou hash, o path e o método do endpoint de rotação, se a secret é retornada em `GET`/listagem, e o código de erro específico para URL inválida (apenas o exemplo `WEBHOOK_INVALID_URL` foi citado em [09:28] Bruno). Esses pontos precisam ser fechados no FDD/RFC.

## Alternativas Consideradas

### Secret global da plataforma

Sofia descartou usar uma secret global: se ela vazar, vaza tudo; cada endpoint de webhook do cliente precisa ter uma secret única ([09:21] Sofia). A alternativa foi rejeitada por ampliar o impacto de um vazamento.

### Aceitar URLs HTTP

Sofia definiu TLS obrigatório e recusa de URLs `http` com erro de validação ([09:23] Sofia). A alternativa de aceitar `http` foi descartada porque o payload trafega fora da infraestrutura da plataforma e precisa de proteção em trânsito.

### Manter a secret fixa, sem rotação

Sofia afirmou que a secret “tem que ser rotacionável” e definiu o grace period de 24 horas para a antiga ([09:21] Sofia; [09:22] Sofia). A alternativa de manter uma secret fixa foi descartada porque não permitiria ao cliente trocar uma credencial potencialmente vazada sem interrupção.

## Consequências

### Positivas

- O cliente consegue validar autenticidade e integridade do payload recebido ([09:19] Sofia; [09:20] Sofia).
- A secret por endpoint limita o impacto de um vazamento a um único cadastro ([09:21] Sofia).
- A rotação com grace period de 24 horas permite migrar sistemas sem interrupção imediata ([09:21] Sofia).
- HTTPS obrigatório protege o transporte dos dados de pedidos ([09:23] Sofia).
- A validação de URL reutiliza o padrão Zod e o middleware de validação já existentes ([09:23] Sofia; `src/middlewares/validate.middleware.ts`).

### Negativas

- O cliente precisa implementar HMAC-SHA256, validar a assinatura e gerenciar a rotação da secret ([09:20] Sofia; [09:21] Sofia).
- Durante 24 horas após a rotação, a secret antiga continua válida, ampliando a janela de exposição ([09:21] Sofia).
- O gerenciamento de secrets por endpoint adiciona complexidade operacional e de armazenamento.
- A plataforma precisa garantir a geração e a devolução da secret sem expor credenciais indevidamente; os detalhes de armazenamento e retorno não foram definidos ([09:31] Marcos).

### Trade-offs

- **Segurança versus complexidade de integração:** HMAC-SHA256 é padrão de mercado e verificável pelo cliente, mas exige implementação do lado dele ([09:20] Sofia).
- **Secret por endpoint versus secret global:** a secret por endpoint isola vazamentos, ao custo de mais credenciais para gerenciar ([09:21] Sofia).
- **Grace period de 24 horas versus revogação imediata:** o grace period facilita a migração do cliente, mas mantém a secret antiga válida por mais tempo ([09:21] Sofia).
- **HTTPS obrigatório versus flexibilidade de cadastro:** exigir `https` reduz riscos de transporte, mas recusa integrações que ainda usem `http` ([09:23] Sofia).
