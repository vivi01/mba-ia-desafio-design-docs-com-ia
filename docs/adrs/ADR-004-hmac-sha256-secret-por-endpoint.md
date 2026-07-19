# ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

## Status

Aceito — reunião de quinta-feira, 09:00 (`TRANSCRICAO.md`). Decidido em `[09:22]` por Sofia ("Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h").

## Contexto

Os webhooks expõem eventos com dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir validar que a requisição veio realmente da plataforma e que ninguém adulterou o payload no caminho (`[09:19] Sofia`). Há histórico concreto de risco: um cliente já vazou uma secret em log da aplicação dele (`[09:22] Diego`) — o desenho precisa limitar o estrago de um vazamento.

## Decisão

Adotaremos **assinatura HMAC-SHA256 sobre o corpo do request**, que o cliente verifica do lado dele (`[09:20] Sofia`); a assinatura viaja no header `X-Signature` (proposto como "um header tipo X-Signature" em `[09:20] Sofia` e confirmado nominalmente na lista de headers de `[09:44] Diego`). HMAC-SHA256 é o padrão de mercado e "todo cliente sério tem biblioteca pra isso" (`[09:20] Sofia`).

- **Secret única por endpoint** de webhook — nunca uma secret global da plataforma (`[09:21] Sofia`). A secret é gerada pela plataforma e devolvida na criação do webhook (`[09:31] Marcos`).
- **Rotação via API**: o cliente pode pedir nova secret; a antiga permanece válida por **24 horas em paralelo** para dar tempo de migração, e depois morre (`[09:21] Sofia`).
- Complementos de segurança definidos na mesma reunião: header `X-Timestamp` com o timestamp do envio, para o cliente poder detectar replay attack (`[09:44] Diego`), e TLS obrigatório — URL de webhook precisa ser `https`, recusada com erro de validação no schema Zod caso contrário (`[09:23] Sofia`; classificado na reunião como validação/requisito, não como decisão arquitetural separada).

## Alternativas Consideradas

### Secret global da plataforma

- **Descrição:** a plataforma usaria uma única secret, compartilhada com todos os clientes, para assinar todos os webhooks enviados.
- **Por que foi descartada:** vazamento de secret é um cenário realista — já houve cliente que expôs uma secret em log da própria aplicação (`[09:22] Diego`). Com uma secret global, o descuido de um único cliente comprometeria a autenticidade dos webhooks de **todos** os clientes de uma vez ("se vaza uma, vaza tudo" — `[09:21] Sofia`), e a resposta ao incidente exigiria rotação emergencial coordenada com a base inteira. Com secret por endpoint, o mesmo incidente fica contido àquele único cadastro e se resolve com uma rotação pontual.

> Nenhum outro mecanismo de autenticação/assinatura foi debatido na reunião — HMAC entrou como "o padrão" e HMAC-SHA256 como "o padrão de mercado", sem contraproposta (`[09:20] Sofia`). A única alternativa realmente discutida foi o **escopo** da secret (global vs. por endpoint), registrada acima.

## Consequências

**Positivas:**

- O cliente valida autenticidade e integridade do payload com bibliotecas padrão de mercado (`[09:20] Sofia`).
- Blast radius de vazamento limitado a um endpoint; rotação sem downtime na integração graças ao grace period (`[09:21] Sofia`).
- `X-Timestamp` dá ao cliente a opção de rejeitar replays (`[09:44] Diego`).

**Negativas:**

- Durante as 24 h de grace period existem duas secrets válidas por endpoint — a verificação e o ciclo de vida das secrets ficam mais complexos.
- Geração, armazenamento e rotação de secrets viram superfície sensível: a revisão de segurança da Sofia (≥ 2 dias úteis antes do deploy) é pré-requisito exatamente por isso (`[09:46] Sofia`).
- A verificação da assinatura é responsabilidade do cliente; a proteção só vale se ele de fato validar.

## Referências

- `TRANSCRICAO.md`: `[09:19]`–`[09:23]`, `[09:31]`, `[09:44]`, `[09:46]`
- Código: `src/shared/logger/index.ts` (padrão de redação de campos sensíveis em log, a estender para secrets de webhook)
- Relacionados: [ADR-005](ADR-005-at-least-once-x-event-id.md) (headers do request), [ADR-006](ADR-006-reuso-padroes-existentes.md) (schemas Zod)
