# ADR-005: Garantia at-least-once com deduplicação por X-Event-Id

## Status

Aceito — reunião de quinta-feira, 09:00 (`TRANSCRICAO.md`). Decidido em `[09:26]` por Larissa ("At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.").

## Contexto

Com outbox + worker + retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-polling.md), [ADR-003](ADR-003-retry-backoff-dlq.md)), "pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado" (`[09:24] Diego`). (Cenário ilustrativo, não citado na reunião: o envio sucede, mas a marcação de "entregue" no banco falha — ao retomar, o worker reenvia o mesmo evento.) A alternativa — garantir que cada evento chegue exatamente uma vez — exigiria coordenação entre os dois lados da integração (`[09:25] Diego`).

## Decisão

Assumiremos **garantia de entrega at-least-once** (`[09:24] Diego`). Cada evento carrega um **`event_id` (UUID) gerado quando entra na outbox, único por evento, enviado no header `X-Event-Id`**. Se o cliente receber o mesmo evento duas vezes, ele deduplica pelo `event_id` do lado dele (`[09:25] Diego`).

O comportamento será documentado com destaque no portal de desenvolvedor para os clientes (`[09:26] Marcos`).

## Alternativas Consideradas

### Garantia exactly-once

- **Descrição:** assegurar que cada evento seja entregue exatamente uma vez, sem duplicatas.
- **Por que foi descartada:** "exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" (`[09:25] Diego`) — ou seja, o custo de complexidade recairia sobre a plataforma **e** sobre cada cliente integrado, para eliminar um problema (duplicata ocasional) que a deduplicação por `event_id` já resolve de forma barata.

> Nenhuma outra semântica de entrega foi considerada na reunião; o debate em `[09:24]`–`[09:25]` foi exclusivamente entre garantir exactly-once ou assumir at-least-once com deduplicação.

## Consequências

**Positivas:**

- Solução simples e alinhada ao padrão de mercado: "Stripe faz assim, GitHub faz assim" (`[09:25] Diego`).
- Combinada com o retry do [ADR-003](ADR-003-retry-backoff-dlq.md), nenhuma transição commitada deixa de gerar notificação — duplicar é aceitável, perder não.
- O `X-Event-Id` também serve de chave de correlação para suporte e debug de entregas.

**Negativas:**

- Joga a responsabilidade de deduplicação para o cliente (`[09:25] Sofia`).
- Exige documentação destacada no portal de desenvolvedor para que os clientes implementem a dedup corretamente (`[09:26] Marcos`).
- Duplicatas reais vão ocorrer em cenários de falha parcial; clientes que ignorarem a orientação processarão eventos repetidos.

## Referências

- `TRANSCRICAO.md`: `[09:24]`–`[09:26]`
- Relacionados: [ADR-003](ADR-003-retry-backoff-dlq.md), [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) (demais headers do request)
