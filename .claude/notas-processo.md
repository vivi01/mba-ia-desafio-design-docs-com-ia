# Notas de processo (log de trabalho)

Este arquivo NÃO faz parte da entrega em `docs/`. Ele alimenta o README final
(seções "Prompts customizados", "Iterações e ajustes" e "Workflow adotado").

---

## Sessão 1 — Contextualização (2026-07-19)

**Prompt do usuário:** "vamos começar pela contextualização"

**O que foi feito:** leitura integral de `TRANSCRICAO.md` (155 falas, 09:00–09:53)
e mapeamento do código nos pontos de integração listados no CLAUDE.md
(`order.service.ts`, `order.status.ts`, erros, `auth.middleware.ts`,
`error.middleware.ts`, logger Pino, `schema.prisma`, `routes/index.ts`,
`server.ts`, estrutura de `src/modules/`).

### Lista 1 — Decisões fechadas (com timestamp do martelo)

| # | Decisão | Martelo batido | Contexto/apoio |
| --- | --- | --- | --- |
| D1 | Webhooks **outbound only** (só saem da nossa plataforma) | `[09:02] Marcos` | `[09:03] Sofia` |
| D2 | **Padrão outbox no MySQL** — inserção na `webhook_outbox` dentro da mesma transação SQL do `changeStatus`; nada síncrono, nada de Redis | `[09:08] Larissa` ("Tá decidido então: outbox em MySQL") | `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Diego` |
| D3 | **Worker lê por polling a cada 2 segundos**; latência mínima de 2s aceita | `[09:10] Larissa` ("Vamos registrar isso como uma decisão") | `[09:09] Diego`, `[09:10] Marcos` |
| D4 | **Worker em processo separado** — entry-point nova `src/worker.ts` + script `npm run worker`; mesmo banco/mesma stack, nunca o mesmo processo | `[09:12] Larissa` ("Anotado") | `[09:11] Diego`, `[09:11] Larissa`, `[09:11] Bruno` |
| D5 | PrismaClient **separado por processo** no worker (mesma `DATABASE_URL`) | `[09:30] Bruno` | `[09:29] Diego` (pergunta) |
| D6 | **Ordering**: garantia só por `order_id` e enquanto single-worker; documentada como limitação conhecida, sem ordering global | `[09:13] Larissa` | `[09:12] Diego`, `[09:14] Marcos` |
| D7 | **Retry: 5 tentativas, backoff exponencial 1m/5m/30m/2h/12h** (~15h da primeira falha à última tentativa) | `[09:17] Larissa` ("Decidido") | `[09:15] Diego`, `[09:16] Diego`, `[09:17] Diego` |
| D8 | **DLQ em tabela separada `webhook_dead_letter`** (payload, motivo da falha, timestamp) | `[09:18] Diego` + `[09:19] Larissa` ("Anota também") | `[09:17] Larissa` (pergunta) |
| D9 | **Replay manual de DLQ via endpoint admin** `POST /admin/webhooks/dead-letter/:id/replay` (recoloca na outbox como pendente) | `[09:19] Larissa` | `[09:18] Diego` |
| D10 | **HMAC-SHA256 sobre o corpo do request, secret única por endpoint, rotação com grace period de 24h** | `[09:22] Sofia` ("Decidido") | `[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Diego` |
| D11 | **TLS obrigatório** — URL só `https`, recusa `http` com erro de validação no schema Zod | `[09:23] Sofia` | — |
| D12 | **Limite de payload 64KB, erro (não trunca) se ultrapassar** — registrado como requisito não funcional, não ADR | `[09:24] Larissa` | `[09:23] Sofia`, `[09:24] Diego` |
| D13 | **At-least-once com `X-Event-Id`** (UUID gerado na entrada da outbox) para dedup do lado do cliente | `[09:26] Larissa` ("Decisão") | `[09:24] Diego`, `[09:25] Diego` |
| D14 | **Módulo `src/modules/webhooks`** seguindo padrão controller/service/repository/routes/schemas; lógica do worker em arquivo do módulo (`webhook.worker.ts` ou `webhook.processor.ts`) | `[09:28] Diego` ("Beleza") | `[09:27] Bruno`, `[09:28] Bruno` |
| D15 | **Códigos de erro com prefixo `WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), seguindo padrão `AppError` | `[09:29] Larissa` | `[09:28] Bruno` |
| D16 | **Reuso máximo do existente**: AppError, Pino, error middleware, padrão de módulos, schemas Zod, códigos de erro | `[09:30] Larissa` ("Decisão: reuso máximo") | `[09:29] Bruno` |
| D17 | `customer_id` **não vem do JWT** — passado no body ou path; endpoint autenticado normal | `[09:32] Larissa` | `[09:32] Bruno`, `[09:32] Marcos` |
| D18 | **Filtro de eventos aplicado na inserção da outbox** (se nenhum webhook do customer quer o status, nem insere) | `[09:34] Diego` ("Concordo") | `[09:33] Marcos`, `[09:34] Bruno` |
| D19 | **Replay de DLQ exige role ADMIN** (reuso do `requireRole`) + **log de auditoria de quem fez** | `[09:36] Larissa` ("Decidido") | `[09:36] Sofia` |
| D20 | CRUD de configuração de webhook: **qualquer role autenticada** (por enquanto) | `[09:37] Sofia` | `[09:36] Marcos` |
| D21 | **Integração no `changeStatus`**: inserção na `webhook_outbox` na mesma transação; falha na outbox ⇒ rollback; via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o tx client | `[09:41] Diego` ("Boa, função pura recebendo o tx") | `[09:40] Bruno`, `[09:41] Bruno` |
| D22 | **Timeout do HTTP call do worker: 10 segundos** — sem resposta em 10s = falha, marca pra retry | `[09:42] Diego` + `[09:42] Sofia` ("Bom") | — |
| D23 | **Payload JSON enxuto**: event_id, event_type `order.status_changed`, timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, campos básicos (ex.: total_cents); **sem items** (cliente busca `GET /orders/:id`) | `[09:43] Diego` + `[09:44] Bruno` | — |
| D24 | **Headers do request**: `X-Event-Id`, `X-Signature`, `X-Timestamp` (detecção de replay attack), `X-Webhook-Id`, `Content-Type: application/json` | `[09:45] Diego` ("Boa, X-Webhook-Id também") | `[09:44] Diego`, `[09:44] Sofia` |
| D25 | **Prazo: 3 sprints**, incluindo revisão de segurança da Sofia (≥ 2 dias úteis para revisar HMAC e geração de secret) | `[09:47] Larissa` ("Combinado") | `[09:46] Larissa`, `[09:46] Sofia` |
| D26 | **id da outbox: UUID**, seguindo padrão do projeto | `[09:51] Larissa` | `[09:51] Diego` |
| D27 | **Payload snapshot renderizado na inserção** (evento reflete o estado do momento da mudança de status) | `[09:52] Bruno` ("Beleza, snapshot. Decidido") | `[09:52] Larissa`, `[09:52] Diego` |

Resumo confirmado por todos em `[09:48] Larissa` (fechado por `[09:49] Diego/Bruno/Sofia`).

### Lista 2 — Requisitos explícitos

**Funcionais:**

| # | Requisito | Fonte |
| --- | --- | --- |
| RF1 | Notificar clientes B2B em tempo real quando o status de pedidos muda (demanda de Atlas Comercial, MaxDistribuição e Nova Cargo, que hoje fazem polling no `GET /orders`) | `[09:00] Marcos` |
| RF2 | Cadastrar webhook: `POST` com url + lista de status desejados; **secret gerada pela plataforma e devolvida na criação** | `[09:31] Marcos` |
| RF3 | `PATCH` para editar, `DELETE` para remover, `GET` para listar webhooks de um customer | `[09:33] Bruno` |
| RF4 | Filtro de eventos por endpoint: lista de status que o webhook quer ouvir | `[09:33] Marcos` |
| RF5 | Histórico de entregas: `GET /webhooks/:id/deliveries` — últimos 100 envios com sucesso/falha, payload, response, tempo de resposta | `[09:34] Marcos` |
| RF6 | Replay manual de DLQ: `POST /admin/webhooks/dead-letter/:id/replay` (role ADMIN, auditado) | `[09:18] Diego`, `[09:36] Sofia` |
| RF7 | Rotação de secret via API, antiga válida por 24h em paralelo | `[09:21] Sofia` |
| RF8 | Evento gerado dentro da transação do `changeStatus` (outbox); retry com backoff; DLQ após esgotar | `[09:40] Bruno`, `[09:15] Diego` |
| RF9 | Entrega at-least-once com `X-Event-Id` para dedup no cliente | `[09:25] Diego` |
| RF10 | Tabela de configuração: url + secret + customer_id + estado ativo | `[09:21] Bruno` (confirmado por Sofia) |

**Não funcionais:**

| # | Requisito | Fonte |
| --- | --- | --- |
| RNF1 | Latência de notificação **< 10 segundos** ("tempo real" para os clientes) | `[09:02] Marcos` |
| RNF2 | Mudança de status não pode ser travada/bloqueada pelo envio HTTP (nada síncrono) | `[09:04] Bruno` |
| RNF3 | Consistência: status mudou ⇔ evento registrado (transação atômica; rollback conjunto) | `[09:06] Diego`, `[09:40] Bruno` |
| RNF4 | URL de webhook obrigatoriamente `https` (TLS) | `[09:23] Sofia` |
| RNF5 | Payload máximo 64KB; acima disso, erro | `[09:24] Larissa` |
| RNF6 | Timeout de 10s por chamada HTTP do worker | `[09:42] Diego` |
| RNF7 | Ordering garantida só por order_id, enquanto single-worker (limitação documentada) | `[09:13] Larissa` |
| RNF8 | Secret única por endpoint (nunca global), rotacionável | `[09:21] Sofia` |
| RNF9 | Auditoria: log de quem executou replay de DLQ | `[09:36] Sofia` |
| RNF10 | Outbox indexada por status (pendente/processando/falhou/entregue) e created_at; worker lê pendentes em batch pequeno | `[09:08] Diego` |

### Lista 3 — Itens descartados ou adiados (protege "Fora de escopo" e "Questões em aberto")

| # | Item | Situação | Fonte |
| --- | --- | --- | --- |
| X1 | Webhooks inbound (cliente → plataforma) | Fora de escopo | `[09:02] Marcos` |
| X2 | Disparo síncrono dentro do service de orders | Descartado ("fora de questão") | `[09:04] Bruno`, `[09:06] Diego` |
| X3 | Redis Streams / fila dedicada | Descartado (mais infra; overengineering p/ time pequeno) | `[09:07] Larissa`, `[09:07] Diego` |
| X4 | Trigger de banco para notificar o worker | Descartado (MySQL sem NOTIFY/LISTEN; improviso esquisito) | `[09:09] Diego` |
| X5 | Arquivamento de linhas entregues da outbox (~30 dias) | Fora do escopo da feature | `[09:08] Diego` |
| X6 | Múltiplos workers em paralelo (particionar por order_id / lock pessimista) | Adiado ("problema do futuro") | `[09:13] Diego` |
| X7 | Retry indefinido com backoff | Descartado (evento pendurado pra sempre) | `[09:15] Diego` |
| X8 | 3 tentativas de retry | Descartado (pouco; não cobre indisponibilidade de 2h) | `[09:16] Diego` |
| X9 | Marcar "failed" na própria outbox em vez de tabela DLQ separada | Descartado (leitura da outbox mais limpa; evidence p/ debug) | `[09:18] Diego` |
| X10 | Secret global da plataforma | Descartado ("se vaza uma, vaza tudo") | `[09:21] Sofia` |
| X11 | Truncar payload acima do limite (vs. erro) | Descartado | `[09:23] Sofia`, `[09:24] Larissa` |
| X12 | Garantia exactly-once | Descartado (coordenação dos dois lados; complexidade) | `[09:25] Diego` |
| X13 | Email de aviso ao cliente quando webhook falha repetidamente | Adiado ("próxima fase, depois de medir impacto") | `[09:37] Larissa` |
| X14 | Rate limiting de envio para o cliente | Em aberto ("observar e decidir depois") | `[09:39] Diego`, `[09:39] Larissa` |
| X15 | Dashboard visual para o cliente | Fora de escopo (projeto separado do frontend) | `[09:40] Larissa` |
| X16 | Endurecer roles do CRUD de configuração | Adiado ("mais pra frente") | `[09:37] Sofia` |

**Questões em aberto candidatas ao RFC:** X6 (escala multi-worker/ordering), X14 (rate limiting), X13 (notificação por email — adiada), X16 (endurecimento de roles), X5 (política de arquivamento da outbox).

### Mapa do código (verificado — todos os caminhos existem)

| Ponto de integração | Arquivo | Observação relevante |
| --- | --- | --- |
| Transação `changeStatus` (ponto de inserção do outbox) | `src/modules/orders/order.service.ts` | `prisma.$transaction` já faz: update `orders` + insert `order_status_history` + débito/reposição de estoque; `TxClient = Prisma.TransactionClient` — assinatura compatível com `publishWebhookEvent(tx, ...)` |
| Máquina de estados | `src/modules/orders/order.status.ts` | `OrderStatus`: PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED; `canTransition`, `isTerminal` |
| Erros base | `src/shared/errors/app-error.ts` | `AppError(message, statusCode, errorCode, details)` |
| Erros derivados | `src/shared/errors/http-errors.ts` | Padrão de códigos: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION` etc. — modelo para `WEBHOOK_*` |
| Auth | `src/middlewares/auth.middleware.ts` | `authenticate` (JWT Bearer) + `requireRole(...roles)`; roles `ADMIN` \| `OPERATOR` |
| Error middleware | `src/middlewares/error.middleware.ts` | Trata `AppError`, `ZodError`, Prisma P2002/P2025; formato `{ error: { code, message, details? } }` |
| Logger | `src/shared/logger/index.ts` | Pino com redact (authorization, password, token) — candidato natural a redigir secret de webhook |
| Schema Prisma | `prisma/schema.prisma` | MySQL; ids `@default(uuid()) @db.Char(36)`; `@@map` snake_case; `@@index` — padrão para `webhook_outbox`/`webhook_dead_letter` |
| Registro de rotas | `src/routes/index.ts` | `buildApiRouter(controllers)` com `router.use('/x', buildXRouter(...))` — webhooks entra aqui |
| Padrão de módulo | `src/modules/orders/` | controller/service/repository/routes/schemas — modelo para `src/modules/webhooks/` |
| Entry-point HTTP | `src/server.ts` | Modelo para a entry `src/worker.ts` (a criar) com graceful shutdown |

