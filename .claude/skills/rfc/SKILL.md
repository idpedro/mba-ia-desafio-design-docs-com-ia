---
name: rfc
description: RFC (Request for Comments) — escrever ou revisar a proposta técnica de uma feature submetida ao time para revisão.
---

# RFC

Um RFC propõe uma solução no **nível de arquitetura** e a abre para revisão. Ele responde *o que propomos, por quê, o que descartamos e o que ainda está em aberto*. Tem de 1.000 a 2.000 palavras (2 a 4 páginas). Cada tipo de conteúdo tem um único lugar:

| Conteúdo | Onde fica |
| --- | --- |
| Abordagem geral, alternativas, questões em aberto, riscos | RFC |
| Por que uma decisão fechada é assim (contexto e consequências) | ADR, e o RFC só põe o link |
| Fluxos detalhados, payloads, schemas, códigos de erro, timeouts | Documento de design, e o RFC só põe o link |

Todo RFC é **rastreável**: cada fato, número, alternativa, questão e risco cita sua fonte.

| Fonte | Formato da citação |
| --- | --- |
| Transcrição de reunião | `[hh:mm] Nome` |
| Código | caminho do arquivo, ex.: `src/modules/orders/order.service.ts` |
| Outro documento | link relativo, ex.: `[ADR-001](adrs/ADR-001-....md)` |

Uma afirmação sem fonte sai do texto.

## Passos

1. **Levantar as fontes.** Leia a fonte inteira (transcrição, notas ou issue), incluindo resumos e confirmações finais. Abra os arquivos de código que a proposta toca e liste os ADRs em `docs/adrs/`. Termina quando você tem uma lista de itens da fonte, cada um com sua citação.
2. **Classificar.** Coloque cada item da lista em exatamente uma categoria:
   - **Decisão fechada:** vira uma linha na Proposta Técnica, com link para o ADR.
   - **Alternativa descartada:** vai para Alternativas Consideradas, com o trade-off que levou ao descarte.
   - **Adiado ou não decidido:** vai para Questões em Aberto.
   - **Fora de escopo:** entra como uma linha no Contexto ou nas Questões em Aberto, e nunca na Proposta.
   - **Detalhe de implementação:** fica no documento de design.
   - **Restrição, risco ou premissa:** vai para a seção correspondente.

   Termina quando cada item está em uma única categoria. Uma decisão fechada sem ADR é anotada no resumo final como ADR que falta escrever.
3. **Escrever** em `docs/RFC.md`, ou no caminho indicado pelo usuário, seguindo o template.
4. **Verificar** cada item do checklist. Termina quando todos passam.

## Template

```markdown
# RFC: {Título afirmativo da proposta}

| Campo | Valor |
| --- | --- |
| Autor | {quem conduz a proposta} |
| Status | Em revisão |
| Data | AAAA-MM-DD |
| Revisores | {Nome (papel)}, … |
| Impacto | Alto / Médio / Baixo — {justificativa em uma linha} |
| Relacionados | [PRD](PRD.md), [Design](FDD.md), [ADRs](adrs/README.md) |

## Resumo (TL;DR)

{3 a 5 frases: o problema, a proposta e o principal trade-off aceito.}

## Contexto e Problema

{Estado atual, com referência ao código. O problema, com um dado concreto e citação. Por que agora. O custo de não fazer nada.}

## Premissas

| Premissa | Fonte | Gatilho de invalidação |
| --- | --- | --- |
| {o que se assume como verdadeiro} | {citação} | {o que faria revisitar} |

## Critérios de Decisão

| Critério | Peso | Fonte |
| --- | --- | --- |
| {requisito ou restrição} | Obrigatório / Alto / Médio | {citação} |

## Proposta Técnica

{Visão geral em 1 ou 2 parágrafos: componentes e como se conectam.}

{Fluxo de alto nível em 4 a 7 passos numerados, ou num diagrama Mermaid.}

Decisões estruturais:
- {Decisão em uma linha} — [ADR-NNN](adrs/ADR-NNN-....md)

## Alternativas Consideradas

### {Alternativa}

{O que é, em 1 ou 2 frases, com citação.}

**Por que foi descartada:** {o trade-off e o critério que ela não atende, com citação.}

## Questões em Aberto

| Questão | Estado | Gatilho para revisitar | Fonte |
| --- | --- | --- | --- |
| {questão} | Adiado / Não decidido / Em observação | {condição} | {citação} |

## Impacto e Riscos

**Impacto:** {sistemas, times e clientes afetados}.

| Risco | Probabilidade | Impacto | Mitigação | Fonte |
| --- | --- | --- | --- | --- |
| {risco} | Alta / Média / Baixa | Alto / Médio / Baixo | {mitigação} | {citação} |

## Decisões Relacionadas

- [ADR-NNN: Título](adrs/ADR-NNN-....md) — {resumo em uma linha}

## Próximos Passos

{Estimativa, revisões combinadas e marcos citados na fonte.}
```

**Status** pode ser `Rascunho`, `Em revisão`, `Aceito`, `Rejeitado` ou `Retirado`. **Revisores** são os participantes da discussão de origem, com seus papéis. **Próximos Passos** só entra quando a fonte traz estimativas, revisões ou marcos.

## Checklist

- [ ] Os metadados têm autor, status, data e revisores com papel.
- [ ] O TL;DR tem até 5 frases e nomeia o principal trade-off.
- [ ] Os Critérios de Decisão vêm antes da Proposta e das Alternativas.
- [ ] Há pelo menos duas alternativas descartadas na fonte, cada uma com o trade-off do descarte citado. Quando a fonte descreve o estado atual, ele aparece como a alternativa "manter como está".
- [ ] Há pelo menos duas questões em aberto vindas da fonte, cada uma com um gatilho para revisitar.
- [ ] Cada risco tem probabilidade, impacto e mitigação.
- [ ] Cada decisão fechada aponta para seu ADR, e todo link para ADR resolve (confira com `ls docs/adrs/`).
- [ ] Payloads, schemas, códigos de erro e parâmetros finos aparecem só como link para o documento de design.
- [ ] Cada item aparece em uma única categoria. Nada descartado ou adiado aparece na Proposta.
- [ ] Todo `[hh:mm] Nome` citado existe na transcrição com aquele falante (confira com `grep -n`), e todo caminho de código existe (confira com `ls`).
- [ ] O corpo tem entre 1.000 e 2.000 palavras.
