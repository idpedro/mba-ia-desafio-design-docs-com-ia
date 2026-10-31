---
name: research
description: Research — coleta de contexto inicial antes de escrever design docs: mapear o código, inventariar uma fonte (transcrição, notas, issue), verificar as afirmações externas contra fontes primárias e levantar as decisões candidatas a ADR.
---

# Research

A pesquisa produz o **contexto de base** que os design docs (PRD, RFC, ADRs, FDD e tracker) consomem depois. Ela **inventaria**, não resume: preserva números, nomes, falantes e nuances da fonte e reorganiza tudo em categorias fáceis de consultar. O que for perdido aqui vira lacuna, ou invenção, nos documentos seguintes.

A saída são quatro arquivos em `docs/research/`, ou no caminho indicado pelo usuário:

| Arquivo | Conteúdo |
| --- | --- |
| `mapeamento-codigo.md` | Retrato do código existente: stack, módulos, padrões e pontos de extensão |
| `inventario-fonte.md` | Cada item da fonte, classificado e com citação |
| `verificacao-externa.md` | Cada afirmação sobre o mundo externo checada contra fontes primárias, com veredito |
| `decisoes-candidatas.md` | As decisões que merecem ADR, com evidências, alternativas, vereditos e perguntas em aberto |

Cada item cita sua origem: `[hh:mm] Nome` para transcrições, o caminho do arquivo para o código e `arquivo#seção` para outros documentos. A citação aponta para a **fonte dona**: na transcrição, quem fechou o ponto; no código, o arquivo onde o elemento é definido (`arquivo.ts:linha`), com um ponto de uso ao lado quando o uso importa.

## Verificação externa

Uma **afirmação verificável** é um fato sobre o mundo fora do repositório que a discussão usa como premissa. Alguns tipos:

- a capacidade ou limitação de uma tecnologia ("o banco X não tem notificação nativa");
- a situação de um projeto ou produto: se ainda é mantido, se foi descontinuado, se mudou de licença ou de distribuição;
- compatibilidade e versões;
- uma prática de mercado ("a empresa Y faz assim");
- um número de desempenho ou de limite.

Preferências do time, decisões e fatos internos ficam de fora: eles são checados contra o próprio código. A verificação olha o **estado atual**, porque o que era verdade na época da conversa pode ter mudado. Por isso ela confere o status de manutenção, a data da última release, mudanças de licença, avisos de depreciação e a alternativa que o próprio mantenedor recomenda.

| Veredito | Quando |
| --- | --- |
| `Confirmada` | A fonte primária sustenta a afirmação como foi dita |
| `Parcial` | Verdadeira em parte: falta uma ressalva que muda o peso do argumento |
| `Desatualizada` | Era verdade, mas mudou. Registre o que mudou e quando |
| `Refutada` | A fonte primária contradiz a afirmação |
| `Inconclusiva` | Não há fonte primária suficiente |

**Fontes primárias** são a documentação oficial, o repositório oficial (README, releases, LICENSE, issues de anúncio), comunicados do mantenedor e especificações (RFC, ISO, W3C). Uma fonte secundária serve só para chegar à primária. Cada evidência registra a URL, a data de acesso e um trecho literal curto.

A verificação **sustenta ou alerta**, e a decisão continua sendo a que o time fechou. Uma afirmação `Parcial`, `Desatualizada` ou `Refutada` vira uma pergunta em aberto e um risco na decisão candidata que se apoia nela. O time decide o que fazer com isso.

## Passos

1. **Delimitar.** Identifique os arquivos de fonte e escreva, em uma frase, a feature a ser documentada. Pergunte ao usuário só quando a fonte não puder ser localizada. Termina com a lista de fontes e a frase da feature.
2. **Despachar dois subagentes em paralelo**, cada um escrevendo um arquivo. Assim a leitura pesada fica fora do contexto principal. Cada subagente recebe um briefing completo: o objetivo do arquivo, o caminho de saída, o template copiado desta skill, a regra da fonte dona e o critério de conclusão dele. Cada um devolve só um resumo, com contagens, ausências, correções e dúvidas.
   - **Subagente de código → `mapeamento-codigo.md`.** Lê os pontos de entrada, a configuração, o schema do banco, um módulo completo de referência e os componentes transversais. Quando algo não existe no código (filas, eventos, cache), registra a ausência como achado. Termina quando cada seção do template tem um caminho real ou uma ausência registrada.
   - **Subagente de fonte → `inventario-fonte.md`.** Lê a fonte do início ao fim, na ordem. Toda fala com conteúdo de produto ou técnico vira um item. Uma fala que muda algo dito antes vira uma **correção** e aponta para o item original. Cada afirmação verificável é listada na seção própria do template. Termina quando a última fala da fonte foi processada e cada item tem categoria e citação.

   O passo termina quando os dois arquivos existem e os dois resumos voltaram.
