# PRD: Sistema de Webhooks de Notificação de Pedidos

> Documento de produto. A proposta de arquitetura está no [RFC](RFC.md), as decisões nos [ADRs](adrs/README.md) e o detalhamento de implementação no [FDD](FDD.md). Rastreabilidade completa em [TRACKER](TRACKER.md).

## Resumo e contexto da feature

Clientes B2B da plataforma de gestão de pedidos passarão a ser notificados automaticamente, em tempo real, sempre que o status de um pedido deles mudar — via webhooks HTTP cadastrados por eles próprios pela API. A feature nasce de um pedido formal de três clientes (`[09:00] Marcos`) e foi desenhada em reunião técnica com produto, engenharia e segurança; todas as decisões desta página têm origem nessa reunião (`TRANSCRICAO.md`) ou no código existente.

## Problema e motivação

Hoje os clientes descobrem mudanças de status "batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles" (`[09:00] Marcos`). Não existe nenhum mecanismo de notificação ativa.

O custo de não resolver é concreto: a Atlas Comercial "chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente" (`[09:00] Marcos`). Para esses clientes, "tempo real" significa qualquer coisa abaixo de 10 segundos — "o importante é que não fique pendurado e eles tenham que ficar atualizando manualmente" (`[09:02] Marcos`).

## Público-alvo e cenários de uso

**Público primário:** clientes B2B integrados à plataforma via API — os solicitantes nomeados são **Atlas Comercial, MaxDistribuição e Nova Cargo** (`[09:00] Marcos`). O cadastro e a gestão dos webhooks são feitos pela nossa API, autenticados com JWT do nosso sistema, por usuários que representam o cliente (`[09:32] Marcos`).

**Público secundário:** administradores internos da plataforma, responsáveis por reprocessar notificações com falha permanente (`[09:35]`–`[09:36]`).

**Cenários de uso:**

1. **Integração reativa:** o cliente cadastra um webhook escolhendo os status que lhe interessam — "só quero saber quando vira SHIPPED e DELIVERED" (`[09:33] Marcos`) — e seu sistema reage à notificação em segundos, sem polling.
2. **Diagnóstico da integração:** o cliente consulta o histórico das últimas entregas feitas a ele — "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" (`[09:34] Marcos`).
3. **Recuperação operacional:** uma notificação esgotou as retentativas (ex.: endpoint do cliente ficou fora do ar por longo período); um administrador a reprocessa manualmente após o cliente se restabelecer (`[09:18] Diego`).
4. **Higiene de credencial:** o cliente rotaciona a credencial do webhook pela API e migra seus sistemas dentro da janela de transição de 24 horas (`[09:21] Sofia`).

## Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- |
| Notificar mudanças de status em tempo real, eliminando a necessidade de polling | Tempo entre a mudança de status e a notificação entregue | **< 10 segundos** (latência mínima de 2 s do ciclo de leitura aceita) | `[09:02] Marcos`, `[09:10] Larissa` |
| Entregar dentro da janela comercial que retém os clientes solicitantes | Prazo de entrega da feature | **3 sprints** (fim de novembro, prazo da Atlas), incluindo revisão de segurança | `[09:45] Marcos`, `[09:46]`–`[09:47] Larissa` |
| Nenhuma notificação perdida | Toda mudança de status assinada gera notificação entregue ou falha permanente registrada e reprocessável — nunca descarte silencioso | 100% dos eventos | `[09:40] Bruno`, `[09:15] Diego` |

## Escopo

### Incluso

- Cadastro, edição, remoção e listagem de webhooks por cliente, via API autenticada (`[09:31]`–`[09:33]`).
- Escolha, por webhook, de quais status de pedido geram notificação (`[09:33] Marcos`).
- Notificação automática de mudança de status com identificação do pedido e da transição ocorrida (`[09:43] Diego`).
- Credencial de validação por webhook, gerada pela plataforma, com rotação sob demanda e janela de transição de 24 h (`[09:21] Sofia`).
- Retentativas automáticas em caso de falha do endpoint do cliente e registro de falha permanente reprocessável por administrador (`[09:15]`–`[09:19]`).
- Histórico consultável das últimas 100 entregas por webhook (`[09:34] Marcos`).
- Documentação de integração no portal do desenvolvedor (`[09:26]`, `[09:40] Marcos`).

### Fora de escopo

Itens **explicitamente descartados ou adiados na reunião** — não fazem parte desta entrega:

