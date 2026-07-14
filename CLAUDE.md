# CLAUDE.md — Desafio: Design Docs Gerados por IA

## O que é este repositório

Este é o desafio "Da Reunião ao Documento": transformar a transcrição de uma reunião técnica (`TRANSCRICAO.md`) e o código de um Order Management System (Node.js + TypeScript + Prisma/MySQL) em um **pacote de design docs** para a feature **Sistema de Webhooks de Notificação de Pedidos**.

**A entrega é 100% documental.** Nenhuma linha de código será escrita. O enunciado completo do desafio está preservado em `README.md` (será substituído ao final pelo README do processo — ver `.claude/rules/workflow.md`).

## Entregáveis

| Arquivo | Conteúdo |
| --- | --- |
| `docs/PRD.md` | Produto: problema, escopo, requisitos, métricas |
| `docs/RFC.md` | Arquitetura: proposta técnica, alternativas, questões em aberto (2–4 páginas) |
| `docs/FDD.md` | Implementação: fluxos, contratos, erros `WEBHOOK_*`, integração com o código |
| `docs/adrs/ADR-NNN-titulo-em-kebab-case.md` | 5 a 8 decisões arquiteturais (formato MADR) |
| `docs/TRACKER.md` | Tabela de rastreabilidade de cada item à transcrição ou ao código |
| `README.md` | Documentação do processo de produção (por último) |

## Regras (leitura obrigatória)

@.claude/rules/restricoes.md
@.claude/rules/fronteiras-documentos.md
@.claude/rules/rastreabilidade.md
@.claude/rules/workflow.md

Antes do push final, valide item por item com `.claude/rules/checklist-aceite.md`.
Templates com as seções obrigatórias de cada documento: `.claude/templates/`.

## Fontes da verdade (as únicas duas)

1. **`TRANSCRICAO.md`** — reunião de ~55 min, formato `[hh:mm] Nome: fala`, 155 falas. Participantes: Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança).
2. **O código existente** — mapa dos pontos de integração que os documentos devem referenciar:

| Componente | Caminho |
| --- | --- |
| Transação `changeStatus` (ponto de inserção do outbox) | `src/modules/orders/order.service.ts` |
| Máquina de estados do pedido | `src/modules/orders/order.status.ts` |
| Classes de erro (`AppError` e derivadas) | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| Middleware de autorização `requireRole` | `src/middlewares/auth.middleware.ts` |
| Error middleware centralizado | `src/middlewares/error.middleware.ts` |
| Logger Pino | `src/shared/logger/index.ts` |
| Schema do banco (novas tabelas do outbox referenciam este padrão) | `prisma/schema.prisma` |
| Registro modular de rotas | `src/routes/index.ts` |
| Padrão de módulo (controller/service/repository/routes/schemas) | `src/modules/orders/` |

## As 6 decisões principais da reunião

Os ADRs devem cobrir pelo menos 5 destas:

1. Padrão Outbox no MySQL (vs. Redis Streams/fila dedicada)
2. Política de retry com backoff e DLQ
3. Autenticação HMAC-SHA256 com secret por endpoint
4. Garantia at-least-once com header `X-Event-Id`
5. Worker em processo separado, lendo por polling
6. Reuso dos padrões existentes do projeto

## Idioma

Toda a documentação entregue é escrita em **português brasileiro**, no mesmo tom técnico da transcrição.