**Observação de consistência (código × transcrição):** Bruno descreve em `[09:04]`
a transação como "atualiza orders, insere na order_status_history, decrementa
stock_quantity" — confere exatamente com `changeStatus` em `order.service.ts`
(linhas 126–179). A fala `[09:28]` de Bruno sobre `AppError` /
`InsufficientStockError` / `InvalidStatusTransitionError` confere com
`src/shared/errors/http-errors.ts`.

### Mapeamento das 6 decisões principais → ADRs planejados

1. Outbox no MySQL → D2 (alternativas: X2 síncrono, X3 Redis Streams)
2. Retry backoff + DLQ → D7, D8 (alternativas: X7 retry infinito, X8 3 tentativas, X9 flag na outbox)
3. HMAC-SHA256 secret por endpoint → D10 (alternativa: X10 secret global)
4. At-least-once com X-Event-Id → D13 (alternativa: X12 exactly-once)
5. Worker separado em polling → D3, D4 (alternativas: X4 trigger de banco, worker no processo da API)
6. Reuso de padrões existentes → D14, D15, D16

Candidatos a ADR extra (se couber em ≤ 8): snapshot do payload na inserção (D27) e/ou filtro na inserção da outbox (D18).

### Próximos passos (ordem do workflow)

2. ADRs (esqueleto das decisões) → 3. RFC → 4. FDD → 5. PRD → 6. varredura do Tracker (alimentado desde o passo 2) → 7. README do processo → 8. checklist de aceite.

