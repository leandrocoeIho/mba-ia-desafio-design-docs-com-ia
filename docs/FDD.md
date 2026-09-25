# FDD: Sistema de Webhooks de Notificação de Pedidos

> Especifica o "como construir" desta feature, em nível acionável para implementação. Para o "o quê e por quê", ver o [PRD](PRD.md); para a proposta de arquitetura e alternativas, ver o [RFC](RFC.md); para o racional de cada decisão isolada, ver os [ADRs](adrs/).

## 1. Contexto e Motivação Técnica

O OMS precisa notificar clientes B2B sobre mudanças de status de pedido abaixo de 10 segundos ([09:02] Marcos), sem comprometer a performance nem a atomicidade da transação de `OrderService.changeStatus()` ([src/modules/orders/order.service.ts:126](../src/modules/orders/order.service.ts)). A solução adotada — Outbox no MySQL + worker em polling + retry/DLQ + HMAC + at-least-once — está descrita em nível de arquitetura no [RFC](RFC.md) e decidida em detalhe nos [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md) a [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao-do-evento.md). Este documento traduz essas decisões em fluxos, contratos e critérios técnicos concretos.

## 2. Objetivos Técnicos

- Garantir que todo evento de mudança de status relevante seja registrado de forma atômica com a própria mudança (nunca status mudado sem evento, nem evento sem status mudado) — [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md).
- Entregar eventos ao cliente abaixo de 10 segundos no caminho feliz (SLA acordado, [09:02] Marcos), com o worker operando em ciclo de polling de 2 segundos ([ADR-002](adrs/ADR-002-worker-processo-separado-em-polling.md)).
- Garantir semântica at-least-once com deduplicação possível do lado do cliente via `X-Event-Id` ([ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).
- Autenticar cada evento entregue via HMAC-SHA256 com secret por endpoint ([ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md)).
- Reutilizar integralmente os padrões de módulo, erro, log e autenticação já existentes no projeto, sem introduzir dependências novas ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).

## 3. Escopo e Exclusões

**Incluído:**
- CRUD de configuração de webhook por cliente (`url`, secret gerada pelo OMS, lista de status assinados, estado ativo/inativo) ([09:31]-[09:33] Marcos/Bruno).
- Rotação de secret com grace period de 24h ([09:21] Sofia).
- Histórico de entregas por webhook (por padrão, as últimas 100 — ver seção 5.6) ([09:34] Marcos).
- Endpoint administrativo de replay manual de eventos em DLQ, restrito a role `ADMIN` ([09:18]-[09:19], [09:35]-[09:36]).
- Worker separado, outbox, retry com backoff, DLQ.

**Explicitamente fora de escopo desta feature** (ver PRD para o detalhamento completo):
- Notificação por e-mail em caso de falhas recorrentes — adiado para fase futura ([09:37] Larissa).
- Rate limiting de envio ao cliente — registrado como ponto em aberto, não implementado agora ([09:38]-[09:39] Diego).
- Dashboard visual para o cliente acompanhar webhooks — fora de escopo, projeto de frontend separado ([09:39]-[09:40] Larissa).
- Suporte a múltiplos workers em paralelo — adiado explicitamente: "problema do futuro, não agora" ([09:13] Diego). *(Não confundir com a ausência de ordering global entre pedidos diferentes, que não é um item adiado, mas uma limitação aceita deliberadamente — "Documentamos como limitação conhecida" [09:13] Larissa; ver seção 4.2, item 6, e seção 12.)*
- Arquivamento de eventos entregues na outbox — fora do escopo desta feature ([09:08] Diego).

## 4. Fluxos Detalhados

### 4.1 Criação do evento na outbox

