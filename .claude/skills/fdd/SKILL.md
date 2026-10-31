---
name: fdd
description: FDD (Feature Design Document) — escrever ou revisar a especificação de implementação de uma feature: fluxos, contratos, erros, resiliência, observabilidade e integração com o código.
---

# FDD

Um FDD é a **especificação de implementação** de uma feature. Ele faz a ponte entre a arquitetura decidida e o código: como a feature se comporta em runtime, quais contratos expõe, como falha, como é observada e onde se encaixa no código existente. Um desenvolvedor pega o FDD e começa a codar. O nível é de contrato e de comportamento verificável, sem prescrever código linha a linha. Cada tipo de conteúdo tem um único lugar:

| Conteúdo | Onde fica |
| --- | --- |
| Fluxos, contratos, erros, configuração, resiliência, observabilidade, integração | FDD |
| Por que uma decisão é assim | ADR. O FDD herda a decisão, põe o link e detalha como ela se comporta |
| Problema, público, métricas de negócio | PRD, e o FDD só põe o link |

Todo FDD é **rastreável**. Números, limites, nomes, headers e regras citam a fonte em linha. Um detalhe de design que a fonte não traz (um status HTTP, o formato de um campo, o nome de uma métrica) é válido quando segue um padrão existente no código, e cita o arquivo desse padrão. Um detalhe sem fonte e sem padrão vira questão em aberto em Riscos, ou sai do texto.

| Fonte | Formato da citação |
| --- | --- |
| Transcrição de reunião | `[hh:mm] Nome` |
| Código | caminho do arquivo, ex.: `src/shared/errors/http-errors.ts` |
| Outro documento | link relativo, ex.: `[ADR-002](adrs/ADR-002-....md)` |

## Passos

1. **Levantar as decisões.** Leia os ADRs, o RFC e o PRD que existirem. Cada decisão fechada é herdada como está. Termina quando você tem a lista das decisões com o link de cada uma.
2. **Mapear o código.** Leia o código que a feature toca: os pontos de entrada, o módulo de referência mais parecido (controller, service, repository, routes, schemas), as classes e os códigos de erro, o middleware de erro, a autenticação e as roles, o logger, o acesso a banco e as transações, e o schema do banco. Termina quando cada ponto de integração tem um caminho real e uma frase sobre o que existe ali hoje.
3. **Levantar a fonte.** Leia a fonte inteira atrás de números, limites, endpoints, campos, headers, regras e casos de erro. Termina quando cada um tem uma citação.
4. **Escrever** em `docs/FDD.md`, ou no caminho indicado pelo usuário, seguindo o template.
5. **Verificar** cada item do checklist. Termina quando todos passam.

## Template

````markdown
# FDD: {Nome da feature}

| Campo | Valor |
| --- | --- |
| Status | Rascunho / Em revisão / Aprovado |
| Data | AAAA-MM-DD |
| Autor | {nome} |
| Relacionados | [PRD](PRD.md), [RFC](RFC.md), [ADRs](adrs/README.md) |

## 1. Contexto e Motivação Técnica

{O estado atual do código e a lacuna técnica que a feature preenche, com caminhos de arquivo.}

## 2. Objetivos Técnicos

- {Objetivo verificável, com citação}

## 3. Escopo e Exclusões

**Incluso:** {…}
**Excluído:** {item} — {citação}

## 4. Modelo de Dados

{Cada tabela ou entidade nova: campos, tipos, índices, estados. Siga as convenções do schema existente, como o tipo de id e a nomenclatura.}

## 5. Fluxos Detalhados

### 5.1 {Fluxo}

{Passos numerados: gatilho, cada efeito, transação, condição de saída.}

```mermaid
sequenceDiagram
  …
```

{Para entidades com ciclo de vida, um `stateDiagram-v2` com todas as transições e o que dispara cada uma.}

## 6. Contratos Públicos

{Toda interface que outro sistema consome: endpoints expostos e chamadas que a feature faz para fora.}

### 6.N `MÉTODO /caminho`

- **Autenticação/autorização:** {…}
- **Headers:** {…}
- **Request:**
```json
{ … }
```
- **Responses:**
  - `201` — {semântica}
```json
{ … }
```
  - `4xx` — {quando} → `{CODIGO_DE_ERRO}`

## 7. Matriz de Erros

| Código | HTTP | Quando ocorre | Classe base | Mensagem |
| --- | --- | --- | --- | --- |
| `{PREFIXO}_{ERRO}` | 4xx | {condição} | {classe existente estendida} | {texto} |

## 8. Configuração

| Parâmetro | Valor padrão | Fonte |
| --- | --- | --- |

## 9. Estratégias de Resiliência

{Timeouts, retries, backoff com a tabela de tentativas, destino de falha permanente, fallback, concorrência e ordenação, idempotência.}

## 10. Segurança

{Autenticação das chamadas, assinatura, gestão de segredos, validação de entrada, dados que nunca vão para logs.}

## 11. Observabilidade

- **Métricas:** | Nome | Tipo | Labels | O que indica |
- **Logs:** | Evento | Nível | Campos |
- **Tracing:** {spans, atributos e como o contexto se propaga entre processos}

## 12. Integração com o Sistema Existente

| Arquivo | O que existe hoje | Como a feature se integra |
| --- | --- | --- |
| `caminho/real.ts` | {…} | {estende / reutiliza / cria ao lado} |

## 13. Dependências e Compatibilidade

{Bibliotecas, serviços, variáveis de ambiente, migrations, compatibilidade com o comportamento atual.}

## 14. Critérios de Aceite Técnicos

- [ ] **CAT-01:** {condição testável, como contrato, erro, resiliência ou desempenho}

## 15. Riscos e Mitigação

| Risco | Impacto | Mitigação | Fonte |
| --- | --- | --- | --- |
````

**Códigos de erro:** use o prefixo do módulo e siga o padrão das classes de erro existentes. Cada código estende uma classe que já existe e é tratado pelo middleware de erro atual.

## Checklist

- [ ] As 15 seções existem e estão preenchidas.
- [ ] Cada decisão fechada aparece com o mesmo valor do ADR e com o link dele.
- [ ] Cada fluxo tem passos numerados e um diagrama, cobrindo o caminho feliz, a falha transitória e a falha permanente.
- [ ] Cada endpoint tem autorização, headers, request de exemplo, response de exemplo e status codes com semântica.
- [ ] Cada código de erro tem o prefixo do módulo, um status HTTP e uma classe base que existe no código.
- [ ] Observabilidade lista métricas, logs e tracing, e nenhum segredo ou dado sensível aparece neles.
- [ ] Integração lista pelo menos 4 arquivos reais, cada um com o que existe hoje e o que muda.
- [ ] Cada critério de aceite pode ser verificado por um teste.
- [ ] Nada que a fonte descartou ou adiou aparece como comportamento da feature.
- [ ] Todo `[hh:mm] Nome` citado existe na transcrição com aquele falante (confira com `grep -n`), e todo caminho de código existe (confira com `ls`).
