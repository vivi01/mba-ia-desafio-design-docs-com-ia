# TRACKER — Rastreabilidade dos design docs

Cada requisito, decisão, restrição, trade-off ou métrica registrado no pacote de design docs
(PRD, RFC, FDD, ADRs) tem uma linha aqui apontando para a fonte que o sustenta:

- **Fonte `TRANSCRICAO`** — `TRANSCRICAO.md`, localização no formato `[hh:mm] Nome` (fala que sustenta o item).
- **Fonte `CODIGO`** — caminho real de arquivo existente no repositório.

**Rotina de verificação executada em 2026-07-19:** 130 linhas, 0 IDs duplicados;
109 linhas com Fonte = TRANSCRICAO (84%, meta ≥ 70%) — todas as citações `[hh:mm] Nome`
conferidas por grep contra `TRANSCRICAO.md` (0 não encontradas); 21 linhas com Fonte =
CODIGO (meta ≥ 5) — todos os caminhos conferidos (os 3 caminhos futuros citados nos
documentos estão sempre marcados "(a criar)"); todos os caminhos da coluna Documento existem.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL: evento inserido na `webhook_outbox` na mesma transação da mudança de status | TRANSCRICAO | `[09:08] Larissa` |
| ADR-001-DEC-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox com índices por status (pendente/processando/falhou/entregue) e `created_at`; worker lê pendentes em batch pequeno | TRANSCRICAO | `[09:08] Diego` |
| ADR-001-DEC-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Filtro de status aplicado na inserção da outbox (se ninguém assina, nem insere) | TRANSCRICAO | `[09:34] Bruno, [09:34] Diego` |
| ADR-001-DEC-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | id da outbox em UUID, seguindo padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono dentro do service de orders (cliente lento trava status; rollback impossível) | TRANSCRICAO | `[09:04] Bruno, [09:06] Diego` |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams / fila dedicada (mais infra; overengineering para time pequeno) | TRANSCRICAO | `[09:07] Larissa, [09:07] Diego` |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Filtrar eventos na hora do envio em vez da inserção (geraria linhas que ninguém assina) | TRANSCRICAO | `[09:34] Diego, [09:34] Bruno` |
| ADR-001-ALT-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | id auto incremental na outbox (destoaria do padrão UUID do projeto) | TRANSCRICAO | `[09:51] Diego, [09:51] Larissa` |
| ADR-001-REST-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Arquivamento de linhas entregues (~30 dias) fora do escopo da feature | TRANSCRICAO | `[09:08] Diego` |
| ADR-001-INT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Integração | Transação `changeStatus` existente (update orders + history + estoque) é o ponto de inserção do outbox | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | docs/adrs/ADR-002-worker-separado-polling.md | Decisão | Worker em processo separado, entry-point nova `src/worker.ts` (a criar) + script `npm run worker` | TRANSCRICAO | `[09:11] Diego, [09:11] Larissa` |
| ADR-002-DEC-02 | docs/adrs/ADR-002-worker-separado-polling.md | Decisão | Polling da outbox a cada 2 segundos; latência mínima de 2s aceita | TRANSCRICAO | `[09:09] Diego, [09:10] Larissa` |
| ADR-002-DEC-03 | docs/adrs/ADR-002-worker-separado-polling.md | Decisão | PrismaClient separado por processo no worker, mesma `DATABASE_URL` | TRANSCRICAO | `[09:30] Bruno` |
| ADR-002-REST-01 | docs/adrs/ADR-002-worker-separado-polling.md | Restrição | Ordering garantida só por order_id e enquanto single-worker (limitação conhecida documentada) | TRANSCRICAO | `[09:12] Diego, [09:13] Larissa` |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-polling.md | Alternativa Descartada | Trigger de banco para acionar o worker (MySQL não notifica processo externo) | TRANSCRICAO | `[09:09] Bruno, [09:09] Diego` |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-polling.md | Alternativa Descartada | Worker dentro do mesmo processo da API (restart da API derruba o worker) | TRANSCRICAO | `[09:11] Diego` |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-polling.md | Alternativa Descartada | Múltiplos workers em paralelo (particionar por order_id / lock pessimista) — adiado, "problema do futuro" | TRANSCRICAO | `[09:13] Diego` |
| ADR-002-INT-01 | docs/adrs/ADR-002-worker-separado-polling.md | Integração | Entry-point `src/server.ts` existente como modelo da nova entry do worker | CODIGO | `src/server.ts` |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-dlq.md | Decisão | Retry com backoff exponencial: envio inicial + 5 retentativas (1m/5m/30m/2h/12h, ~15h entre primeira falha e última tentativa) | TRANSCRICAO | `[09:17] Diego, [09:17] Larissa` |
| ADR-003-DEC-02 | docs/adrs/ADR-003-retry-backoff-dlq.md | Decisão | DLQ em tabela separada `webhook_dead_letter` com payload, motivo da falha e timestamp | TRANSCRICAO | `[09:18] Diego, [09:19] Larissa` |
| ADR-003-DEC-03 | docs/adrs/ADR-003-retry-backoff-dlq.md | Decisão | Replay manual via `POST /admin/webhooks/dead-letter/:id/replay`, recoloca evento na outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-backoff-dlq.md | Alternativa Descartada | Retry indefinido com backoff (evento pendurado para sempre se o cliente sumiu) | TRANSCRICAO | `[09:15] Diego` |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-backoff-dlq.md | Alternativa Descartada | 3 tentativas (mataria em ~30min; não cobre indisponibilidade real de 2h) | TRANSCRICAO | `[09:16] Bruno, [09:16] Diego` |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-backoff-dlq.md | Alternativa Descartada | Marcar "failed" na própria outbox em vez de tabela DLQ separada | TRANSCRICAO | `[09:17] Larissa, [09:18] Diego` |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo do request, assinatura no header `X-Signature` | TRANSCRICAO | `[09:20] Sofia, [09:22] Sofia` |
| ADR-004-DEC-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Secret única por endpoint, rotação via API com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-DEC-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Header `X-Timestamp` no envio para detecção de replay attack pelo cliente | TRANSCRICAO | `[09:44] Diego` |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa Descartada | Secret global da plataforma ("se vaza uma, vaza tudo") | TRANSCRICAO | `[09:21] Sofia` |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | Garantia at-least-once com `X-Event-Id` (UUID gerado na entrada na outbox) para dedup no cliente | TRANSCRICAO | `[09:25] Diego, [09:26] Larissa` |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Alternativa Descartada | Exactly-once (exigiria coordenação dos dois lados; muito mais complexo) | TRANSCRICAO | `[09:25] Diego` |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso máximo do existente: AppError, Pino, error middleware, padrão de módulos, schemas Zod, códigos de erro | TRANSCRICAO | `[09:30] Larissa` |
| ADR-006-DEC-02 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Códigos de erro do módulo com prefixo `WEBHOOK_` (ex.: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED) | TRANSCRICAO | `[09:28] Bruno, [09:29] Larissa` |
| ADR-006-DEC-03 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Módulo `src/modules/webhooks` (a criar) no padrão controller/service/repository/routes/schemas; worker como `webhook.worker.ts`/`webhook.processor.ts` | TRANSCRICAO | `[09:27] Bruno, [09:28] Bruno` |
| ADR-006-DEC-04 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Replay de DLQ exige role ADMIN reusando `requireRole`; log de auditoria de quem executou | TRANSCRICAO | `[09:36] Sofia, [09:36] Larissa` |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Alternativa Descartada | Introduzir libs/estruturas novas para o módulo ("não vamos botar nada novo") | TRANSCRICAO | `[09:29] Bruno` |
| ADR-006-INT-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | Classe base `AppError` para os novos erros `WEBHOOK_*` | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-INT-02 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | Padrão de erros derivados com código (INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION) | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-INT-03 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | `requireRole` existente para exigir ADMIN no replay de DLQ | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-INT-04 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | Error middleware centralizado já trata AppError/Zod/Prisma sem mudanças | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-INT-05 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | Logger Pino compartilhado do projeto | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-INT-06 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | Padrão de módulo controller/service/repository/routes/schemas (referência: módulo orders) | CODIGO | `src/modules/orders/` |
| ADR-006-INT-07 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Integração | Registro modular de rotas onde o router de webhooks será montado | CODIGO | `src/routes/index.ts` |
| ADR-007 | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Decisão | Payload gravado já renderizado (snapshot) no momento da inserção na outbox | TRANSCRICAO | `[09:52] Larissa, [09:52] Bruno` |
| ADR-007-DEC-02 | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Decisão | Payload JSON enxuto (event_id, event_type, timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents), sem items | TRANSCRICAO | `[09:43] Diego` |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Alternativa Descartada | Guardar só order_id e renderizar payload na hora do envio ("caso esquisito" se pedido mudar) | TRANSCRICAO | `[09:51] Bruno, [09:52] Larissa` |
| RFC-PROP-01 | docs/RFC.md | Proposta | Solução consolidada: outbox MySQL + worker polling 2s + retry/backoff/DLQ + HMAC + at-least-once (resumo confirmado por todos) | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-02 | docs/RFC.md | Proposta | Demanda de Atlas Comercial, MaxDistribuição e Nova Cargo por notificação em tempo real; risco de churn da Atlas | TRANSCRICAO | `[09:00] Marcos` |
| RFC-PROP-03 | docs/RFC.md | Proposta | Escopo outbound only: plataforma envia, clientes recebem | TRANSCRICAO | `[09:02] Marcos, [09:03] Sofia` |
| RFC-PROP-04 | docs/RFC.md | Proposta | CRUD autenticado de webhooks com secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| RFC-PROP-05 | docs/RFC.md | Proposta | Alteração pontual no changeStatus: publicação do evento recebendo o client da transação atual | TRANSCRICAO | `[09:41] Bruno` |
| RFC-PROP-06 | docs/RFC.md | Métrica | "Tempo real" para os clientes = entrega abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| RFC-PROP-07 | docs/RFC.md | Restrição | Estimativa de 3 sprints incluindo revisão de segurança (≥ 2 dias úteis da Sofia); prazo Atlas fim de novembro | TRANSCRICAO | `[09:46] Larissa, [09:46] Sofia, [09:45] Marcos` |
| RFC-PROP-08 | docs/RFC.md | Restrição | Transação changeStatus existente é o ponto único de mudança no código atual | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono no service de orders | TRANSCRICAO | `[09:04] Bruno, [09:06] Diego` |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / fila dedicada | TRANSCRICAO | `[09:07] Larissa, [09:07] Diego` |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco para acionar o worker | TRANSCRICAO | `[09:09] Bruno, [09:09] Diego` |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Retry indefinido / teto de 3 tentativas | TRANSCRICAO | `[09:15] Diego, [09:16] Diego` |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Secret global da plataforma | TRANSCRICAO | `[09:21] Sofia` |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Garantia exactly-once | TRANSCRICAO | `[09:25] Diego` |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente — "observar e decidir depois" | TRANSCRICAO | `[09:38] Diego, [09:39] Larissa` |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers (particionamento/lock) — "problema do futuro" | TRANSCRICAO | `[09:13] Diego` |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Aviso proativo (email) ao cliente sobre webhook com falhas — próxima fase | TRANSCRICAO | `[09:37] Marcos, [09:37] Larissa` |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Arquivamento das linhas entregues da outbox (~30 dias) — fora do escopo da feature | TRANSCRICAO | `[09:08] Diego` |
| RFC-OPEN-05 | docs/RFC.md | Questão em Aberto | Endurecimento das roles do CRUD de configuração — "mais pra frente" | TRANSCRICAO | `[09:36] Marcos, [09:37] Sofia` |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Evento criado na outbox dentro da transação do changeStatus, via `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | `[09:40] Bruno, [09:41] Bruno` |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtro na inserção: se nenhum webhook do customer assina o status, não insere linha | TRANSCRICAO | `[09:33] Marcos, [09:34] Bruno` |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Worker: polling a cada 2s, batch pequeno dos pendentes mais antigos, processa e marca | TRANSCRICAO | `[09:08] Diego, [09:09] Diego` |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Retry: 5 retentativas nos intervalos 1m/5m/30m/2h/12h (~15h entre primeira falha e última tentativa) | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | DLQ: falha permanente vai para webhook_dead_letter (payload, motivo, timestamp); replay recoloca como pendente | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Payload renderizado (snapshot) no momento da inserção | TRANSCRICAO | `[09:52] Larissa` |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks: url + lista de status; secret gerada pela plataforma e devolvida na criação; customer_id no body | TRANSCRICAO | `[09:31] Marcos, [09:32] Larissa` |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | PATCH para editar, DELETE para remover, GET para listar webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | Endpoint de rotação de secret; antiga válida por 24h em paralelo | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries: últimos 100 envios com sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay, role ADMIN, auditado | TRANSCRICAO | `[09:18] Diego, [09:36] Sofia` |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Headers do envio: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type application/json | TRANSCRICAO | `[09:44] Diego, [09:44] Sofia` |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Payload JSON enxuto (event_id, event_type, timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents), sem items | TRANSCRICAO | `[09:43] Diego` |
| FDD-ERRO-01 | docs/FDD.md | Contrato | Códigos com prefixo WEBHOOK_ no padrão AppError; citados: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | `[09:28] Bruno, [09:29] Larissa` |
| FDD-ERRO-02 | docs/FDD.md | Requisito Não Funcional | URL http recusada com erro de validação (TLS obrigatório, schema Zod) | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-03 | docs/FDD.md | Requisito Não Funcional | Payload acima de 64KB: erro, não trunca, não envia | TRANSCRICAO | `[09:23] Sofia, [09:24] Diego, [09:24] Larissa` |
| FDD-ERRO-04 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s no HTTP call do worker; estourou = falha marcada para retry | TRANSCRICAO | `[09:42] Diego` |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus é o único ponto de alteração em código existente | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados como fonte canônica dos status do filtro e do payload | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | docs/FDD.md | Integração | AppError como classe base dos erros WEBHOOK_* | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-04 | docs/FDD.md | Integração | Padrão de erros derivados com código específico a replicar | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate em todas as rotas; requireRole('ADMIN') no replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | docs/FDD.md | Integração | Error middleware trata os erros do módulo sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-07 | docs/FDD.md | Integração | Logger Pino compartilhado; estender redactPaths para secrets | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-08 | docs/FDD.md | Integração | Novas tabelas webhook_* no padrão do schema (UUID Char(36), @@map, @@index) | CODIGO | `prisma/schema.prisma` |
| FDD-INT-09 | docs/FDD.md | Integração | Registro do router do módulo em buildApiRouter | CODIGO | `src/routes/index.ts` |
| FDD-INT-10 | docs/FDD.md | Integração | Bootstrap/graceful shutdown do server como modelo da entry do worker | CODIGO | `src/server.ts` |
| FDD-INT-11 | docs/FDD.md | Integração | Formato paginado {data, pagination} reutilizado na listagem | CODIGO | `src/shared/http/response.ts` |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados via Pino existente, sem logger novo | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria do replay com o usuário que executou | TRANSCRICAO | `[09:36] Sofia` |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Secrets redigidas em log (redactPaths), motivado por vazamento prévio de secret em log | TRANSCRICAO | `[09:22] Diego` |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Métricas por endpoint (backlog, resultado, duração) dão a base do "observar e decidir depois" do rate limiting | TRANSCRICAO | `[09:39] Diego` |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Notificação em tempo real: entrega < 10 segundos; latência mínima de 2s aceita | TRANSCRICAO | `[09:02] Marcos, [09:10] Larissa` |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Prazo: 3 sprints, fim de novembro (prazo da Atlas), com revisão de segurança incluída | TRANSCRICAO | `[09:45] Marcos, [09:47] Larissa` |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Nenhuma notificação perdida: 100% dos eventos entregues ou registrados como falha reprocessável | TRANSCRICAO | `[09:40] Bruno, [09:15] Diego` |
| PRD-ESC-01 | docs/PRD.md | Restrição | Fora de escopo: webhooks de entrada (cliente → plataforma) | TRANSCRICAO | `[09:02] Marcos` |
| PRD-ESC-02 | docs/PRD.md | Restrição | Fora de escopo (adiado): email de aviso ao cliente sobre webhook com falhas | TRANSCRICAO | `[09:37] Larissa` |
| PRD-ESC-03 | docs/PRD.md | Restrição | Fora de escopo (observar): rate limiting de envio por cliente | TRANSCRICAO | `[09:39] Diego, [09:39] Larissa` |
| PRD-ESC-04 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual (projeto separado do frontend) | TRANSCRICAO | `[09:39] Marcos, [09:40] Larissa` |
| PRD-ESC-05 | docs/PRD.md | Restrição | Fora de escopo: arquivamento automático de notificações entregues | TRANSCRICAO | `[09:08] Diego` |
| PRD-ESC-06 | docs/PRD.md | Restrição | Fora de escopo (adiado): múltiplos workers em paralelo | TRANSCRICAO | `[09:13] Diego` |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL e lista de status desejados | TRANSCRICAO | `[09:31] Marcos, [09:33] Marcos` |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Credencial gerada pela plataforma e devolvida no cadastro | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar, remover e listar webhooks do cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Webhook recebe somente os status que assinou | TRANSCRICAO | `[09:33] Marcos, [09:34] Bruno` |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Toda mudança de status gera notificação com pedido, status anterior e novo | TRANSCRICAO | `[09:40] Bruno, [09:43] Diego` |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas (resultado, conteúdo, resposta, tempo) | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de falhas permanentes, só ADMIN, auditado | TRANSCRICAO | `[09:18] Diego, [09:36] Sofia` |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Identificador único por notificação para dedup no cliente | TRANSCRICAO | `[09:25] Diego` |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Cliente valida autenticidade e integridade da notificação | TRANSCRICAO | `[09:19] Sofia` |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de credencial via API com 24h de transição | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Endereço https obrigatório; não seguro é recusado | TRANSCRICAO | `[09:23] Sofia` |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Identificação do cliente explícita na operação (não vem do JWT) | TRANSCRICAO | `[09:32] Larissa` |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Entrega < 10s no caminho feliz; latência mínima 2s | TRANSCRICAO | `[09:02] Marcos, [09:10] Larissa` |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Mudança de status nunca bloqueada por endpoint de cliente | TRANSCRICAO | `[09:04] Bruno` |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Consistência: status mudou ⇔ notificação registrada | TRANSCRICAO | `[09:06] Diego, [09:40] Bruno` |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Retentativas automáticas por ~15h antes de falha permanente | TRANSCRICAO | `[09:17] Diego` |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | At-least-once: duplicatas possíveis, documentadas com destaque | TRANSCRICAO | `[09:24] Diego, [09:26] Marcos` |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Credencial única por webhook; comprometimento contido | TRANSCRICAO | `[09:21] Sofia` |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB por notificação; acima disso falha explícita | TRANSCRICAO | `[09:23] Sofia, [09:24] Larissa` |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordem garantida só por pedido (não global), limitação documentada | TRANSCRICAO | `[09:13] Larissa, [09:14] Marcos` |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Reprocessamentos administrativos auditáveis | TRANSCRICAO | `[09:36] Sofia` |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso além do trimestre leva Atlas ao concorrente | TRANSCRICAO | `[09:00] Marcos, [09:47] Marcos` |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente vaza credencial (precedente real) | TRANSCRICAO | `[09:22] Diego, [09:21] Sofia` |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente não deduplica e processa notificação repetida | TRANSCRICAO | `[09:25] Sofia, [09:26] Marcos` |
| PRD-RISK-04 | docs/PRD.md | Risco | Endpoint do cliente indisponível por período longo (caso real de 2h) | TRANSCRICAO | `[09:16] Diego, [09:17] Diego` |
