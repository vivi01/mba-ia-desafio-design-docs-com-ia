# FDD: Sistema de Webhooks de Notificação de Pedidos

> Documento de design de implementação. Decisões arquiteturais estão nos [ADRs](adrs/README.md); a visão de arquitetura está no [RFC](RFC.md); requisitos de produto no [PRD](PRD.md). Arquivos marcados **(a criar)** não existem no repositório — são o plano de implementação.

## Contexto e motivação técnica

Hoje o sistema não notifica ninguém quando um pedido muda de status. A transição acontece em `changeStatus` (`src/modules/orders/order.service.ts`): uma transação Prisma que atualiza `orders`, insere em `order_status_history` e debita/repõe estoque conforme a transição (`[09:04] Bruno`). Os clientes B2B descobrem mudanças fazendo polling no `GET /orders`, o que é lento e caro para eles (`[09:00] Marcos`).

A feature adiciona a publicação de um evento `order.status_changed` **dentro dessa mesma transação** (padrão outbox, [ADR-001](adrs/ADR-001-outbox-no-mysql.md)) e um worker em processo separado que entrega esse evento por HTTP aos endpoints cadastrados pelos clientes ([ADR-002](adrs/ADR-002-worker-separado-polling.md)), com assinatura HMAC ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)), retry e DLQ ([ADR-003](adrs/ADR-003-retry-backoff-dlq.md)) e semântica at-least-once ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)).

## Objetivos técnicos

1. Notificação entregue em menos de 10 segundos após a mudança de status, no caminho feliz (`[09:02] Marcos`); com polling de 2 s, a latência mínima é de até 2 s (`[09:10] Larissa`).
2. Atomicidade absoluta: não pode existir status mudado sem evento registrado, nem evento registrado com rollback do status (`[09:41] Diego`).
3. Zero impacto de disponibilidade dos clientes sobre a operação de pedidos (`[09:04] Bruno`).
4. Nenhum evento perdido: falha permanente termina em DLQ reprocessável, nunca em descarte silencioso (`[09:15]`–`[09:18] Diego`).
5. Reuso integral dos padrões do projeto — módulo, erros, logger, middlewares ([ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)).

## Escopo e exclusões

**No escopo:** CRUD de configuração de webhooks por customer; geração e rotação de secret; publicação transacional na outbox com filtro por status assinados; worker de entrega com retry/backoff e DLQ; replay admin; histórico de entregas; assinatura HMAC e headers de segurança.

**Fora do escopo (explicitamente descartado ou adiado na reunião):**

| Exclusão | Origem |
| --- | --- |
| Webhooks inbound (cliente → plataforma) | `[09:02] Marcos` |
| Email de aviso ao cliente quando o webhook falha repetidamente — próxima fase | `[09:37] Larissa` |
| Rate limiting de envio por cliente — "observar e decidir depois" | `[09:39] Larissa` |
| Dashboard visual para o cliente — projeto separado do frontend | `[09:40] Larissa` |
| Arquivamento das linhas entregues da outbox (~30 dias) | `[09:08] Diego` |
| Múltiplos workers em paralelo (particionamento / lock pessimista) | `[09:13] Diego` |
| Endurecimento das roles do CRUD (hoje: qualquer role autenticada) | `[09:37] Sofia` |

## Modelo de dados (a criar em `prisma/schema.prisma`)

Quatro tabelas novas, seguindo o padrão do schema existente (`id` UUID `@db.Char(36)`, `@@map` snake_case, `@@index` — ver `prisma/schema.prisma`):

- **`webhook_endpoints`** — configuração por cliente: `id`, `customerId`, `url` (https), `secret`, `previousSecret` + `previousSecretExpiresAt` (grace de 24 h da rotação, `[09:21] Sofia`), `events` (lista de `OrderStatus` assinados, `[09:33] Marcos`), `active` (`[09:21] Bruno`), timestamps.
- **`webhook_outbox`** — o outbox ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)): `id` UUID — usado também como `event_id` do header `X-Event-Id` (`[09:25] Diego`, `[09:51] Larissa`) —, `webhookEndpointId`, `orderId`, `eventType`, `payload` (JSON snapshot, [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)), `status` (`PENDING` | `PROCESSING` | `FAILED` | `DELIVERED`, `[09:08] Diego`), `attempts`, `nextAttemptAt`, `lastError`, `createdAt`, `deliveredAt`. Índices em `status` e `createdAt` (`[09:08] Diego`) e, derivado do retry, em `nextAttemptAt`.
- **`webhook_dead_letter`** — DLQ ([ADR-003](adrs/ADR-003-retry-backoff-dlq.md)): `id`, `eventId`, `webhookEndpointId`, `payload`, `reason` (motivo da falha), `failedAt` (`[09:18] Diego`).
- **`webhook_deliveries`** — histórico de tentativas, necessário para o `GET /webhooks/:id/deliveries` (sucesso/falha, payload, response, tempo de resposta — `[09:34] Marcos`): `id`, `eventId`, `webhookEndpointId`, `attempt`, `responseStatus`, `responseBody` (truncado), `responseTimeMs`, `success`, `createdAt`.

