# ADR-007: Snapshot do payload renderizado na inserção na outbox

## Status

Aceito — reunião de quinta-feira, 09:00 (`TRANSCRICAO.md`). Decidido em `[09:52]` por Larissa, Diego e Bruno ("Beleza, snapshot. Decidido.").

## Contexto

Dúvida levantada por Bruno no fim da reunião: o evento da outbox guarda o payload já renderizado, ou guarda só o `order_id` e renderiza na hora do envio? (`[09:51] Bruno`). A diferença importa porque entre a transição de status e a entrega pode passar tempo — com retry, até ~15 horas ([ADR-003](ADR-003-retry-backoff-dlq.md)) — e o pedido pode mudar nesse intervalo.

## Decisão

O evento é gravado na `webhook_outbox` **com o payload completo já renderizado no momento da inserção** (snapshot), dentro da mesma transação da mudança de status: "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" (`[09:52] Larissa`).

O conteúdo do payload segue o formato enxuto definido em `[09:43] Diego` — JSON com `event_id`, `event_type` (`order.status_changed`), timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`, **sem items** (quem quiser detalhes consulta `GET /orders/:id`). O contrato completo do payload é detalhado no [FDD](../FDD.md).

## Alternativas Consideradas

### Guardar só o order_id e renderizar no envio

- **Descrição:** a outbox armazenaria apenas a referência ao pedido; o worker montaria o payload na hora de enviar (`[09:51] Bruno`).
- **Por que foi descartada:** se o pedido mudar entre a transição e o envio (ou entre retries), o evento renderizado tardiamente não refletiria mais o estado do momento da mudança de status — o "caso esquisito" apontado em `[09:52] Larissa`.

> A dúvida em `[09:51]` colocou exatamente estas duas opções (payload renderizado vs. só `order_id`); nenhuma terceira foi discutida na reunião.

## Consequências

**Positivas:**

- O evento é imutável e fiel ao estado do pedido no momento exato da transição (`[09:52] Larissa`).
- O worker não precisa reler o pedido para enviar; retries reenviam exatamente o mesmo corpo, o que mantém a assinatura HMAC estável por evento ([ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md)).

**Negativas:**

- O payload completo ocupa mais espaço na `webhook_outbox` (e na DLQ) do que uma simples referência — o que reforça a relevância do limite de 64KB por evento (`[09:24] Larissa`).
- Mudanças futuras no formato do payload conviverão com eventos antigos já renderizados no formato anterior.

## Referências

- `TRANSCRICAO.md`: `[09:43]`, `[09:51]`–`[09:52]`, `[09:24]`
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-dlq.md), [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md)
