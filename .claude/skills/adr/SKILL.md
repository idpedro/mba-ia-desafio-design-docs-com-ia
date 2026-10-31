---
name: adr
description: ADR (Architecture Decision Record) no formato MADR — criar, revisar ou substituir uma decisão em docs/adrs/.
---

# ADR no formato MADR

Um ADR registra **uma** decisão arquitetural fechada: o contexto que forçou a decisão, as opções que estavam na mesa, a escolha feita e o preço pago por ela. Ele responde *por que decidimos exatamente assim*. Tem de 200 a 500 palavras. Detalhes de implementação (payloads completos, matrizes de erro, passo a passo) ficam no documento de design, e o ADR só aponta para lá.

Todo ADR é **rastreável**: cada fato, número, alternativa e consequência cita sua fonte.

| Fonte | Formato da citação |
| --- | --- |
| Transcrição de reunião | `[hh:mm] Nome` |
| Código | caminho do arquivo, ex.: `src/modules/orders/order.service.ts` |
| Outro documento | link relativo, ex.: `[FDD](../FDD.md#retry)` |

Uma afirmação sem fonte vira questão em aberto em "Mais Informações", ou sai do texto. Algo que a fonte descartou ou adiou entra só como alternativa descartada ou como questão em aberto.

Referências externas (documentação oficial, especificações) entram só em "Mais Informações", como respaldo de uma afirmação feita na fonte. Quando a referência diverge do que foi dito (afirmação parcial, desatualizada ou refutada), a divergência vira questão em aberto, e Contexto, Decisão e Consequências continuam refletindo o que o time sabia ao decidir.

Um ADR `Aceito` é um **registro histórico**: a decisão estava certa com o que se sabia na época. Quando a decisão muda, um ADR novo substitui o antigo, e o texto antigo permanece como está.

## Passos

1. **Levantar as fontes.** Leia todo o trecho da fonte em que a decisão foi discutida: a proposta, as objeções, os números, as alternativas levantadas e qualquer resumo ou confirmação final. Abra os arquivos de código que a decisão toca. Termina quando cada item que vai entrar no ADR tem uma citação anotada.
2. **Escolher o arquivo.** Liste `docs/adrs/` e siga o caso que se aplica:
   - **Criar:** o número é o maior `NNN` existente + 1, com 3 dígitos. O arquivo é `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`, com título curto em minúsculas e sem acento.
   - **Revisar um ADR `Proposto`:** edite o mesmo arquivo, mantendo o número e o nome.
   - **Mudar um ADR `Aceito`:** crie um ADR novo pelo caso "Criar", citando o antigo em "Mais Informações". No antigo, altere apenas a linha de Status para `Substituído por [ADR-NNN](...)`.

   Termina com o caminho definido.
3. **Escrever** seguindo o template. Toda seção marcada como obrigatória fica preenchida.
4. **Verificar** cada item do checklist. Termina quando todos passam.
5. **Indexar.** Mantenha a linha de cada ADR criado ou alterado na tabela de `docs/adrs/README.md` (`| ADR | Título | Status |`, com link relativo), criando a tabela se ela não existir.

## Template

Seções obrigatórias: Status, Contexto e Problema, Alternativas Consideradas, Decisão e Consequências. As demais entram quando a fonte tiver material para elas.

```markdown
---
date: AAAA-MM-DD
decision-makers: [quem fechou a decisão]
consulted: [quem opinou]
informed: [quem só foi informado]
---

# ADR-NNN: {Título afirmativo que nomeia a decisão, ex.: "Usar X para Y"}

## Status

{Status} — {onde e quando foi decidido, com citação}.

## Contexto e Problema

{2 a 4 frases sobre a força que obriga a decidir agora, com citação. Pode terminar com a pergunta que a decisão responde.}

## Fatores de Decisão

- {Força ou restrição, com citação}

## Alternativas Consideradas

- {Opção escolhida}
- {Alternativa levantada na fonte, ou plausível no contexto}
- {Outra alternativa}

## Decisão

Opção escolhida: "{título}", porque {justificativa com citação}. {Por que ela vence cada alternativa, ligando o argumento aos Fatores de Decisão.}

{Parâmetros concretos fechados: números, nomes, arquivos.}

## Consequências

- Positiva: {…}
- Negativa: {…}
- Trade-off: {numa frase, o que se aceitou perder e em troca de quê}

### Confirmação

{Como verificar que a implementação segue a decisão: um teste, um ponto de code review, uma revisão específica.}

## Prós e Contras das Alternativas

### {Alternativa}

{Descrição em uma frase, com citação.}

- Bom, porque {…}
- Ruim, porque {…} ← motivo do descarte, com citação

## Mais Informações

- Fontes: {citações}
- Código relacionado: `caminho/real`
- Referências externas: [{título}]({URL}) — {afirmação que respalda, veredito}
- Relacionados: [ADR-NNN](ADR-NNN-....md)
- Substitui / Substituído por: [ADR-NNN](ADR-NNN-....md)
- Questões em aberto e gatilhos para revisitar: {…}
```

**Status** pode ser `Proposto`, `Aceito`, `Rejeitado`, `Depreciado` ou `Substituído por ADR-NNN`. Uma decisão fechada na fonte é `Aceito`.

**`date`** é a data da decisão quando a fonte a informa. Quando não informa, use a data de redação.

## Checklist

- [ ] O título afirma a decisão tomada.
- [ ] Todas as seções obrigatórias existem e estão preenchidas.
- [ ] Há pelo menos duas alternativas além da escolhida, cada uma com o motivo do descarte citado.
- [ ] A Decisão explica por que a opção vence cada alternativa.
- [ ] As Consequências têm pelo menos uma positiva, uma negativa e um trade-off explícito.
- [ ] Todo `[hh:mm] Nome` citado existe na transcrição com aquele falante (confira com `grep -n "\[hh:mm\] Nome"`).
- [ ] Todo caminho de código citado existe (confira com `ls`).
- [ ] Tudo o que a fonte descartou ou adiou aparece só como alternativa descartada ou questão em aberto.
- [ ] O corpo tem entre 200 e 500 palavras, e o detalhe de implementação fica num link para o documento de design.
- [ ] As substituições estão ligadas nos dois sentidos: o ADR novo cita o antigo, e o antigo aponta para o novo.