## Fluxos detalhados

### 1. Criação do evento na outbox (dentro da transação do `changeStatus`)

1. `PATCH /orders/:id/status` chega ao `changeStatus` de `src/modules/orders/order.service.ts`, que abre a transação Prisma existente: valida a transição pela máquina de estados (`src/modules/orders/order.status.ts`), atualiza `orders`, insere em `order_status_history`, debita/repõe estoque.
2. **Novo passo, na mesma transação:** chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:41] Bruno`) — função do módulo de webhooks **(a criar)** que recebe o `Prisma.TransactionClient` atual, sem injeção de repository (`[09:41] Diego`).
3. `publishWebhookEvent` busca os `webhook_endpoints` ativos do `customerId` do pedido cujo filtro `events` contém `toStatus`. **Se nenhum assina o status, não insere nada** — filtro na inserção, economiza linha na tabela (`[09:34] Bruno`).
4. Para cada endpoint que assina: renderiza o payload completo (snapshot do estado atual do pedido, [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)) e insere uma linha `PENDING` na `webhook_outbox` com `id` novo (UUID = futuro `X-Event-Id`).
5. Commit da transação: status + histórico + estoque + evento(s) confirmados juntos. Rollback em qualquer passo desfaz tudo — "se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno`).

### 2. Processamento pelo worker (polling, batch, marcação)

