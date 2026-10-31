---
date: 2026-10-30
decision-makers: [Larissa, Diego]
consulted: [Sofia, Bruno, Marcos]
informed: []
---

# ADR-006: Garantir entrega at-least-once com X-Event-Id para deduplicação no cliente

## Status

Aceito — reunião técnica de webhooks, fechado em [09:26] Larissa ("At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.") e confirmado em [09:48] Larissa.

## Contexto e Problema

Com a outbox ([ADR-001](ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md)) e o retry ([ADR-004](ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md)), um evento pode ser enviado mais de uma vez. Por exemplo, o cliente processa e o timeout estoura antes da resposta. Que garantia de entrega a plataforma promete, e como o cliente lida com duplicatas?

## Fatores de Decisão

- Nenhum evento pode se perder depois que o status mudou ([09:40] Bruno).
- Simplicidade de implementação num time pequeno ([09:07] Diego).
- Alinhamento com o que os clientes já conhecem de outros provedores ([09:25] Diego).

## Alternativas Consideradas

- At-least-once, com `X-Event-Id` para dedup no cliente
- Exactly-once
- At-most-once (um envio, sem retry) (alternativa plausível, não discutida na reunião)

## Decisão

Opção escolhida: "at-least-once com `X-Event-Id`", porque é "o padrão de mercado. Stripe faz assim, GitHub faz assim" ([09:25] Diego). Vence exactly-once porque garantir isso "exigiria coordenação dos dois lados e fica muito mais complexo" ([09:25] Diego). Vence at-most-once porque perderia eventos sempre que o cliente estivesse fora do ar, contra o retry já decidido ([09:17] Larissa).

Parâmetros:
- `X-Event-Id` é um UUID gerado quando o evento entra na outbox, único por evento ([09:25] Diego).
- Demais headers: `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` ([09:44] Diego; [09:44] Sofia).

O contrato completo dos headers fica no [FDD](../FDD.md).

## Consequências

- Positiva: nenhum evento registrado é descartado em silêncio ([09:24] Diego).
- Positiva: a implementação é simples do lado da plataforma ([09:25] Diego).
- Negativa: "Isso joga responsabilidade pro cliente" ([09:25] Sofia). O cliente tem que deduplicar.
- Negativa: a promessa precisa ser comunicada, e Marcos documenta isso "bem destacado no portal de desenvolvedor" ([09:26] Marcos).
- Trade-off: aceita-se transferir a deduplicação ao cliente em troca de não perder eventos e de não coordenar dois lados.

### Confirmação

Um teste reenvia o mesmo evento e verifica que o `X-Event-Id` é idêntico nos dois envios. O portal de desenvolvedor descreve a deduplicação.

## Prós e Contras das Alternativas

### Exactly-once

Garantir uma única entrega por evento ([09:25] Diego).

- Bom, porque o cliente não precisa deduplicar.
- Ruim, porque "exigiria coordenação dos dois lados e fica muito mais complexo" ([09:25] Diego). ← motivo do descarte

### At-most-once

Enviar uma vez e não reenviar. Alternativa plausível, não discutida na reunião.

- Bom, porque nunca duplica.
- Ruim, porque perde o evento quando o cliente cai, contra o retry decidido em [09:17] Larissa. ← motivo do descarte

## Mais Informações

- Fontes: [09:24] Diego a [09:26] Larissa; [09:44] Diego a [09:45] Diego.
- Referências externas:
  - [Stripe — Handle duplicate events](https://docs.stripe.com/webhooks?locale=en-US) — respalda o modelo, Parcial no conjunto.
  - [GitHub — Handling failed webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries) — o GitHub tem id estável, mas não reenvia falhas automaticamente.
- Relacionados: [ADR-004](ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md), [ADR-005](ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md).
- Questões em aberto:
  - Veredito Parcial: a comparação com o GitHub vale para o id, não para o retry.
  - Inconclusiva: não há fonte primária para a complexidade do exactly-once nem para "resolve 99% dos casos" ([09:25] Diego).
  - Não foi dito se o id da linha da outbox é o mesmo valor do `X-Event-Id`, nem se o replay da DLQ mantém esse id.
