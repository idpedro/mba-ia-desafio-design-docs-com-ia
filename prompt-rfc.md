/rfc

Com base nos ADRs em @docs/adrs/ e no research em @docs/research/, escreva o RFC da feature de Sistema de Webhooks de Notificação de Pedidos.

- Revisores: os 5 participantes da reunião, com seus papéis.
- Alternativas consideradas: só as discutidas e descartadas na reunião (síncrono, Redis Streams, trigger do banco, retry indefinido, DLQ como status na outbox), cada uma com o motivo do descarte.
- Questões em aberto: os pontos adiados ou não decididos na reunião (rate limiting, escala para múltiplos workers, endurecer roles do CRUD) e as divergências da verificação externa que afetam decisões, como o X-Timestamp fora da assinatura ([09:44] Diego × [09:22] Sofia).
- Riscos: a autorização do CRUD (customer_id no body/path, qualquer role autenticada, [09:32] Larissa e [09:37] Sofia) em conflito com o código, sem vínculo entre User e Customer (prisma/schema.prisma).
- Cada decisão fechada aparece em uma linha com link para o ADR correspondente; não repita o detalhamento.