---

## Sessão 2 — ADRs (2026-07-19)

**Prompt do usuário:** "pode começar" (após a contextualização).

**O que foi produzido:** 7 ADRs em `docs/adrs/` (formato MADR, template
`.claude/templates/adr.md`), cobrindo as 6 decisões principais + 1 extra:

1. `ADR-001-outbox-no-mysql.md` — decisão principal 1 (inclui filtro na inserção e id UUID)
2. `ADR-002-worker-separado-polling.md` — decisão principal 5
3. `ADR-003-retry-backoff-dlq.md` — decisão principal 2
4. `ADR-004-hmac-sha256-secret-por-endpoint.md` — decisão principal 3
5. `ADR-005-at-least-once-x-event-id.md` — decisão principal 4
6. `ADR-006-reuso-padroes-existentes.md` — decisão principal 6 (referencia 7 caminhos reais do código)
7. `ADR-007-snapshot-payload-na-insercao.md` — extra (D27 da lista de decisões)

`docs/TRACKER.md` alimentado na mesma sessão com 43 linhas (IDs ADR-NNN,
ADR-NNN-DEC-NN, ADR-NNN-ALT-NN, ADR-NNN-REST-NN, ADR-NNN-INT-NN);
9 linhas com Fonte = CODIGO (caminhos verificados), 34 com TRANSCRICAO.
`docs/adrs/README.md` atualizado com índice na convenção do desafio
(prevalece sobre a sugestão antiga `0001-titulo.md`).