| Item | Situação | Origem |
| --- | --- | --- |
| Webhooks de entrada (cliente → plataforma) | Fora de escopo — "só saindo da gente pra eles" | `[09:02] Marcos` |
| Email de aviso ao cliente quando o webhook dele falha repetidamente | Adiado — "talvez próxima fase, depois que a gente medir o impacto" | `[09:37] Larissa` |
| Rate limiting de envio por cliente | Em observação — "a gente observa e implementa se virar problema" | `[09:39] Diego` |
| Dashboard/painel visual para o cliente acompanhar webhooks | Fora de escopo — "painel é projeto separado do time de frontend" | `[09:40] Larissa` |
| Arquivamento automático de notificações antigas já entregues | Fora do escopo da feature | `[09:08] Diego` |
| Processamento paralelo em múltiplos workers | Adiado — "problema do futuro, não agora" | `[09:13] Diego` |

## Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | O cliente pode cadastrar um webhook informando a URL de destino e a lista de status de pedido que deseja receber | `[09:31] Marcos`, `[09:33] Marcos` |
| RF-02 | A credencial de validação é gerada pela plataforma e entregue ao cliente no ato do cadastro | `[09:31] Marcos` |
| RF-03 | O cliente pode editar, remover e listar os webhooks do seu cadastro | `[09:33] Bruno` |
| RF-04 | Cada webhook recebe somente notificações dos status que assinou | `[09:33] Marcos`, `[09:34] Bruno` |
| RF-05 | Toda mudança de status de pedido gera notificação aos webhooks assinantes do cliente dono do pedido, identificando o pedido, o status anterior e o novo | `[09:40] Bruno`, `[09:43] Diego` |
| RF-06 | O cliente pode consultar o histórico das últimas 100 entregas de um webhook, com resultado (sucesso/falha), conteúdo enviado, resposta recebida e tempo de resposta | `[09:34] Marcos` |
| RF-07 | Notificações com falha permanente podem ser reprocessadas manualmente, apenas por administradores, com registro de quem executou | `[09:18] Diego`, `[09:36] Sofia` |
| RF-08 | Cada notificação carrega um identificador único que permite ao cliente reconhecer e descartar duplicatas | `[09:25] Diego` |
| RF-09 | O cliente consegue verificar que a notificação veio da plataforma e que o conteúdo não foi adulterado | `[09:19] Sofia` |
| RF-10 | O cliente pode rotacionar a credencial pela API; a anterior permanece válida por 24 h para migração | `[09:21] Sofia` |
| RF-11 | O cadastro exige endereço seguro (https); endereços não seguros são recusados no ato | `[09:23] Sofia` |
| RF-12 | A identificação do cliente é explícita na operação de cadastro (não inferida da sessão do usuário) | `[09:32] Larissa` |

## Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RNF-01 | Notificação entregue em menos de 10 segundos após a mudança de status (caminho feliz); latência mínima de 2 s do mecanismo de leitura aceita | `[09:02] Marcos`, `[09:10] Larissa` |
| RNF-02 | A operação de mudança de status de pedidos nunca é bloqueada ou atrasada pela indisponibilidade ou lentidão dos endpoints dos clientes | `[09:04] Bruno` |
| RNF-03 | Consistência absoluta: status mudou ⇔ notificação registrada; nunca um sem o outro | `[09:06] Diego`, `[09:40] Bruno` |
| RNF-04 | Falhas de entrega são retentadas automaticamente por uma janela de ~15 horas antes de virarem falha permanente reprocessável | `[09:17] Diego` |
| RNF-05 | Garantia de entrega "pelo menos uma vez": duplicatas são possíveis e o comportamento é documentado com destaque para os clientes | `[09:24] Diego`, `[09:26] Marcos` |
| RNF-06 | Credencial única por webhook — o comprometimento de uma não afeta os demais clientes | `[09:21] Sofia` |
| RNF-07 | Notificações limitadas a 64KB; acima disso a notificação falha explicitamente (nunca é truncada) | `[09:24] Larissa` |
| RNF-08 | Ordem de entrega garantida apenas por pedido (não global), limitação documentada | `[09:13] Larissa`, `[09:14] Marcos` |
| RNF-09 | Ações administrativas de reprocessamento são auditáveis (quem executou fica registrado) | `[09:36] Sofia` |

## Decisões e trade-offs principais

Uma linha por decisão — o registro completo, com alternativas e consequências, está no ADR correspondente:

