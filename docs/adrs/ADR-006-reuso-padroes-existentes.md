# ADR-006: Reuso dos padrões existentes do projeto para o módulo de webhooks

## Status

Aceito — reunião de quinta-feira, 09:00 (`TRANSCRICAO.md`). Decidido em `[09:30]` por Larissa ("Decisão: reuso máximo do que já existe").

## Contexto

A codebase tem um padrão claro e consolidado: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas (`[09:27] Bruno`) — o exemplo de referência é `src/modules/orders/`. A infraestrutura transversal já cobre o que a feature precisa:

- Erros: classe base `AppError` (`src/shared/errors/app-error.ts`) e derivadas com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` (`src/shared/errors/http-errors.ts`) (`[09:28] Bruno`).
- Middleware de erro centralizado que já trata `AppError`, Zod e Prisma (`src/middlewares/error.middleware.ts`) (`[09:29] Bruno`).
- Logger Pino no projeto inteiro (`src/shared/logger/index.ts`) (`[09:29] Bruno`).
- Autorização por role com `requireRole` (`src/middlewares/auth.middleware.ts`) (`[09:36] Larissa`).
- Registro modular de rotas (`src/routes/index.ts`).

Introduzir estruturas ou bibliotecas novas para um único módulo criaria dois padrões concorrentes na mesma codebase.

## Decisão

O módulo de webhooks **reusa ao máximo o que já existe** (`[09:30] Larissa`):

- Novo domínio como módulo padrão em `src/modules/webhooks` (a criar), com controller, service, repository, routes e schemas, igual aos demais (`[09:27] Bruno`); a lógica de processamento do worker vive em arquivo do módulo, tipo `webhook.worker.ts` ou `webhook.processor.ts` (a criar) (`[09:28] Bruno`).
- Erros como derivadas de `AppError`, com códigos prefixados **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) (`[09:28] Bruno`, `[09:29] Larissa`).
- Logger Pino, sem nada novo; o error middleware pega os erros do módulo "sem precisar mudar nada" (`[09:29] Bruno`).
- Validação com schemas Zod, seguindo o padrão dos módulos existentes (`[09:30] Larissa`).
- Replay de DLQ protegido com o `requireRole` já existente, exigindo role ADMIN (`[09:36] Larissa`).
- Worker na mesma stack (Node + Prisma + MySQL), como processo separado (`[09:11] Diego`; ver [ADR-002](ADR-002-worker-separado-polling.md)).

## Alternativas Consideradas

### Introduzir bibliotecas ou estruturas novas para o módulo

- **Descrição:** dar ao módulo de webhooks ferramentas ou organização próprias — logger novo, estrutura de pastas fora do padrão de módulos, classes de erro fora da hierarquia `AppError`.
- **Por que foi descartada:** criaria um segundo padrão convivendo com o atual na mesma codebase, sem ganho que justifique o custo: o Pino "já tá no projeto inteiro" e o middleware de erro centralizado já trata `AppError`, Zod e Prisma — os erros novos seriam capturados "sem precisar mudar nada" (`[09:29] Bruno`). Qualquer ferramenta nova traria configuração, aprendizado e manutenção duplicados para resolver problemas que a infraestrutura existente já resolve. (A variante mais concreta dessa alternativa — infraestrutura de fila nova, Redis Streams — foi descartada como overengineering no [ADR-001](ADR-001-outbox-no-mysql.md), `[09:07] Diego`.)

## Consequências

**Positivas:**

- O middleware de erro existente trata os novos erros `WEBHOOK_*` sem alteração (`[09:29] Bruno`).
- Consistência da codebase: qualquer pessoa do time navega o módulo novo com o mesmo mapa mental dos demais.
- Menos código e menos decisões novas — o esforço fica no domínio (outbox, worker, HMAC), não em infraestrutura.

**Negativas:**

- O padrão completo de módulo (controller/service/repository/routes/schemas) impõe boilerplate mesmo para endpoints simples.
- A feature herda as restrições da stack atual — MySQL sem notificação nativa a processo externo, por exemplo, é o que força o polling ([ADR-002](ADR-002-worker-separado-polling.md)).
- O error middleware centralizado é do pipeline HTTP do Express; o worker, por rodar em processo separado, precisa do próprio tratamento de erros no loop, ainda que reuse `AppError` e Pino.

## Referências

- `TRANSCRICAO.md`: `[09:27]`–`[09:30]`, `[09:36]`
- Código: `src/modules/orders/` (padrão de módulo), `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-polling.md), [ADR-003](ADR-003-retry-backoff-dlq.md)
