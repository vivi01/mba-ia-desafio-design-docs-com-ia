# Da Reunião ao Documento — Processo de Produção dos Design Docs

> Entrega do desafio **"Da Reunião ao Documento: Design Docs Gerados por IA"** do MBA Engenharia de Software com IA (Full Cycle).
> O enunciado original está preservado em [`ENUNCIADO.md`](ENUNCIADO.md) e no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).
>
> **Autora:** Viviane Pereira

## Sobre o desafio

O desafio parte de um cenário realista: uma empresa que opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) decidiu, em uma reunião técnica de ~55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos** — mas nada foi registrado além da transcrição da call (`TRANSCRICAO.md`). Minha tarefa foi transformar essa transcrição, junto com o código existente, em um pacote completo de design docs (PRD, RFC, FDD, 5–8 ADRs e um tracker de rastreabilidade), acionável o suficiente para o time começar a implementar.

A entrega é 100% documental — nenhuma linha de código foi alterada — e tem uma regra de ouro: **toda informação registrada precisa ser rastreável à transcrição (timestamp + falante) ou a um arquivo real do código**. O que a reunião descartou ou adiou não vira requisito; o que ela deixou sem decisão permanece como questão em aberto. O Claude Code foi a ferramenta principal de produção, e o meu papel foi o de maestro: dirigir com prompts específicos, revisar criticamente cada documento e forçar correções até o resultado ficar consistente.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (CLI, modelo Claude Fable 5) | Ferramenta única de produção: leu o repositório e a transcrição, mapeou os pontos de integração do código, gerou e revisou todos os documentos, alimentou o tracker em paralelo e executou as verificações automatizadas (grep das citações contra a transcrição, checagem de existência dos caminhos de arquivo) |
| **Configuração customizada do agente** (`CLAUDE.md` + `.claude/rules/` + `.claude/templates/`) | "Prompt persistente" do projeto: regras anti-alucinação, fronteiras de altura entre documentos, convenções do tracker, workflow em etapas e templates com as seções obrigatórias de cada documento — carregados em toda sessão para manter o agente dentro dos trilhos |

## Workflow adotado

Segui a ordem em que cada documento serve de fundação para o próximo, com o tracker alimentado **em paralelo** (nunca depois) e uma revisão crítica minha entre cada etapa:

1. **Contextualização** — leitura integral da transcrição (155 falas) e mapeamento do código (transação `changeStatus`, máquina de estados, classes de erro, `requireRole`, error middleware, logger Pino, schema Prisma, registro de rotas). Saída: três listas de trabalho — 27 decisões fechadas com o timestamp do "martelo", requisitos explícitos (funcionais e não funcionais) e 16 itens **descartados/adiados** (a lista que protege o "Fora de escopo").
2. **ADRs primeiro** — as 6 decisões principais da reunião + 1 extra (snapshot do payload) viraram 7 ADRs em formato MADR, o esqueleto de todo o resto.
3. **RFC** — consolidação da proposta em cima dos ADRs: alternativas descartadas e questões em aberto moram aqui, sem descer ao detalhe do FDD.
4. **FDD** — o único documento com profundidade de implementação: fluxos outbox → worker → retry → DLQ, 7 contratos HTTP + contrato outbound, matriz de erros `WEBHOOK_*`, observabilidade e a seção "Integração com o sistema existente" com 11 caminhos reais do código.
5. **PRD** — por último entre os grandes documentos: com o pacote técnico pronto, virou consolidação de alto nível em vocabulário de produto.
6. **Varredura final do tracker** — rotina de verificação completa (130 linhas, 84% com fonte na transcrição, 21 linhas ancoradas em código).
7. **README do processo** (este arquivo) — escrito a partir de um log de trabalho (`.claude/notas-processo.md`) mantido durante todo o processo, registrando prompts e correções no momento em que aconteceram.

Duas práticas transversais fizeram diferença: **cada afirmação nova gerava sua linha no tracker na mesma sessão**, e ao final de cada bloco eu rodava uma **verificação automatizada** — extração de todas as citações `[hh:mm] Falante` dos documentos com grep literal contra a `TRANSCRICAO.md` (89 citações únicas, 0 não encontradas) e teste de existência de todo caminho de arquivo citado (os 3 únicos inexistentes são futuros e estão sempre marcados "(a criar)").

## Prompts customizados

Prompt de contextualização dirigida, que estruturou a leitura da transcrição em listas verificáveis em vez de um resumo genérico:

```text
Leia TRANSCRICAO.md na íntegra e mapeie o código nos pontos de integração
listados no CLAUDE.md. Produza três listas de trabalho:
1. decisões fechadas — com o timestamp exato de onde o martelo foi batido
   e quem bateu;
2. requisitos explícitos (funcionais e não funcionais) — cada um com
   timestamp + falante;
3. itens explicitamente descartados ou adiados — com timestamp. Esta lista
   protege o "Fora de escopo" do PRD e as "Questões em aberto" do RFC:
   nada dela pode aparecer como requisito.
```

Prompt de revisão dirigida dos ADRs (escrito depois de eu notar justificativas fracas em dois ADRs — foi o prompt que mais melhorou a qualidade do pacote):

