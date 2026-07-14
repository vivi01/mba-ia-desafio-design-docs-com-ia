# Fronteiras entre os documentos

Cada documento opera em uma **altura** diferente. Conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.

| Documento | Altura | Pergunta que responde |
| --- | --- | --- |
| PRD | Produto / negócio | *Por que e o quê?* |
| RFC | Arquitetura | *Como pretendemos resolver, e o que ainda está em aberto?* |
| ADRs | Decisão pontual | *Por que decidimos exatamente assim?* |
| FDD | Implementação | *Como construir, em detalhe?* |
| Tracker | Transversal | *De onde veio cada coisa?* |

Em uma frase: o **RFC propõe e abre para revisão**, os **ADRs registram cada decisão fechada** e o **FDD detalha como construir**.

## Regras de fronteira (o que fica onde)

- **PRD** fala de problema, público, escopo e métricas. Não descreve tabelas, endpoints nem headers. Se aparecer `HMAC` ou nome de coluna no PRD, está descendo demais — cite a decisão em uma linha e aponte para o ADR.
- **RFC** é conciso (2 a 4 páginas) e fala em **decisão**: visão geral da solução, alternativas descartadas com o trade-off que motivou o descarte, questões em aberto. Não repete o nível de detalhe do FDD — nada de payload de exemplo, matriz de erros ou passo a passo de fluxo no RFC. Referencia os ADRs por link relativo (`adrs/ADR-001-....md`).
- **ADR** = uma decisão por arquivo, formato MADR: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas E negativas, com trade-off explícito). Contexto explica a força que levou à decisão, não resume a feature inteira.
- **FDD** é o único documento com profundidade de implementação: fluxos detalhados (outbox → worker → retry → DLQ), contratos HTTP completos com payloads de exemplo, matriz de erros `WEBHOOK_*`, resiliência, observabilidade. É onde vivem os detalhes técnicos secundários (formato de payload, timeouts, headers) que não viraram ADR.
- **Tracker** não tem conteúdo próprio: apenas mapeia itens dos outros documentos às fontes.

## Convenções de formato

- ADRs em `docs/adrs/`, nomeados `ADR-NNN-titulo-em-kebab-case.md` (ex.: `ADR-001-outbox-no-mysql.md`), numeração sequencial a partir de 001. **Este formato do desafio prevalece** sobre a sugestão `0001-titulo.md` do `docs/adrs/README.md` existente.
- Entre 5 e 8 ADRs — nem menos, nem mais.
- Links entre documentos sempre relativos, para funcionarem no GitHub.
- Códigos de erro da feature sempre com prefixo `WEBHOOK_` (ex.: `WEBHOOK_ENDPOINT_NOT_FOUND`), seguindo o padrão de códigos das classes de erro em `src/shared/errors/`.
- Metadados do RFC usam os participantes reais da reunião como revisores (Larissa, Marcos, Bruno, Diego, Sofia).

## Teste rápido de altura

Antes de escrever uma seção, pergunte: "um leitor procuraria isso NESTE documento?"

- Meta de latência de entrega da notificação → PRD (métrica) e FDD (critério de aceite técnico).
- "Por que outbox e não Redis Streams?" → ADR (decisão) e RFC (alternativa, resumida).
- Corpo do POST de cadastro de endpoint → só FDD.
- "De onde veio o limite de retries?" → Tracker.
