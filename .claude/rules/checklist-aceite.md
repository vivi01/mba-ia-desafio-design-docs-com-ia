# Checklist de critérios de aceite

Rodar item por item antes do push final. Todos são obrigatórios.

## PRD (`docs/PRD.md`)

- [ ] Arquivo existe e está em Markdown
- [ ] Contém TODAS as seções: Resumo e contexto; Problema e motivação; Público-alvo e cenários de uso; Objetivos e métricas de sucesso; Escopo (incluso e fora de escopo); Requisitos funcionais; Requisitos não funcionais; Decisões e trade-offs principais; Dependências; Riscos e mitigação; Critérios de aceitação; Estratégia de testes e validação
- [ ] ≥ 8 requisitos funcionais discutidos na reunião
- [ ] ≥ 1 objetivo com métrica e meta quantitativa
- [ ] "Fora de escopo" com ≥ 2 itens explicitamente descartados ou adiados na reunião
- [ ] "Riscos" com ≥ 2 riscos, cada um com probabilidade, impacto e mitigação

## RFC (`docs/RFC.md`)

- [ ] Arquivo existe e está em Markdown, conciso (2–4 páginas)
- [ ] Contém TODAS as seções: Metadados (autor, status, data, revisores = participantes da reunião); Resumo executivo (TL;DR); Contexto e problema; Proposta técnica; Alternativas consideradas; Questões em aberto; Impacto e riscos; Decisões relacionadas
- [ ] ≥ 2 alternativas reais descartadas na reunião, cada uma com o trade-off que motivou o descarte
- [ ] ≥ 2 questões em aberto levantadas na reunião e não decididas ou adiadas
- [ ] Referencia com link ≥ 2 ADRs do pacote
- [ ] Não duplica o detalhamento do FDD

## FDD (`docs/FDD.md`)

- [ ] Arquivo existe e está em Markdown
- [ ] Contém TODAS as seções: Contexto e motivação técnica; Objetivos técnicos; Escopo e exclusões; Fluxos detalhados (outbox, worker, retry, DLQ); Contratos públicos; Matriz de erros; Estratégias de resiliência; Observabilidade; Dependências e compatibilidade; Critérios de aceite técnicos; Riscos e mitigação; **Integração com o sistema existente**
- [ ] "Contratos públicos" com ≥ 4 endpoints HTTP, cada um com payload de exemplo (request E response) e status codes
- [ ] Matriz de erros com códigos prefixo `WEBHOOK_`
- [ ] "Integração com o sistema existente" referencia ≥ 4 caminhos de arquivo REAIS do código (verificar existência)
- [ ] "Observabilidade" cita métricas, logs E tracing

## ADRs (`docs/adrs/`)

- [ ] Entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md`
- [ ] Cada ADR com as seções: Status, Contexto, Decisão, Alternativas Consideradas (≥ 1 real), Consequências (positivas e negativas, trade-off explícito)
- [ ] Conjunto cobre ≥ 5 das 6 decisões principais (outbox MySQL; retry/backoff/DLQ; HMAC-SHA256 por endpoint; at-least-once com X-Event-Id; worker separado em polling; reuso de padrões existentes)
- [ ] ≥ 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

## Tracker (`docs/TRACKER.md`)

- [ ] Segue o formato de tabela obrigatório (ID, Documento, Tipo, Conteúdo, Fonte, Localização)
- [ ] ≥ 80% dos itens identificáveis dos documentos têm linha
- [ ] ≥ 70% das linhas com Fonte = `TRANSCRICAO` e timestamp válido `[hh:mm] Nome` (verificar via Grep na transcrição)
- [ ] ≥ 5 linhas com Fonte = `CODIGO` e caminho real (verificar via Glob)

## README (`README.md`)

- [ ] Todas as seções: Sobre o desafio; Ferramentas de IA utilizadas; Workflow adotado; Prompts customizados; Iterações e ajustes; Como navegar a entrega
- [ ] ≥ 1 ferramenta de IA listada com o papel de cada uma
- [ ] ≥ 2 prompts customizados em blocos de código
- [ ] ≥ 2 iterações/ajustes concretos descritos

## Consistência geral

- [ ] Nenhum requisito, decisão ou restrição contradiz a transcrição ou o código
- [ ] Nenhum arquivo de código citado nos documentos é inexistente (varrer todos os caminhos citados e verificar via Glob)
- [ ] Nenhum item descartado/adiado na reunião aparece como requisito ou parte da solução
- [ ] Números e valores consistentes entre PRD, RFC, FDD, ADRs e Tracker
- [ ] `src/`, `prisma/`, `tests/`, `TRANSCRICAO.md` e configs intactos (`git status` limpo fora de `docs/`, `README.md` e `.claude/`)