**Escolhas editoriais registradas:**

- O filtro de eventos na inserção (D18) e o id UUID (D26) foram absorvidos
  pelo ADR-001 (modelagem da outbox) em vez de virarem ADRs próprios, para
  manter 1 decisão arquitetural por arquivo sem inflar o conjunto.
- Caminhos futuros (`src/worker.ts`, `src/modules/webhooks/`) aparecem nos
  ADRs sempre marcados com "(a criar)", porque são citações literais da
  reunião; a regra de restrição reserva caminhos futuros ao FDD, então a
  marcação explícita evita que pareçam arquivos existentes.
- TLS/https e limite de 64KB ficaram como requisitos (não ADR), seguindo a
  própria classificação da reunião (`[09:23] Sofia`, `[09:24] Larissa`).

**Iteração/correção nesta sessão (candidata ao README):** na primeira escrita
do tracker, a linha ADR-003-ALT-03 saiu com a Fonte grafada `TRANSSCRICAO`
(typo) — detectado em revisão imediata e corrigido via Edit. Reforça a
necessidade da rotina de verificação por Grep antes do push final.

---

## Sessão 3 — RFC (2026-07-19)

**Prompt do usuário:** "pode seguir pro RFC".

**O que foi produzido:** `docs/RFC.md` (template `.claude/templates/rfc.md`),
~3 páginas, altura de arquitetura:

