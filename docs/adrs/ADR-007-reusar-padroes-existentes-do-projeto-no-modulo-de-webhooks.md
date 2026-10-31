---
date: 2026-10-30
decision-makers: [Larissa]
consulted: [Bruno, Diego]
informed: [Marcos, Sofia]
---

# ADR-007: Reusar os padrões existentes do projeto no módulo de webhooks

## Status

Aceito — reunião técnica de webhooks, fechado em [09:30] Larissa ("Decisão: reuso máximo do que já existe") e confirmado em [09:48] Larissa.

## Contexto e Problema

A feature traz uma área nova: configuração, outbox, worker e DLQ. A codebase já tem convenções de módulo, erros, logging e validação ([09:27] Bruno). A feature segue essas convenções ou cria as próprias?

## Fatores de Decisão

- "A gente tem um padrão claro na codebase" ([09:27] Bruno).
- "Não vamos botar nada novo" ([09:29] Bruno).
- O worker usa "mesmo banco, mesma stack" ([09:11] Diego).
- Time pequeno ([09:07] Diego).

## Alternativas Consideradas

- Módulo `src/modules/webhooks` (novo, a criar) que reusa os padrões existentes
- Serviço de webhooks separado, com stack e repositório próprios (alternativa plausível, não discutida na reunião)
- Módulo no mesmo projeto, mas com bibliotecas próprias de erro, log e fila (alternativa plausível, não discutida na reunião)

## Decisão

Opção escolhida: "módulo que reusa os padrões existentes", porque mantém o webhook "como módulo igual aos outros" ([09:30] Larissa), e o middleware de erro "Vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno). Vence o serviço separado porque contraria o "mesma stack" do worker ([09:11] Diego) e o tamanho do time ([09:07] Diego). Vence as bibliotecas próprias porque "Não vamos botar nada novo" ([09:29] Bruno).

Parâmetros:
- Pasta `src/modules/webhooks` (nova, a criar) com controller, service, repository, routes e schemas ([09:27] Bruno).
- Erros como subclasses de `AppError`, com prefixo `WEBHOOK_`, por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno; [09:29] Larissa).
- Logger Pino, o error middleware centralizado e schemas Zod ([09:29] Bruno; [09:30] Larissa).
- Ids UUID, como no resto do projeto ([09:51] Larissa).

A estrutura de arquivos fica no [FDD](../FDD.md).

## Consequências

- Positiva: quem conhece um módulo do projeto entende o de webhooks.
- Positiva: os erros novos saem no mesmo formato HTTP sem mudar o middleware ([09:29] Bruno).
- Negativa: a feature herda os limites desses padrões. O middleware só trata os erros Prisma `P2002` e `P2025`, e o resto vira 500.
- Negativa: o Pino não é usado em nenhum service hoje, então o log de auditoria do replay ([09:36] Sofia) não tem um padrão a seguir.
- Trade-off: aceita-se herdar os limites dos padrões atuais em troca de consistência e de nenhuma dependência nova.

### Confirmação

No code review, conferir os cinco arquivos do padrão, que todo erro novo estende `AppError` com código `WEBHOOK_` e que `package.json` não ganhou dependências de log, erro ou fila.

## Prós e Contras das Alternativas

### Serviço separado

Webhooks em serviço e stack próprios. Alternativa plausível, não discutida na reunião.

- Bom, porque isola o deploy.
- Ruim, porque o worker deve usar "mesmo banco, mesma stack" ([09:11] Diego), e o time é pequeno ([09:07] Diego). ← motivo do descarte

### Bibliotecas próprias no módulo

Por exemplo, uma lib de fila ou de logging nova. Alternativa plausível, não discutida na reunião.

- Bom, porque daria ferramentas específicas para jobs.
- Ruim, porque "Não vamos botar nada novo" ([09:29] Bruno), e infraestrutura de fila já foi descartada ([09:07] Diego). ← motivo do descarte

## Mais Informações

- Fontes: [09:27] Bruno a [09:30] Larissa; [09:48] Larissa; [09:51] Larissa.
- Código relacionado: `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/modules/orders/`.
- Relacionados: [ADR-001](ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md), [ADR-002](ADR-002-processar-outbox-com-worker-separado-em-polling.md).
- Questões em aberto:
  - Conflito: "O middleware de erro centralizado já trata AppError, Zod e Prisma" ([09:29] Bruno), mas só cobre dois códigos Prisma.
  - Conflito: o Pino "já tá no projeto inteiro" ([09:29] Bruno), mas só bootstrap e middlewares logam.