1. Um caller autenticado chama `PATCH /api/v1/orders/:id/status`, tratado por `OrderController.changeStatus` → `OrderService.changeStatus()` ([src/modules/orders/order.service.ts:126-179](../src/modules/orders/order.service.ts)).
2. Dentro da mesma transação Prisma (`this.prisma.$transaction`) que já atualiza `orders`, ajusta estoque e insere em `order_status_history`, o serviço passa a chamar uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, recebendo o `tx` da transação corrente ([09:41] Bruno/Diego).
3. `publishWebhookEvent` busca os `WebhookEndpoint` ativos do `customerId` do pedido cuja lista de status assinados contenha o `toStatus` de destino. Se nenhum endpoint estiver inscrito naquele status, **nenhuma linha é inserida** — a filtragem acontece na inserção, não no envio ([09:33]-[09:34] Marcos/Bruno/Diego). **Decisão de implementação não discutida na reunião:** a estrutura de armazenamento dessa lista de status por endpoint — a reunião só definiu "uma lista dos status que ele quer receber" ([09:33] Marcos), sem especificar o formato. Como o MySQL/Prisma não suporta nativamente uma coluna de lista de enums, a recomendação é um campo `Json` (há precedente no projeto em `Customer.address`, [prisma/schema.prisma](../prisma/schema.prisma)) ou uma tabela de relação `webhook_endpoint_events`.
4. **Decisão de implementação não discutida na reunião:** se um pedido tem mais de um endpoint inscrito no mesmo `toStatus` (fan-out), a reunião não discutiu esse cenário — esta especificação assume que se insere uma linha por endpoint elegível, cada uma com seu próprio `event_id`. Para cada linha, monta-se o payload completo do evento (ver seção 5, "Contrato do webhook enviado ao cliente") como **snapshot no momento da inserção** — não recalculado depois ([ADR-007](adrs/ADR-007-payload-snapshot-na-insercao-do-evento.md)) — com status `PENDING`, um `event_id` (UUID) e o `webhook_endpoint_id` de destino.
5. Se a inserção falhar, toda a transação (incluindo a mudança de status já escrita no banco, mas ainda não commitada) sofre rollback — não pode existir mudança de status sem o evento correspondente registrado ([09:40]-[09:41] Bruno/Diego).
6. **Decisão de implementação em aberto, não resolvida na reunião:** onde exatamente o limite de 64KB é verificado. A reunião só definiu o limite e que deve haver erro se ultrapassado ([09:23]-[09:24] Sofia/Diego/Larissa), sem dizer se a checagem ocorre na inserção (bloqueando a transação de `changeStatus`) ou no envio pelo worker (seção 5.8 assume esta segunda leitura: "requisições que ultrapassem esse limite não são enviadas"). Esta especificação recomenda validar no **envio pelo worker**, não na inserção — para não repetir, com o próprio payload do OMS, o mesmo problema que motivou descartar o disparo síncrono (acoplar a transação de negócio a uma falha externa a ela, [09:04] Bruno). Um evento que exceda 64KB no envio é registrado como falha permanente e vai direto para `webhook_dead_letter` (código `WEBHOOK_PAYLOAD_TOO_LARGE` no motivo da falha), sem consumir tentativas de retry — reenviar não mudaria o tamanho do payload. **Este ponto deve ser confirmado com o time antes da implementação.**

### 4.2 Processamento pelo worker

1. `src/worker.ts` (novo arquivo, a ser criado) é um entry-point separado (`npm run worker`), com seu próprio `PrismaClient`, conectado à mesma `DATABASE_URL` da API ([09:29]-[09:30] Diego/Bruno; [ADR-002](adrs/ADR-002-worker-processo-separado-em-polling.md)).
2. `webhook_outbox` usa os quatro estados citados na reunião — pendente, processando, falhou, entregue ([09:08] Diego) — mapeados aqui como `PENDING`, `PROCESSING`, `FAILED` e `DELIVERED` (nomenclatura de implementação, não literal da transcrição). `PENDING` cobre tanto um evento nunca tentado quanto um evento aguardando retry; `FAILED` é reservado para o instante transitório em que as tentativas se esgotam, imediatamente antes da linha ser movida para `webhook_dead_letter` (seção 4.4) — não é um estado de espera. A cada 2 segundos, o worker consulta eventos com `status = PENDING` **e** (`next_attempt_at IS NULL OR next_attempt_at <= now()`), ordenados por `created_at` ascendente, em lote pequeno (ex.: 50 eventos por ciclo — tamanho de lote não definido na reunião, valor de referência a validar em implementação).
3. Para cada evento do lote, o worker marca-o como `PROCESSING`, monta a requisição HTTP (ver seção 5) e a envia ao `url` do `WebhookEndpoint` associado, com timeout de 10 segundos ([09:42] Sofia/Diego).
4. **Sucesso** (resposta 2xx dentro do timeout): o worker marca o evento como `DELIVERED`. **Decisão de implementação não discutida na reunião:** a existência de uma tabela `webhook_deliveries` para o histórico de entregas e o critério de "sucesso = resposta 2xx" — a reunião só pediu o histórico ("os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta", [09:34] Marcos), sem especificar a modelagem. Esta especificação assume uma tabela dedicada, com uma linha por tentativa (para alimentar o histórico de entregas, seção 5).
5. **Falha** (timeout, erro de conexão, ou resposta não-2xx): se ainda houver tentativas de retry disponíveis, o evento volta a `PENDING` com `next_attempt_at` recalculado (seção 4.3); se as tentativas se esgotaram, o evento passa brevemente por `FAILED` e é movido para a DLQ (seção 4.4). Em nenhum momento um evento aguardando retry fica com status diferente de `PENDING` — isso evita que ele "desapareça" da consulta do item 2.
6. Como o worker é único (single-worker) e processa a fila principal em ordem de `created_at`, a ordem de entrega por `order_id` é preservada **apenas enquanto nenhum evento anterior daquele pedido estiver aguardando retry** ([09:12]-[09:13] Diego/Larissa). **Limitação não coberta na reunião:** se o evento de uma transição (ex.: `PAID`) falhar e for agendado para retry, e o pedido avançar novamente antes desse retry (ex.: para `PROCESSING`), o worker pode entregar o evento mais recente antes do mais antigo, quebrando a ordem por `order_id` que a reunião presumiu ao decidir a topologia single-worker. Ver seção 12 (Riscos).