- TL;DR de 5 linhas; contexto com as duas forças técnicas (não acoplar
  disponibilidade do cliente à transação; não perder eventos).
- Proposta em 4 blocos, cada um apontando para o ADR correspondente +
  diagrama mermaid do fluxo outbox → worker → retry → DLQ → replay.
- Seção "O que muda no sistema existente" em 1 parágrafo (guard-rail contra
  descer ao nível do FDD — contratos/payloads/matriz de erros ficaram
  explicitamente delegados ao FDD).
- 6 alternativas reais descartadas em tabela, com trade-off e link do ADR.
- 5 questões em aberto, todas com timestamp (rate limiting, multi-worker,
  email de alerta, arquivamento da outbox, endurecimento de roles).
- Impacto/riscos em nível de arquitetura; links para os 7 ADRs.
- Revisores = participantes reais da reunião, conforme regra de fronteiras.

Tracker: +19 linhas (RFC-PROP-01..08, RFC-ALT-01..06, RFC-OPEN-01..05);
1 com Fonte = CODIGO (`order.service.ts`), 18 com TRANSCRICAO.

**Cuidados de fronteira aplicados (candidatos ao README):**

- O RFC não repete payload de exemplo, matriz de erros nem passo a passo —
  apenas nomeia os blocos e delega ao FDD; alternativas resumidas em 1 linha
  cada porque o registro completo é dos ADRs.
- Caminhos futuros de arquivo não aparecem no RFC (regra: só no FDD marcados
  "a criar"); a proposta fala em "novo módulo no padrão existente" sem citar
  path a criar, e só referencia arquivos existentes
  (`order.service.ts`, `routes/index.ts`).

**Ajuste do autor:** o campo Autor do RFC foi corrigido manualmente pela autora
para "Viviane Pereira" (a IA havia inferido o sobrenome errado a partir do
email). Usar "Viviane Pereira" nos demais documentos.

---

## Sessão 4 — FDD (2026-07-19)

**Prompt do usuário:** "pode seguir pro FDD".

**Preparação:** leitura adicional de `src/shared/http/response.ts` (formato
paginado `{data, pagination}`) e `src/modules/orders/order.routes.ts`
(padrão `authenticate` + `validate` por rota) para que os contratos do FDD
sigam exatamente o formato do projeto.

**O que foi produzido:** `docs/FDD.md` completo com todas as seções do
template, incluindo:

- Modelo de dados com 4 tabelas (`webhook_endpoints`, `webhook_outbox`,
  `webhook_dead_letter`, `webhook_deliveries`), todas marcadas "(a criar)".
- 4 fluxos detalhados (outbox na transação / worker / retry / DLQ).
- 8 contratos: 7 endpoints HTTP com request + response + status codes, mais o
  contrato outbound do webhook entregue ao cliente (checklist pede ≥ 4).
- Matriz de erros `WEBHOOK_*` com coluna "Origem" separando códigos citados
  na reunião ([09:28]) dos derivados de regras decididas (64KB, timeout 10s).
- Observabilidade com métricas + logs + tracing/correlação (os três,
  obrigatórios); tracing honesto: sem APM no projeto, correlação por
  req.id → eventId → X-Event-Id.
- Integração com o sistema existente: 11 caminhos reais (checklist pede ≥ 4)
  + lista separada de arquivos a criar.
- 9 critérios de aceite técnicos verificáveis; riscos com mitigação.

Tracker: +32 linhas (FDD-FLUXO-01..06, FDD-CONTRATO-01..07, FDD-ERRO-01..04,
FDD-INT-01..11, FDD-OBS-01..04); 11 com Fonte = CODIGO.

**Decisões de interpretação registradas (candidatas ao README):**

- **Ambiguidade "5 tentativas" vs 5 intervalos:** a reunião fala em "5
  tentativas" e lista 5 intervalos (1m/5m/30m/2h/12h) com "quase 15 horas
  entre primeira falha e última tentativa". A única leitura aritmeticamente
  consistente é envio inicial + 5 retentativas (até 6 envios). O FDD adota
  essa semântica com uma "Nota de interpretação" explícita no fluxo de retry,
  em vez de esconder a ambiguidade.
- Detalhes não fechados na reunião foram marcados como "a confirmar na
  revisão de segurança da Sofia": encoding da assinatura (hex) e dupla
  assinatura durante o grace period de rotação.
- Nomes de rota não ditos literalmente (ex.: `/webhooks/:id/rotate-secret`)
  seguem o padrão REST do módulo orders; os ditos literalmente
  (`GET /webhooks/:id/deliveries`, `POST /admin/webhooks/dead-letter/:id/replay`)
  foram mantidos exatamente como na transcrição.

---

## Sessão 5 — Revisão dos ADRs apontada pela autora (2026-07-19)

