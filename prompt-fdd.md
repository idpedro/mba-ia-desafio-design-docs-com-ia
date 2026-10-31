/fdd

Com base nos ADRs em @docs/adrs/, no RFC em @docs/RFC.md e no research em @docs/research/, escreva o FDD da feature de Sistema de Webhooks de Notificação de Pedidos.

- Contratos públicos: todos os endpoints discutidos na reunião (CRUD de webhooks, rotação de secret, GET /webhooks/:id/deliveries, POST /admin/webhooks/dead-letter/:id/replay) e a chamada de saída para o cliente com payload e headers. As rotas existentes ficam sob /api/v1 (src/app.ts).
- Códigos de erro com prefixo WEBHOOK_, estendendo as classes de src/shared/errors/.
- Integração com o sistema existente: descreva o que o código faz hoje, não o que foi dito na reunião quando os dois divergirem (ex.: o estoque só muda em PENDING→PAID e no cancelamento, src/modules/orders/order.status.ts). Arquivos novos aparecem marcados como "a criar".
- Retry: envio inicial + 5 retentativas, como fecha "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego).
- Sugestões da verificação externa que a reunião não discutiu (jitter, assinar o timestamp, connection_limit, cifrar a secret) entram só como risco ou questão em aberto.
