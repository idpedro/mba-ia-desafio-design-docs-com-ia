---
name: tracker
description: Tracker de rastreabilidade — montar ou atualizar a tabela que liga cada item dos design docs à sua origem na transcrição ou no código, apontando o que não tem origem.
---

# Tracker de rastreabilidade

O tracker é uma **referência cruzada**: para cada item dos design docs (PRD, RFC, FDD e ADRs), ele mostra de onde o item veio. Também serve de detector de **alucinação**: um item sem origem identificável foi inventado e precisa ser corrigido no documento de onde veio, não maquiado no tracker.

## Formato

```markdown
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RF-001 | `docs/PRD.md` | Requisito Funcional | {resumo em uma linha} | TRANSCRICAO | [hh:mm] Nome |
| FDD-INT-01 | `docs/FDD.md` | Integração | {resumo em uma linha} | CODIGO | `caminho/do/arquivo.ts` |
```

| Coluna | Regra |
| --- | --- |
| **ID** | O ID que o documento já dá ao item, prefixado pelo documento (`RF-001` no PRD vira `PRD-RF-001`). Se o documento não numera o item, crie no padrão `DOC-TIPO-NN`: `RFC-ALT-01`, `RFC-QA-01`, `FDD-CONTRATO-01`, `FDD-ERRO-01`, `FDD-INT-01`. Cada ADR é o próprio número: `ADR-001`. |
| **Documento** | O caminho do arquivo onde o item aparece. |
| **Tipo** | Um destes: Requisito Funcional, Requisito Não Funcional, Objetivo, Escopo, Fora de Escopo, Decisão, Alternativa Descartada, Questão em Aberto, Restrição, Trade-off, Risco, Critério de Aceite, Contrato, Erro, Integração. |
| **Conteúdo** | Uma linha. |
| **Fonte** | `TRANSCRICAO` ou `CODIGO`. |
| **Localização** | Para `TRANSCRICAO`, `[hh:mm] Nome`. Para `CODIGO`, o caminho do arquivo. Várias origens vão separadas por `; `, com a principal primeiro. |

Referências externas são respaldo, não origem: elas não viram linha. A origem do item continua sendo a fala ou o código que o trouxe.

Quando o item nasce da transcrição e toca o código, a linha principal é a da transcrição. Quando a conexão com o código é o que define o item, acrescente uma segunda linha com Fonte `CODIGO`.

## Passos

1. **Inventariar.** Percorra cada documento e liste todo item identificável: requisitos, objetivos, itens de escopo e de fora de escopo, decisões, alternativas, questões em aberto, restrições, riscos, critérios de aceite, contratos, códigos de erro, pontos de integração e cada ADR. Termina com uma contagem de itens por documento.
2. **Localizar a origem.** Para cada item, encontre o trecho exato com `grep -n` na transcrição, ou o arquivo no código. Use o falante que disse ou fechou o ponto, não quem só perguntou. Termina quando cada item tem uma localização ou está marcado como **órfão**.
3. **Tratar os órfãos.** Liste os órfãos para o usuário com a correção sugerida no documento de origem: remover o item, reescrever conforme a fonte, ou mover para questão em aberto. Termina quando o usuário decidiu cada órfão e o documento foi corrigido.
4. **Escrever** `docs/TRACKER.md`: um parágrafo curto sobre o propósito, a tabela ordenada por documento (PRD, RFC, FDD e ADRs em ordem numérica) e depois por ID, e um rodapé com a cobertura.
5. **Verificar** cada item do checklist. Termina quando todos passam.

## Rodapé de cobertura

```markdown
## Cobertura

| Documento | Itens identificados | Itens rastreados | Cobertura |
| --- | --- | --- | --- |

Linhas por fonte: TRANSCRICAO {n} ({%}) · CODIGO {n} ({%})
```

## Checklist

- [ ] Cada item do inventário tem uma linha, ou é um órfão já resolvido no documento.
- [ ] Cada ID é único e bate com o ID usado no documento.
- [ ] Toda localização `[hh:mm] Nome` existe na transcrição exatamente com aquele timestamp e falante (confira cada uma com `grep -n "\[hh:mm\] Nome"`).
- [ ] Todo caminho com Fonte `CODIGO` existe (confira cada um com `ls`).
- [ ] O conteúdo de cada linha diz o mesmo que o trecho citado. Valor, número e sentido conferem.
- [ ] O rodapé de cobertura reflete as contagens reais.