**Prompt do usuário (resumo):** antes do PRD, revisar ADR-003 (a justificativa
de descarte de "marcar failed na própria outbox" não parecia boa) e ADR-004
(o "por que foi descartada" da secret global ficou confuso).

**Diagnóstico (candidato forte ao README, seção "Iterações e ajustes"):**
a IA tinha cometido o mesmo vício nos dois pontos — justificar o descarte da
alternativa citando os *benefícios da opção escolhida* (frase do Diego sobre a
tabela separada; frase solta da Sofia) em vez de explicar o *problema da
alternativa em si*.

**Correções aplicadas:**

- **ADR-003 / "failed" na própria outbox:** reescrito invertendo o argumento
  do Diego [09:18]: falhas permanentes se acumulariam na tabela que o worker
  consulta a cada 2s (leitura suja, linhas mortas crescendo) e a evidência de
  falha ficaria sem lugar dedicado para debug/reprocessamento.
- **ADR-004 / secret global:** reescrito em cadeia causal: vazamento é cenário
  realista (precedente do cliente que vazou secret em log [09:22] Diego) →
  com secret global, o descuido de UM cliente compromete a autenticidade dos
  webhooks de TODOS ("se vaza uma, vaza tudo" [09:21] Sofia) → resposta ao
  incidente exigiria rotação coordenada da base inteira; por endpoint, uma
  rotação pontual resolve.

**Varredura do mesmo vício nos demais ADRs:** ADR-001 (síncrono, Redis),
ADR-002 (trigger, mesmo processo, multi-worker), ADR-003 (retry indefinido,
3 tentativas), ADR-005 (exactly-once) e ADR-007 (renderizar no envio) já
explicavam o problema da alternativa — sem alteração. ADR-006 usa a citação
direta "não vamos botar nada novo" com o contexto de padrões concorrentes na
seção Contexto — mantido.

---

## Sessão 6 — Segunda revisão dos ADRs pedida pela autora (2026-07-19)

**Prompt do usuário (resumo):** varrer TODOS os ADRs procurando ambiguidades e
o mesmo defeito do ADR-003/004; reavaliar ADRs com uma única alternativa e
verificar se a reunião discutiu opções que ficaram de fora; tudo claro, sem
ambiguidade, com o motivo real — sem criar nada fora da transcrição/código.

**Problemas encontrados e corrigidos (candidatos ao README):**

1. **ADR-001 — duas alternativas REAIS da reunião estavam faltando:**
   - Filtrar eventos no envio vs. na inserção (`[09:34] Diego` perguntou;
     `[09:34] Bruno` decidiu na inserção, "economiza linha na tabela");
   - id auto incremental vs. UUID (`[09:51] Diego` perguntou; `[09:51]
     Larissa` fechou UUID pelo padrão do projeto).
   Ambas adicionadas com o problema real da opção descartada; +2 linhas no
   tracker (ADR-001-ALT-03/04).
2. **ADR-001 — justificativa do Redis** tinha extrapolação minha ("perderia a
   atomicidade") apresentada como motivo do descarte; reescrita ancorada na
   garantia descrita por Diego em [09:06].
3. **ADR-003 — mesma ambiguidade "5 tentativas × 5 intervalos" do FDD**
   existia no ADR sem nota; adicionada a "Semântica adotada (desambiguação)"
   com a mesma leitura do FDD (envio inicial + 5 retentativas).
4. **ADR-005 — risco de falsa atribuição:** meu exemplo ilustrativo de
   duplicata (worker falha ao marcar entregue) estava colado na citação de
   Diego como se fosse fala dele; agora a citação é literal e o exemplo está
   marcado como "cenário ilustrativo, não citado na reunião". Justificativa
   do descarte de exactly-once expandida com o trade-off real.
5. **ADR-006 — alternativa vaga:** descrição e descarte reescritos (segundo
   padrão concorrente na codebase; custo duplicado sem ganho), mantendo as
   citações de [09:29].
6. **ADR-004/005/007 — uma única alternativa:** confirmado na transcrição que
   a reunião só debateu UMA alternativa nesses casos (escopo da secret;
   exactly-once; renderizar no envio). Em vez de inventar alternativas
   "plausíveis" (vetado pelo prompt da autora), cada ADR ganhou nota
   explícita dizendo que nenhuma outra opção foi discutida na reunião.
7. **ADR-002 — aprovado sem mudanças** na varredura (3 alternativas reais,
   todas com o problema da alternativa explicado).

---

## Sessão 7 — Auditoria de coerência ADRs + RFC + FDD (2026-07-19)

**Prompt do usuário (resumo):** revisar TODO o conteúdo de ADRs, RFC e FDD e
garantir coerência com a transcrição e o código, sem ambiguidade.

**Método:** releitura dos 9 documentos no estado atual do disco, conferência
manual de cada citação/afirmação contra a transcrição (em contexto na íntegra)
e o código já mapeado, e duas verificações automatizadas em Bash:

1. extração de todas as citações `[hh:mm] Falante` de `docs/` e grep literal
   contra `TRANSCRICAO.md` → **82 citações únicas, 0 não encontradas**;
2. extração de todos os caminhos `src/`, `prisma/`, `tests/` citados em
   `docs/` e teste de existência → únicos inexistentes são os 3 futuros
   (`src/worker.ts`, `src/modules/webhooks`, `webhook.controller.ts`), todos
   marcados "(a criar)" no texto (tracker ADR-002 ganhou a marcação também).

**Problemas encontrados e corrigidos (candidatos ao README):**

1. **Inconsistência numérica RFC × FDD (a mais séria):** o RFC dizia "duas
   tabelas novas" em dois lugares; o FDD define QUATRO (configuração, outbox,
   DLQ, histórico de entregas). O próprio FDD dizia "três + uma" no modelo de
   dados e "duas + duas auxiliares" em Dependências. Unificado para "quatro
   tabelas novas" nos 4 pontos.