### 4.3 Retry com backoff exponencial

1. Em caso de falha, o evento volta a `PENDING` com um campo `next_attempt_at` calculado pela progressão de backoff: **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** entre tentativas sucessivas ([09:15]-[09:17] Diego/Larissa; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)), e um contador de tentativas é incrementado (nomes de campo são decisão de implementação, não literais da transcrição).
2. O worker só volta a considerar um evento em falha quando `next_attempt_at <= now()` (ver seção 4.2, item 2).
3. Cada tentativa (sucesso ou falha) é registrada em `webhook_deliveries`, permitindo reconstruir o histórico completo de tentativas de um evento (mesma ressalva de implementação da seção 4.2, item 4).
4. **Nota de ambiguidade herdada da fonte** (mesma nota do [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)): a transcrição não deixa claro se as "5 tentativas" contam a partir da primeira falha (1 envio inicial + 5 retries = 6 chamadas HTTP no total) ou se o envio inicial já conta como uma das 5 (5 chamadas no total). Esta especificação assume 6 chamadas no total, por ser a leitura mais consistente com os 5 intervalos de backoff enumerados por Diego ([09:17]). **Este ponto deve ser confirmado com Larissa/Diego antes da implementação** (ver [RFC](RFC.md), Questões em Aberto).

### 4.4 Dead Letter Queue e replay manual

1. Esgotadas as tentativas de retry, o evento é removido de `webhook_outbox` e inserido em `webhook_dead_letter`, contendo o payload completo, o motivo da última falha e o timestamp ([09:18] Diego).
2. Um administrador (role `ADMIN`) pode reprocessar manualmente um evento em DLQ via `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, que recoloca o evento em `webhook_outbox` com status `PENDING` ([09:18]-[09:19] Diego: "Manual via endpoint admin... Recoloca na outbox como pendente"). **Decisão de implementação não discutida na reunião:** se o contador de tentativas é zerado no replay, e se o `event_id` original é preservado (o que importa para a deduplicação do cliente via [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)) ou um novo é gerado. Esta especificação recomenda **zerar o contador** e **preservar o `event_id` original**, para que um cliente que já viu aquele evento consiga deduplicar corretamente mesmo após o replay.
3. Esse endpoint exige role `ADMIN` (reutilizando `requireRole('ADMIN')` de [src/middlewares/auth.middleware.ts:49](../src/middlewares/auth.middleware.ts)) e deve registrar em log estruturado (Pino) o `userId` do administrador que executou o replay, para fins de auditoria ([09:35]-[09:36] Sofia/Larissa).

## 5. Contratos Públicos

Todos os endpoints ficam sob `/api/v1`, seguindo o prefixo já usado pelos demais módulos ([src/routes/index.ts](../src/routes/index.ts)), autenticados via `authenticate` (JWT, header `Authorization: Bearer <token>`) salvo indicação contrária.

Todo erro segue o formato já produzido por [error.middleware.ts](../src/middlewares/error.middleware.ts), sem alteração — por exemplo, `WEBHOOK_NOT_FOUND`:
```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

### 5.1 `POST /api/v1/webhooks` — Cadastrar webhook

Cria um cadastro de webhook para um cliente. A secret é gerada pelo OMS e devolvida na criação ([09:31] Marcos: "secret é gerada pela gente e devolvida na criação"). **Decisão de implementação não discutida na reunião:** se a secret volta a aparecer em respostas de leitura posteriores (este FDD assume que não, ver 5.2) — a reunião não afirmou "apenas na criação" literalmente.

Request:
```json
{
  "customerId": "8f14e45f-ceea-4c8b-8cc7-1f1b1a1b1a1b",
  "url": "https://cliente.example.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

`customerId` no corpo é a leitura assumida por esta especificação; a reunião deixou explicitamente em aberto se viria "no body ou no path" ([09:32] Larissa).

Response `201 Created`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "customerId": "8f14e45f-ceea-4c8b-8cc7-1f1b1a1b1a1b",
  "url": "https://cliente.example.com/webhooks/oms",
  "secret": "whsec_9f8a7b6c5d4e3f2a1b0c",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-09-24T14:00:00.000Z"
}
```
*(o formato exato da secret, `whsec_...`, é ilustrativo — não foi definido na reunião)*

Erros possíveis: `VALIDATION_ERROR` (400, inclui URL não-HTTPS e status inválido em `events` — ver seção 6), `WEBHOOK_CUSTOMER_NOT_FOUND` (404).

### 5.2 `GET /api/v1/webhooks?customerId=...` — Listar webhooks de um cliente

Segue o padrão de paginação já usado por `GET /api/v1/orders` ([src/shared/http/response.ts](../src/shared/http/response.ts)). A `secret` **não é retornada** na listagem (decisão de implementação para reduzir exposição de segredo em respostas de leitura — não discutida explicitamente na reunião).