1. Entry-point `src/worker.ts` **(a criar)**, processo Node separado da API (`npm run worker`), com PrismaClient próprio (instância por processo, mesma `DATABASE_URL` — `[09:30] Bruno`) e graceful shutdown no padrão de `src/server.ts`.
2. Loop de polling: a cada **2 segundos** (`[09:09] Diego`), busca em batch pequeno (`[09:08] Diego`) os eventos elegíveis mais antigos, em ordem de `createdAt`: `status = PENDING` ou (`status = FAILED` e `nextAttemptAt <= now`).
3. Cada evento é marcado `PROCESSING` antes do envio (evita reenvio no mesmo ciclo).
4. Envio: `POST` para a `url` do endpoint com o payload snapshot e os headers do contrato (ver [Contratos públicos](#contratos-públicos)), timeout de **10 segundos** (`[09:42] Diego`).
5. Toda tentativa (sucesso ou falha) gera uma linha em `webhook_deliveries` com status HTTP, corpo da resposta truncado e tempo de resposta.
6. Resposta 2xx → evento marcado `DELIVERED` (`deliveredAt` preenchido). Não-2xx, timeout ou erro de rede → fluxo de retry.
7. Sendo single-worker, o processamento em ordem de `createdAt` preserva a ordem por `order_id` (`[09:12] Diego`); ordering global não é garantida (`[09:13] Larissa`).

### 3. Retry (backoff, limites)

1. Falha na tentativa: `attempts` é incrementado, `lastError` registrado, `status = FAILED` e `nextAttemptAt = now + intervalo[attempts]`.
2. Progressão dos intervalos entre tentativas consecutivas: **1 min, 5 min, 30 min, 2 h, 12 h** (`[09:17] Diego`) — **5 retentativas** após a falha do envio inicial, somando ~14 h 36 min ("quase 15 horas entre primeira falha e última tentativa", `[09:17] Diego`).
   > **Nota de interpretação:** a reunião fixou "5 tentativas" e 5 intervalos (`[09:17]`, `[09:48]`). A única leitura aritmeticamente consistente com "quase 15 horas entre primeira falha e última tentativa" é: envio inicial + 5 retentativas nos intervalos acima (até 6 envios no total). É essa a semântica adotada aqui.
3. O evento em `FAILED` com `nextAttemptAt` futuro fica invisível ao polling até vencer o horário; depois volta a ser elegível.
4. Timeout de 10 s conta como falha normal de tentativa (`[09:42] Diego`).

### 4. DLQ (quando entra, o que acontece depois)

1. Falha na última retentativa → falha permanente: o worker move o evento para `webhook_dead_letter` com payload, motivo da última falha e timestamp (`[09:18] Diego`) e remove a linha da outbox (leitura da outbox principal fica limpa — `[09:18] Diego`).
2. Reprocessamento é **manual**: `POST /admin/webhooks/dead-letter/:id/replay`, restrito a role `ADMIN` via `requireRole` (`[09:36] Larissa`), recoloca o evento na `webhook_outbox` como `PENDING` com `attempts` zerado (`[09:18] Diego`).
3. O replay gera log de auditoria com o usuário que o executou (`[09:36] Sofia`).
4. Não há aviso proativo ao cliente sobre eventos na DLQ nesta fase (email adiado — `[09:37] Larissa`).

## Contratos públicos

Todos os endpoints da API usam `authenticate` (JWT Bearer — `src/middlewares/auth.middleware.ts`); qualquer role autenticada, exceto o replay (ADMIN) (`[09:36]`–`[09:37]`). Erros seguem o formato do error middleware existente: `{ "error": { "code", "message", "details?" } }`. O `customer_id` é sempre explícito no body ou query — não vem do JWT (`[09:32] Larissa`). Rotas novas montadas em `/webhooks` e `/admin/webhooks` via `src/routes/index.ts`.

### 1. `POST /webhooks` — cadastrar webhook

A secret é **gerada pela plataforma e devolvida somente na criação** (`[09:31] Marcos`); URLs `http` são recusadas na validação Zod (`[09:23] Sofia`).

Request:

```json
{
  "url": "https://integracao.atlascomercial.com.br/webhooks/pedidos",
  "customerId": "3f8a2c1e-5b7d-4e9f-a1b2-c3d4e5f6a7b8",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "9c1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
  "url": "https://integracao.atlascomercial.com.br/webhooks/pedidos",
  "customerId": "3f8a2c1e-5b7d-4e9f-a1b2-c3d4e5f6a7b8",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f86d081884c7d659a2feaa0c55ad015",
  "createdAt": "2026-07-19T12:00:00.000Z"
}
```

Status codes: `201` sucesso; `400` `WEBHOOK_INVALID_URL` (url não-https) ou `VALIDATION_ERROR` (Zod); `401` sem JWT; `404` `NOT_FOUND` (customer inexistente).

### 2. `GET /webhooks?customerId=...` — listar webhooks do customer

Response `200 OK` (padrão paginado de `src/shared/http/response.ts`; a secret **não** é retornada em leituras):

```json
{
  "data": [
    {
      "id": "9c1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
      "url": "https://integracao.atlascomercial.com.br/webhooks/pedidos",
      "customerId": "3f8a2c1e-5b7d-4e9f-a1b2-c3d4e5f6a7b8",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-19T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status codes: `200`; `400` `VALIDATION_ERROR` (query inválida); `401`.

### 3. `PATCH /webhooks/:id` — editar webhook

Request (campos opcionais — url, filtro de eventos, ativação):

```json
{
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

Response `200 OK`: o webhook atualizado (mesmo shape do GET, sem secret).

```json
{
  "id": "9c1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
  "url": "https://integracao.atlascomercial.com.br/webhooks/pedidos",
  "customerId": "3f8a2c1e-5b7d-4e9f-a1b2-c3d4e5f6a7b8",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "createdAt": "2026-07-19T12:00:00.000Z"
}
```

Status codes: `200`; `400` `WEBHOOK_INVALID_URL` / `VALIDATION_ERROR`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 4. `DELETE /webhooks/:id` — remover webhook

Response `204 No Content` (sem corpo). Status codes: `204`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 5. `POST /webhooks/:id/rotate-secret` — rotacionar secret

A antiga permanece válida por 24 h em paralelo (`[09:21] Sofia`). Request sem corpo.

Response `200 OK`:

```json
{
  "id": "9c1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
  "secret": "whsec_2c26b46b68ffc68ff99b453c1d304134",
  "previousSecretExpiresAt": "2026-07-20T12:00:00.000Z"
}
```

Status codes: `200`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 6. `GET /webhooks/:id/deliveries` — histórico de entregas

Últimos 100 envios, com sucesso/falha, payload, response e tempo de resposta (`[09:34] Marcos`).

Response `200 OK`:

```json
{
  "data": [
    {
      "eventId": "b7e6a1d0-2f3c-4a5b-8c9d-0e1f2a3b4c5d",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "responseTimeMs": 184,
      "payload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "createdAt": "2026-07-19T12:00:04.000Z"
    }
  ]
}
```

Status codes: `200`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 7. `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar evento da DLQ

Exige role `ADMIN` (`requireRole('ADMIN')` — `[09:36] Larissa`); auditado em log. Request sem corpo.

Response `202 Accepted`:

```json
{
  "eventId": "b7e6a1d0-2f3c-4a5b-8c9d-0e1f2a3b4c5d",
  "status": "PENDING",
  "requeuedAt": "2026-07-19T14:30:00.000Z"
}
```

Status codes: `202`; `401`; `403` `FORBIDDEN` (role insuficiente); `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 8. Contrato do webhook entregue ao cliente (outbound)

`POST` na URL cadastrada. Headers (`[09:44] Diego`, `[09:44] Sofia`):

| Header | Conteúdo |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID único do evento (dedup no cliente — [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)) |
| `X-Signature` | HMAC-SHA256 do corpo bruto, codificação hex (detalhe de encoding a confirmar na revisão de segurança — `[09:46] Sofia`) |
| `X-Timestamp` | Timestamp do envio (detecção de replay attack pelo cliente) |
| `X-Webhook-Id` | id do cadastro de webhook que originou o envio |

Body — payload enxuto, snapshot do momento da transição, **sem items** (detalhes via `GET /orders/:id` — `[09:43] Diego`):

```json
{
  "event_id": "b7e6a1d0-2f3c-4a5b-8c9d-0e1f2a3b4c5d",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-19T12:00:02.000Z",
  "order_id": "7a8b9c0d-1e2f-3a4b-5c6d-7e8f9a0b1c2d",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3f8a2c1e-5b7d-4e9f-a1b2-c3d4e5f6a7b8",
  "total_cents": 152990
}
```

Resposta esperada do cliente: qualquer `2xx` em até 10 s = entregue; qualquer outra coisa = falha e retry. Durante o grace period de rotação, o request carrega assinaturas da secret nova e da antiga (proposta de implementação a validar na revisão de segurança da Sofia).

## Matriz de erros

Códigos com prefixo `WEBHOOK_`, seguindo o padrão de `src/shared/errors/` (`[09:28] Bruno`, `[09:29] Larissa`). Os três primeiros foram citados nominalmente na reunião; os demais derivam das regras decididas (origem indicada).

| Código | HTTP | Quando ocorre | Classe reutilizada (base em `src/shared/errors/http-errors.ts`) | Origem |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `:id` de webhook inexistente em GET/PATCH/DELETE/rotate/deliveries | `NotFoundError` | `[09:28] Bruno` |
| `WEBHOOK_INVALID_URL` | 400 | URL inválida ou não-`https` no cadastro/edição | `ValidationError` | `[09:28] Bruno` + regra TLS `[09:23] Sofia` |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Estado inconsistente: webhook ativo sem secret válida ao assinar | `UnprocessableEntityError` | `[09:28] Bruno` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de `:id` inexistente na DLQ | `NotFoundError` | derivado do replay `[09:18] Diego` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (interno) | Payload renderizado excede 64KB: evento não é enviado, falha com este motivo | — (motivo em `lastError`/DLQ) | limite `[09:24] Larissa` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | Cliente não respondeu em 10 s; conta como falha de tentativa | — (motivo em `webhook_deliveries`/`lastError`) | timeout `[09:42] Diego` |
| `WEBHOOK_DELIVERY_FAILED` | — (interno) | Resposta não-2xx ou erro de rede na tentativa | — (motivo em `webhook_deliveries`/`lastError`) | fluxo de retry `[09:15] Diego` |

Códigos genéricos existentes continuam valendo fora do domínio: `VALIDATION_ERROR` (Zod), `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND` (customer inexistente) — tratados pelo error middleware sem mudança (`[09:29] Bruno`).

## Estratégias de resiliência

- **Atomicidade na origem:** evento gravado na transação do `changeStatus`; rollback desfaz status e evento juntos ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
- **Timeout:** 10 s por chamada HTTP; estourou, é falha comum de tentativa (`[09:42] Diego`).
- **Retry/backoff:** 5 retentativas em 1m/5m/30m/2h/12h (~15 h de janela) ([ADR-003](adrs/ADR-003-retry-backoff-dlq.md)); cobre indisponibilidades reais de clientes, como manutenção de 2 h (`[09:16] Diego`).
- **DLQ:** falha permanente preservada com payload + motivo + timestamp; replay manual por ADMIN recoloca como `PENDING`.
- **Idempotência:** at-least-once; duplicatas são possíveis e o cliente deduplica pelo `X-Event-Id` ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)); o snapshot garante corpo idêntico em todo reenvio ([ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)).
- **Isolamento de processo:** queda da API não para entregas; queda do worker não para a API — eventos acumulam na outbox e são drenados quando o worker volta ([ADR-002](adrs/ADR-002-worker-separado-polling.md)).
- **Limite de payload:** eventos acima de 64KB não são enviados — falha explícita, nunca truncamento (`[09:23] Sofia`, `[09:24] Larissa`).

## Observabilidade

**Métricas** (expostas pelo worker; nomes indicativos):

- `webhook_outbox_pending` — tamanho do backlog (gauge); é o indicador de worker parado ou cliente em massa fora do ar.
- `webhook_delivery_duration_ms` — tempo de resposta por endpoint (histograma; alimenta o requisito de < 10 s).
- `webhook_delivery_total` — contador por resultado (sucesso, falha, timeout) e por endpoint.
- `webhook_dead_letter_total` — eventos que viraram falha permanente.
- `webhook_poll_cycle_duration_ms` — duração do ciclo de polling (detecta batch grande demais ou banco lento).

**Logs** — via logger Pino existente (`src/shared/logger/index.ts`), estruturados:

- Worker: um log por tentativa com `eventId`, `webhookId`, `attempt`, `responseStatus`, `responseTimeMs`, resultado; nível `warn` para falha de tentativa, `error` para movimentação à DLQ.
- API: criação/edição/remoção/rotação logadas com `webhookId` e `customerId`.
- **Auditoria obrigatória:** replay de DLQ loga o usuário que executou (`[09:36] Sofia`).
- A lista `redactPaths` do logger deve ser estendida para `*.secret` / `*.previousSecret` — secrets nunca aparecem em log (motivado pelo incidente citado em `[09:22] Diego`).

**Tracing/correlação:** o projeto não tem APM; a correlação é feita por identificadores propagados — `req.id` do request que mudou o status → `eventId` da outbox → linhas de `webhook_deliveries` → `X-Event-Id` recebido pelo cliente. Isso permite reconstruir a cadeia completa de uma entrega ponta a ponta a partir de qualquer elo (e serve de base para instrumentação OpenTelemetry futura, se o projeto adotar).

## Dependências e compatibilidade

- **Runtime:** Node + TypeScript + Express + Prisma/MySQL já existentes; nenhuma lib nova de infraestrutura ([ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)). HMAC-SHA256 via `crypto` nativo do Node.
- **Banco:** quatro tabelas novas via migration Prisma **(a criar)**; nenhuma alteração em tabelas existentes — compatível com o schema atual.
- **API existente:** nenhum contrato existente muda. O `PATCH /orders/:id/status` ganha um insert a mais na transação, transparente para quem chama.
- **Worker:** mesmo `DATABASE_URL`, PrismaClient próprio por processo (`[09:30] Bruno`); deploy passa a ter dois processos (`npm run dev`/API e `npm run worker`).
- **Clientes:** precisam expor endpoint `https`, validar HMAC e deduplicar por `X-Event-Id`; documentação no portal de desenvolvedor (`[09:26] Marcos`, `[09:40] Marcos`).

## Integração com o sistema existente

Caminhos **reais** do repositório e como o módulo se integra a cada um:

| Arquivo existente | Integração |
| --- | --- |
| `src/modules/orders/order.service.ts` | Único ponto de alteração em código existente: `changeStatus` passa a chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação atual (`[09:40]`–`[09:41] Bruno/Diego`). |
| `src/modules/orders/order.status.ts` | Fonte canônica dos status (`OrderStatus`) usados no filtro `events` e nos campos `from_status`/`to_status` do payload; a máquina de estados define quais transições podem gerar evento. |
| `src/shared/errors/app-error.ts` | Classe base dos novos erros `WEBHOOK_*` (mesmo contrato `statusCode`/`errorCode`/`details`). |
| `src/shared/errors/http-errors.ts` | Padrão a seguir: erros derivados com código específico, como `InsufficientStockError` → ex.: `WebhookNotFoundError` **(a criar)**. |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas do módulo; `requireRole('ADMIN')` no replay de DLQ (`[09:36] Larissa`). |
| `src/middlewares/error.middleware.ts` | Trata os erros do módulo sem alteração — `AppError`, Zod e Prisma já cobertos (`[09:29] Bruno`). |
| `src/shared/logger/index.ts` | Logger Pino compartilhado por API e worker; estender `redactPaths` para secrets de webhook. |
| `prisma/schema.prisma` | Recebe os quatro models novos no padrão existente (UUID `@db.Char(36)`, `@@map`, `@@index`). |
| `src/routes/index.ts` | Registro do router do módulo (`/webhooks`, `/admin/webhooks`) no `buildApiRouter`. |
| `src/server.ts` | Modelo de bootstrap/graceful shutdown replicado na entry do worker. |

Arquivos **a criar** (nenhum existe hoje): `src/worker.ts`; `src/modules/webhooks/webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, `webhook.worker.ts` (ou `webhook.processor.ts` — nome final em aberto na reunião, `[09:28] Bruno`); migration Prisma das tabelas `webhook_*`; script `worker` no `package.json`.

## Critérios de aceite técnicos

1. Transição de status commitada com webhook assinante ⇒ linha na `webhook_outbox` na mesma transação; rollback ⇒ nenhuma linha (teste ponta a ponta).
2. Evento pendente entregue em < 10 s no caminho feliz, com polling de 2 s.
3. Endpoint que responde não-2xx/timeout: retentativas exatamente nos offsets 1m/5m/30m/2h/12h, e na 5ª falha de retentativa o evento está em `webhook_dead_letter` com motivo e fora da outbox.
4. `X-Signature` verificável pelo cliente com a secret do endpoint; após rotação, requests validam com a secret nova e a antiga por 24 h, e a antiga é recusada depois.
5. Cadastro com URL `http` retorna `400 WEBHOOK_INVALID_URL`; payload > 64KB nunca é enviado e falha com `WEBHOOK_PAYLOAD_TOO_LARGE`.
6. Mesmo `X-Event-Id` em todo reenvio de um evento; corpo byte a byte idêntico entre tentativas.
7. Replay sem role ADMIN retorna `403`; com ADMIN, evento volta a `PENDING` e o log registra o usuário executor.
8. Para um mesmo `order_id`, eventos entregues na ordem das transições (single-worker).
9. `GET /webhooks/:id/deliveries` reflete cada tentativa com status HTTP e tempo de resposta.

## Riscos e mitigação

| Risco | Mitigação |
| --- | --- |
| Crescimento contínuo da `webhook_outbox` degrada o polling (arquivamento ficou fora de escopo — `[09:08] Diego`) | Índices em `status`/`created_at`/`nextAttemptAt` + batch pequeno; DLQ separada mantém a outbox limpa; monitorar `webhook_outbox_pending`; arquivamento registrado como questão em aberto no [RFC](RFC.md) |
| Worker parado acumula backlog silenciosamente | Métrica de backlog + alerta; processo separado permite restart sem afetar a API; eventos não se perdem (ficam `PENDING`) |
| Insert adicional torna a transação de status mais lenta | Payload enxuto (`[09:43] Diego`) e renderização a partir de dados já carregados na transação; sem chamadas externas dentro da transação |
| Secret vazada pelo cliente (precedente real — `[09:22] Diego`) | Secret por endpoint limita blast radius; rotação com grace 24 h; secrets redigidas nos logs; revisão de segurança da Sofia antes do deploy (`[09:46] Sofia`) |
| Cliente não deduplica e processa evento duplicado | Semântica at-least-once documentada em destaque no portal (`[09:26] Marcos`); `X-Event-Id` estável em todo reenvio |
| Rajadas de eventos bombardeiam o cliente (rate limiting fora de escopo — `[09:39] Diego`) | Aceito nesta fase: "observar e implementar se virar problema"; métricas por endpoint dão a visibilidade para essa decisão |
