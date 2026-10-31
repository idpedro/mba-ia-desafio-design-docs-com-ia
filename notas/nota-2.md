# Nota 2: Execução do processo (research → tracker)

## Como cada etapa rodou

- **Duas sessões:** uma de **execução**, que roda as skills, e outra de **revisão**, que confere cada saída contra a transcrição, o código e os critérios do enunciado.
- **`/clear` entre um documento e outro.** Cada etapa lê o que precisa dos arquivos (`docs/research/`, `docs/adrs/`…), não da conversa.
- **Correções na mesma sessão da geração**, antes do `/clear`.
- **Prompts guardados na raiz:** `prompt-1.md` (research), `prompt-ard.md` (ADR), `prompt-rfc.md`, `prompt-fdd.md`, `prompt-prd.md`, `prompt-tracker.md` e `prompt-revisao.md`.
  - Os prompts de **research** e de **ADR** são meus.
  - Os de **RFC, FDD, PRD, tracker e revisão** foram estruturados com ajuda da IA a partir do que a revisão de cada etapa encontrou.
- **Revisão automática em cada documento:** existência de cada `[hh:mm] Nome` na transcrição, existência de cada caminho de código, links entre os documentos e uma busca por termos que não deveriam aparecer (jitter, CDC, lock, "plausível").

## Commits (branch `docs/design-docs-skills`)

| Commit | Conteúdo |
| --- | --- |
| `4e5aaba` | As 6 skills |
| `392d7d9` | 8 ADRs e o índice em `docs/adrs/README.md` |
| `0fef591` | RFC e `docs/research/` (o RFC aponta para o research) |
| `6a792b7` | FDD |
| `6f8d350` | PRD |

## Etapa 1: Research

**O prompt teve 5 versões**, até a final em `prompt-1.md`:

1. "Mapeamento da demanda… produzir documentos que auxiliem a tomada de decisão". **Problema:** as decisões já tinham sido tomadas na reunião. Pedir ajuda para decidir convida a IA a reabrir ou inventar decisões. A versão também não dizia o que filtrar.
2. "Criar documentos contendo as decisões". **Problema:** isso é o resultado final, não o research. Ficou parecido com o "gere um PRD a partir da transcrição" que o enunciado critica.
3. "Levantar e organizar o contexto…". O verbo ficou certo, mas sobrou "detalhamento técnico", que convida a IA a completar o que ninguém disse.
4. A pergunta "então nem preciso da skill?" me levou a entender a divisão: a skill é o **como**, o prompt é o **o quê desta vez**. O prompt podia ser mais curto.
5. **Final:** `/research`, as categorias (decisões, restrições, RF, requisitos de negócio, descartados/adiados) e "referências à codebase e à pessoa e ao momento de cada fala, sem modificar nada".

**O que o research encontrou** (`docs/research/`):
- **Fala × código:**
  - o estoque só muda em PENDING→PAID e no cancelamento, e não a cada mudança de status;
  - não existe vínculo entre `User` e `Customer`, mesmo com "usuários que representam o cliente";
  - as rotas ficam sob `/api/v1`;
  - o middleware só trata os erros Prisma P2002 e P2025;
  - `POST /auth/register` é público e aceita `role: 'ADMIN'`.
- **Verificação externa:** o `X-Timestamp` não protege contra replay, porque o HMAC cobre só o corpo. Também apareceram "99% dos casos" sem fonte, o retry quebrando o ordering mesmo com worker único e o `z.string().url()` aceitando `http`.
- **Correções no meio da reunião:** `customer_id` "implícito do JWT" virou "não vem do JWT", e "mesmo Prisma client" virou "PrismaClient separado".

## Etapa 2: ADRs

**O prompt passou por várias versões** até `prompt-ard.md`:
- **Primeira tentativa:** um passo intermediário só para listar os ADRs. O problema foi que a IA decidia o corte sem critério.
- **Erro recorrente:** colar trechos do enunciado no prompt (formato MADR, "pelo menos 2 itens", "5 das 6", checklist de aceite). Isso enfraquecia a skill, que exige 2 alternativas, e abria uma brecha.
- **A brecha:** o research classificou o **reuso dos padrões** como "detalhe", porque ninguém discutiu alternativas. Com "no mínimo 5 das 6", a IA podia deixar de fora uma das decisões obrigatórias.
- **Correção:** "todas as 6 decisões" e a linha "o reuso aparece no research como 'detalhe', mas é uma decisão do time ([09:30] Larissa) e deve virar ADR", mais "Justifique cada escolha".

**Resultado:** 8 ADRs.
- 001 outbox, 002 worker com polling, 003 worker único e ordering, 004 retry com DLQ, 005 HMAC, 006 at-least-once, 007 reuso, 008 snapshot.
- Ficaram para o FDD: headers, timeout, 64KB, https, campos do payload, filtro na inserção e códigos `WEBHOOK_`.
- A autorização do CRUD ficou como risco no RFC.

**Correções depois da revisão:**
- **ADR-007:** as alternativas "serviço separado" e "bibliotecas próprias" foram criadas como plausíveis, mas o texto as apresentava como se tivessem sido discutidas. Foram marcadas como "plausível, não discutida na reunião".
- **Arquivos novos:** `src/worker.ts` e `src/modules/webhooks` apareciam sem indicação de que ainda não existem. Foram marcados como "novo, a criar", porque o enunciado proíbe citar arquivos inexistentes.
- **`docs/adrs/README.md`:** dizia `0001-titulo.md`. Foi alinhado ao formato `ADR-NNN-titulo.md`.

**Aceito sem mudar:**
- Os ADRs têm de 500 a 650 palavras, acima do limite de 500 da skill. O limite não previa a seção de referências, e o enunciado não fixa tamanho.
- A data é a de redação, porque a transcrição só diz "quinta-feira".