2. **Nomenclatura do retry desalinhada:** RFC (texto, TL;DR e diagrama
   mermaid) falava "5 tentativas" enquanto ADR-003/FDD adotam a semântica
   desambiguada "envio inicial + 5 retentativas". RFC e linha ADR-003 do
   tracker alinhados.
3. **ADR-004:** o nome `X-Signature` era citado como definido em [09:20],
   mas a Sofia disse "um header TIPO X-Signature" (indicativo); quem confirma
   o nome é a lista de headers do Diego em [09:44]. Citação corrigida.

**Verificado e aprovado sem mudança:** `npm run dev`/`npm run worker` (dev
existe no package.json; worker marcado a criar), rota real
`PATCH /orders/:id/status` (order.routes.ts), formato `ORD-NNNNNN`
(reserveOrderNumber), enum de estados da outbox (FDD usa PENDING/…/DELIVERED
como definição técnica dos "pendente/processando/falhou/entregue" de [09:08]),
todas as citações dos ADRs 001–003, 005–007 e do FDD.

---

## Sessão 8 — PRD (2026-07-19)

**Prompt do usuário:** "pode começar o PRD".

**O que foi produzido:** `docs/PRD.md` (template `.claude/templates/prd.md`),
altura de produto — consolidação de alto nível com RFC/FDD/ADRs já prontos:

- Todas as 12 seções obrigatórias do checklist.
- 12 requisitos funcionais e 9 não funcionais, todos com timestamp
  (checklist pede ≥ 8 RFs).
- 3 objetivos com métrica e meta quantitativa (< 10s; 3 sprints/fim de
  novembro; 100% dos eventos) — checklist pede ≥ 1.
- Fora de escopo com 6 itens explicitamente descartados/adiados, com
  timestamp (checklist pede ≥ 2).
- 4 riscos com probabilidade + impacto + mitigação (checklist pede ≥ 2).
- Cenários de uso com os 3 clientes nomeados (Atlas, MaxDistribuição,
  Nova Cargo) e o cenário admin de reprocessamento.

**Guarda de fronteira aplicada:** o PRD não cita tabela, endpoint, header nem
nome de coluna. Vocabulário traduzido para produto: "credencial" (não
"secret HMAC"), "identificador único" (não "X-Event-Id"), "registro
transacional interno" (não "webhook_outbox"); a única menção a HMAC-SHA256 é
a linha da decisão apontando para o ADR-004, como a regra de fronteiras
permite. "Decisões e trade-offs" = 1 linha por decisão + link do ADR.

Tracker: +34 linhas (PRD-OBJ-01..03, PRD-ESC-01..06, PRD-FR-01..12,
PRD-NFR-01..09, PRD-RISK-01..04). Verificação automatizada re-rodada após o
PRD: 89 citações únicas em docs/, 0 não encontradas.

---

## Sessão 9 — Varredura final do tracker (etapa 6 do workflow, 2026-07-19)

**Prompt do usuário:** "pode seguir o próximo passo do workflow após o PRD".

**Rotina de verificação de `rastreabilidade.md` executada (via Bash):**

1. **IDs duplicados:** 0 em 130 linhas.
2. **Fontes:** 109 TRANSCRICAO (84% ≥ meta de 70%) + 21 CODIGO (≥ meta de 5).
   Todas as 89 citações únicas `[hh:mm] Nome` de docs/ (incluindo o tracker)
   conferidas por grep literal contra TRANSCRICAO.md — 0 não encontradas.
3. **Caminhos:** todos os caminhos da coluna Documento existem; todos os
   caminhos de código citados existem, exceto os 3 futuros, sempre marcados
   "(a criar)".
