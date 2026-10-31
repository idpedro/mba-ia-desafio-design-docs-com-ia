/prd

Com base nos ADRs em @docs/adrs/, no RFC, no FDD e no research em @docs/research/, escreva o PRD da feature de Sistema de Webhooks de Notificação de Pedidos.

- O PRD fica no nível de produto: decisões técnicas aparecem em uma linha com link para o ADR, e detalhes de endpoint e payload ficam no FDD.
- Fora de escopo: todos os itens descartados ou adiados na reunião, cada um com o motivo.
- Métrica principal: latência abaixo de 10 segundos, que é o "tempo real" para os clientes ([09:02] Marcos).
- O customer_id não vem do JWT ([09:32] Larissa); registre o conflito com o código (User sem vínculo com Customer, prisma/schema.prisma) como risco.
