---
date: 2026-10-30
decision-makers: [Larissa, Diego]
consulted: [Bruno]
informed: [Marcos, Sofia]
---

# ADR-001: Usar padrão outbox no MySQL para registrar eventos de webhook

## Status

Aceito — reunião técnica de webhooks, fechado em [09:08] Larissa ("Tá decidido então: outbox em MySQL") e confirmado no resumo de [09:48] Larissa.

## Contexto e Problema

Três clientes B2B querem ser notificados quando o status dos pedidos muda e hoje fazem polling no `GET /orders` ([09:00] Marcos). A transação de mudança de status "hoje já é pesada": atualiza `orders`, insere em `order_status_history` e mexe no estoque ([09:04] Bruno; `src/modules/orders/order.service.ts`). Como garantir que toda mudança de status gere um evento, sem acoplar a transação à disponibilidade do cliente?

## Fatores de Decisão

- Um cliente lento não pode travar a mudança de status de outros pedidos ([09:04] Bruno).
- "Não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).
- O time é pequeno e não quer subir infraestrutura nova ([09:07] Diego).

## Alternativas Consideradas

- Outbox no MySQL existente, na mesma transação
- Disparo HTTP síncrono no service de orders
- Redis Streams como fila

## Decisão

Opção escolhida: "outbox no MySQL existente", porque, na mesma transação SQL, "se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto" ([09:06] Diego). Ela vence o disparo síncrono porque tira o HTTP da transação. Vence o Redis porque usa o banco que já existe ("Outbox no MySQL existente resolve", [09:07] Diego).

Parâmetros: tabela `webhook_outbox`, inserida dentro do `$transaction` de `changeStatus`. Se a inserção falhar, a transação inteira dá rollback ([09:40] Bruno). A inserção é feita pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o tx client, em vez de injetar um repository no `OrderService` ([09:41] Bruno; [09:41] Diego). O modelo da tabela e os índices ([09:08] Diego) ficam no [FDD](../FDD.md).

## Consequências

- Positiva: o evento só existe quando o status mudou de fato, e vice-versa, sem infraestrutura nova ([09:41] Diego; [09:07] Diego).
- Negativa: a entrega passa a depender de um leitor da tabela ([ADR-002](ADR-002-processar-outbox-com-worker-separado-em-polling.md)).
- Negativa: a transação de `changeStatus`, que já era pesada, ganha mais um insert ([09:04] Bruno).
- Negativa: a tabela cresce sem arquivamento, que ficou fora de escopo ([09:08] Diego).
- Trade-off: aceita-se o atraso do polling e o peso extra na transação em troca de atomicidade sem infraestrutura nova.

### Confirmação

Um teste força uma falha no insert da outbox e verifica que o status não mudou. No code review, `publishWebhookEvent` recebe o `tx`, nunca o `prisma` global.

## Prós e Contras das Alternativas

### Disparo síncrono

Chamar o endpoint do cliente dentro do service de orders ([09:03] Larissa).

- Bom, porque a latência fica mínima.
- Ruim, porque "qualquer cliente lento vai travar mudança de status pra outros pedidos" ([09:04] Bruno). ← motivo do descarte

### Redis Streams

Fila dedicada fora do banco ([09:07] Larissa).

- Bom, porque é reativo, sem polling.
- Ruim, porque "a gente acabaria precisando subir mais infra" ([09:07] Larissa); "Subir Redis Cluster pra isso é overengineering" ([09:07] Diego). ← motivo do descarte

## Mais Informações

- Fontes: [09:03] Larissa a [09:08] Larissa; [09:40] Bruno a [09:41] Diego; [09:48] Larissa.
- Código relacionado: `src/modules/orders/order.service.ts`, `src/config/database.ts`.
- Referências externas: [Pattern: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html) — respalda o registro atômico, veredito Parcial: "The Message relay might publish a message more than once". [Redis streams](https://redis.io/docs/latest/develop/data-types/streams/) — Parcial: Streams roda em Redis standalone, sem exigir Cluster.
- Relacionados: [ADR-002](ADR-002-processar-outbox-com-worker-separado-em-polling.md), [ADR-006](ADR-006-garantir-entrega-at-least-once-com-x-event-id.md), [ADR-008](ADR-008-gravar-payload-como-snapshot-na-insercao.md).
- Questões em aberto:
  - "Não tem inconsistência possível" ([09:06] Diego) vale para o registro; a entrega pode duplicar.
  - A criação do pedido grava o `PENDING` sem passar por `changeStatus`, e não foi dito se ela gera evento.
  - `changeStatus` não trava a linha do pedido, então mudanças concorrentes podem gerar eventos com o mesmo `from_status`.
  - Ver [decisões candidatas](../research/decisoes-candidatas.md).
