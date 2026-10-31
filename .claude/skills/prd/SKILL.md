---
name: prd
description: PRD (Product Requirements Document) de feature — escrever ou revisar o documento de produto que define problema, público, escopo, requisitos e métricas.
---

# PRD de feature

Um PRD de feature fica no **nível de produto**. Ele responde *por que construir e o quê*: o problema, para quem, o que entra e o que fica fora, os requisitos e como medir o sucesso. A escrita é em bullets curtos e densos, com cada ideia isolada para ficar fácil de citar. Cada tipo de conteúdo tem um único lugar:

| Conteúdo | Onde fica |
| --- | --- |
| Problema, público, objetivos, escopo, requisitos, aceitação | PRD |
| Abordagem técnica e alternativas | RFC, e o PRD só põe o link |
| Por que uma decisão fechada é assim | ADR, e o PRD só põe o link |
| Endpoints, payloads, tabelas, códigos de erro | Documento de design, e o PRD só põe o link |

Todo PRD é **rastreável**: cada requisito, número, escopo e risco cita sua fonte.

| Fonte | Formato da citação |
| --- | --- |
| Transcrição de reunião | `[hh:mm] Nome` |
| Código | caminho do arquivo, ex.: `src/modules/orders/order.service.ts` |
| Outro documento | link relativo, ex.: `[ADR-001](adrs/ADR-001-....md)` |

Um valor que a fonte não traz vira questão em aberto em Considerações Gerais, ou sai do texto. Requisito é só o que a fonte pediu. O que ela descartou ou adiou vai para Fora de Escopo.

## Passos

1. **Levantar as fontes.** Leia a fonte inteira e os documentos que já existem (RFC, ADRs e documento de design), além do código que define o estado atual. Termina quando você tem a lista de pedidos, números, limites, exclusões e adiamentos da fonte, cada um com sua citação.
2. **Classificar.** Coloque cada item em uma única seção do template. Pedido do usuário ou do negócio vira RF. Restrição de qualidade (latência, limite, segurança, disponibilidade) vira RNF. Descartado ou adiado vai para Fora de Escopo. Decisão técnica fechada vira uma linha em Decisões, com link para o ADR. Termina quando cada item tem uma seção.
3. **Escrever** em `docs/PRD.md`, ou no caminho indicado pelo usuário, seguindo o template.
4. **Checar a consistência.** Os objetivos cabem no escopo? Cada critério de aceitação aponta para um RF ou RNF? Algum item fora de escopo reaparece como requisito? Termina quando cada contradição foi corrigida, ou anotada como questão em aberto.
5. **Verificar** cada item do checklist. Termina quando todos passam.

## Template

```markdown
# PRD: {Nome da feature}

| Campo | Valor |
| --- | --- |
| Status | Rascunho / Em revisão / Aprovado |
| Data | AAAA-MM-DD |
| Responsável de produto | {nome} |
| Stakeholders | {Nome (papel)}, … |
| Relacionados | [RFC](RFC.md), [Design](FDD.md), [ADRs](adrs/README.md) |

## 1. Resumo e Contexto

{3 a 5 linhas: o que a feature faz, por que existe e que partes do sistema ela toca.}

## 2. Problema e Motivação

- {Dor atual, com dado concreto e citação}
- {Por que agora: prazo, risco, pedido formal}

## 3. Público-alvo e Cenários de Uso

**Público:** {quem usa ou é afetado, incluindo papéis internos}

**Cenários:**
- **CEN-01 — {nome}:** {quem faz o quê e com que resultado}

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta | Fonte |
| --- | --- | --- | --- |
| {resultado esperado} | {indicador observável} | {valor numérico} | {citação} |

## 5. Escopo

**Incluso:**
- {capacidade}

**Fora de escopo:**
- {item} — {descartado / adiado / outro time}, {citação}

## 6. Requisitos Funcionais

### RF-001 — {nome}

- **Descrição:** {capacidade observável, em uma frase}
- **Fluxo principal:** {passos curtos}
- **Alternativas e erros:** {comportamento esperado}
- **Prioridade:** Alta / Média / Baixa
- **Fonte:** {citação}

## 7. Requisitos Não Funcionais

| ID | Categoria | Requisito | Fonte |
| --- | --- | --- | --- |
| RNF-001 | Desempenho / Segurança / Confiabilidade / … | {restrição verificável} | {citação} |

## 8. Decisões e Trade-offs

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| {decisão em uma linha} | {o que se perde e em troca de quê} | [ADR-NNN](adrs/ADR-NNN-....md) |

## 9. Dependências

- {Sistema, módulo, time ou decisão prévia} — {por que a feature depende disso}, {citação}

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação | Fonte |
| --- | --- | --- | --- | --- |
| {risco concreto da feature} | Alta / Média / Baixa | Alto / Médio / Baixo | {ação} | {citação} |

## 11. Critérios de Aceitação

- [ ] **CA-01** ({RF-001}): {condição verificável}

## 12. Estratégia de Testes e Validação

- **Automatizados:** {o que cada tipo de teste cobre, em ligação com os RFs}
- **Validação com usuários:** {piloto, clientes, revisão de segurança}
- **Pós-lançamento:** {como as métricas da seção 4 serão acompanhadas}

## 13. Considerações Gerais

- {Nuance que não cabe acima, ou questão em aberto com citação}
```

**Fluxo do usuário** entra como uma seção a mais, depois dos Cenários, quando a ordem de uso entre os RFs importa.

## Checklist

- [ ] As seções de 1 a 12 existem e estão preenchidas.
- [ ] Cada RF tem ID `RF-NNN`, descrição observável, prioridade e fonte citada.
- [ ] Cada RNF é verificável: tem número, limite ou condição testável.
- [ ] Pelo menos um objetivo tem métrica e meta numérica vindas da fonte.
- [ ] Fora de Escopo lista todo item que a fonte descartou ou adiou, cada um com citação.
- [ ] Cada risco tem probabilidade, impacto e mitigação.
- [ ] Cada critério de aceitação aponta para um RF ou RNF.
- [ ] Decisões técnicas aparecem em uma linha com link para o ADR, e o detalhe de implementação fica no documento de design.
- [ ] Todo `[hh:mm] Nome` citado existe na transcrição com aquele falante (confira com `grep -n`), e todo caminho de código existe (confira com `ls`).
