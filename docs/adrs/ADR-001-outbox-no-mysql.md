# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito — reunião de quinta-feira, 09:00 (`TRANSCRICAO.md`). Decidido em `[09:08]` por Larissa ("Tá decidido então: outbox em MySQL").

## Contexto

A transação de mudança de status de pedido (`changeStatus` em `src/modules/orders/order.service.ts`) já é pesada: atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos (`[09:04] Bruno`). Acrescentar uma chamada HTTP dentro dessa transação faria qualquer cliente lento travar a mudança de status de outros pedidos, e um cliente fora do ar criaria um dilema sem saída — não dá para dar rollback na mudança de status por falha de notificação (`[09:04] Bruno`).

Ao mesmo tempo, a notificação não pode se perder: "se ficar fora da transação, perde a garantia toda" (`[09:41] Diego`). É preciso que status mudado ⇔ evento registrado, sem inconsistência possível (`[09:06] Diego`).

O time é pequeno e não quer operar infraestrutura nova só para isso (`[09:07] Diego`).

## Decisão

Adotaremos o **padrão outbox sobre o MySQL existente**: quando o status do pedido muda, dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserimos uma linha na tabela `webhook_outbox` com o evento. Um worker separado (ver [ADR-002](ADR-002-worker-separado-polling.md)) lê essa tabela e dispara as chamadas HTTP (`[09:06] Diego`).

Detalhes de modelagem fechados na reunião:

- A tabela tem índice no campo de status (pendente, processando, falhou, entregue) e em `created_at`; o worker lê só os pendentes em batch pequeno (`[09:08] Diego`).
- O `id` da outbox é UUID, seguindo o padrão do restante do projeto (`[09:51] Larissa`; ver `prisma/schema.prisma`).
- O filtro de eventos por endpoint é aplicado **na inserção**: se nenhum webhook do customer assina aquele status, a linha nem é inserida (`[09:34] Bruno`, `[09:34] Diego`).
- O payload é gravado já renderizado no momento da inserção (ver [ADR-007](ADR-007-snapshot-payload-na-insercao.md)).

O arquivamento de linhas entregues (após ~30 dias) ficou explicitamente fora do escopo desta feature (`[09:08] Diego`).

## Alternativas Consideradas

### Disparo síncrono dentro do service de orders

- **Descrição:** fazer a chamada HTTP ao cliente diretamente no `changeStatus`, na mesma requisição que muda o status (`[09:03] Larissa`).
- **Por que foi descartada:** "Síncrono está fora de questão" (`[09:06] Diego`). Acopla a latência e a disponibilidade do endpoint do cliente à transação de status — cliente lento trava mudanças de status de outros pedidos, e cliente fora do ar não tem tratamento possível (rollback do status não é aceitável) (`[09:04] Bruno`).

### Redis Streams / fila dedicada

- **Descrição:** publicar os eventos em uma fila externa como Redis Streams (`[09:07] Larissa`).
- **Por que foi descartada:** exigiria subir e operar infraestrutura nova; para um time pequeno, "subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" (`[09:07] Diego`). Além disso, uma fila externa fica fora da transação SQL — a garantia central buscada ("se a transação principal commitou, o evento foi registrado", `[09:06] Diego`) teria que ser reconstruída por outros meios.

### Filtrar eventos no envio, e não na inserção

- **Descrição:** inserir uma linha na outbox a cada mudança de status e deixar o worker decidir, na hora de mandar, se algum webhook do customer assina aquele status (`[09:34] Diego`: "Filtra na inserção do outbox ou na hora de mandar?").
- **Por que foi descartada:** geraria linhas na outbox para eventos que nenhum webhook quer receber — crescimento inútil de uma tabela cujo acúmulo já era preocupação de performance (`[09:07] Bruno`). Filtrando na inserção, "se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela" (`[09:34] Bruno`).

### id auto incremental na outbox

- **Descrição:** usar chave auto incremental na tabela `webhook_outbox` em vez de UUID (`[09:51] Diego`).
- **Por que foi descartada:** destoaria do padrão uniforme de chaves do projeto — todos os models existentes usam UUID (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`): "UUID, segue o padrão do resto do projeto. Tudo é uuid" (`[09:51] Larissa`).

## Consequências

**Positivas:**

- Consistência garantida por construção: se a transação commitou, o evento foi registrado; se deu rollback, o evento some junto — "não tem inconsistência possível" (`[09:06] Diego`).
- Nenhuma infraestrutura nova para operar; reusa MySQL e Prisma já existentes (`[09:07] Diego`).
- A mudança de status não é bloqueada nem afetada pela disponibilidade dos endpoints dos clientes.

**Negativas:**

- A entrega deixa de ser imediata: a latência mínima passa a ser a do ciclo de polling do worker (2 s no pior caso, aceito em `[09:10] Larissa` — ver [ADR-002](ADR-002-worker-separado-polling.md)).
- A tabela `webhook_outbox` cresce continuamente; a política de arquivamento ficou fora do escopo e precisará ser tratada depois (`[09:08] Diego`).
- Ordering global não é garantida — apenas por `order_id` e enquanto houver um único worker (`[09:12] Diego`, `[09:13] Larissa`); escalar workers exigirá particionamento ou lock ("problema do futuro", `[09:13] Diego`).

## Referências

- `TRANSCRICAO.md`: `[09:03]`–`[09:08]`, `[09:34]`, `[09:40]`–`[09:41]`, `[09:51]`
- Código: `src/modules/orders/order.service.ts` (transação `changeStatus`), `prisma/schema.prisma` (padrão de ids UUID e índices)
- Relacionados: [ADR-002](ADR-002-worker-separado-polling.md), [ADR-003](ADR-003-retry-backoff-dlq.md), [ADR-007](ADR-007-snapshot-payload-na-insercao.md)