```text
Revise as demais ADRs e procure por ambiguidades e se existe algum problema
identificado na ADR-003 e ADR-004. Também notei que em algumas ADRs tem
apenas uma alternativa avaliada: reavalie e verifique se não existem outras
opções que não foram incluídas na ADR. Todas as informações na ADR DEVEM
ser claras, sem ambiguidade e com o real motivo de sua escolha. Não crie
nada que não seja justificado pela transcrição ou código.
```

Prompt de auditoria de coerência cruzada, usado antes do PRD para travar inconsistências entre documentos:

```text
Revise todo o conteúdo das ADRs, RFC e FDD e certifique-se de que todas as
informações são coerentes com o que foi discutido na transcrição e com o
que existe no código. Não pode existir ambiguidade. Verifique cada citação
[hh:mm] Falante contra a transcrição e cada caminho de arquivo contra o
repositório.
```

## Iterações e ajustes

O processo levou **3 ciclos de revisão sobre os ADRs** (geração inicial → revisão pontual → varredura completa) mais uma auditoria cruzada de todo o pacote antes do PRD. As correções mais relevantes:

1. **Justificativas de descarte invertidas (ADR-003 e ADR-004).** A IA justificou o descarte de alternativas citando os *benefícios da opção escolhida* (ex.: "a tabela separada deixa a leitura mais limpa") em vez do *problema da alternativa em si*. Apontei os dois casos; a correção reescreveu o descarte de "marcar failed na própria outbox" (falhas permanentes se acumulariam na tabela que o worker consulta a cada 2s) e o da "secret global" (cadeia causal: precedente real de vazamento → um cliente descuidado comprometeria todos → rotação emergencial coordenada). O mesmo vício foi então caçado nos outros cinco ADRs.
2. **Alternativas reais da reunião estavam faltando no ADR-001.** Na revisão seguinte, pedi para reavaliar os ADRs com alternativa única contra a transcrição — e a varredura encontrou duas alternativas debatidas e descartadas na reunião que a IA não tinha registrado: *filtrar eventos no envio vs. na inserção* (`[09:34]`) e *id auto incremental vs. UUID* (`[09:51]`). Nos casos em que a reunião realmente só debateu uma alternativa (ADR-004/005/007), a instrução "não crie nada sem lastro" levou a notas explícitas de que nenhuma outra opção foi discutida — em vez de alternativas "plausíveis" inventadas.
3. **Inconsistência numérica entre RFC e FDD.** A auditoria cruzada flagrou o RFC dizendo "duas tabelas novas" enquanto o FDD define **quatro** (configuração, outbox, DLQ, histórico de entregas) — e o próprio FDD oscilava entre "três + uma" e "duas + duas auxiliares". Unificado para "quatro" nos quatro pontos, honrando a regra de números consistentes entre documentos.
4. **Ambiguidade real da transcrição tratada às claras.** A reunião fixa "5 tentativas" mas lista **5 intervalos** de backoff com "quase 15 horas entre primeira falha e última tentativa" — as duas coisas não fecham aritmeticamente. Em vez de esconder, o ADR-003 e o FDD ganharam uma nota de desambiguação com a única leitura consistente (envio inicial + 5 retentativas) e a justificativa da escolha.
5. **Falsa atribuição de fala.** No ADR-005, um exemplo ilustrativo criado pela IA (worker falha ao marcar como entregue) estava emendado na citação do Diego como se fosse fala dele. Corrigido: citação literal separada e exemplo marcado como "cenário ilustrativo, não citado na reunião".
6. **Detalhes menores pegos pelas verificações:** typo `TRANSSCRICAO` numa linha do tracker; o header `X-Signature` atribuído a `[09:20]` quando a Sofia disse "um header *tipo* X-Signature" (o nome definitivo vem da lista do Diego em `[09:44]`).

## Como navegar a entrega

```
.
├── README.md                  ← este arquivo (processo de produção)
├── ENUNCIADO.md               ← enunciado original do desafio
├── TRANSCRICAO.md             ← fonte primária (reunião, intocada)
├── docs/
│   ├── PRD.md                 ← produto: problema, escopo, requisitos, métricas
│   ├── RFC.md                 ← arquitetura: proposta, alternativas, questões em aberto
│   ├── FDD.md                 ← implementação: fluxos, contratos, erros, integração
│   ├── TRACKER.md             ← rastreabilidade: 130 itens → transcrição ou código
│   └── adrs/                  ← 7 decisões em formato MADR (ADR-001 a ADR-007)
│       └── README.md          ← índice dos ADRs
└── src/, prisma/, tests/      ← código existente (contexto e referência, intocado)
```

**Ordem de leitura sugerida** (da altura de negócio para a de implementação):

1. [`docs/PRD.md`](docs/PRD.md) — por que a feature existe e o que ela entrega;
2. [`docs/RFC.md`](docs/RFC.md) — como propomos resolver, o que foi descartado e o que segue em aberto;
3. [`docs/adrs/`](docs/adrs/README.md) — o porquê de cada decisão, uma a uma;
4. [`docs/FDD.md`](docs/FDD.md) — como construir, em detalhe acionável;
5. [`docs/TRACKER.md`](docs/TRACKER.md) — de onde veio cada afirmação (consulte quando quiser conferir a origem de qualquer item).

