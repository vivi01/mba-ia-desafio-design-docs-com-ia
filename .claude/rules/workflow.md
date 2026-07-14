# Workflow de produção

## Ordem de execução

1. **Contextualização** — ler `TRANSCRICAO.md` na íntegra e mapear o código (estrutura modular, `changeStatus`, máquina de estados, classes de erro, `requireRole`, error middleware, logger Pino). Produzir três listas de trabalho a partir da transcrição:
   - decisões fechadas (com timestamp de onde foram batidas o martelo);
   - requisitos explícitos (funcionais e não funcionais);
   - itens **descartados ou adiados** (com timestamp) — esta lista protege o "Fora de escopo" e as "Questões em aberto".
2. **ADRs primeiro** — as decisões formam o esqueleto. Usar `.claude/templates/adr.md`.
3. **RFC** — consolida a proposta em cima dos ADRs; alternativas descartadas e questões em aberto moram aqui. Referenciar os ADRs já escritos. Usar `.claude/templates/rfc.md`.
4. **FDD** — o desenho técnico se constrói sobre as decisões formalizadas. Não esquecer a seção obrigatória "Integração com o sistema existente" (≥ 4 caminhos reais). Usar `.claude/templates/fdd.md`.
5. **PRD** — por último entre os grandes documentos; com RFC, FDD e ADRs prontos, vira consolidação de alto nível. Usar `.claude/templates/prd.md`.
6. **Tracker** — alimentado em paralelo desde o passo 2 (ver `rastreabilidade.md`); no fim, fazer a varredura de cobertura.
7. **README do processo** — substituir o conteúdo de `README.md` pela documentação da jornada. Seções obrigatórias: Sobre o desafio; Ferramentas de IA utilizadas; Workflow adotado; Prompts customizados (≥ 2, em blocos de código); Iterações e ajustes (≥ 2 correções concretas); Como navegar a entrega. Manter link/menção ao enunciado original. **Importante:** registrar durante todo o processo os prompts usados e os momentos em que a geração saiu errada e foi corrigida — esse material é insumo obrigatório do README e não dá para reconstituir depois.
8. **Revisão final** — passar por `.claude/rules/checklist-aceite.md` item por item, rodar a rotina de verificação do tracker e conferir consistência cruzada entre documentos.

## Como trabalhar cada documento

- Gerar por seção ou por documento, **revisar criticamente, ajustar e regenerar**. O desafio espera 3 a 5 ciclos de iteração; um documento aceito na primeira geração provavelmente está genérico.
- Sinais de documento fraco a caçar ativamente: frases que serviriam para qualquer feature de webhooks genérica; requisito sem timestamp possível; seção que repete outra de outro documento; número redondo que ninguém falou na reunião.
- Ao revisar, preferir prompts dirigidos ("liste tudo que foi explicitamente descartado entre 09:00 e 09:30, com timestamp") a pedidos genéricos ("melhore o documento").
- Cada afirmação nova em um documento gera, na mesma sessão, sua linha no tracker.

## Registro do processo (para o README final)

Manter um log de trabalho em `.claude/notas-processo.md` (fora da entrega em `docs/`), anotando a cada sessão:

- prompts relevantes usados (candidatos à seção "Prompts customizados");
- erros/superficialidades geradas pela IA e como foram corrigidas (candidatos à seção "Iterações e ajustes");
- ordem real do que foi produzido.