3. **Despachar o subagente de verificação → `verificacao-externa.md`.** O briefing inclui a seção "Verificação externa" desta skill, o template e a lista de afirmações verificáveis do inventário. O subagente busca na web e lê as fontes primárias. Enquanto ele trabalha, siga para o passo 4. Termina quando cada afirmação tem um veredito com evidência.
4. **Cruzar fonte e código.** Para cada menção da fonte a algo do código (arquivo, método, classe, tabela, middleware), confirme que existe e anote o caminho. Quando a fonte afirma algo que o código desmente, registre um **conflito**. Termina quando cada menção está confirmada ou é um conflito.
5. **Levantar as decisões candidatas** e escrever `decisoes-candidatas.md` pelo template, depois que a verificação voltar. Uma decisão candidata é estrutural, tem efeito duradouro e foi discutida com alternativas ou trade-off. Parâmetros finos, formatos, validações e regras locais ficam como detalhe para o documento de design, a menos que a fonte os trate como decisão própria. Classifique cada uma como `Documentar` (a evidência sustenta um ADR) ou `Considerar` (precisa de julgamento humano). Ligue a cada candidata os vereditos das afirmações em que ela se apoia. Termina quando cada item `Decisão` do inventário virou candidata ou foi apontado como detalhe.
6. **Verificar** cada item do checklist.
7. **Relatar** ao usuário: a contagem de itens por categoria, as candidatas a ADR por classificação, os conflitos e as correções encontradas, e cada afirmação que não ficou `Confirmada`, com a decisão que ela afeta.

## Templates

### `mapeamento-codigo.md`

```markdown
# Mapeamento do Código

## Visão Geral
{Propósito do sistema, em 2 ou 3 linhas.}

## Stack
| Camada | Tecnologia | Evidência |
| --- | --- | --- |

## Estrutura e Módulos
| Módulo | Caminho | Responsabilidade |
| --- | --- | --- |

## Padrões
| Padrão | Como funciona | Arquivo de referência |
| --- | --- | --- |
{Inclua organização de módulo, validação, erros e códigos de erro, autenticação e roles, logging, acesso a dados e transações, respostas HTTP e testes.}

## Fluxos Relevantes para a Feature
{Para cada fluxo que a feature vai tocar: os passos atuais com caminho e linha, por exemplo `arquivo.ts:126`.}

## Pontos de Extensão
| Ponto | Caminho | Como a feature pode se encaixar |
| --- | --- | --- |

## Ausências
- {O que a feature precisa e que não existe hoje}
```

### `inventario-fonte.md`

```markdown
# Inventário da Fonte

| ID | Categoria | Item | Citação | Status |
| --- | --- | --- | --- | --- |
| INV-001 | {categoria} | {o que foi dito, com os números e nomes exatos} | [hh:mm] Nome | {Fechado / Aberto / Corrigido por INV-NNN} |

## Correções
- INV-NNN corrige INV-MMM: {o que mudou}

## Conflitos com o Código
- INV-NNN: {o que a fonte afirma} × `caminho`: {o que o código mostra}

## Afirmações Verificáveis
- INV-NNN: "{a afirmação como foi dita}" — {tipo: capacidade / status do projeto / compatibilidade / prática de mercado / número}
```

As **categorias** são estas: Contexto de Negócio, Decisão, Requisito Funcional, Requisito Não Funcional, Restrição, Alternativa Descartada (com o motivo do descarte), Fora de Escopo, Adiado, Questão em Aberto, Gancho com o Código, Detalhe Técnico, Prazo/Estimativa e Risco. Na Citação, use o falante que fechou o ponto. Quando a proposta e o fechamento vêm de falantes diferentes, cite os dois, separados por `; `.

### `verificacao-externa.md`

```markdown
# Verificação Externa

Data da verificação: AAAA-MM-DD

## VER-001 — {afirmação resumida}

- **Afirmação:** "{como foi dita}" — INV-NNN, [hh:mm] Nome
- **Veredito:** Confirmada / Parcial / Desatualizada / Refutada / Inconclusiva
- **Evidência:** [{título}]({URL}) (acesso em AAAA-MM-DD) — "{trecho literal curto}"
- **O que mudou:** {para Parcial, Desatualizada ou Refutada: a ressalva ou mudança e desde quando}
- **Impacto:** {qual decisão se apoia nisso e o que fica em risco}
```

### `decisoes-candidatas.md`

```markdown
# Decisões Candidatas a ADR

## {Título afirmativo da decisão}

- **Classificação:** Documentar / Considerar
- **Categoria:** {persistência, mensageria, segurança, integração, padrão de código…}
- **O que foi decidido:** {com parâmetros exatos}
- **Evidências:** {INV-NNN, citações e caminhos de código}
- **Verificação externa:** {VER-NNN — veredito}
- **Alternativas levantadas:** {alternativa — motivo do descarte, com citação}
- **Trade-off:** {o que se ganha e o que se perde}
- **Perguntas em aberto:** {inclui cada VER-NNN que não ficou Confirmada}
```

## Checklist

- [ ] Todas as falas com conteúdo da fonte estão no inventário, sem nenhum trecho pulado entre o início e o fim.
- [ ] Os itens preservam os valores exatos da fonte: números, nomes, headers, caminhos e limites.
- [ ] Toda citação `[hh:mm] Nome` existe na fonte com aquele falante (confira com `grep -n`).
- [ ] Todo caminho de código citado existe (confira com `ls`) e é o arquivo que define o elemento, não só um lugar que o usa (confira com `grep -n "export .*Nome"`).
- [ ] Cada item descartado, adiado ou fora de escopo está nessa categoria, com o motivo citado.
- [ ] Cada correção aponta para o item corrigido, e o item corrigido tem status `Corrigido por INV-NNN`.
- [ ] Cada afirmação verificável tem um veredito, e a evidência é uma fonte primária com URL, data de acesso e trecho literal.
- [ ] Cada veredito diferente de `Confirmada` aparece nas Perguntas em aberto da decisão candidata que ele afeta.
- [ ] Cada decisão candidata tem evidência na fonte, e no código quando o código for afetado.