Request:
```
GET /api/v1/webhooks?customerId=8f14e45f-ceea-4c8b-8cc7-1f1b1a1b1a1b&page=1&pageSize=20
Authorization: Bearer <jwt>
```

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "customerId": "8f14e45f-ceea-4c8b-8cc7-1f1b1a1b1a1b",
      "url": "https://cliente.example.com/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-09-24T14:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

### 5.3 `PATCH /api/v1/webhooks/:id` — Editar webhook

Request (todos os campos opcionais):
```
PATCH /api/v1/webhooks/3fa85f64-5717-4562-b3fc-2c963f66afa6
Authorization: Bearer <jwt>
```
```json
{
  "url": "https://cliente.example.com/webhooks/oms-v2",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true
}
```

Response `200 OK`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "customerId": "8f14e45f-ceea-4c8b-8cc7-1f1b1a1b1a1b",
  "url": "https://cliente.example.com/webhooks/oms-v2",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "createdAt": "2026-09-24T14:00:00.000Z"
}
```
*(mesmo formato do item 5.2, sem `secret`)*

Erros possíveis: `WEBHOOK_NOT_FOUND` (404), `VALIDATION_ERROR` (400, inclui URL não-HTTPS e status inválido em `events`).

### 5.4 `DELETE /api/v1/webhooks/:id` — Remover webhook

Response `204 No Content`. Erros possíveis: `WEBHOOK_NOT_FOUND` (404).

### 5.5 `POST /api/v1/webhooks/:id/rotate-secret` — Rotacionar secret

Gera uma nova secret; a secret anterior permanece válida por 24h em paralelo ([09:21] Sofia: "a antiga fica válida por 24 horas em paralelo"). **Decisão de implementação não discutida na reunião:** o caminho exato deste endpoint e o nome do campo `previousSecretValidUntil` — a reunião definiu apenas que deve existir um "endpoint pro cliente conseguir pedir nova secret pela API" ([09:21] Sofia), sem especificar rota ou payload.

Request (sem corpo):
```
POST /api/v1/webhooks/3fa85f64-5717-4562-b3fc-2c963f66afa6/rotate-secret
Authorization: Bearer <jwt>
```

Response `200 OK`:
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "secret": "whsec_1a2b3c4d5e6f7a8b9c0d",
  "previousSecretValidUntil": "2026-09-25T14:00:00.000Z"
}
```

Erros possíveis: `WEBHOOK_NOT_FOUND` (404).

### 5.6 `GET /api/v1/webhooks/:id/deliveries` — Histórico de entregas

Retorna, por padrão, as últimas 100 entregas do webhook, com sucesso/falha, payload enviado, resposta recebida e tempo de resposta — atendendo ao pedido de Marcos: "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" ([09:34] Marcos; o número "100" vem desse exemplo dado por Marcos, não de um requisito numérico fechado formalmente).

Request:
```
GET /api/v1/webhooks/3fa85f64-5717-4562-b3fc-2c963f66afa6/deliveries?page=1&pageSize=100
Authorization: Bearer <jwt>
```

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "b6e1a6d0-1234-4a1b-9c8d-abcdef123456",
      "eventId": "c1d2e3f4-5678-4a1b-9c8d-abcdef654321",
      "status": "FAILED",
      "requestPayload": { "event_id": "c1d2e3f4-5678-4a1b-9c8d-abcdef654321", "event_type": "order.status_changed" },
      "httpStatusCode": 503,
      "responseBody": "Service Unavailable",
      "responseTimeMs": 420,
      "attemptNumber": 2,
      "attemptedAt": "2026-09-24T14:05:00.000Z"
    },
    {
      "id": "f1e2d3c4-5678-4a1b-9c8d-abcdef000111",
      "eventId": "c1d2e3f4-5678-4a1b-9c8d-abcdef654321",
      "status": "FAILED",
      "requestPayload": { "event_id": "c1d2e3f4-5678-4a1b-9c8d-abcdef654321", "event_type": "order.status_changed" },
      "httpStatusCode": null,
      "responseBody": null,
      "responseTimeMs": 10000,
      "attemptNumber": 1,
      "attemptedAt": "2026-09-24T14:00:12.000Z",
      "failureReason": "timeout"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```
*(o segundo exemplo mostra uma falha por timeout — sem `httpStatusCode`, já que a chamada foi abortada antes de qualquer resposta)*

Erros possíveis: `WEBHOOK_NOT_FOUND` (404).

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — Reprocessar evento em DLQ

Restrito a role `ADMIN` ([09:35]-[09:36] Sofia/Larissa). Recoloca o evento como `PENDING` na outbox.

Request (sem corpo):
```
POST /api/v1/admin/webhooks/dead-letter/d4e5f6a7-8901-4a1b-9c8d-abcdef789012/replay
Authorization: Bearer <jwt-de-um-usuario-ADMIN>
```

Response `200 OK` (formato de resposta é decisão de implementação, não discutido na reunião — apenas o comportamento "recoloca na outbox como pendente" foi definido, [09:18] Diego):
```json
{ "id": "d4e5f6a7-8901-4a1b-9c8d-abcdef789012", "status": "requeued" }
```

Erros possíveis: `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404), `FORBIDDEN` (403, role diferente de `ADMIN`).

