# Rastreabilidade e Tracker

`docs/TRACKER.md` é a defesa contra alucinação. Ele é preenchido **junto** com cada documento, não depois: ao registrar um requisito, decisão, restrição ou trade-off em qualquer documento, adicione a linha correspondente no tracker na mesma sessão.

## Formato obrigatório

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |

- **ID**: único, com prefixo do documento e tipo. Convenção:
  - `PRD-FR-NN` (requisito funcional), `PRD-NFR-NN` (não funcional), `PRD-OBJ-NN` (objetivo/métrica), `PRD-ESC-NN` (escopo/fora de escopo), `PRD-RISK-NN` (risco)
  - `RFC-PROP-NN` (proposta), `RFC-ALT-NN` (alternativa descartada), `RFC-OPEN-NN` (questão em aberto)
  - `FDD-FLUXO-NN`, `FDD-CONTRATO-NN`, `FDD-ERRO-NN`, `FDD-INT-NN` (integração com código), `FDD-OBS-NN`
  - `ADR-NNN` (uma linha por ADR, no mínimo; decisão + alternativa podem virar linhas separadas)
- **Documento**: caminho do arquivo onde o item aparece (ex.: `docs/PRD.md`, `docs/adrs/ADR-002-retry-backoff-dlq.md`)
- **Tipo**: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, Alternativa Descartada, Questão em Aberto, Risco, Métrica, Contrato, Integração
- **Conteúdo (resumo)**: uma linha
- **Fonte**: `TRANSCRICAO` ou `CODIGO`
- **Localização**:
  - Para `TRANSCRICAO`: timestamp + falante, exatamente no formato `[hh:mm] Nome` (ex.: `[09:06] Diego`). O timestamp deve existir na transcrição e a fala citada deve realmente sustentar o item. Se o item vem de um trecho de diálogo, cite o timestamp da fala mais decisiva; múltiplos timestamps separados por vírgula são permitidos.
  - Para `CODIGO`: caminho real do arquivo (ex.: `src/modules/orders/order.service.ts`). Verifique a existência antes de registrar.

## Metas quantitativas (critérios de aceite)

- ≥ 80% dos itens identificáveis dos documentos com linha no tracker — na prática, mire 100%.
- ≥ 70% das linhas com Fonte = `TRANSCRICAO` e timestamp válido.
- ≥ 5 linhas com Fonte = `CODIGO` e caminho real.

## Rotina de verificação (rodar antes do push final)

1. Para cada linha com Fonte = `TRANSCRICAO`, confirmar via Grep que o timestamp + falante existem em `TRANSCRICAO.md` e que a fala sustenta o resumo.
2. Para cada linha com Fonte = `CODIGO`, confirmar via Glob que o arquivo existe.
3. Varrer PRD, RFC, FDD e ADRs procurando afirmações concretas (requisitos, números, decisões) sem linha no tracker → adicionar linha ou remover a afirmação.
4. Conferir que nenhum ID está duplicado e que os caminhos na coluna Documento existem.
