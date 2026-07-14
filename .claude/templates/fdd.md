<!-- Template FDD. Arquivo: docs/FDD.md — o documento mais técnico do pacote.
     Acionável: um dev pega e começa a codar. Altura: implementação. -->

# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

<!-- Por que a solução técnica é necessária, ancorada no código existente (o vácuo de notificação). -->

## Objetivos técnicos

## Escopo e exclusões

<!-- O que este design cobre e o que fica explicitamente fora (com origem na transcrição). -->

## Fluxos detalhados

<!-- Obrigatório cobrir os 4, passo a passo (sequência, estados, transações):
     1. Criação do evento na outbox (dentro da transação do changeStatus)
     2. Processamento pelo worker (polling, batch, marcação de status)
     3. Retry (backoff, limites)
     4. DLQ (quando entra, o que acontece depois) -->

## Contratos públicos

<!-- ≥ 4 endpoints HTTP. Para CADA um: método + rota, headers, payload de exemplo de REQUEST,
     payload de exemplo de RESPONSE, status codes de sucesso e erro, semântica.
     Incluir também o contrato do webhook entregue ao cliente (headers X-Event-Id, assinatura HMAC). -->

## Matriz de erros

<!-- Tabela: Código (prefixo WEBHOOK_) | HTTP status | Quando ocorre | Classe de erro reutilizada.
     Seguir o padrão de src/shared/errors/. -->

## Estratégias de resiliência

<!-- Timeouts, retries, backoff, fallback, idempotência (at-least-once + X-Event-Id). -->

## Observabilidade

<!-- Obrigatório citar os três: métricas (quais), logs (o que logar, via logger Pino existente), tracing. -->

## Dependências e compatibilidade

## Integração com o sistema existente

<!-- SEÇÃO OBRIGATÓRIA DO DESAFIO. ≥ 4 caminhos de arquivo REAIS do código base, cada um com a
     descrição de como o módulo de webhooks se integra. Candidatos:
     - src/modules/orders/order.service.ts (extensão do changeStatus)
     - src/modules/orders/order.status.ts (eventos por transição de estado)
     - src/shared/errors/ (reuso das classes e do padrão de códigos)
     - src/middlewares/auth.middleware.ts (requireRole nos endpoints de gestão)
     - src/middlewares/error.middleware.ts (tratamento centralizado)
     - src/shared/logger/index.ts (logs do worker)
     - prisma/schema.prisma (novas tabelas webhook_*)
     - src/routes/index.ts (registro das rotas do módulo)
     Arquivos A CRIAR devem ser marcados como tal, nunca apresentados como existentes. -->

## Critérios de aceite técnicos

## Riscos e mitigação
