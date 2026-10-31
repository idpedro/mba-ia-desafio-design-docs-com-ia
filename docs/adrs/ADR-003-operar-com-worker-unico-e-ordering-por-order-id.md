---
date: 2026-10-30
decision-makers: [Larissa, Diego]
consulted: [Bruno, Marcos]
informed: [Sofia]
---

# ADR-003: Operar com um único worker e garantir ordering apenas por order_id

## Status

Aceito — reunião técnica de webhooks, fechado em [09:13] Larissa ("Documentamos como limitação conhecida").

## Contexto e Problema

Um pedido pode passar por PAID, PROCESSING e SHIPPED em sequência rápida ([09:12] Larissa). Os clientes "nunca pediram garantia de ordering global, eles só querem saber se cada pedido deles mudou" ([09:14] Marcos). Que garantia de ordem a plataforma oferece, e com quantos workers?

## Fatores de Decisão

- O cliente precisa receber, para cada pedido, as mudanças na ordem em que aconteceram ([09:12] Larissa).
- Não há requisito de ordem global ([09:14] Marcos).
- O time é pequeno e prefere a solução mais simples ([09:07] Diego).

## Alternativas Consideradas

- Um único worker, com ordering por `order_id`
- Múltiplos workers com partição por `order_id`
- Múltiplos workers com lock pessimista

## Decisão

Opção escolhida: "um único worker, com ordering por `order_id`", porque "Se a gente tem um único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em ordem" ([09:12] Diego). Vence as duas opções multi-worker porque elas resolvem um problema de escala que não existe hoje: "isso é problema do futuro, não agora" ([09:13] Diego). Também não há demanda por ordem global ([09:14] Marcos).

Limitação a publicar: "Não é garantia de ordering global, só por order_id e enquanto for single-worker" ([09:13] Larissa). Como manter a ordem por pedido durante retries fica para o [FDD](../FDD.md).

## Consequências

- Positiva: não há coordenação, lock nem partição entre workers ([09:12] Diego).
- Positiva: atende o que os clientes pediram ([09:14] Marcos).
- Negativa: a vazão de entrega fica limitada a um processo, e esse processo é ponto único de falha.
- Negativa: escalar exige revisitar esta decisão, porque com múltiplos workers "perde a garantia" ([09:12] Diego).
- Trade-off: aceita-se um teto de vazão e um ponto único de falha em troca de ordem por pedido sem coordenação.

### Confirmação

O deploy roda exatamente uma instância de `npm run worker`. Um teste cria PAID → PROCESSING → SHIPPED para o mesmo pedido e verifica a ordem de entrega.

## Prós e Contras das Alternativas

### Partição por order_id

Vários workers, cada um com uma faixa de pedidos ([09:13] Diego).

- Bom, porque escala mantendo a ordem por pedido.
- Ruim, porque traz uma complexidade de distribuição desnecessária agora ("problema do futuro", [09:13] Diego). ← motivo do adiamento

### Lock pessimista

Vários workers disputando linhas com lock ([09:13] Diego).

- Bom, porque escala sem atribuição fixa.
- Ruim, porque também é "problema do futuro" ([09:13] Diego) e sozinho não garante ordem por pedido. ← motivo do adiamento

## Mais Informações

- Fontes: [09:12] Larissa a [09:14] Marcos; [09:48] Larissa.
- Código relacionado: `src/modules/orders/order.status.ts` (transições de status).
- Referências externas: [MySQL — Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html) — respalda `FOR UPDATE`/`SKIP LOCKED`, Confirmada; `SKIP LOCKED` não preserva ordem por pedido. [MySQL — Fractional Seconds](https://dev.mysql.com/doc/refman/8.0/en/fractional-seconds.html) — base da ressalva Parcial sobre empates em `created_at`.
- Relacionados: [ADR-002](ADR-002-processar-outbox-com-worker-separado-em-polling.md), [ADR-004](ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md).
- Questões em aberto:
  - Veredito Parcial: mesmo com um único worker, o backoff ([ADR-004](ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md)) permite entregar o evento N+1 antes do evento N que falhou, a menos que o worker segure os eventos seguintes do mesmo pedido. Isso não foi discutido.
  - Empates em `created_at` não têm ordem definida, e o UUID não desempata.
  - Gatilho para revisitar: a vazão de um worker deixar de dar conta do volume.