4. **Varredura de cobertura:** números concretos dos documentos (10s, 2s,
   5 retentativas, 1m/5m/30m/2h/12h, ~15h, 24h, 64KB, 100 entregas,
   3 sprints, 2 dias úteis, ~30 dias, "50 pedidos", "3 falhas seguidas")
   têm linha no tracker ou estão dentro de item rastreado (RFC-OPEN-01/03
   cobrem os exemplos numéricos das questões em aberto).

Resumo por documento: ADRs 45 linhas, RFC 19, FDD 32, PRD 34 (total 130).
Registrado o resultado da rotina no cabeçalho do próprio TRACKER.md.

**Próximo passo:** etapa 7 — README do processo (substitui o enunciado em
README.md; manter link/menção ao enunciado original).

---

## Sessão 10 — README do processo (etapa 7, 2026-07-19)

**Prompt do usuário:** "pode seguir com o README".

**O que foi feito:**

- Enunciado original preservado integralmente em `ENUNCIADO.md` (raiz),
  linkado no novo README junto com o repositório base — atende o "manter
  link/menção ao enunciado original".
- `README.md` substituído pela documentação do processo, com as 6 seções
  obrigatórias: Sobre o desafio; Ferramentas de IA utilizadas (Claude Code +
  configuração customizada do agente, com papel de cada); Workflow adotado
  (7 etapas + práticas transversais de tracker paralelo e verificação
  automatizada); Prompts customizados (3 em blocos de código — o de
  contextualização em 3 listas, o de revisão dirigida dos ADRs escrito pela
  autora, e o de auditoria cruzada); Iterações e ajustes (6 correções
  concretas, extraídas deste log); Como navegar a entrega (árvore + ordem de
  leitura + como auditar).
- Todo o conteúdo do README saiu deste log — nada reconstruído de memória.

**Restante:** etapa 8 — revisão final pelo checklist-aceite.md item por item.

---

## Sessão 11 — Revisão final pelo checklist (etapa 8, 2026-07-19)

**Prompt do usuário:** "pode executar a revisão final".

**Correção encontrada pelo próprio checklist:** `ENUNCIADO.md` tinha sido
criado na RAIZ, violando o item "git status limpo fora de docs/, README.md e
.claude/". Movido para `docs/ENUNCIADO.md`; links e árvore do README
ajustados. (Candidato ao README? já coberto — o README lista iterações
principais; esta fica registrada aqui.)

**Resultados, item por item:**

- **PRD:** 14 cabeçalhos = todas as 12 seções (Escopo com subseções);
  12 RFs (≥8); 3 objetivos quantitativos (≥1); 6 itens fora de escopo (≥2);
  4 riscos com P/I/M (≥2). ✓
- **RFC:** 8 seções ✓; 1.352 palavras ≈ 3 páginas (2–4) ✓; 6 alternativas
  com trade-off (≥2) ✓; 5 questões em aberto (≥2) ✓; 7 ADRs linkados (≥2) ✓;
  sem payload/matriz/passo a passo (fronteira com FDD) ✓.
- **FDD:** 12 seções obrigatórias + "Modelo de dados" extra ✓; 7 endpoints +
  contrato outbound com payloads e status codes (≥4) ✓; matriz WEBHOOK_ ✓;
  integração com 10+ caminhos reais (≥4) ✓; observabilidade com métricas,
  logs E tracing ✓.
- **ADRs:** 7 arquivos ADR-NNN-kebab (5–8) ✓; 5 seções MADR em cada
  (verificado por grep: 5/5 nos 7) ✓; cobre 6/6 decisões principais (≥5) ✓;
  ADR-001/002/004/006 referenciam código real (≥1) ✓.
- **Tracker:** formato ✓; 130 linhas, 0 IDs duplicados; 84% TRANSCRICAO
  (≥70%); 21 CODIGO (≥5); caminhos da coluna Documento existem ✓.
- **README:** 6 seções ✓; 2 ferramentas com papel (≥1) ✓; 3 prompts em
  blocos (≥2) ✓; 6 iterações concretas (≥2) ✓; link ao enunciado
  (docs/ENUNCIADO.md + repositório base) ✓.
- **Consistência geral:** 89 citações [hh:mm] Falante verificadas por grep,
  0 não encontradas (varridos README + docs, excluindo ENUNCIADO) ✓;
  únicos caminhos inexistentes = 3 futuros, sempre "(a criar)" ✓;
  itens descartados aparecem SÓ em Fora de escopo/Questões em aberto/
  Alternativas/Exclusões (grep por email/dashboard/rate limiting conferido) ✓;
  sem valores divergentes ("duas/três tabelas" e "3 tentativas" fora de
  contexto de alternativa: 0 ocorrências) ✓; git status limpo fora de
  docs/, README.md e .claude/ ✓ (src/, prisma/, tests/, TRANSCRICAO.md e
  configs intactos).

**Entrega completa.** Todas as 8 etapas do workflow executadas.
