---
date: 2026-10-30
decision-makers: [Larissa, Diego]
consulted: [Bruno, Marcos, Sofia]
informed: []
---

# ADR-004: Fazer retry com backoff exponencial em 5 tentativas e mover falhas para uma DLQ em tabela separada

## Status

Aceito — reunião técnica de webhooks, fechado em [09:17] Larissa ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"), [09:19] Larissa (replay manual) e [09:36] Larissa (ADMIN no replay). Confirmado em [09:48] Larissa.

## Contexto e Problema

O endpoint do cliente pode estar fora do ar ([09:14] Larissa). Já houve cliente "com indisponibilidade de duas horas em manutenção planejada" ([09:16] Diego). Quantas vezes a plataforma tenta de novo, em que intervalos, e o que acontece com o evento que esgota as tentativas?

## Fatores de Decisão

- Cobrir indisponibilidades de horas ([09:16] Diego).
- Não deixar evento "pendurado pra sempre se o cliente sumiu" ([09:15] Diego).
- Preservar a falha "como evidence pra debug e reprocessamento" ([09:18] Diego).

## Alternativas Consideradas

- 5 tentativas com backoff exponencial e DLQ em tabela separada
- Retry indefinido com backoff
- 3 tentativas
- Marcar como "failed" na própria outbox, sem DLQ

## Decisão

Opção escolhida: "5 tentativas com backoff exponencial e DLQ em tabela separada", porque cinco tentativas cobrem uma janela de horas sem tornar o evento eterno ([09:15] Diego). Vence o retry indefinido porque o evento teria um fim. Vence as 3 tentativas porque "retentaria três vezes em 30 minutos e mataria" ([09:16] Diego). Vence o "failed" na outbox porque a tabela separada deixa "mais limpa a leitura da outbox principal" ([09:18] Diego).

Parâmetros:
- Progressão 1m, 5m, 30m, 2h, 12h, "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego).
- Tabela `webhook_dead_letter` com payload, motivo da falha e timestamp ([09:18] Diego).
- Replay manual por `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
- O replay exige role ADMIN via `requireRole` ([09:36] Larissa) e loga quem o fez, para auditoria ([09:36] Sofia).

O cálculo do próximo envio fica no [FDD](../FDD.md).

## Consequências

- Positiva: cobre indisponibilidades de várias horas ([09:16] Diego).
- Positiva: todo evento tem um fim, entregue ou na DLQ ([09:15] Diego).
- Negativa: um evento pode levar ~15h para chegar, o que Marcos aceitou ([09:17] Marcos).
- Negativa: depois das 5 tentativas, recuperar o evento exige ação humana de um ADMIN ([09:18] Diego).
- Trade-off: aceita-se até ~15h de atraso e reprocessamento manual em troca de um limite claro e de evidência preservada.

### Confirmação

Um teste unitário verifica os intervalos de cada tentativa e a ida para a DLQ após a última. Um teste de rota verifica que OPERATOR recebe 403 no replay.

## Prós e Contras das Alternativas

### Retry indefinido

Continuar tentando com intervalos crescentes ([09:15] Diego).

- Bom, porque nunca desiste de um cliente que volta.
- Ruim, porque o evento fica "pendurado pra sempre se o cliente sumiu" ([09:15] Diego). ← motivo do descarte

### 3 tentativas

Teto "mais agressivo" ([09:16] Bruno).

- Bom, porque a falha é detectada mais cedo.
- Ruim, porque "3 é pouco" ([09:16] Diego). ← motivo do descarte

## Mais Informações

- Fontes: [09:14] Larissa a [09:19] Larissa; [09:35] Diego a [09:36] Larissa.
- Código relacionado: `src/middlewares/auth.middleware.ts` (`requireRole`), `src/modules/auth/auth.schemas.ts`.
- Referências externas: [Stripe — Automatic retries](https://docs.stripe.com/webhooks?locale=en-US) — comparação: a Stripe retenta por até 3 dias.
- Relacionados: [ADR-002](ADR-002-processar-outbox-com-worker-separado-em-polling.md), [ADR-006](ADR-006-garantir-entrega-at-least-once-com-x-event-id.md).
- Questões em aberto:
  - "12 ou 24 horas" ([09:15] Diego) não bate com "quase 15 horas" ([09:17] Diego).
  - Não ficou dito se o envio inicial conta entre as 5 tentativas.
  - `POST /auth/register` é público e aceita `role: 'ADMIN'`, então a exigência de ADMIN não protege o replay por si só.
  - Aviso por email após falhas ficou adiado ([09:37] Larissa).
