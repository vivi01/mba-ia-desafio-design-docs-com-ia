# ADR-002: Worker de entrega em processo separado, lendo a outbox por polling de 2 segundos

## Status

Aceito — reunião de quinta-feira, 09:00 (`TRANSCRICAO.md`). Decidido em `[09:10]` por Larissa (polling de 2 s: "Vamos registrar isso como uma decisão") e em `[09:11]`–`[09:12]` por Diego e Larissa (processo separado).

## Contexto

Com a outbox no MySQL ([ADR-001](ADR-001-outbox-no-mysql.md)), algo precisa consumir a tabela e disparar os HTTP calls. Duas forças moldaram o formato:

- **Reatividade:** os clientes consideram "tempo real" qualquer entrega abaixo de 10 segundos (`[09:02] Marcos`). Porém o MySQL não tem mecanismo de notificação a processo externo tipo NOTIFY/LISTEN do Postgres — triggers só executam SQL (`[09:09] Diego`).
- **Resiliência:** se o worker rodar dentro da mesma instância da API, um restart da API derruba o worker junto (`[09:11] Diego`).

O projeto hoje tem uma única entry-point HTTP (`src/server.ts`), que serve de modelo para uma segunda entry-point.

## Decisão

O worker rodará como **processo Node separado**, com uma entry-point nova no projeto — `src/worker.ts` (a criar) e um script `npm run worker` (`[09:11] Larissa`). Ele lê a outbox por **polling em loop: a cada 2 segundos busca os eventos pendentes mais antigos, processa e marca** (`[09:09] Diego`).

- Mesmo banco, mesma stack — mas nunca o mesmo processo (`[09:11] Diego`).
- O worker abre um **PrismaClient próprio**: a instância é por processo, com a mesma `DATABASE_URL` (`[09:30] Bruno`).
- Por enquanto **single-worker**, com ordering implícita por `order_id` via `created_at` da outbox (`[09:12] Diego`). Documentado como limitação conhecida: não há garantia de ordering global, só por pedido e enquanto houver um único worker (`[09:13] Larissa`).

A latência mínima de 2 segundos no pior caso foi aceita explicitamente (`[09:10] Larissa`, `[09:10] Marcos`).

## Alternativas Consideradas

### Trigger de banco para acionar o worker

- **Descrição:** usar trigger do MySQL para reagir a inserções na outbox e avisar o worker (`[09:09] Bruno`).
- **Por que foi descartada:** trigger do MySQL não notifica processo externo, só executa SQL; para avisar o worker seria preciso improvisar (escrever em arquivo, bater em endpoint), o que "fica esquisito". Polling de 2 s atende o requisito de menos de 10 s com folga (`[09:09] Diego`).

### Worker dentro do mesmo processo da API

- **Descrição:** rodar o loop de entrega na mesma instância Node da API.
- **Por que foi descartada:** se a API reinicia, perde o worker (`[09:11] Diego`).

### Múltiplos workers em paralelo

- **Descrição:** escalar o consumo da outbox horizontalmente, particionando por `order_id` ou usando lock pessimista (`[09:13] Bruno`, `[09:13] Diego`).
- **Por que foi descartada (adiada):** "é problema do futuro, não agora" (`[09:13] Diego`); com múltiplos workers a garantia de ordering se perde (`[09:12] Diego`).

## Consequências

**Positivas:**

- Falhas e deploys da API não afetam a entrega de webhooks (e vice-versa) (`[09:11] Diego`).
- Polling de 2 s atende o requisito de "abaixo de 10 segundos" com folga (`[09:09] Diego`).
- Nenhuma dependência de mecanismos externos de notificação; só MySQL + Node já existentes.

**Negativas:**

- Latência mínima de 2 segundos no pior caso, aceita na reunião (`[09:10] Larissa`).
- O polling gera consultas constantes ao banco, mesmo sem eventos pendentes.
- Ordering global não garantida; a garantia por `order_id` desaparece se um dia houver múltiplos workers (`[09:12] Diego`, `[09:13] Larissa`).
- Um processo a mais para operar, monitorar e manter vivo em produção.

## Referências

- `TRANSCRICAO.md`: `[09:02]`, `[09:08]`–`[09:14]`, `[09:29]`–`[09:30]`
- Código: `src/server.ts` (modelo de entry-point e graceful shutdown), `src/config/database.ts` (instanciação do PrismaClient)
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-dlq.md), [ADR-006](ADR-006-reuso-padroes-existentes.md)
