# ADR-003: Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

## Status

Aceito — reunião de quinta-feira, 09:00 (`TRANSCRICAO.md`). Decidido em `[09:17]` por Larissa ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h") e em `[09:18]`–`[09:19]` por Diego e Larissa (DLQ em tabela separada + replay manual).

## Contexto

Endpoints de clientes ficam fora do ar — houve caso real de cliente com indisponibilidade de duas horas em manutenção planejada (`[09:16] Diego`). O sistema precisa reagir a falhas de entrega sem perder o evento e sem deixá-lo pendurado para sempre: retry indefinido "traz o problema de evento ficar pendurado pra sempre se o cliente sumiu" (`[09:15] Diego`). Também conta como falha o cliente que não responde dentro do timeout de 10 segundos do HTTP call (`[09:42] Diego`).

## Decisão

Adotaremos **retry com backoff exponencial em 5 tentativas, com progressão 1m / 5m / 30m / 2h / 12h** — quase 15 horas entre a primeira falha e a última tentativa (`[09:17] Diego`, `[09:17] Larissa`).

**Semântica adotada (desambiguação):** a reunião fixou "5 tentativas" e listou 5 intervalos; a única leitura em que os 5 intervalos somam as "quase 15 horas entre primeira falha e última tentativa" ditas em `[09:17]` é contá-los **entre tentativas consecutivas, após a falha do envio inicial** — ou seja, envio inicial + 5 retentativas (até 6 envios no total). É essa a semântica deste ADR; o detalhamento operacional (estados, `nextAttemptAt`) está no [FDD](../FDD.md).

Esgotadas as tentativas, o evento é considerado **falha permanente e movido para uma tabela separada `webhook_dead_letter`**, com a payload, o motivo da falha e o timestamp (`[09:18] Diego`).

O reprocessamento é **manual, via endpoint admin** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente (`[09:18] Diego`). O endpoint exige role ADMIN e registra em log quem executou o replay, para auditoria (`[09:36] Sofia`, `[09:36] Larissa`).

## Alternativas Consideradas

### Retry indefinido com backoff

- **Descrição:** nunca desistir da entrega, apenas espaçando as tentativas (`[09:15] Diego`).
- **Por que foi descartada:** evento fica pendurado para sempre se o cliente sumiu (`[09:15] Diego`).

### 3 tentativas (mais agressivo)

- **Descrição:** teto menor de tentativas, proposto por Bruno (`[09:16] Bruno`).
- **Por que foi descartada:** "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria" — não cobre o caso real de 2 h de manutenção (`[09:16] Diego`).

### Marcar "failed" na própria outbox (sem tabela DLQ)

- **Descrição:** em vez de tabela separada, marcar o evento como `failed` na própria `webhook_outbox` (`[09:17] Larissa`).
- **Por que foi descartada:** falhas permanentes se acumulariam na mesma tabela que o worker consulta a cada 2 segundos — toda busca por eventos pendentes teria que conviver com (e filtrar) linhas mortas, que só crescem com o tempo, sujando a leitura da outbox principal. Além disso, a evidência da falha (payload, motivo, timestamp) ficaria misturada ao estado operacional da fila, sem um lugar dedicado para debug e reprocessamento — os dois usos que motivaram a tabela separada (`[09:18] Diego`: "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento").

## Consequências

**Positivas:**

- A janela de ~15 horas cobre indisponibilidades reais dos clientes, incluindo manutenções longas (`[09:16] Diego`); "se um cliente meu cair por 15 horas, ele já tá com problema sério dele" (`[09:17] Marcos`).
- Outbox principal permanece enxuta; a DLQ preserva payload, motivo e timestamp como evidência para debug (`[09:18] Diego`).
- Nenhum evento é descartado silenciosamente: falha permanente fica registrada e é reprocessável.

**Negativas:**

- No pior caso a entrega acontece ~15 horas depois da transição — aceito pelo produto (`[09:17] Marcos`).
- O replay é manual e depende de um ADMIN: não há automação nem aviso proativo ao cliente (email de alerta ficou para uma próxima fase — `[09:37] Larissa`).
- Duas tabelas para modelar e manter (`webhook_outbox` e `webhook_dead_letter`).

## Referências

- `TRANSCRICAO.md`: `[09:14]`–`[09:19]`, `[09:35]`–`[09:36]`, `[09:42]`
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-polling.md), [ADR-005](ADR-005-at-least-once-x-event-id.md), [ADR-006](ADR-006-reuso-padroes-existentes.md) (requireRole no replay)