- Registro transacional dos eventos no banco existente (padrão outbox), em vez de fila externa ou disparo síncrono → [ADR-001](adrs/ADR-001-outbox-no-mysql.md).
- Entrega por um processo trabalhador separado da API, lendo eventos a cada 2 s → [ADR-002](adrs/ADR-002-worker-separado-polling.md).
- Retentativas com intervalos crescentes (janela de ~15 h) e fila de falhas permanentes com reprocessamento manual → [ADR-003](adrs/ADR-003-retry-backoff-dlq.md).
- Assinatura criptográfica (HMAC-SHA256) com credencial por webhook e rotação com 24 h de transição → [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).
- Garantia "pelo menos uma vez" com identificador único por notificação para deduplicação no cliente → [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md).
- Reuso integral dos padrões do projeto existente (módulos, erros, logs, autorização) → [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md).
- Conteúdo da notificação congelado no momento da mudança de status (snapshot) → [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md).

## Dependências

- **Autenticação existente:** o cadastro usa a autenticação JWT atual da plataforma; a identificação do cliente é explícita na operação (`[09:32] Marcos`, `[09:32] Larissa`).
- **Revisão de segurança:** ao menos 2 dias úteis da engenheira de segurança antes do deploy, já contados nas 3 sprints (`[09:46] Sofia`, `[09:47] Larissa`).
- **Portal do desenvolvedor:** a documentação de integração (incluindo o comportamento de duplicatas) é responsabilidade do PM e condição para os clientes integrarem (`[09:26]`, `[09:40] Marcos`).
- **Adaptação dos clientes:** cada cliente precisa expor um endereço https, validar a assinatura e tratar duplicatas pelo identificador único (`[09:23] Sofia`, `[09:25] Diego`).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Atraso além do trimestre leva a Atlas a migrar para o concorrente (`[09:00] Marcos`) | Média | Alto | Estimativa fechada em 3 sprints já incluindo a revisão de segurança (`[09:46]`–`[09:47]`); PM confirma o prazo com os clientes imediatamente (`[09:47] Marcos`) |
| Cliente vaza a credencial do webhook — há precedente real (`[09:22] Diego`) | Média | Alto | Credencial por webhook limita o estrago a um único cadastro; rotação com 24 h de transição (`[09:21] Sofia`); revisão de segurança dedicada (`[09:46] Sofia`) |
| Cliente não implementa a deduplicação e processa notificações repetidas (`[09:25] Sofia`) | Média | Médio | Comportamento documentado com destaque no portal do desenvolvedor (`[09:26] Marcos`); identificador único estável em toda reentrega |
| Endpoint do cliente fica indisponível por período longo (caso real de 2 h de manutenção — `[09:16] Diego`) | Alta | Baixo | Janela de retentativas de ~15 h cobre indisponibilidades reais; falha permanente fica registrada e é reprocessável por administrador (`[09:17]`–`[09:19]`) |

## Critérios de aceitação

1. Cliente cadastra um webhook com endereço https e lista de status, e recebe a credencial no ato (RF-01, RF-02, RF-11).
2. Mudança de status assinada gera notificação entregue em menos de 10 segundos no caminho feliz (RNF-01).
3. Webhook que assina apenas alguns status não recebe notificações dos demais (RF-04).
4. Indisponibilidade do endpoint do cliente não atrasa nem impede mudanças de status na plataforma (RNF-02).
5. Falha de entrega passa por retentativas automáticas (~15 h) e, esgotadas, fica registrada como falha permanente reprocessável por administrador, com auditoria (RNF-04, RF-07, RNF-09).
6. Cliente valida a autenticidade de cada notificação e reconhece duplicatas pelo identificador único (RF-08, RF-09).
7. Rotação de credencial mantém a anterior válida por exatamente 24 h (RF-10).
8. Histórico das últimas 100 entregas disponível por webhook, com resultado, conteúdo, resposta e tempo (RF-06).

## Estratégia de testes e validação

- **Testes ponta a ponta da integração com pedidos**, previstos na própria estimativa da reunião ("Integração no order.service e testes ponta a ponta é mais meio [sprint]" — `[09:46] Larissa`): mudança de status commitada gera notificação; rollback não gera; filtro de status respeitado.
- **Validação do ciclo de falha:** simulação de endpoint indisponível para verificar a janela de retentativas, o registro de falha permanente e o reprocessamento por administrador (fluxos detalhados e critérios técnicos no [FDD](FDD.md)).
- **Revisão de segurança como gate de deploy:** a engenheira de segurança revisa geração de credencial e assinatura antes de subir — "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy" (`[09:46] Sofia`); a sessão é agendada como parte do plano (`[09:49] Sofia`).
- **Validação com os clientes:** o PM comunica os clientes e a documentação do portal orienta a integração (deduplicação em destaque) (`[09:26]`, `[09:47] Marcos`); a revisão do desenho com o time antes de codar foi combinada na própria reunião (`[09:50] Larissa`).