### 5.8 Contrato do webhook enviado ao cliente (chamada de saída)

Não é um endpoint do OMS, mas o contrato da chamada HTTP que o worker faz ao endpoint do cliente ([09:43]-[09:45] Diego/Sofia).

Headers:
| Header | Conteúdo |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID único do evento, gerado na inserção na outbox |
| `X-Signature` | HMAC-SHA256 do corpo da requisição, usando a secret do endpoint. **Em aberto:** a reunião não definiu a codificação do valor (hex ou base64) nem um prefixo de esquema (ex.: `sha256=...`, usado por outras plataformas) — necessário para o cliente conseguir validar |
| `X-Timestamp` | Timestamp do momento do envio. A reunião definiu ISO 8601 para o campo `timestamp` do corpo ([09:43] Diego), mas não confirmou explicitamente o formato deste header — esta especificação assume o mesmo formato ISO 8601, para consistência |
| `X-Webhook-Id` | ID do cadastro de webhook que originou o envio |

Body:
```json
{
  "event_id": "c1d2e3f4-5678-4a1b-9c8d-abcdef654321",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-24T14:00:02.150Z",
  "order_id": "9f8e7d6c-5b4a-3c2d-1e0f-abcdef123456",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "8f14e45f-ceea-4c8b-8cc7-1f1b1a1b1a1b",
  "total_cents": 15990
}
```

Não inclui os itens do pedido — o cliente pode consultar `GET /orders/:id` para detalhes completos ([09:43]-[09:44] Diego). Limite de tamanho: 64KB; requisições que ultrapassem esse limite não são enviadas ([09:23]-[09:24] Sofia/Diego).

## 6. Matriz de Erros

Nem toda validação vira um código `WEBHOOK_*`: seguindo o padrão existente, uma falha de validação de schema Zod é capturada por [validate.middleware.ts:26-31](../src/middlewares/validate.middleware.ts) e sempre vira o código genérico `VALIDATION_ERROR`, com o detalhe do campo em `details` — inclusive a validação de URL HTTPS, que a própria Sofia descreveu como "só uma validação no schema Zod" ([09:23] Sofia). Os `WEBHOOK_*` abaixo são erros de negócio, lançados explicitamente pelo `WebhookService`, cada um exigindo uma subclasse própria de `AppError` com `errorCode` fixo, seguindo o padrão de `InvalidStatusTransitionError`/`InsufficientStockError` em [http-errors.ts:45-63](../src/shared/errors/http-errors.ts). Note que `NotFoundError` genérico ([http-errors.ts:27-31](../src/shared/errors/http-errors.ts)) não aceita código customizado — por isso `WEBHOOK_NOT_FOUND`/`WEBHOOK_CUSTOMER_NOT_FOUND` exigem subclasses dedicadas, e não instâncias diretas de `NotFoundError`. Já `ConflictError` ([http-errors.ts:33-37](../src/shared/errors/http-errors.ts)) aceita um código customizado no construtor (é assim que `order.service.ts` usa `INVALID_STATUS_TRANSITION`), mas nenhum erro `WEBHOOK_*` desta matriz é semanticamente um conflito, por isso nenhum o reutiliza aqui.