## Etapa 3: RFC

**Pontos específicos do prompt:**
- só alternativas discutidas na reunião;
- questões em aberto vindas da reunião e das divergências da verificação, como o X-Timestamp;
- a autorização do CRUD como risco;
- uma linha com link para cada ADR.

**Resultado:** 5 alternativas, todas discutidas (síncrono, Redis Streams, trigger, retry indefinido, "failed" na outbox), 5 questões em aberto com gatilho e riscos com probabilidade, impacto e mitigação.

**Correções:**
- **Risco inventado:** "as três sprints **não cabem** até fim de novembro". A reunião não informa a data nem a duração da sprint. Foi reescrito como incerteza, com o Marcos confirmando o prazo com a Atlas ([09:47] Marcos).
- **Arquivo novo:** `src/modules/webhooks` foi marcado como "a criar".
- **Revisores:** a Larissa era autora e revisora ao mesmo tempo. Ficou só como autora, porque em [09:50] ela é quem diz que vai abrir o documento de design.

## Etapa 4: FDD

**Pontos específicos do prompt:**
- todos os endpoints discutidos, sob `/api/v1`;
- erros `WEBHOOK_` estendendo `src/shared/errors/`;
- descrever o código como ele é quando diverge da fala;
- retry como envio inicial mais 5 retentativas, porque "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego) só fecha assim;
- sugestões que a reunião não discutiu entram só como risco.

**Resultado:**
- as 15 seções;
- 8 endpoints mais a chamada de saída, com 10 exemplos JSON;
- 13 códigos `WEBHOOK_`;
- 16 arquivos reais na integração, com os novos marcados como "A criar";
- métricas derivadas de logs. O tracing foi resolvido por correlação de ids, sem lib nova, para manter o ADR-007.

**Correções:**
- **"Redirects não são seguidos":** era uma regra sem origem. Ganhou a justificativa de que seguir um redirect pode levar o envio para `http` e furar o https obrigatório ([09:23] Sofia).
- **"Webhook inativo vai para a DLQ":** marcado como "decisão deste FDD, não discutida na reunião", com a justificativa.

## Etapa 5: PRD

**Pontos específicos do prompt:**
- nível de produto, com decisões em uma linha e link para o ADR;
- todos os itens fora de escopo com o motivo;
- a métrica de menos de 10s ([09:02] Marcos);
- o `customer_id` fora do JWT como risco.

**Resultado:** 10 RFs, 4 objetivos com meta numérica (< 10s, 3 de 3 clientes, fim de novembro, ~15h de janela de retry), 10 itens de produto fora de escopo e 9 riscos.

**Correções:**
- **Duplicação:** o Fora de Escopo trazia de novo as 12 alternativas técnicas que já estão no RFC e nos ADRs. Virou uma linha com link, seguindo a regra do enunciado de que "conteúdo duplicado é sinal de algo no lugar errado".
- **Detalhe técnico:** o nome técnico da métrica, que é coisa do FDD, saiu do objetivo.

## Etapa 6: Tracker

**O prompt** pedia a lista de órfãos **antes** de escrever o tracker, e dizia que a verificação externa não vira linha.

**Órfãos encontrados:** 7 itens que já tinham passado pelas revisões anteriores.
1. **"O Time de Plataforma opera o worker"** (PRD e RFC). Ninguém atribuiu isso a um time. O texto foi reescrito, e "quem opera o worker" virou questão em aberto.
2. **Meta "3 de 3 clientes com webhook ativo"** (PRD). A reunião traz o pedido dos três clientes, não uma meta de adoção. A meta saiu.
3. **Recusar uma segunda rotação de secret dentro das 24h** (FDD). Foi marcado como "decisão deste FDD, não discutida", e virou a Q-13.
4. **Recusar replay de item já reprocessado** (FDD). Mesma marcação, e virou a Q-14.
5. **`GET /webhooks/:id`** (FDD). Ninguém pediu. Foi mantido como "proposta deste FDD", com origem CODIGO no padrão `GET /orders/:id` (`src/modules/orders/order.routes.ts`).
6. **Alternativas não discutidas e sem aviso** no ADR-005 e no ADR-006. Foram marcadas como plausíveis, como no ADR-007.
7. **Fontes que apontavam só para outro documento.** Foram trocadas pela fala ou pelo arquivo de origem.

**Resultado:**
- 321 linhas e 283 itens, com 100% de cobertura nos 4 documentos.
- Por fonte: TRANSCRICAO 264 (82%) e CODIGO 57 (18%).
- A conferência mecânica passou: todos os timestamps existem, todos os caminhos CODIGO existem e não há IDs duplicados.

**Para o README:** é a iteração mais forte. O tracker funcionou como detector de alucinação, que é o papel que o enunciado dá a ele.

## Lições para o README

- **O prompt carrega o que é específico da execução, e a skill carrega as regras.** Colar o enunciado no prompt duplica e às vezes contradiz a skill.
- **Um prompt não pode conter o que só se sabe depois de ler a fonte.** O prompt do research foi escrito só com o enunciado e o código.
- **A IA tende a apresentar o "plausível" como "discutido"**, e a transformar uma inferência em fato (o prazo). A revisão pegou os dois casos.
- **Quando a fala da reunião contradiz o código**, o documento descreve o código e registra o conflito.
- **A verificação externa** não justifica as decisões, mas encontra falhas nelas. O caso concreto é o X-Timestamp.
- **6 ciclos principais** (research, ADR, RFC, FDD, PRD, tracker), cada um com geração, revisão e correção. O tracker ainda forçou uma rodada de correção em todos os documentos.
