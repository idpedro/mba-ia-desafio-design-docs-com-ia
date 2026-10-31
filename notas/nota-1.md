# Nota 1: Skills criadas antes do processo

## Por que criar as skills primeiro

- Pedir cada documento com um prompt solto gera saídas diferentes a cada execução e documentos genéricos.
- A skill guarda o **como**: passos, template, limites com os outros documentos, formato de citação e checklist. O prompt traz só o **o quê desta vez**: feature, fontes e pontos de atenção.
- Assim os prompts ficam curtos, e as regras ficam no repositório, versionadas, onde o avaliador consegue ver.

## As 6 skills (`.claude/skills/`)

| Skill | Papel | Saída |
| --- | --- | --- |
| `research` | Coleta de contexto antes de qualquer documento | `docs/research/`: mapeamento do código, inventário da fonte, verificação externa e decisões candidatas |
| `adr` | Uma decisão fechada por arquivo, no formato MADR | `docs/adrs/ADR-NNN-titulo.md` e o índice em `docs/adrs/README.md` |
| `rfc` | Proposta técnica no nível de arquitetura: alternativas e questões em aberto | `docs/RFC.md` |
| `fdd` | Especificação de implementação: fluxos, contratos, erros, observabilidade e integração com o código | `docs/FDD.md` |
| `prd` | Nível de produto: problema, público, escopo, requisitos e métricas | `docs/PRD.md` |
| `tracker` | Liga cada item à origem (`TRANSCRICAO` ou `CODIGO`) e detecta os órfãos | `docs/TRACKER.md` |

Ordem de uso: research → adr → rfc → fdd → prd → tracker.

## Regras comuns a todas

- **Rastreável:** todo item cita a origem, `[hh:mm] Nome` para a transcrição ou o caminho do arquivo para o código. Item sem origem sai do texto ou vira questão em aberto.
- **Um lugar para cada conteúdo:** o RFC não repete o FDD, e PRD e RFC linkam o ADR em vez de explicar a decisão de novo.
- **O que foi descartado ou adiado** aparece só como alternativa descartada, fora de escopo ou questão em aberto, nunca como requisito.
- **Checklist com verificação mecânica:** `grep -n` para os timestamps e `ls` para os caminhos.
- **Autocontidas:** nenhuma skill fala do desafio nem depende de link externo. O contexto específico vai no prompt.

## Como cada uma evoluiu

1. **`adr`:** começou como `madr`. A primeira versão citava o desafio, a reunião e o site do MADR, e foi reescrita para se sustentar sozinha. Depois da comparação com a `create-adr` (tech-leads-club), ganhou título afirmativo, ADR aceito imutável (só pode ser substituído), corpo de 200 a 500 palavras, a Decisão explicando por que vence cada alternativa e no mínimo 2 alternativas. Também foi renomeada para `adr`.
2. **`rfc`:** comparada com a `create-rfc` (tech-leads-club). Trouxe premissas com gatilho de invalidação, critérios de decisão antes das alternativas, "manter como está" como alternativa e riscos com probabilidade, impacto e mitigação. Ganhou um passo próprio de **classificação**, em que cada item da fonte vai para uma única categoria.
3. **`prd` e `fdd`:** inspiradas no conteúdo do curso. O PRD tem bullets, RFs numerados (`RF-001`), objetivos com métrica e meta, e uma checagem de consistência. O FDD tem um template de 15 seções como lista de completude, herda as decisões dos ADRs sem reabri-las e permite detalhe de design só quando segue um padrão que já existe no código.
4. **`tracker`:** IDs derivados dos documentos (`RF-001` vira `PRD-RF-001`), órfãos tratados no documento de origem e um rodapé de cobertura.
5. **`research`:** criada por último e comparada com a skill de research do Matt Pocock.
   - Ganhou **subagentes em paralelo** (código e fonte) para não encher o contexto principal.
   - Passou a exigir a citação da **fonte dona**: no código, o arquivo que define o elemento, não um lugar que só o usa.
   - Ganhou a **verificação externa**. A primeira ideia era usar fontes externas para reforçar os ADRs, e foi recusada: isso seria justificar a decisão depois do fato, e o tracker só aceita `TRANSCRICAO` ou `CODIGO`. O formato final checa só as **afirmações factuais feitas na reunião**, com fontes primárias e veredito (Confirmada, Parcial, Desatualizada, Refutada ou Inconclusiva). Divergência vira questão em aberto, e a decisão continua sendo a do time. O exemplo que motivou: "MinIO é o mais usado e compatível" pode ser uma meia-verdade se o projeto mudou de situação.
   - A regra foi propagada: no `adr`, referência externa entra só em "Mais Informações"; no `tracker`, referência externa não vira linha.

## Referências usadas para escrever as skills

- `writing-for-agents` (mattpocock-skills): critério de conclusão por passo, *leading words*, instruções positivas, um lugar para cada conteúdo e poda do que não muda comportamento.
- `create-adr` e `create-rfc` ([tech-leads-club/agent-skills](https://github.com/tech-leads-club/agent-skills)).
- `research` (mattpocock-skills).
- Conteúdo do curso sobre PRD, FDD, deep research e mapeamento de ADRs, usado como inspiração sem ser citado nas skills.

## Commit

- Branch `docs/design-docs-skills`, commit `4e5aaba`, com as 6 skills na versão final, incluindo a verificação externa.
