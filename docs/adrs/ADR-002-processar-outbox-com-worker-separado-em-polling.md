---
date: 2026-10-30
decision-makers: [Larissa, Diego]
consulted: [Bruno, Marcos]
informed: [Sofia]
---

# ADR-002: Processar a outbox com um worker em processo separado, por polling a cada 2 segundos

## Status

Aceito — reunião técnica de webhooks, fechado em [09:10] Larissa ("Vamos registrar isso como uma decisão. Worker em polling, 2s") e confirmado em [09:48] Larissa.

## Contexto e Problema

A outbox ([ADR-001](ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md)) precisa de um leitor que dispare as chamadas HTTP. Para os clientes, "qualquer coisa abaixo de 10 segundos já é 'tempo real'" ([09:02] Marcos). Como o leitor descobre que há eventos novos, e onde ele roda?

## Fatores de Decisão

- Latência abaixo de 10 segundos ([09:02] Marcos).
- Nenhuma infraestrutura nova ([09:07] Diego).
- O worker não pode morrer junto com a API: "se a API reinicia, perde o worker" ([09:11] Diego).

## Alternativas Consideradas

- Polling a cada 2s, num processo separado
- Trigger do banco para acordar o worker
- Worker dentro do processo da API

## Decisão

Opção escolhida: "polling a cada 2s, num processo separado", porque "Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo" ([09:09] Diego) sem exigir mecanismo de notificação que o MySQL não tem. Vence a trigger porque ela "não notifica processo externo, ela só executa SQL" ([09:09] Diego). Vence o worker embutido porque um restart da API o derrubaria ([09:11] Diego).

Parâmetros:
- Entry-point `src/worker.ts` e script `npm run worker`, ambos novos, a criar ([09:11] Larissa).
- Mesmo banco e mesma stack, em outro processo ([09:11] Diego).
- PrismaClient próprio com a mesma `DATABASE_URL`, porque "PrismaClient é por processo" ([09:30] Bruno).
- Cada ciclo busca os pendentes mais antigos em batch pequeno ([09:09] Diego).

O loop, o batch e o timeout de 10s ([09:42] Diego) ficam no [FDD](../FDD.md).

## Consequências

- Positiva: atende o requisito de latência com margem ([09:10] Marcos).
- Positiva: o worker é isolado da API, e cada um reinicia sem afetar o outro ([09:11] Diego).
- Negativa: cada evento espera até 2s antes de ser lido ([09:10] Larissa).
- Negativa: o banco recebe consultas periódicas mesmo sem eventos.
- Negativa: passa a haver um segundo processo para subir e monitorar.
- Trade-off: aceita-se a latência e o custo de uma consulta a cada 2s em troca de não depender de infraestrutura nova.

### Confirmação

Em code review, conferir que `src/worker.ts` cria a própria instância com `createPrismaClient()` e não importa `src/app.ts`. Um teste mede que um evento inserido é processado em menos de 10s.

## Prós e Contras das Alternativas

### Trigger do banco

Acordar o worker a partir de uma trigger ([09:09] Bruno).

- Bom, porque seria mais reativo.
- Ruim, porque "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres", e avisar o worker exigiria "escrever em arquivo ou bater num endpoint, fica esquisito" ([09:09] Diego). ← motivo do descarte

### Worker no processo da API

Rodar o loop dentro de `src/server.ts`.

- Bom, porque é um processo a menos.
- Ruim, porque "se a API reinicia, perde o worker" ([09:11] Diego). ← motivo do descarte

## Mais Informações

- Fontes: [09:09] Bruno a [09:12] Larissa; [09:29] Diego; [09:30] Bruno; [09:42] Diego.
- Código relacionado: `src/server.ts`, `src/config/database.ts`, `package.json`.
- Referências externas:
  - [PostgreSQL NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html) e [MySQL CREATE TRIGGER](https://dev.mysql.com/doc/refman/8.0/en/create-trigger.html) — respaldam a ausência de listener no MySQL, Confirmada.
  - [Prisma — Connection pool](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-pool) — respalda o pool por instância, Confirmada.
- Relacionados: [ADR-003](ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md), [ADR-004](ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md).
- Questões em aberto:
  - "A latência mínima vai ser 2 segundos no pior caso" ([09:10] Larissa) mistura mínimo e pior caso.
  - O pool do worker soma conexões às da API e precisa caber em `max_connections`.
  - O arquivo de lógica (novo, a criar em `src/modules/webhooks`) ficou entre `webhook.worker.ts` e `webhook.processor.ts` ([09:28] Bruno).