Para auditar qualquer afirmação: pegue o timestamp citado (ex.: `[09:17] Diego`) e procure-o na `TRANSCRICAO.md` — todas as 89 citações únicas do pacote foram verificadas dessa forma antes da entrega.

## Critérios de Aceite
   
Checklist do [enunciado](ENUNCIADO.md), verificado item por item na revisão final (com apoio de verificações automatizadas por grep):

### PRD (`docs/PRD.md`)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias (Resumo e contexto; Problema e motivação; Público-alvo e cenários de uso; Objetivos e métricas; Escopo incluso e fora de escopo; Requisitos funcionais; Requisitos não funcionais; Decisões e trade-offs; Dependências; Riscos e mitigação; Critérios de aceitação; Estratégia de testes)
- [x] Identifica no mínimo 8 requisitos funcionais discutidos na reunião — **12 RFs**, todos com timestamp
- [x] Inclui pelo menos 1 objetivo com métrica e meta quantitativa — **3 objetivos** (entrega < 10 s; 3 sprints/fim de novembro; 100% dos eventos)
- [x] "Fora de escopo" com pelo menos 2 itens explicitamente descartados/adiados na reunião — **6 itens**, com timestamp
- [x] "Riscos" com pelo menos 2 riscos com probabilidade, impacto e mitigação — **4 riscos**

### RFC (`docs/RFC.md`)

- [x] Arquivo existe e está em Markdown, conciso (2–4 páginas — ~1.350 palavras, ≈3 páginas)
- [x] Contém todas as seções obrigatórias (Metadados com os participantes reais como revisores; TL;DR; Contexto e problema; Proposta técnica; Alternativas consideradas; Questões em aberto; Impacto e riscos; Decisões relacionadas)
- [x] Pelo menos 2 alternativas reais descartadas na reunião, com o trade-off do descarte — **6 alternativas**
- [x] Pelo menos 2 questões em aberto levantadas na reunião — **5 questões**
- [x] Referencia com link pelo menos 2 ADRs — **os 7 ADRs**

### FDD (`docs/FDD.md`)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias, incluindo os 4 fluxos detalhados (outbox, worker, retry, DLQ)
- [x] "Contratos públicos" com pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes — **7 endpoints + o contrato outbound do webhook**
- [x] Matriz de erros com códigos no padrão `WEBHOOK_*`
- [x] "Integração com o sistema existente" com pelo menos 4 caminhos de arquivo reais — **10 caminhos verificados** (arquivos futuros sempre marcados "a criar")
- [x] "Observabilidade" cita métricas, logs e tracing

### ADRs (`docs/adrs/`)

- [x] Entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md` — **7 ADRs**
- [x] Cada ADR com Status, Contexto, Decisão, Alternativas Consideradas e Consequências (positivas e negativas, com trade-off explícito)
- [x] Conjunto cobre pelo menos 5 das 6 decisões principais — **cobre as 6** (outbox MySQL; retry/backoff/DLQ; HMAC-SHA256 por endpoint; at-least-once com X-Event-Id; worker separado em polling; reuso de padrões) + 1 extra (snapshot do payload)
- [x] Pelo menos 1 ADR referencia arquivos/módulos/classes do código base — **ADR-001, 002, 004 e 006**

### Tracker (`docs/TRACKER.md`)

- [x] Segue o formato de tabela obrigatório (ID, Documento, Tipo, Conteúdo, Fonte, Localização)
- [x] Pelo menos 80% dos itens identificáveis dos documentos têm linha — **130 linhas**, varredura de cobertura executada
- [x] Pelo menos 70% das linhas com Fonte = `TRANSCRICAO` e timestamp válido — **84%** (109/130), todas conferidas por grep
- [x] Pelo menos 5 linhas com Fonte = `CODIGO` e caminho real — **21 linhas**, caminhos verificados

### README (`README.md`)

- [x] Todas as seções obrigatórias (Sobre o desafio; Ferramentas de IA; Workflow adotado; Prompts customizados; Iterações e ajustes; Como navegar a entrega; Critérios de Aceite)
- [x] Pelo menos 1 ferramenta de IA listada com o papel — **2 listadas**
- [x] Pelo menos 2 prompts customizados em blocos de código — **3 prompts**
- [x] Pelo menos 2 iterações/ajustes concretos descritos — **6 iterações**

### Consistência geral

- [x] Nenhum requisito, decisão ou restrição contradiz a transcrição ou o código (auditoria de coerência dedicada)
- [x] Nenhum arquivo de código citado é inexistente — os 3 caminhos futuros estão sempre marcados "(a criar)"
- [x] Nenhum item descartado/adiado aparece como requisito — só em "Fora de escopo", "Questões em aberto", "Alternativas" e "Exclusões"
- [x] Números e valores consistentes entre PRD, RFC, FDD, ADRs e Tracker
- [x] `src/`, `prisma/`, `tests/`, `TRANSCRICAO.md` e configurações intocados
