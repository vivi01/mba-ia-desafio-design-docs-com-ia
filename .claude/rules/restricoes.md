# Restrições absolutas

## Não alterar (nunca, em nenhuma hipótese)

- `src/` — o código é contexto e referência, não entregável
- `prisma/` — schema, migrations e seed
- `tests/` — suíte de testes existente
- `TRANSCRICAO.md` — fonte primária do desafio
- Arquivos de configuração do boilerplate (`package.json`, `tsconfig*.json`, `docker-compose.yml`, `.eslintrc.json`, `.prettierrc`, `.env.example`, etc.)

Se uma tarefa parecer exigir mudança nesses arquivos, a tarefa está errada — o entregável é **puramente documental**. Documentar como o código *seria* estendido (ex.: "o `changeStatus` passará a inserir na `webhook_outbox` dentro da mesma transação") é o objetivo; alterar o código, não.

## Proibição de invenção (anti-alucinação)

Toda informação registrada nos documentos deve ser rastreável a uma das duas fontes:

- **`TRANSCRICAO`** — com timestamp e falante no formato `[hh:mm] Nome`
- **`CODIGO`** — com caminho de arquivo real e existente no repositório

Regras derivadas:

1. **Não inventar** requisitos, decisões, restrições, métricas, nomes de clientes, prazos ou números que não estejam na transcrição ou no código.
2. **Não referenciar arquivos inexistentes.** Antes de citar um caminho em qualquer documento, confirme que ele existe (Glob/Read). Caminhos *futuros* (ex.: `src/modules/webhooks/webhook.service.ts`) só aparecem no FDD claramente marcados como "a criar".
3. **Não contradizer as fontes.** Se um documento afirmar algo, a transcrição ou o código deve sustentar exatamente aquilo — não uma versão "melhorada".
4. **Teste do tracker:** se não for possível preencher a coluna Localização para um item, o item foi inventado. Corrija a origem ou remova o item.
5. **O que foi descartado ou adiado na reunião NÃO vira requisito.** Itens explicitamente descartados/adiados aparecem apenas em: "Fora de escopo" (PRD), "Alternativas consideradas" e "Questões em aberto" (RFC), "Alternativas Consideradas" (ADRs) e "Escopo e exclusões" (FDD). Se um desses itens aparecer como requisito funcional ou parte da solução, é erro grave.
6. Detalhes que a reunião deixou explicitamente sem decisão permanecem como **questão em aberto** — não decida pelo time.

## Higiene de conteúdo

- Nada de frases genéricas de preenchimento ("garantir alta qualidade", "seguir boas práticas") sem conteúdo concreto por trás.
- Números, timeouts, limites e códigos citados nos documentos devem bater entre si (PRD ↔ RFC ↔ FDD ↔ ADRs ↔ Tracker). Ao mudar um valor em um documento, propague para os demais e para o tracker.