Dois códigos cogitados por Bruno como exemplo durante a reunião ([09:28]) foram deliberadamente **excluídos** desta matriz, por não sobreviverem a decisões tomadas mais adiante: `WEBHOOK_SECRET_REQUIRED`, porque a secret é sempre gerada pelo OMS e nunca enviada pelo cliente ([09:31] Marcos); e `WEBHOOK_INVALID_URL`, porque a validação de HTTPS é tratada como `VALIDATION_ERROR` genérico de schema, não como erro de negócio (ver parágrafo acima). Ver também [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

| Código | HTTP | Descrição | Origem |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Cadastro de webhook não encontrado | Nova subclasse de `AppError`, no padrão de [http-errors.ts](../src/shared/errors/http-errors.ts) |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` informado não existe | Nova subclasse de `AppError`; cenário análogo à checagem de customer em `OrderService.create` ([order.service.ts:60](../src/modules/orders/order.service.ts)), que hoje usa o `NotFoundError` genérico |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Evento de DLQ não encontrado para replay | Nova subclasse de `AppError`; [09:18]-[09:19] Diego |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (não é resposta HTTP a um caller) | Payload do evento ultrapassa 64KB no momento do envio pelo worker; registrado como motivo de falha em `webhook_deliveries`/`webhook_dead_letter`, não retornado a nenhum endpoint (ver seção 4.1, item 6) | [09:23]-[09:24] Sofia/Diego/Larissa |
| `VALIDATION_ERROR` | 400 | Corpo/query inválidos, incluindo URL não-HTTPS e status inválido em `events` (reutilizado, não é `WEBHOOK_*`) | Middleware [validate.middleware.ts](../src/middlewares/validate.middleware.ts), sem alteração |
| `FORBIDDEN` | 403 | Role diferente de `ADMIN` tentando reprocessar DLQ (reutilizado) | `requireRole` já existente ([src/middlewares/auth.middleware.ts:49](../src/middlewares/auth.middleware.ts)); [09:35]-[09:36] Sofia/Larissa |
| `UNAUTHORIZED` | 401 | Token ausente, inválido ou expirado (reutilizado, qualquer endpoint autenticado) | `authenticate` já existente ([src/middlewares/auth.middleware.ts:27](../src/middlewares/auth.middleware.ts)), sem alteração |
| `CONFLICT` | 409 | Violação de unicidade no Prisma (`P2002`, reutilizado) | [src/middlewares/error.middleware.ts](../src/middlewares/error.middleware.ts), sem alteração |

## 7. Estratégias de Resiliência

- **Timeout:** 10 segundos por chamada HTTP do worker ao endpoint do cliente ([09:42] Sofia/Diego).
- **Retry:** backoff exponencial 1m/5m/30m/2h/12h, ver seção 4.3 e [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).
- **Fallback:** esgotado o retry, o evento vai para DLQ e aguarda replay manual — não há fallback automático (ex.: e-mail) nesta fase ([09:37] Larissa).
- **Isolamento de falhas:** o worker roda em processo separado, então um reinício da API não derruba o worker: "se a API reinicia, perde o worker" era o cenário evitado ([09:11] Diego). O inverso (falha do worker não afetar a API) é inferência razoável da mesma separação de processos, não uma afirmação literal da reunião — ver [ADR-002](adrs/ADR-002-worker-processo-separado-em-polling.md).
- **Idempotência do lado do cliente:** garantida pelo `X-Event-Id`, não pelo OMS — o cliente deve deduplicar (ver [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).
- **Healthcheck e política de restart do worker:** a reunião não definiu esse ponto ([ADR-002](adrs/ADR-002-worker-processo-separado-em-polling.md) e o [RFC](RFC.md) o remetem a este FDD). Recomendação deste documento, para orientar a implementação: o worker escreve um heartbeat (timestamp da última iteração do loop de polling) em log estruturado a cada ciclo; o orquestrador de processos usado em produção (a definir na infraestrutura de deploy, fora do escopo desta feature) reinicia o processo se nenhum heartbeat for observado por um intervalo múltiplo do polling (ex.: 30 segundos, 15 ciclos sem heartbeat).

## 8. Observabilidade

O projeto hoje não tem biblioteca de métricas (ex.: Prometheus/`prom-client`) nem tracing distribuído configurados — não há essas dependências em `package.json`. A observabilidade desta feature parte do que já existe (Pino) e propõe extensões consistentes com esse padrão, não a introdução de ferramentas novas (consistente com [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)):

- **Logs:** eventos estruturados via Pino ([src/shared/logger/index.ts](../src/shared/logger/index.ts)), seguindo o padrão já usado em `src/server.ts` (`logger.info({...}, 'server_started')`) e no `request-logger.middleware.ts` (`logger.info({...}, 'http_request')`). Eventos sugeridos: `webhook_event_enqueued`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_retry_scheduled`, `webhook_dead_lettered`, `webhook_dead_letter_replayed` (este último incluindo `userId` do administrador, para auditoria).
- **Correlação:** o `event_id` (UUID) gerado na inserção na outbox deve ser incluído em todo log relacionado ao processamento daquele evento pelo worker, cumprindo o mesmo papel que o `X-Request-Id`/`requestId` já cumpre nas requisições HTTP da API ([src/middlewares/request-logger.middleware.ts](../src/middlewares/request-logger.middleware.ts)).
- **Métricas (a implementar, sem biblioteca definida):** contagem de eventos pendentes na outbox, taxa de sucesso/falha de entrega, latência de entrega, contagem de eventos em DLQ. A escolha da ferramenta de métricas está fora do escopo decidido na reunião e deve ser tratada como uma decisão de implementação separada.
- **Tracing:** não há tracing distribuído (ex.: OpenTelemetry) no projeto hoje; não é proposto nesta feature.

## 9. Dependências e Compatibilidade

- **Runtime:** Node ≥20 (já exigido em `package.json`, campo `engines`) — o worker pode usar o `fetch` nativo do Node para as chamadas HTTP de saída, sem adicionar nova dependência de cliente HTTP.
- **Banco de dados:** MySQL via Prisma 5.22.0 (já em uso); exige novas migrations para as tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.
- **Sem novas dependências de runtime** para autenticação (usa `jsonwebtoken`/`crypto` nativo do Node para HMAC), validação (`zod`, já em uso) ou logging (`pino`, já em uso) — consistente com [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).
- **Compatibilidade:** não requer alteração de contrato (request/response) em nenhum endpoint existente. A extensão de código existente mais sensível é `OrderService.changeStatus()`; os demais pontos de integração (seção 10) são todos aditivos — novos arquivos, novas linhas de composição, sem alterar comportamento existente.

## 10. Integração com o Sistema Existente

Esta feature se integra ao código já existente nos seguintes pontos:

1. **[src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts) — `changeStatus()` (linha 126):** o método passa a chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação já existente (`this.prisma.$transaction`), logo após a atualização de `orders` e antes do commit — sem alterar a assinatura pública do método nem seu contrato de erros atuais (`NotFoundError`, `ConflictError`/`InvalidStatusTransitionError` das linhas 141-149, `InsufficientStockError` lançado por `debitStock` na linha 223, continuam funcionando exatamente como hoje).
2. **[src/shared/errors/http-errors.ts](../src/shared/errors/http-errors.ts):** novas classes de erro do módulo de webhooks (`WebhookNotFoundError`, `WebhookCustomerNotFoundError`, `WebhookDeadLetterNotFoundError`) estendem `AppError` diretamente, cada uma fixando seu próprio `errorCode` — seguindo o mesmo padrão de `InvalidStatusTransitionError`/`InsufficientStockError` (linhas 45-63), que também estendem `AppError`/uma de suas subclasses para carregar um código específico. Erros de validação de schema (URL, `events`) **não** ganham classe própria — seguem como `VALIDATION_ERROR` genérico (ver seção 6). Nenhuma classe existente precisa mudar.
3. **[src/middlewares/error.middleware.ts](../src/middlewares/error.middleware.ts):** nenhuma alteração necessária — o middleware já trata qualquer instância de `AppError` e `ZodError` genericamente, cobrindo os novos erros `WEBHOOK_*` automaticamente.
4. **[src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts):** o endpoint `POST /admin/webhooks/dead-letter/:id/replay` reutiliza `requireRole('ADMIN')` (linha 49) — o mesmo padrão já usado hoje em [src/modules/users/user.routes.ts:15](../src/modules/users/user.routes.ts) para restringir endpoints a administradores. Nenhuma alteração no middleware.
5. **[src/shared/logger/index.ts](../src/shared/logger/index.ts):** o worker e o módulo de webhooks reutilizam a instância `logger` já exportada. Ponto de atenção: `redactPaths` hoje cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken` (linhas 5-11), mas não um campo `secret` — recomenda-se adicionar `*.secret` a essa lista antes de logar qualquer objeto de configuração de webhook. Atenção: o padrão `*.secret` da biblioteca `pino` redige apenas a chave `secret` um nível abaixo de qualquer objeto — não cobre `secret` na raiz do objeto logado nem aninhamentos mais profundos; a lista pode precisar de entradas adicionais dependendo de como o objeto é logado na prática.
6. **[src/routes/index.ts](../src/routes/index.ts) e [src/app.ts](../src/app.ts):** um novo `buildWebhookRouter` é montado em `/webhooks` (e um router administrativo em `/admin/webhooks`), seguindo exatamente o padrão de composição já usado para `orders`, `customers` e `products`; `buildControllers` em `app.ts` ganha a instanciação de `WebhookRepository`/`WebhookService`/`WebhookController`.
7. **[prisma/schema.prisma](../prisma/schema.prisma):** adição dos modelos `WebhookEndpoint`, `WebhookOutboxEvent`, `WebhookDelivery` e `WebhookDeadLetter`, seguindo a convenção já usada (chave primária `String @id @default(uuid()) @db.Char(36)`, `@@map` para nome de tabela em snake_case, índices em campos de filtro frequente).
8. **[package.json](../package.json):** novo script `"worker": "tsx watch --env-file=.env src/worker.ts"` (dev) e equivalente de produção, espelhando os scripts `dev`/`start` já existentes para `src/server.ts`.

## 11. Critérios de Aceite Técnicos

- [ ] Uma mudança de status que gera evento elegível insere a linha em `webhook_outbox` na mesma transação de `changeStatus`; forçar uma falha após a mudança de status (mas antes do commit) reverte também a mudança de status.
- [ ] Uma mudança de status para um valor que nenhum webhook do cliente assina não gera nenhuma linha em `webhook_outbox`.
- [ ] O worker entrega um evento pendente em até ~2 segundos após sua inserção, no caminho feliz.
- [ ] Um endpoint que responde com erro ou não responde em 10s tem o evento reagendado conforme a tabela de backoff (1m/5m/30m/2h/12h).
- [ ] Esgotadas as tentativas, o evento aparece em `webhook_dead_letter` e some de `webhook_outbox`.
- [ ] `POST /admin/webhooks/dead-letter/:id/replay` com um usuário não-`ADMIN` retorna 403; com um `ADMIN`, recoloca o evento como pendente e gera log de auditoria com o `userId`.
- [ ] A assinatura `X-Signature` enviada é validável pelo cliente usando a secret retornada na criação do webhook.
- [ ] `GET /webhooks/:id/deliveries` retorna entregas com sucesso e falha, incluindo código de status HTTP (quando houver) e tempo de resposta.
- [ ] Cadastrar um webhook com `url` não-HTTPS retorna `VALIDATION_ERROR` (400) e não cria o registro.
- [ ] Todas as tentativas (inicial e retries) de um mesmo evento são enviadas com o mesmo `X-Event-Id`.
- [ ] Rotacionar a secret de um webhook mantém a secret anterior válida por 24h e a invalida após esse período.
- [ ] Toda chamada de saída ao cliente inclui os 5 headers do contrato (seção 5.8): `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.

> Nota: o critério de backoff acima assume a leitura de 6 chamadas HTTP no total (envio inicial + 5 retries) — ver a ambiguidade registrada na seção 4.3 e no [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md), a confirmar antes da implementação. O critério de latência de "~2 segundos" mede apenas o ciclo de coleta do worker; o SLA de ponta a ponta acordado com o cliente é "abaixo de 10 segundos" ([09:02] Marcos).

## 12. Riscos e Mitigação

- **Risco:** a ambiguidade sobre a contagem exata de "5 tentativas" (seção 4.3) pode gerar uma implementação divergente do que o time realmente pretendia. **Mitigação:** confirmar com Larissa/Diego antes de implementar o contador de tentativas.
- **Risco:** o mecanismo de assinatura do OMS durante o grace period de rotação de secret não foi definido na reunião (ver [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md)). **Mitigação:** decidir explicitamente (ex.: assinar com a secret mais nova e aceitar verificação contra ambas do lado do cliente) antes da implementação do endpoint de rotação.
- **Risco:** a lista de redação de logs do Pino (`redactPaths` em [src/shared/logger/index.ts](../src/shared/logger/index.ts)) não cobre um campo `secret`, criando risco de a secret de um webhook vazar em log de aplicação do próprio OMS — o mesmo tipo de risco que já se concretizou do lado de um cliente, cujo log de aplicação vazou uma secret ("a gente já teve cliente que vazou secret em log de aplicação **dele**", [09:22] Diego). **Mitigação:** atualizar `redactPaths` antes de qualquer log que manipule objetos de configuração de webhook.
- **Risco:** ausência de rate limiting de saída pode gerar rajadas de chamadas a um mesmo cliente em picos de mudança de status. **Mitigação:** nenhuma nesta fase — ponto explicitamente deixado em aberto pelo time ([09:38]-[09:39] Diego), a ser observado em produção.
- **Risco:** a garantia de ordem de entrega por `order_id` (seção 4.2, item 6) pode ser quebrada quando um evento entra em retry e um evento mais recente do mesmo pedido é entregue antes dele — cenário não coberto na reunião. **Mitigação:** nenhuma definida; considerar, na implementação, bloquear o processamento de eventos de um `order_id` enquanto houver um evento anterior daquele pedido aguardando retry.
- **Risco:** o worker não tem healthcheck nem política de restart definidos (ver seção 7) — uma falha silenciosa do processo pararia toda a entrega de webhooks sem alarme automático. **Mitigação:** definir mecanismo de liveness e restart antes do deploy em produção; fora do escopo decidido na reunião.
- **Risco:** por ser um único worker processando o lote em sequência (seção 4.2), um cliente lento (até o timeout de 10s) pode atrasar a entrega dos demais eventos do lote, empurrando-os para além do SLA de 10 segundos combinado com os outros clientes — o mesmo tipo de problema que motivou descartar o disparo síncrono ([09:04] Bruno), agora reintroduzido dentro do worker. **Mitigação:** nenhuma definida na reunião; considerar, na implementação, paralelizar o envio dentro do lote (com um limite de concorrência) em vez de enviar sequencialmente.
- **Risco:** se o processo do worker cair entre marcar um evento como `PROCESSING` (seção 4.2, item 3) e registrar o resultado da tentativa, esse evento fica preso em `PROCESSING` indefinidamente, sem ser reprocessado pela consulta do item 2 (que só considera `PENDING`). **Mitigação:** nenhuma definida na reunião; considerar, na implementação, um timeout de "PROCESSING órfão" (ex.: reverter para `PENDING` se `PROCESSING` há mais tempo que o timeout de 10s mais uma margem).
- **Risco:** assinar um webhook ao status `PENDING` nunca dispararia nenhum evento — `changeStatus()` nunca transiciona **para** `PENDING` (esse status só é atribuído na criação do pedido, [order.status.ts](../src/modules/orders/order.status.ts)). **Mitigação:** ao validar a lista de `events` no cadastro do webhook, considerar alertar ou impedir a inclusão de `PENDING`, já que não é um destino de transição válido.
