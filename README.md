# Design Docs com IA: Sistema de Webhooks de Notificação de Pedidos

> Enunciado original: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)

## Sobre o desafio

Um OMS em produção (Node.js, TypeScript, Prisma e MySQL) vai ganhar notificação de mudança de status de pedido por webhook, a pedido de três clientes B2B. As decisões foram tomadas numa reunião de ~55 minutos, e o único registro dela é a transcrição (`TRANSCRICAO.md`).

A tarefa é transformar essa transcrição e o código existente num pacote de design docs: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade. Cada documento fica numa altura diferente, e todo item precisa ter origem na transcrição ou no código. A IA produziu o texto. O meu papel foi definir o processo, escrever os prompts, revisar cada saída e corrigir o que veio inventado, fora do lugar ou em contradição com o código.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code, sessão de execução** | Rodou as skills (`/research`, `/adr`, `/rfc`, `/fdd`, `/prd`, `/tracker`) e aplicou as correções |
| **Claude Code, sessão de revisão** | Criou as skills comigo e revisou cada artefato contra a transcrição, o código e os critérios do enunciado, com conferência mecânica de timestamps, caminhos e links |
| **Subagentes do Claude Code** | Na skill `research`: mapeamento do código, inventário da transcrição e verificação externa em paralelo |
| **Skill `writing-for-agents`** (mattpocock-skills) | Boas práticas para escrever as skills |
| **Skills `create-adr` e `create-rfc`** ([tech-leads-club/agent-skills](https://github.com/tech-leads-club/agent-skills)) e **`research`** (mattpocock-skills) | Referências comparadas com as nossas skills |

## Workflow adotado

1. **Skills antes dos documentos.** Criei uma skill por artefato em [`.claude/skills/`](.claude/skills/): `research`, `adr`, `rfc`, `fdd`, `prd` e `tracker`. A skill guarda o *como* (passos, template, limites com os outros documentos, citação `[hh:mm] Nome` ou caminho, checklist). O prompt traz só o *o quê desta vez*.
2. **Research como base.** Gerou [`docs/research/`](docs/research/):
   - o mapeamento do código;
   - o inventário da transcrição, com correções e conflitos fala × código;
   - a verificação externa das afirmações factuais da reunião;
   - as decisões candidatas a ADR.
3. **Ordem dos documentos:** ADRs → RFC → FDD → PRD → Tracker → README.
4. **Duas sessões:** uma executa e a outra revisa. Uso `/clear` entre um documento e outro, porque cada etapa lê o que precisa dos arquivos. As correções acontecem na mesma sessão da geração.
5. **Revisão de cada documento:** conferência mecânica de citações, caminhos e links, e leitura crítica contra a transcrição e o código.

## Prompts customizados

Os dois prompts abaixo são meus. Os de RFC, FDD, PRD e tracker (`prompt-rfc.md`, `prompt-fdd.md`, `prompt-prd.md` e `prompt-tracker.md`) foram estruturados com ajuda da IA, a partir do que a revisão de cada etapa encontrou.

**Research** ([`prompt-1.md`](prompt-1.md)). Escrevi antes de ler a reunião, então ele só usa o que o enunciado e o código já diziam. O foco é extrair e filtrar, sem modificar nada.

```text
/research

Seu objetivo é levantar e organizar o contexto da @TRANSCRICAO.md, contendo as decisões tomadas pelo time para a feature de Sistema de Webhooks de Notificação de Pedidos.

O resultado produzido deve espelhar:
- decisões
- restrições
- requisitos funcionais
- requisitos de negócio
- pontos descartados ou adiados

O resultado deve conter referências à codebase e à pessoa e ao momento de cada fala, sempre buscando trazer o contexto sem modificar nada.

Deve documentar, mas não se aprofundar, no que foi descartado ou adiado.
```

**ADRs** ([`prompt-ard.md`](prompt-ard.md)). Escrevi depois de revisar o research. A linha sobre o reuso corrige uma classificação do próprio research, que tinha tratado uma das 6 decisões obrigatórias como "detalhe".

```text
/adr

analise o documento @docs/research/decisoes-candidatas.md e sintetize em 5 a no maximo 8 adrs seguindo as regas:

As Adrs devem conter as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 2 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

O conjunto de ADRs deve cobrir todas as 6 decições pincipais. 

- Padrão Outbox no MySQL
- Política de retry com backoff e DLQ
- Autenticação HMAC-SHA256 com secret por endpoint
- Garantia at-least-once com X-Event-Id
- Worker em processo separado em polling
- Reuso dos padrões existentes do projeto

O reuso dos padrões aparece no research como "detalhe", mas é uma decisão do time ([09:30] Larissa) e deve virar ADR.

Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais ou ficar apenas no FDD, conforme você considerar mais adequado.

Justifique cada escolha.
```

## Iterações e ajustes

Foram **6 ciclos principais** (research, ADR, RFC, FDD, PRD e tracker), cada um com geração, revisão e correção. O tracker ainda forçou uma rodada final de correção em todos os documentos. O detalhe está em [`notas/nota-1.md`](notas/nota-1.md) e [`notas/nota-2.md`](notas/nota-2.md).

**Skills**
1. A primeira skill de ADR citava o desafio e o site do MADR. Foi reescrita para se sustentar sozinha, e o contexto passou a ir no prompt.
2. A verificação externa foi questionada antes de entrar. Usá-la para reforçar decisões seria justificar depois do fato. Ela ficou restrita a checar as afirmações factuais da reunião, com veredito, e uma divergência vira questão em aberto.

**Prompts**
3. O prompt do research teve 5 versões. Começou pedindo documentos "que auxiliem a tomada de decisão", o que convidava a IA a reabrir decisões, e terminou pedindo coleta, com a pessoa e o momento de cada fala, sem modificar nada.
4. No prompt dos ADRs, colar trechos do enunciado ("no mínimo 5 das 6") abria espaço para a IA deixar de fora o reuso de padrões. A correção foi "todas as 6", mais a citação [09:30] Larissa.

**Research**
5. O cruzamento entre a fala e o código achou contradições:
   - o estoque só muda em PENDING→PAID e no cancelamento;
   - não existe vínculo entre `User` e `Customer`;
   - as rotas ficam sob `/api/v1`;
   - o registro público aceita `role: 'ADMIN'`.

   Os documentos descrevem o código e registram o conflito.
6. A verificação externa encontrou uma falha no desenho: o `X-Timestamp` não protege contra replay, porque o HMAC cobre só o corpo. O ponto virou questão em aberto para a revisão de segurança.

**Documentos**
7. **ADRs:**
   - alternativas plausíveis apareciam como se tivessem sido discutidas e foram marcadas;
   - arquivos novos (`src/worker.ts`, `src/modules/webhooks`) ganharam a marcação "a criar".
8. **RFC:** a IA afirmou que "as três sprints não cabem até fim de novembro", mas a reunião não informa data nem duração da sprint. Foi reescrito como incerteza.
9. **FDD:** regras de design sem origem (não seguir redirects, webhook inativo ir para a DLQ) foram justificadas ou marcadas como "decisão deste FDD, não discutida na reunião".
10. **PRD:** o Fora de Escopo repetia as 12 alternativas técnicas do RFC e dos ADRs. Virou um link.
11. **Tracker:** encontrou 7 itens sem origem que tinham passado pelas revisões anteriores. Entre eles, uma meta de adoção "3 de 3 clientes", "o Time de Plataforma opera o worker" e duas regras de 409 no FDD. Todos foram corrigidos ou marcados antes do tracker ser escrito. O resultado ficou em 100% de cobertura, 82% das linhas com fonte TRANSCRICAO e 57 linhas com fonte CODIGO.

## Como navegar a entrega

| Ordem | Arquivo | O que é |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](docs/PRD.md) | Problema, público, escopo, requisitos e métricas |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta técnica, alternativas descartadas e questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/README.md) | 8 decisões, uma por arquivo |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Especificação de implementação e integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | Origem de cada item na transcrição ou no código |
| — | [`docs/research/`](docs/research/) | Artefatos intermediários do research |
| — | [`.claude/skills/`](.claude/skills/) | Skills usadas para produzir cada documento |
| — | [`notas/`](notas/) e `prompt-*.md` | Notas do processo e prompts usados |
