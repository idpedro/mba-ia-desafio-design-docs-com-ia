---
date: 2026-10-30
decision-makers: [Larissa, Bruno]
consulted: [Diego]
informed: []
---

# ADR-008: Gravar o payload do evento como snapshot na inserção da outbox

## Status

Aceito — conversa pós-reunião, fechado em [09:52] Bruno ("Beleza, snapshot. Decidido."), depois que Marcos e Sofia saíram da call.

## Contexto e Problema

O evento pode ser enviado minutos ou horas depois da mudança de status, por causa do polling ([ADR-002](ADR-002-processar-outbox-com-worker-separado-em-polling.md)) e do retry ([ADR-004](ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md)). Bruno perguntou: "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?" ([09:51] Bruno).

## Fatores de Decisão

- O evento precisa refletir "o estado de quando o status mudou" ([09:52] Larissa).
- O payload deve ficar enxuto ([09:44] Bruno).
- Há um limite de 64KB por payload, com erro se ultrapassar ([09:24] Larissa).

## Alternativas Consideradas

- Snapshot do payload renderizado na inserção
- Guardar só o `order_id` e renderizar no envio
- Snapshot que inclui os items do pedido

## Decisão

Opção escolhida: "snapshot na inserção", porque "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" ([09:52] Larissa; [09:52] Diego). Vence a renderização no envio porque, nela, um evento atrasado mostraria o pedido de outro momento. Vence a inclusão de items porque "Não manda items pra não inflar", e o cliente busca detalhes em `GET /orders/:id` ([09:43] Diego).

Conteúdo: JSON com `event_id`, `event_type` (ex.: `order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents` ([09:43] Diego). O schema completo fica no [FDD](../FDD.md).

## Consequências

- Positiva: retries e replays da DLQ enviam o mesmo conteúdo do momento da mudança.
- Positiva: o worker não precisa reler o pedido para enviar.
- Negativa: cada linha da outbox guarda o payload inteiro, o que pesa no armazenamento e na transação de `changeStatus`.
- Negativa: o formato fica congelado nos eventos já gravados, e mudar o schema não afeta o que já está na fila.
- Trade-off: aceita-se mais armazenamento e um formato congelado em troca de fidelidade ao momento da mudança.

### Confirmação

Um teste muda o status, altera o pedido antes do envio e verifica que o payload enviado é o do momento da mudança.

## Prós e Contras das Alternativas

### Renderizar no envio

Guardar só o `order_id` e montar o payload no worker ([09:51] Bruno).

- Bom, porque a linha da outbox fica pequena.
- Ruim, porque "Senão tem caso esquisito" ([09:52] Larissa). ← motivo do descarte

### Incluir os items

Enviar o pedido completo.

- Bom, porque o cliente não precisa de uma segunda chamada.
- Ruim, porque infla o payload ([09:43] Diego). ← motivo do descarte

## Mais Informações

- Fontes: [09:43] Diego; [09:44] Bruno; [09:51] Bruno a [09:52] Bruno; [09:23] Sofia a [09:24] Larissa.
- Código relacionado: `prisma/schema.prisma` (as colunas são camelCase, como `totalCents` e `orderNumber`, enquanto o payload usa snake_case).
- Relacionados: [ADR-001](ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md), [ADR-005](ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md).
- Questões em aberto:
  - A decisão foi tomada sem Sofia, embora o payload seja o conteúdo assinado ([ADR-005](ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md)).
  - A lista de "campos básicos" além de `total_cents` não foi enumerada ([09:43] Diego).
