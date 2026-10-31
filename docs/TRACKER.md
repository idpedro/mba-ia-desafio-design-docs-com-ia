# Tracker de rastreabilidade

Este tracker liga cada item do [PRD](PRD.md), do [RFC](RFC.md), do [FDD](FDD.md) e dos [ADRs](adrs/README.md) à sua origem: uma fala da [transcrição da reunião](../TRANSCRICAO.md) ou um arquivo do código. Um item sem origem identificável é tratado como invenção e corrigido no documento de onde veio. Referências externas da [verificação externa](research/verificacao-externa.md) servem de respaldo e não viram linha. Quando o item nasce na reunião e a ligação com o código o define, uma segunda linha com sufixo `-C` registra a origem `CODIGO`. Itens que o documento marca como "decisão deste FDD" ou "não discutida na reunião" são rastreados pela fala que originou a questão.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CA-01 | `docs/PRD.md` | Critério de Aceite | Cadastro com URL `https` devolve a secret; com `http`, recusa com erro de validação | TRANSCRICAO | [09:23] Sofia; [09:31] Marcos |
| PRD-CA-02 | `docs/PRD.md` | Critério de Aceite | Editar, remover e listar funcionam por customer; só criação e rotação devolvem a secret | TRANSCRICAO | [09:33] Bruno; [09:31] Marcos |
| PRD-CA-03 | `docs/PRD.md` | Critério de Aceite | Webhook que assina só `DELIVERED` não recebe evento quando o pedido vira `SHIPPED` | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-CA-04 | `docs/PRD.md` | Critério de Aceite | Mudança assinada chega em menos de 10s, com os dados do momento e sem items | TRANSCRICAO | [09:02] Marcos; [09:52] Larissa; [09:43] Diego |
| PRD-CA-05 | `docs/PRD.md` | Critério de Aceite | A assinatura confere com o HMAC-SHA256 do corpo com a secret do endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-CA-06 | `docs/PRD.md` | Critério de Aceite | Após rotação, envios das 24h seguintes verificáveis com a antiga e a nova; depois, só a nova | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-07 | `docs/PRD.md` | Critério de Aceite | Falha ou timeout de 10s gera envio inicial e 5 retentativas em 1m/5m/30m/2h/12h, depois DLQ com motivo | TRANSCRICAO | [09:17] Diego; [09:42] Diego; [09:18] Diego |
| PRD-CA-08 | `docs/PRD.md` | Critério de Aceite | Evento acima de 64KB não é enviado e vai para a DLQ com o motivo | TRANSCRICAO | [09:24] Larissa; [09:23] Sofia |
| PRD-CA-09 | `docs/PRD.md` | Critério de Aceite | Histórico mostra até 100 entregas com sucesso/falha, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-10 | `docs/PRD.md` | Critério de Aceite | Replay por OPERATOR é recusado; por ADMIN, volta à fila e o log registra quem fez | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| PRD-CA-11 | `docs/PRD.md` | Critério de Aceite | Identificador do evento igual no envio inicial, nas retentativas e após o replay | TRANSCRICAO | [09:25] Diego; [09:18] Diego |
| PRD-CA-12 | `docs/PRD.md` | Critério de Aceite | Se o registro do evento falha, a mudança de status é desfeita | TRANSCRICAO | [09:40] Bruno |
| PRD-CA-13 | `docs/PRD.md` | Critério de Aceite | Com o endpoint do cliente fora do ar, a mudança de status responde normalmente | TRANSCRICAO | [09:04] Bruno |
| PRD-CA-14 | `docs/PRD.md` | Critério de Aceite | Três mudanças seguidas do mesmo pedido chegam na ordem em que aconteceram | TRANSCRICAO | [09:12] Diego; [09:12] Larissa |
| PRD-DEC-01 | `docs/PRD.md` | Trade-off | Outbox no MySQL na mesma transação: mais um insert em troca de atomicidade sem infra nova | TRANSCRICAO | [09:08] Larissa; [09:06] Diego |
| PRD-DEC-02 | `docs/PRD.md` | Trade-off | Worker separado com polling de 2s: até 2s de espera e um processo a mais, sem infra nova | TRANSCRICAO | [09:10] Larissa; [09:09] Diego; [09:11] Diego |
| PRD-DEC-03 | `docs/PRD.md` | Trade-off | Worker único: teto de vazão e ponto único de falha, em troca de ordem por pedido | TRANSCRICAO | [09:13] Larissa; [09:12] Diego |
| PRD-DEC-04 | `docs/PRD.md` | Trade-off | 5 retentativas e DLQ com replay ADMIN: até ~15h de atraso, em troca de fim definido | TRANSCRICAO | [09:17] Larissa; [09:18] Diego; [09:36] Larissa |
| PRD-DEC-05 | `docs/PRD.md` | Trade-off | HMAC-SHA256 com secret por endpoint e 24h de convivência: secret recuperável, vazamento isolado | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-06 | `docs/PRD.md` | Trade-off | At-least-once com identificador: o cliente deduplica, em troca de não perder eventos | TRANSCRICAO | [09:26] Larissa; [09:25] Diego |
| PRD-DEC-07 | `docs/PRD.md` | Trade-off | Reuso dos padrões do projeto: herda os limites, em troca de consistência sem dependência nova | TRANSCRICAO | [09:30] Larissa; [09:29] Bruno |
| PRD-DEC-08 | `docs/PRD.md` | Trade-off | Payload como snapshot: mais armazenamento e formato congelado, em troca de fidelidade ao momento | TRANSCRICAO | [09:52] Bruno; [09:52] Larissa |
| PRD-DEP-01 | `docs/PRD.md` | Restrição | O evento nasce dentro de `changeStatus`, que precisa ser alterado pelo time de Pedidos | TRANSCRICAO | [09:40] Bruno |
| PRD-DEP-01-C | `docs/PRD.md` | Restrição | O evento nasce dentro de `changeStatus`, que precisa ser alterado pelo time de Pedidos | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-02 | `docs/PRD.md` | Restrição | O worker é um segundo processo em produção, no mesmo banco; quem o opera não foi definido | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-03 | `docs/PRD.md` | Restrição | O MySQL existente é a única infraestrutura da feature | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-03-C | `docs/PRD.md` | Restrição | O MySQL existente é a única infraestrutura da feature | CODIGO | `docker-compose.yml` |
| PRD-DEP-04 | `docs/PRD.md` | Restrição | Revisão de segurança da Sofia, com pelo menos dois dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia; [09:49] Sofia |
| PRD-DEP-05 | `docs/PRD.md` | Restrição | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-06 | `docs/PRD.md` | Restrição | Marcos documenta a integração e a deduplicação no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-07 | `docs/PRD.md` | Restrição | Os clientes implementam a validação da assinatura e a deduplicação | TRANSCRICAO | [09:25] Sofia; [09:25] Diego |
| PRD-DEP-08 | `docs/PRD.md` | Restrição | A role ADMIN e o `requireRole` existentes são a base da restrição do replay | TRANSCRICAO | [09:36] Larissa |
| PRD-DEP-08-C | `docs/PRD.md` | Restrição | A role ADMIN e o `requireRole` existentes são a base da restrição do replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-ESC-01 | `docs/PRD.md` | Escopo | Cadastro, edição, remoção e listagem de webhooks por customer, com filtro de status | TRANSCRICAO | [09:31] Marcos; [09:33] Bruno |
| PRD-ESC-02 | `docs/PRD.md` | Escopo | Notificação assinada de cada mudança de status assinada pelo cliente | TRANSCRICAO | [09:33] Marcos; [09:20] Sofia |
| PRD-ESC-03 | `docs/PRD.md` | Escopo | Rotação de secret com 24 horas de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-ESC-04 | `docs/PRD.md` | Escopo | Retentativa automática e DLQ | TRANSCRICAO | [09:15] Diego; [09:18] Diego |
| PRD-ESC-05 | `docs/PRD.md` | Escopo | Histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-06 | `docs/PRD.md` | Escopo | Reprocessamento manual da DLQ por ADMIN, com auditoria | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| PRD-ESC-07 | `docs/PRD.md` | Escopo | Documentação de integração no portal, com a deduplicação em destaque | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-FORA-01 | `docs/PRD.md` | Fora de Escopo | Webhooks de entrada: os clientes "querem receber, não mandar" | TRANSCRICAO | [09:02] Marcos |
| PRD-FORA-02 | `docs/PRD.md` | Fora de Escopo | Email de aviso quando o webhook falha, adiado para a próxima fase | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| PRD-FORA-03 | `docs/PRD.md` | Fora de Escopo | Dashboard visual: projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-FORA-04 | `docs/PRD.md` | Fora de Escopo | Rate limiting de envio por cliente, em observação | TRANSCRICAO | [09:39] Diego; [09:39] Larissa |
| PRD-FORA-05 | `docs/PRD.md` | Fora de Escopo | Arquivamento das entregas antigas | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-06 | `docs/PRD.md` | Fora de Escopo | Restringir por role o CRUD de webhooks, adiado | TRANSCRICAO | [09:37] Sofia |
| PRD-FORA-07 | `docs/PRD.md` | Fora de Escopo | Garantia de ordem global entre pedidos, não pedida | TRANSCRICAO | [09:14] Marcos |
| PRD-FORA-08 | `docs/PRD.md` | Fora de Escopo | Entrega exactly-once, descartada pela complexidade | TRANSCRICAO | [09:25] Diego |
| PRD-FORA-09 | `docs/PRD.md` | Fora de Escopo | Items do pedido no payload; o cliente usa `GET /orders/:id` | TRANSCRICAO | [09:43] Diego |
| PRD-FORA-10 | `docs/PRD.md` | Fora de Escopo | Truncar eventos acima do limite de tamanho | TRANSCRICAO | [09:23] Sofia |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo | Notificar em "tempo real": primeiro envio em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo | Não perder a Atlas: feature em produção até o fim de novembro | TRANSCRICAO | [09:45] Marcos; [09:00] Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Absorver indisponibilidades sem ação humana: ~15h de retentativa antes da DLQ | TRANSCRICAO | [09:17] Diego; [09:17] Marcos |
| PRD-QA-01 | `docs/PRD.md` | Questão em Aberto | A meta de 10 segundos não tem percentil definido | TRANSCRICAO | [09:02] Marcos |
| PRD-QA-02 | `docs/PRD.md` | Questão em Aberto | "Latência mínima ... no pior caso" mistura mínimo e pior caso | TRANSCRICAO | [09:10] Larissa |
| PRD-QA-03 | `docs/PRD.md` | Questão em Aberto | Janela de retentativa "12 ou 24 horas" contra "quase 15 horas" | TRANSCRICAO | [09:15] Diego; [09:17] Diego |
| PRD-QA-04 | `docs/PRD.md` | Questão em Aberto | Prazo da Atlas como "fim do trimestre" e como "fim de novembro" | TRANSCRICAO | [09:00] Marcos; [09:45] Marcos |
| PRD-QA-05 | `docs/PRD.md` | Questão em Aberto | Sem linha de base nem meta para a redução do polling em `GET /orders` | TRANSCRICAO | [09:00] Marcos |
| PRD-QA-06 | `docs/PRD.md` | Questão em Aberto | O resumo final não define piloto com os clientes antes da liberação | TRANSCRICAO | [09:48] Larissa |
| PRD-QA-07 | `docs/PRD.md` | Questão em Aberto | Sem meta de adoção: o pedido é dos três clientes, mas sem número como objetivo | TRANSCRICAO | [09:00] Marcos |
| PRD-QA-08 | `docs/PRD.md` | Questão em Aberto | Quem opera o worker em produção não foi definido | TRANSCRICAO | [09:11] Diego |
| PRD-QA-09 | `docs/PRD.md` | Questão em Aberto | A criação do pedido (`PENDING`) não passa por `changeStatus` e não gera evento | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-QA-10 | `docs/PRD.md` | Questão em Aberto | Snapshot e id UUID decididos depois do resumo, sem Marcos e Sofia | TRANSCRICAO | [09:51] Larissa; [09:52] Bruno |
| PRD-QA-11 | `docs/PRD.md` | Questão em Aberto | Não foi escolhido se o `customer_id` vai no body ou no path | TRANSCRICAO | [09:32] Larissa |
| PRD-RF-001 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook com URL https e status; a plataforma gera e devolve a secret | TRANSCRICAO | [09:31] Marcos; [09:32] Larissa; [09:23] Sofia |
| PRD-RF-002 | `docs/PRD.md` | Requisito Funcional | Editar, remover e listar os webhooks de um customer | TRANSCRICAO | [09:33] Bruno; [09:28] Bruno |
| PRD-RF-003 | `docs/PRD.md` | Requisito Funcional | Filtrar por status assinado; sem assinante, nenhum evento é gerado | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-RF-004 | `docs/PRD.md` | Requisito Funcional | Enviar `order.status_changed` com dados básicos, status anterior e novo, sem items | TRANSCRICAO | [09:43] Diego; [09:00] Marcos; [09:44] Diego; [09:44] Sofia |
| PRD-RF-005 | `docs/PRD.md` | Requisito Funcional | O cliente valida origem e integridade pela assinatura com a secret do endpoint | TRANSCRICAO | [09:19] Sofia; [09:20] Sofia; [09:21] Sofia |
| PRD-RF-006 | `docs/PRD.md` | Requisito Funcional | Rotacionar a secret pela API; a antiga vale 24h em paralelo | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| PRD-RF-007 | `docs/PRD.md` | Requisito Funcional | Retentar em 1m/5m/30m/2h/12h e mover para a DLQ com o motivo | TRANSCRICAO | [09:17] Diego; [09:15] Diego; [09:18] Diego |
| PRD-RF-008 | `docs/PRD.md` | Requisito Funcional | Consultar as últimas 100 entregas de um webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-009 | `docs/PRD.md` | Requisito Funcional | ADMIN recoloca evento da DLQ na fila, com registro de quem fez | TRANSCRICAO | [09:18] Diego; [09:36] Sofia; [09:36] Larissa |
| PRD-RF-010 | `docs/PRD.md` | Requisito Funcional | Identificador único e estável por evento para deduplicação no cliente | TRANSCRICAO | [09:25] Diego; [09:24] Diego |
| PRD-RISCO-01 | `docs/PRD.md` | Risco | `customer_id` do body ou path, sem vínculo `User`↔`Customer`: qualquer usuário acessa webhooks de qualquer customer | TRANSCRICAO | [09:32] Larissa; [09:32] Marcos; [09:37] Sofia |
| PRD-RISCO-01-C | `docs/PRD.md` | Risco | `customer_id` do body ou path, sem vínculo `User`↔`Customer`: qualquer usuário acessa webhooks de qualquer customer | CODIGO | `prisma/schema.prisma`; `src/middlewares/auth.middleware.ts` |
| PRD-RISCO-02 | `docs/PRD.md` | Risco | O registro público aceita `role: 'ADMIN'` e anula a restrição do replay | TRANSCRICAO | [09:36] Sofia |
| PRD-RISCO-02-C | `docs/PRD.md` | Risco | O registro público aceita `role: 'ADMIN'` e anula a restrição do replay | CODIGO | `src/modules/auth/auth.schemas.ts`; `src/modules/auth/auth.routes.ts` |
| PRD-RISCO-03 | `docs/PRD.md` | Risco | Três sprints podem não caber até "fim de novembro" | TRANSCRICAO | [09:46] Larissa; [09:45] Marcos; [09:00] Marcos; [09:47] Marcos |
| PRD-RISCO-04 | `docs/PRD.md` | Risco | Clientes que não deduplicam processam o mesmo evento duas vezes | TRANSCRICAO | [09:25] Sofia; [09:26] Marcos |
| PRD-RISCO-05 | `docs/PRD.md` | Risco | Clientes lentos no mesmo lote atrasam os demais além dos 10s | TRANSCRICAO | [09:42] Diego; [09:02] Marcos |
| PRD-RISCO-06 | `docs/PRD.md` | Risco | Um evento em retentativa pode ser ultrapassado pelo seguinte do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| PRD-RISCO-07 | `docs/PRD.md` | Risco | O timestamp não entra na assinatura e não protege contra replay | TRANSCRICAO | [09:44] Diego; [09:22] Sofia |
| PRD-RISCO-08 | `docs/PRD.md` | Risco | Sem rate limiting, o cliente recebe rajadas de chamadas | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| PRD-RISCO-09 | `docs/PRD.md` | Risco | Secrets recuperáveis no banco: um vazamento de leitura expõe todas | TRANSCRICAO | [09:20] Sofia |
| PRD-RISCO-09-C | `docs/PRD.md` | Risco | Secrets recuperáveis no banco: um vazamento de leitura expõe todas | CODIGO | `src/modules/auth/auth.service.ts` |
| PRD-RNF-001 | `docs/PRD.md` | Requisito Não Funcional | Primeiro envio em menos de 10s após a mudança de status | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| PRD-RNF-002 | `docs/PRD.md` | Requisito Não Funcional | Evento e mudança de status atômicos; falha no registro desfaz a mudança | TRANSCRICAO | [09:40] Bruno; [09:41] Diego; [09:06] Diego |
| PRD-RNF-003 | `docs/PRD.md` | Requisito Não Funcional | Cliente lento ou fora do ar não atrasa nem desfaz a mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-004 | `docs/PRD.md` | Requisito Não Funcional | URL obrigatoriamente `https`; `http` recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-005 | `docs/PRD.md` | Requisito Não Funcional | HMAC-SHA256 sobre o corpo, com secret única por endpoint | TRANSCRICAO | [09:22] Sofia; [09:20] Sofia; [09:21] Sofia |
| PRD-RNF-006 | `docs/PRD.md` | Requisito Não Funcional | Evento acima de 64KB não é enviado e gera erro, sem truncar | TRANSCRICAO | [09:24] Larissa; [09:23] Sofia; [09:24] Diego |
| PRD-RNF-007 | `docs/PRD.md` | Requisito Não Funcional | Resposta acima de 10s conta como falha e entra em retentativa | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-008 | `docs/PRD.md` | Requisito Não Funcional | Envio inicial + 5 retentativas em 1m/5m/30m/2h/12h (quase 15h) | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| PRD-RNF-009 | `docs/PRD.md` | Requisito Não Funcional | At-least-once: nenhum evento registrado é descartado sem ir para a DLQ | TRANSCRICAO | [09:24] Diego; [09:26] Larissa |
| PRD-RNF-010 | `docs/PRD.md` | Requisito Não Funcional | Ordem por pedido enquanto houver um único worker; sem ordem global | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| PRD-RNF-011 | `docs/PRD.md` | Requisito Não Funcional | Replay exige ADMIN; CRUD aceita qualquer usuário autenticado nesta fase | TRANSCRICAO | [09:36] Sofia; [09:37] Sofia |
| PRD-RNF-012 | `docs/PRD.md` | Requisito Não Funcional | Todo replay da DLQ registra quem o executou | TRANSCRICAO | [09:36] Sofia |
| PRD-RNF-013 | `docs/PRD.md` | Requisito Não Funcional | Secret só exibida na criação e na rotação, nunca em log | TRANSCRICAO | [09:31] Marcos; [09:21] Sofia |
| PRD-RNF-013-C | `docs/PRD.md` | Requisito Não Funcional | Secret só exibida na criação e na rotação, nunca em log | CODIGO | `src/shared/logger/index.ts` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | Webhook síncrono no service de orders: transação pesada e cliente lento travaria a mudança | TRANSCRICAO | [09:04] Bruno; [09:03] Larissa; [09:06] Diego |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa Descartada | Redis Streams: exigiria subir mais infra, "overengineering" para time pequeno | TRANSCRICAO | [09:07] Diego; [09:07] Larissa |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa Descartada | Trigger do banco para acordar o worker: MySQL não tem listener nativo | TRANSCRICAO | [09:09] Diego; [09:09] Bruno |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa Descartada | Retry indefinido: evento "pendurado pra sempre se o cliente sumiu" | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa Descartada | DLQ como status "failed" na outbox: tabela separada deixa a leitura mais limpa | TRANSCRICAO | [09:18] Diego; [09:17] Larissa |
| RFC-CRIT-01 | `docs/RFC.md` | Restrição | Critério obrigatório: notificação em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CRIT-02 | `docs/RFC.md` | Restrição | Critério obrigatório: cliente lento ou fora do ar não trava nem desfaz a mudança | TRANSCRICAO | [09:04] Bruno |
| RFC-CRIT-03 | `docs/RFC.md` | Restrição | Critério obrigatório: se o status mudou, o evento existe | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| RFC-CRIT-04 | `docs/RFC.md` | Restrição | Critério obrigatório: o cliente valida origem e integridade | TRANSCRICAO | [09:19] Sofia |
| RFC-CRIT-05 | `docs/RFC.md` | Restrição | Critério alto: não subir infraestrutura nova | TRANSCRICAO | [09:07] Diego; [09:07] Larissa |
| RFC-CRIT-06 | `docs/RFC.md` | Restrição | Critério alto: cobrir indisponibilidades de algumas horas | TRANSCRICAO | [09:16] Diego |
| RFC-CRIT-07 | `docs/RFC.md` | Restrição | Critério alto: caber no prazo da Atlas | TRANSCRICAO | [09:45] Marcos |
| RFC-CRIT-08 | `docs/RFC.md` | Restrição | Critério médio: reusar os padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| RFC-DEC-01 | `docs/RFC.md` | Decisão | Outbox no MySQL, gravada na mesma transação da mudança de status (ADR-001) | TRANSCRICAO | [09:08] Larissa |
| RFC-DEC-02 | `docs/RFC.md` | Decisão | Worker em processo separado, polling a cada 2s (ADR-002) | TRANSCRICAO | [09:10] Larissa |
| RFC-DEC-03 | `docs/RFC.md` | Decisão | Worker único, ordering só por `order_id` (ADR-003) | TRANSCRICAO | [09:13] Larissa |
| RFC-DEC-04 | `docs/RFC.md` | Decisão | Retry com backoff em 5 tentativas, DLQ em tabela separada, replay por ADMIN (ADR-004) | TRANSCRICAO | [09:17] Larissa; [09:36] Larissa |
| RFC-DEC-05 | `docs/RFC.md` | Decisão | HMAC-SHA256 com secret por endpoint, rotação com grace period de 24h (ADR-005) | TRANSCRICAO | [09:22] Sofia |
| RFC-DEC-06 | `docs/RFC.md` | Decisão | At-least-once com `X-Event-Id` (ADR-006) | TRANSCRICAO | [09:26] Larissa |
| RFC-DEC-07 | `docs/RFC.md` | Decisão | Reuso de AppError, Pino, error middleware e padrão de módulos (ADR-007) | TRANSCRICAO | [09:30] Larissa |
| RFC-DEC-08 | `docs/RFC.md` | Decisão | Payload como snapshot na inserção da outbox (ADR-008) | TRANSCRICAO | [09:52] Bruno |
| RFC-FORA-01 | `docs/RFC.md` | Fora de Escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| RFC-FORA-02 | `docs/RFC.md` | Fora de Escopo | Arquivamento das linhas entregues | TRANSCRICAO | [09:08] Diego |
| RFC-FORA-03 | `docs/RFC.md` | Fora de Escopo | Aviso por email quando o webhook falha | TRANSCRICAO | [09:37] Larissa |
| RFC-FORA-04 | `docs/RFC.md` | Fora de Escopo | Dashboard visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| RFC-PASSO-01 | `docs/RFC.md` | Restrição | Estimativa de três sprints, incluindo a revisão de segurança | TRANSCRICAO | [09:46] Larissa; [09:47] Larissa |
| RFC-PASSO-02 | `docs/RFC.md` | Restrição | Pelo menos dois dias úteis para a revisão de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| RFC-PASSO-03 | `docs/RFC.md` | Restrição | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-PASSO-04 | `docs/RFC.md` | Restrição | Marcos confirma o prazo com a Atlas e atualiza os clientes | TRANSCRICAO | [09:47] Marcos; [09:49] Marcos |
| RFC-PREM-01 | `docs/RFC.md` | Restrição | Premissa: menos de 10s conta como "tempo real" | TRANSCRICAO | [09:02] Marcos |
| RFC-PREM-02 | `docs/RFC.md` | Restrição | Premissa: os clientes não precisam de ordering global | TRANSCRICAO | [09:14] Marcos |
| RFC-PREM-03 | `docs/RFC.md` | Restrição | Premissa: os clientes conseguem deduplicar pelo `X-Event-Id` | TRANSCRICAO | [09:25] Diego; [09:26] Marcos |
| RFC-PREM-04 | `docs/RFC.md` | Restrição | Premissa: indisponibilidade acima de ~15h é problema do cliente | TRANSCRICAO | [09:17] Marcos |
| RFC-PREM-05 | `docs/RFC.md` | Restrição | Premissa: um único worker dá conta do volume | TRANSCRICAO | [09:13] Diego; [09:12] Diego |
| RFC-QA-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting de envio por cliente, em observação | TRANSCRICAO | [09:39] Larissa; [09:38] Diego |
| RFC-QA-02 | `docs/RFC.md` | Questão em Aberto | Escalar para múltiplos workers, por partição ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | `docs/RFC.md` | Questão em Aberto | Restringir por role o CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-04 | `docs/RFC.md` | Questão em Aberto | `X-Timestamp` fora do conteúdo assinado não detecta replay | TRANSCRICAO | [09:44] Diego; [09:22] Sofia |
| RFC-QA-05 | `docs/RFC.md` | Questão em Aberto | Quem opera o worker em produção | TRANSCRICAO | [09:11] Diego |
| RFC-QA-06 | `docs/RFC.md` | Questão em Aberto | Evento em retry pode ser ultrapassado pelo seguinte do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| RFC-RISCO-01 | `docs/RFC.md` | Risco | Qualquer usuário autenticado acessa webhooks de qualquer customer | TRANSCRICAO | [09:32] Larissa; [09:37] Sofia; [09:46] Sofia |
| RFC-RISCO-01-C | `docs/RFC.md` | Risco | Qualquer usuário autenticado acessa webhooks de qualquer customer | CODIGO | `src/middlewares/auth.middleware.ts` |
| RFC-RISCO-02 | `docs/RFC.md` | Risco | "Usuários que representam o cliente" sem apoio no código: `User` sem relação com `Customer` | TRANSCRICAO | [09:32] Marcos |
| RFC-RISCO-02-C | `docs/RFC.md` | Risco | "Usuários que representam o cliente" sem apoio no código: `User` sem relação com `Customer` | CODIGO | `prisma/schema.prisma`; `src/middlewares/auth.middleware.ts` |
| RFC-RISCO-03 | `docs/RFC.md` | Risco | `POST /auth/register` é público e aceita `role: 'ADMIN'` | TRANSCRICAO | [09:36] Sofia |
| RFC-RISCO-03-C | `docs/RFC.md` | Risco | `POST /auth/register` é público e aceita `role: 'ADMIN'` | CODIGO | `src/modules/auth/auth.routes.ts`; `src/modules/auth/auth.schemas.ts` |
| RFC-RISCO-04 | `docs/RFC.md` | Risco | Três sprints podem não caber até "fim de novembro" | TRANSCRICAO | [09:46] Larissa; [09:45] Marcos; [09:46] Sofia; [09:47] Marcos |
| FDD-CAT-01 | `docs/FDD.md` | Critério de Aceite | Webhook que assina `SHIPPED` gera uma linha `PENDING` com `eventId` igual a `payload.event_id` | TRANSCRICAO | [09:33] Marcos; [09:25] Diego |
| FDD-CAT-02 | `docs/FDD.md` | Critério de Aceite | Webhook que assina só `DELIVERED` não gera linha na mudança para `SHIPPED` | TRANSCRICAO | [09:34] Bruno |
| FDD-CAT-03 | `docs/FDD.md` | Critério de Aceite | Dois webhooks do customer geram duas linhas, uma por `webhookId` | TRANSCRICAO | [09:44] Sofia; [09:21] Sofia |
| FDD-CAT-04 | `docs/FDD.md` | Critério de Aceite | Insert na outbox forçado a falhar: status, histórico e estoque ficam como antes | TRANSCRICAO | [09:40] Bruno |
| FDD-CAT-05 | `docs/FDD.md` | Critério de Aceite | `PENDING → PAID` sem estoque responde 422 `INSUFFICIENT_STOCK` sem linha na outbox | CODIGO | `src/modules/orders/order.service.ts`; `src/shared/errors/http-errors.ts` |
| FDD-CAT-06 | `docs/FDD.md` | Critério de Aceite | `http` → 400 `WEBHOOK_INVALID_URL`; `events` vazio ou com `PENDING` → `VALIDATION_ERROR` | TRANSCRICAO | [09:23] Sofia; [09:28] Bruno |
| FDD-CAT-06-C | `docs/FDD.md` | Critério de Aceite | `http` → 400 `WEBHOOK_INVALID_URL`; `events` vazio ou com `PENDING` → `VALIDATION_ERROR` | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-CAT-07 | `docs/FDD.md` | Critério de Aceite | POST devolve secret de 64 hex; GET e PATCH não devolvem secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CAT-08 | `docs/FDD.md` | Critério de Aceite | `customerId` inexistente responde 404 `WEBHOOK_CUSTOMER_NOT_FOUND`, não 500 | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-CAT-09 | `docs/FDD.md` | Critério de Aceite | Evento inserido chega ao servidor de teste em menos de 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-CAT-10 | `docs/FDD.md` | Critério de Aceite | Request traz os cinco headers, HMAC do rawBody em hex e corpo sem items | TRANSCRICAO | [09:44] Diego; [09:44] Sofia; [09:22] Sofia; [09:43] Diego |
| FDD-CAT-11 | `docs/FDD.md` | Critério de Aceite | Cliente com 500 recebe envios em 1m/5m/30m/2h/12h e vai para a DLQ com `attempts = 6` | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| FDD-CAT-12 | `docs/FDD.md` | Critério de Aceite | Timeout de 10s gera delivery com `error = "timeout"` e outbox `FAILED` | TRANSCRICAO | [09:42] Diego |
| FDD-CAT-13 | `docs/FDD.md` | Critério de Aceite | Resposta 302 conta como falha e o redirect não é seguido | TRANSCRICAO | [09:23] Sofia |
| FDD-CAT-14 | `docs/FDD.md` | Critério de Aceite | Payload acima de 65536 bytes vai para a DLQ sem requisição ao cliente | TRANSCRICAO | [09:24] Larissa; [09:23] Sofia |
| FDD-CAT-15 | `docs/FDD.md` | Critério de Aceite | `X-Event-Id` idêntico no envio inicial, nas retentativas e após o replay | TRANSCRICAO | [09:25] Diego |
| FDD-CAT-16 | `docs/FDD.md` | Critério de Aceite | Pedido alterado depois da inserção é enviado com o payload do momento da mudança | TRANSCRICAO | [09:52] Larissa |
| FDD-CAT-17 | `docs/FDD.md` | Critério de Aceite | Replay: OPERATOR 403; ADMIN 202 com mesmo `eventId` e log com `userId` | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| FDD-CAT-18 | `docs/FDD.md` | Critério de Aceite | Segundo replay responde 409; replays simultâneos criam uma só linha (decisão do FDD, Q-14) | TRANSCRICAO | [09:18] Diego |
| FDD-CAT-19 | `docs/FDD.md` | Critério de Aceite | Rotação: duas assinaturas por 24h, depois só a nova; nova rotação no período responde 409 (Q-13) | TRANSCRICAO | [09:21] Sofia |
| FDD-CAT-20 | `docs/FDD.md` | Critério de Aceite | Linha presa em `PROCESSING` volta a `PENDING` quando o worker reinicia | TRANSCRICAO | [09:08] Diego |
| FDD-CAT-21 | `docs/FDD.md` | Critério de Aceite | Deliveries com no máximo 100 itens em ordem decrescente; `pageSize=101` → 400 | TRANSCRICAO | [09:34] Marcos |
| FDD-CAT-21-C | `docs/FDD.md` | Critério de Aceite | Deliveries com no máximo 100 itens em ordem decrescente; `pageSize=101` → 400 | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-CAT-22 | `docs/FDD.md` | Critério de Aceite | Nenhum log da API ou do worker contém secret ou assinatura | CODIGO | `src/shared/logger/index.ts` |
| FDD-CAT-23 | `docs/FDD.md` | Critério de Aceite | `src/worker.ts` usa `createPrismaClient()`, não importa `src/app.ts`, sem dependência nova | TRANSCRICAO | [09:30] Bruno; [09:29] Bruno |
| FDD-CAT-23-C | `docs/FDD.md` | Critério de Aceite | `src/worker.ts` usa `createPrismaClient()`, não importa `src/app.ts`, sem dependência nova | CODIGO | `src/config/database.ts` |
| FDD-CONFIG-01 | `docs/FDD.md` | Restrição | `WEBHOOK_POLL_INTERVAL_MS` = 2000 | TRANSCRICAO | [09:09] Diego; [09:10] Larissa |
| FDD-CONFIG-02 | `docs/FDD.md` | Restrição | `WEBHOOK_HTTP_TIMEOUT_MS` = 10000 | TRANSCRICAO | [09:42] Diego |
| FDD-CONFIG-03 | `docs/FDD.md` | Restrição | `WEBHOOK_BATCH_SIZE` = 10, proposta sobre o "batch pequeno" (Q-04) | TRANSCRICAO | [09:08] Diego |
| FDD-CONFIG-04 | `docs/FDD.md` | Restrição | `RETRY_DELAYS_MS` = 1m, 5m, 30m, 2h, 12h | TRANSCRICAO | [09:17] Diego |
| FDD-CONFIG-05 | `docs/FDD.md` | Restrição | `MAX_SENDS` = 6 (envio inicial + 5 retentativas) | TRANSCRICAO | [09:17] Diego |
| FDD-CONFIG-06 | `docs/FDD.md` | Restrição | `MAX_PAYLOAD_BYTES` = 65536 | TRANSCRICAO | [09:24] Diego; [09:24] Larissa |
| FDD-CONFIG-07 | `docs/FDD.md` | Restrição | `SECRET_GRACE_PERIOD_MS` = 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONFIG-08 | `docs/FDD.md` | Restrição | `SECRET_BYTES` = 32, proposta a revisar (Q-03) | TRANSCRICAO | [09:46] Sofia |
| FDD-CONFIG-09 | `docs/FDD.md` | Restrição | `DELIVERIES_MAX_PAGE_SIZE` = 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CONFIG-10 | `docs/FDD.md` | Restrição | `RESPONSE_BODY_MAX_CHARS` a definir (Q-05) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONFIG-11 | `docs/FDD.md` | Restrição | `EVENT_TYPE` = `order.status_changed` | TRANSCRICAO | [09:43] Diego |
| FDD-CONFIG-12 | `docs/FDD.md` | Restrição | `DATABASE_URL` igual à da API | TRANSCRICAO | [09:30] Bruno |
| FDD-CONFIG-12-C | `docs/FDD.md` | Restrição | `DATABASE_URL` igual à da API | CODIGO | `src/config/env.ts` |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | `POST /api/v1/webhooks`: `url` https, `events` sem `PENDING`, secret gerada, 201 | TRANSCRICAO | [09:31] Marcos; [09:23] Sofia; [09:33] Marcos |
| FDD-CONTRATO-01-C | `docs/FDD.md` | Contrato | `POST /api/v1/webhooks`: `url` https, `events` sem `PENDING`, secret gerada, 201 | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | `GET /api/v1/webhooks?customerId=`: lista paginada, sem secrets | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-02-C | `docs/FDD.md` | Contrato | `GET /api/v1/webhooks?customerId=`: lista paginada, sem secrets | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | `GET /api/v1/webhooks/:id`: proposta deste FDD, no padrão de `GET /orders/:id` | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | `PATCH /api/v1/webhooks/:id`: edita `url`, `events` e `active` | TRANSCRICAO | [09:33] Bruno; [09:34] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | `DELETE /api/v1/webhooks/:id`: 204, com cascade | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05-C | `docs/FDD.md` | Contrato | `DELETE /api/v1/webhooks/:id`: 204, com cascade | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | `POST /api/v1/webhooks/:id/rotate-secret`: nova secret, antiga válida 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | `GET /api/v1/webhooks/:id/deliveries`: até 100 envios, mais recentes primeiro | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: ADMIN, 202 | TRANSCRICAO | [09:18] Diego; [09:36] Sofia; [09:36] Larissa |
| FDD-CONTRATO-08-C | `docs/FDD.md` | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: ADMIN, 202 | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | Chamada de saída: headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`; payload sem items; timeout 10s | TRANSCRICAO | [09:44] Diego; [09:44] Sofia; [09:43] Diego; [09:42] Diego |
| FDD-ERRO-01 | `docs/FDD.md` | Erro | `WEBHOOK_NOT_FOUND` 404 | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | `docs/FDD.md` | Erro | `WEBHOOK_INVALID_URL` 400, lançado no service | TRANSCRICAO | [09:28] Bruno; [09:23] Sofia |
| FDD-ERRO-02-C | `docs/FDD.md` | Erro | `WEBHOOK_INVALID_URL` 400, lançado no service | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-ERRO-03 | `docs/FDD.md` | Erro | `WEBHOOK_CUSTOMER_NOT_FOUND` 404, para não cair no `P2003` → 500 | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-ERRO-04 | `docs/FDD.md` | Erro | `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` 409, decisão deste FDD (Q-13) | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-05 | `docs/FDD.md` | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` 404 | TRANSCRICAO | [09:29] Larissa; [09:18] Diego |
| FDD-ERRO-06 | `docs/FDD.md` | Erro | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` 409, decisão deste FDD (Q-14) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-07 | `docs/FDD.md` | Erro | `WEBHOOK_INACTIVE` 409 no replay, decisão deste FDD (Q-12) | TRANSCRICAO | [09:21] Bruno |
| FDD-ESC-01 | `docs/FDD.md` | Escopo | Módulo `src/modules/webhooks` com CRUD, rotação, histórico e replay | TRANSCRICAO | [09:27] Bruno; [09:33] Bruno; [09:34] Marcos; [09:18] Diego |
| FDD-ESC-02 | `docs/FDD.md` | Escopo | Tabelas `webhooks`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter` | TRANSCRICAO | [09:06] Diego; [09:21] Bruno; [09:34] Marcos; [09:18] Diego |
| FDD-ESC-03 | `docs/FDD.md` | Escopo | `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro de `changeStatus` | TRANSCRICAO | [09:41] Bruno |
| FDD-ESC-04 | `docs/FDD.md` | Escopo | Processo `src/worker.ts` e script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| FDD-FORA-01 | `docs/FDD.md` | Fora de Escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| FDD-FORA-02 | `docs/FDD.md` | Fora de Escopo | Arquivamento das linhas entregues da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-FORA-03 | `docs/FDD.md` | Fora de Escopo | Email de aviso quando o webhook falha | TRANSCRICAO | [09:37] Larissa |
| FDD-FORA-04 | `docs/FDD.md` | Fora de Escopo | Rate limiting de envio, em observação | TRANSCRICAO | [09:39] Larissa |
| FDD-FORA-05 | `docs/FDD.md` | Fora de Escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-FORA-06 | `docs/FDD.md` | Fora de Escopo | Múltiplos workers | TRANSCRICAO | [09:13] Diego |
| FDD-FORA-07 | `docs/FDD.md` | Fora de Escopo | Restrição por role do CRUD, adiada | TRANSCRICAO | [09:37] Sofia |
| FDD-FORA-08 | `docs/FDD.md` | Fora de Escopo | Vínculo `User`↔`Customer`, inexistente no código e não decidido | CODIGO | `prisma/schema.prisma` |
| FDD-FORA-09 | `docs/FDD.md` | Fora de Escopo | Evento para o `PENDING` inicial, gravado em `create` sem `changeStatus` (Q-07) | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-01 | `docs/FDD.md` | Integração | `changeStatus` chama `publishWebhookEvent` após o `orderStatusHistory.create` | TRANSCRICAO | [09:40] Bruno; [09:41] Bruno |
| FDD-INT-01-C | `docs/FDD.md` | Integração | `changeStatus` chama `publishWebhookEvent` após o `orderStatusHistory.create` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Mapa de transições define os valores de `events` (sem `PENDING`); regras de estoque intactas | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | `buildControllers` instancia repository, service e controller de webhooks | CODIGO | `src/app.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | `buildApiRouter` monta `/webhooks` e `/admin` | CODIGO | `src/routes/index.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | `authenticate` no router de webhooks e `requireRole('ADMIN')` no replay | TRANSCRICAO | [09:36] Larissa |
| FDD-INT-05-C | `docs/FDD.md` | Integração | `authenticate` no router de webhooks e `requireRole('ADMIN')` no replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | `validate` para body, query e params dos endpoints novos | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | `errorMiddleware` reutilizado sem mudança | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-07-C | `docs/FDD.md` | Integração | `errorMiddleware` reutilizado sem mudança | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Classes `WEBHOOK_*` estendem as classes de erro existentes | TRANSCRICAO | [09:28] Bruno |
| FDD-INT-08-C | `docs/FDD.md` | Integração | Classes `WEBHOOK_*` estendem as classes de erro existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | `redactPaths` ganha secret e assinatura; worker usa `logger.child` | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-09-C | `docs/FDD.md` | Integração | `redactPaths` ganha secret e assinatura; worker usa `logger.child` | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | `envSchema` ganha três variáveis `WEBHOOK_*` com default | CODIGO | `src/config/env.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Worker chama `createPrismaClient()` e não usa o singleton | TRANSCRICAO | [09:30] Bruno |
| FDD-INT-11-C | `docs/FDD.md` | Integração | Worker chama `createPrismaClient()` e não usa o singleton | CODIGO | `src/config/database.ts` |
| FDD-INT-12 | `docs/FDD.md` | Integração | `src/server.ts` serve de modelo para o worker, sem mudar | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-12-C | `docs/FDD.md` | Integração | `src/server.ts` serve de modelo para o worker, sem mudar | CODIGO | `src/server.ts` |
| FDD-INT-13 | `docs/FDD.md` | Integração | Schema ganha 4 models e o enum `WebhookOutboxStatus` | CODIGO | `prisma/schema.prisma` |
| FDD-INT-14 | `docs/FDD.md` | Integração | Migration nova ao lado de `20260519182739_init` | CODIGO | `prisma/migrations/` |
| FDD-INT-15 | `docs/FDD.md` | Integração | Scripts `worker` e `start:worker`, sem dependência nova | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-15-C | `docs/FDD.md` | Integração | Scripts `worker` e `start:worker`, sem dependência nova | CODIGO | `package.json` |
| FDD-INT-16 | `docs/FDD.md` | Integração | `tests/setup.ts` apaga as tabelas de webhook antes de `customer` e `user` | CODIGO | `tests/setup.ts` |
| FDD-INT-17 | `docs/FDD.md` | Integração | `src/worker.ts` a criar, entry-point do worker | TRANSCRICAO | [09:11] Larissa; [09:28] Bruno |
| FDD-INT-18 | `docs/FDD.md` | Integração | Arquivos de routes, controller, service, repository e schemas do módulo, a criar | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-18-C | `docs/FDD.md` | Integração | Arquivos de routes, controller, service, repository e schemas do módulo, a criar | CODIGO | `src/modules/orders/` |
| FDD-INT-19 | `docs/FDD.md` | Integração | `webhook.publisher.ts` a criar, com `publishWebhookEvent` | TRANSCRICAO | [09:41] Bruno |
| FDD-INT-20 | `docs/FDD.md` | Integração | `webhook.processor.ts` a criar, com ciclo, envio, retry e DLQ | TRANSCRICAO | [09:28] Bruno |
| FDD-INT-21 | `docs/FDD.md` | Integração | `webhook.signature.ts`, `webhook.errors.ts` e `webhook.constants.ts` a criar | TRANSCRICAO | [09:20] Sofia; [09:28] Bruno; [09:29] Larissa |
| FDD-MODELO-01 | `docs/FDD.md` | Decisão | Convenções do schema: id UUID `Char(36)`, `@@map` snake_case | TRANSCRICAO | [09:51] Larissa |
| FDD-MODELO-01-C | `docs/FDD.md` | Decisão | Convenções do schema: id UUID `Char(36)`, `@@map` snake_case | CODIGO | `prisma/schema.prisma` |
| FDD-MODELO-02 | `docs/FDD.md` | Decisão | `webhooks`: url, secret, customerId, ativo e lista de status | TRANSCRICAO | [09:21] Bruno; [09:33] Marcos; [09:44] Sofia |
| FDD-MODELO-03 | `docs/FDD.md` | Decisão | `webhook_outbox`: status pendente/processando/falhou/entregue, índices e `eventId` UUID | TRANSCRICAO | [09:08] Diego; [09:06] Diego; [09:25] Diego |
| FDD-MODELO-03-C | `docs/FDD.md` | Decisão | `webhook_outbox`: status pendente/processando/falhou/entregue, índices e `eventId` UUID | CODIGO | `src/modules/orders/order.service.ts`; `src/middlewares/error.middleware.ts` |
| FDD-MODELO-04 | `docs/FDD.md` | Decisão | `webhook_deliveries`: sucesso/falha, payload, resposta e tempo por envio | TRANSCRICAO | [09:34] Marcos |
| FDD-MODELO-05 | `docs/FDD.md` | Decisão | `webhook_dead_letter`: payload, motivo, timestamp e quem fez o replay | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-OBJ-01 | `docs/FDD.md` | Objetivo | `changeStatus` que comita gera uma linha por webhook assinante; rollback não deixa linha | TRANSCRICAO | [09:40] Bruno; [09:06] Diego |
| FDD-OBJ-02 | `docs/FDD.md` | Objetivo | Primeiro envio em menos de 10s, com polling de 2s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| FDD-OBJ-03 | `docs/FDD.md` | Objetivo | Nenhum HTTP dentro da transação nem no processo da API | TRANSCRICAO | [09:04] Bruno; [09:11] Diego |
| FDD-OBJ-04 | `docs/FDD.md` | Objetivo | Todo evento termina entregue ou na DLQ após o envio inicial e 5 retentativas | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| FDD-OBJ-05 | `docs/FDD.md` | Objetivo | `X-Signature` HMAC-SHA256 do corpo e `X-Event-Id` estável | TRANSCRICAO | [09:22] Sofia; [09:25] Diego |
| FDD-OBJ-06 | `docs/FDD.md` | Objetivo | Nenhuma dependência nova: `fetch` nativo e `node:crypto` | TRANSCRICAO | [09:29] Bruno |
| FDD-OBJ-06-C | `docs/FDD.md` | Objetivo | Nenhuma dependência nova: `fetch` nativo e `node:crypto` | CODIGO | `package.json` |
| FDD-Q-01 | `docs/FDD.md` | Questão em Aberto | Caminho do endpoint de rotação, não nomeado na reunião | TRANSCRICAO | [09:21] Sofia |
| FDD-Q-02 | `docs/FDD.md` | Questão em Aberto | Formato de `X-Signature` e uma ou duas assinaturas durante a rotação | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia |
| FDD-Q-03 | `docs/FDD.md` | Questão em Aberto | Tamanho e formato da secret gerada | TRANSCRICAO | [09:46] Sofia |
| FDD-Q-04 | `docs/FDD.md` | Questão em Aberto | Tamanho do lote ("batch pequeno") | TRANSCRICAO | [09:08] Diego |
| FDD-Q-05 | `docs/FDD.md` | Questão em Aberto | Limite de truncamento de `responseBody` | TRANSCRICAO | [09:34] Marcos |
| FDD-Q-06 | `docs/FDD.md` | Questão em Aberto | Como o ADMIN descobre o id da DLQ sem listagem | TRANSCRICAO | [09:18] Diego |
| FDD-Q-07 | `docs/FDD.md` | Questão em Aberto | A criação do pedido (`null → PENDING`) deve gerar evento? | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-Q-08 | `docs/FDD.md` | Questão em Aberto | Adotar biblioteca de métricas ou derivar dos logs | CODIGO | `package.json` |
| FDD-Q-09 | `docs/FDD.md` | Questão em Aberto | Só 2xx conta como sucesso; redirects não seguidos por causa do `https` | TRANSCRICAO | [09:23] Sofia |
| FDD-Q-10 | `docs/FDD.md` | Questão em Aberto | "Campos básicos" do payload além de `total_cents` | TRANSCRICAO | [09:43] Diego |
| FDD-Q-11 | `docs/FDD.md` | Questão em Aberto | Confirmar a DLQ com `WEBHOOK_INACTIVE` para webhook inativo no envio | TRANSCRICAO | [09:21] Bruno; [09:15] Diego; [09:50] Larissa |
| FDD-Q-12 | `docs/FDD.md` | Questão em Aberto | Confirmar o 409 no replay para webhook inativo | TRANSCRICAO | [09:21] Bruno; [09:50] Larissa |
| FDD-Q-13 | `docs/FDD.md` | Questão em Aberto | Confirmar a recusa de nova rotação durante o grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-Q-14 | `docs/FDD.md` | Questão em Aberto | Confirmar a recusa de replay de linha da DLQ já reprocessada | TRANSCRICAO | [09:18] Diego |
| FDD-R-01 | `docs/FDD.md` | Risco | Qualquer usuário autenticado acessa webhooks, secrets e deliveries de qualquer customer | TRANSCRICAO | [09:32] Larissa; [09:37] Sofia |
| FDD-R-01-C | `docs/FDD.md` | Risco | Qualquer usuário autenticado acessa webhooks, secrets e deliveries de qualquer customer | CODIGO | `prisma/schema.prisma` |
| FDD-R-02 | `docs/FDD.md` | Risco | `X-Timestamp` fora da assinatura não detecta replay | TRANSCRICAO | [09:44] Diego; [09:22] Sofia |
| FDD-R-03 | `docs/FDD.md` | Risco | Sem jitter, eventos que falham juntos voltam juntos a cada degrau | TRANSCRICAO | [09:17] Diego; [09:39] Larissa |
| FDD-R-04 | `docs/FDD.md` | Risco | Evento em `FAILED` pode ser ultrapassado; empates em `createdAt` sem ordem | TRANSCRICAO | [09:12] Diego |
| FDD-R-05 | `docs/FDD.md` | Risco | Envios sequenciais: clientes em timeout atrasam a fila além dos 10s | TRANSCRICAO | [09:42] Diego; [09:02] Marcos |
| FDD-R-06 | `docs/FDD.md` | Risco | Secrets em texto no banco; cifragem em repouso não discutida | TRANSCRICAO | [09:46] Sofia; [09:20] Sofia |
| FDD-R-06-C | `docs/FDD.md` | Risco | Secrets em texto no banco; cifragem em repouso não discutida | CODIGO | `src/modules/users/user.service.ts` |
| FDD-R-07 | `docs/FDD.md` | Risco | `POST /auth/register` público aceita `role: 'ADMIN'` | TRANSCRICAO | [09:36] Sofia |
| FDD-R-07-C | `docs/FDD.md` | Risco | `POST /auth/register` público aceita `role: 'ADMIN'` | CODIGO | `src/modules/auth/auth.schemas.ts`; `src/modules/auth/auth.routes.ts` |
| FDD-R-08 | `docs/FDD.md` | Risco | Pool do worker soma conexões ao da API | TRANSCRICAO | [09:30] Bruno |
| FDD-R-09 | `docs/FDD.md` | Risco | `DELETE /webhooks/:id` apaga em cascata e perde a evidência de debug | TRANSCRICAO | [09:18] Diego; [09:50] Larissa |
| FDD-R-10 | `docs/FDD.md` | Risco | `changeStatus` sem lock: `PATCH` concorrentes geram eventos com o mesmo `from_status` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-R-11 | `docs/FDD.md` | Risco | `webhook_outbox` e `webhook_deliveries` crescem sem arquivamento | TRANSCRICAO | [09:08] Diego |
| ADR-001 | `docs/adrs/ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md` | Decisão | Outbox no MySQL existente, inserida na transação de `changeStatus` via `publishWebhookEvent(tx, ...)` | TRANSCRICAO | [09:08] Larissa; [09:06] Diego; [09:41] Bruno; [09:48] Larissa |
| ADR-001-C | `docs/adrs/ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md` | Decisão | Outbox no MySQL existente, inserida na transação de `changeStatus` via `publishWebhookEvent(tx, ...)` | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md` | Alternativa Descartada | Disparo HTTP síncrono: cliente lento travaria a mudança de status | TRANSCRICAO | [09:04] Bruno; [09:03] Larissa |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md` | Alternativa Descartada | Redis Streams: subir mais infra é "overengineering" | TRANSCRICAO | [09:07] Diego; [09:07] Larissa |
| ADR-002 | `docs/adrs/ADR-002-processar-outbox-com-worker-separado-em-polling.md` | Decisão | Worker em processo separado, polling a cada 2s, PrismaClient próprio | TRANSCRICAO | [09:10] Larissa; [09:09] Diego; [09:11] Diego; [09:30] Bruno |
| ADR-002-C | `docs/adrs/ADR-002-processar-outbox-com-worker-separado-em-polling.md` | Decisão | Worker em processo separado, polling a cada 2s, PrismaClient próprio | CODIGO | `src/server.ts`; `src/config/database.ts` |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-processar-outbox-com-worker-separado-em-polling.md` | Alternativa Descartada | Trigger do banco: não notifica processo externo | TRANSCRICAO | [09:09] Diego; [09:09] Bruno |
| ADR-002-ALT-02 | `docs/adrs/ADR-002-processar-outbox-com-worker-separado-em-polling.md` | Alternativa Descartada | Worker no processo da API: "se a API reinicia, perde o worker" | TRANSCRICAO | [09:11] Diego |
| ADR-003 | `docs/adrs/ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md` | Decisão | Um único worker, ordering só por `order_id`, sem ordem global | TRANSCRICAO | [09:13] Larissa; [09:12] Diego; [09:14] Marcos |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md` | Alternativa Descartada | Múltiplos workers com partição por `order_id`, adiado | TRANSCRICAO | [09:13] Diego |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md` | Alternativa Descartada | Múltiplos workers com lock pessimista, adiado | TRANSCRICAO | [09:13] Diego |
| ADR-004 | `docs/adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md` | Decisão | 5 tentativas em 1m/5m/30m/2h/12h, DLQ em `webhook_dead_letter`, replay por ADMIN com log | TRANSCRICAO | [09:17] Larissa; [09:17] Diego; [09:18] Diego; [09:36] Larissa; [09:36] Sofia |
| ADR-004-C | `docs/adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md` | Decisão | 5 tentativas em 1m/5m/30m/2h/12h, DLQ em `webhook_dead_letter`, replay por ADMIN com log | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md` | Alternativa Descartada | Retry indefinido: evento "pendurado pra sempre" | TRANSCRICAO | [09:15] Diego |
| ADR-004-ALT-02 | `docs/adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md` | Alternativa Descartada | 3 tentativas: "3 é pouco" | TRANSCRICAO | [09:16] Diego; [09:16] Bruno |
| ADR-004-ALT-03 | `docs/adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md` | Alternativa Descartada | Status "failed" na própria outbox, sem tabela de DLQ | TRANSCRICAO | [09:18] Diego; [09:17] Larissa |
| ADR-005 | `docs/adrs/ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia; [09:20] Sofia; [09:21] Sofia |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md` | Alternativa Descartada | Secret global: "se vaza uma, vaza tudo" | TRANSCRICAO | [09:21] Sofia |
| ADR-005-ALT-02 | `docs/adrs/ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md` | Alternativa Descartada | Só TLS, sem assinatura (não discutida): não prova a origem | TRANSCRICAO | [09:19] Sofia |
| ADR-006 | `docs/adrs/ADR-006-garantir-entrega-at-least-once-com-x-event-id.md` | Decisão | At-least-once com `X-Event-Id` UUID gerado na entrada da outbox | TRANSCRICAO | [09:26] Larissa; [09:25] Diego; [09:24] Diego |
| ADR-006-ALT-01 | `docs/adrs/ADR-006-garantir-entrega-at-least-once-com-x-event-id.md` | Alternativa Descartada | Exactly-once: "exigiria coordenação dos dois lados" | TRANSCRICAO | [09:25] Diego |
| ADR-006-ALT-02 | `docs/adrs/ADR-006-garantir-entrega-at-least-once-com-x-event-id.md` | Alternativa Descartada | At-most-once (não discutida): perde eventos, contra o retry decidido | TRANSCRICAO | [09:17] Larissa |
| ADR-007 | `docs/adrs/ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md` | Decisão | Módulo `src/modules/webhooks` reusando AppError, prefixo `WEBHOOK_`, Pino, error middleware e Zod | TRANSCRICAO | [09:30] Larissa; [09:27] Bruno; [09:28] Bruno; [09:29] Bruno; [09:29] Larissa |
| ADR-007-C | `docs/adrs/ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md` | Decisão | Módulo `src/modules/webhooks` reusando AppError, prefixo `WEBHOOK_`, Pino, error middleware e Zod | CODIGO | `src/modules/orders/`; `src/shared/errors/app-error.ts`; `src/middlewares/error.middleware.ts`; `src/shared/logger/index.ts` |
| ADR-007-ALT-01 | `docs/adrs/ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md` | Alternativa Descartada | Serviço separado (não discutida): contraria "mesma stack" e o time pequeno | TRANSCRICAO | [09:11] Diego; [09:07] Diego |
| ADR-007-ALT-02 | `docs/adrs/ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md` | Alternativa Descartada | Bibliotecas próprias de erro, log ou fila (não discutida): "Não vamos botar nada novo" | TRANSCRICAO | [09:29] Bruno |
| ADR-008 | `docs/adrs/ADR-008-gravar-payload-como-snapshot-na-insercao.md` | Decisão | Payload renderizado como snapshot na inserção da outbox | TRANSCRICAO | [09:52] Bruno; [09:52] Larissa; [09:52] Diego |
| ADR-008-ALT-01 | `docs/adrs/ADR-008-gravar-payload-como-snapshot-na-insercao.md` | Alternativa Descartada | Renderizar no envio: o evento mostraria o pedido de outro momento | TRANSCRICAO | [09:52] Larissa; [09:51] Bruno |
| ADR-008-ALT-02 | `docs/adrs/ADR-008-gravar-payload-como-snapshot-na-insercao.md` | Alternativa Descartada | Incluir os items: infla o payload | TRANSCRICAO | [09:43] Diego |

## Cobertura

| Documento | Itens identificados | Itens rastreados | Cobertura |
| --- | --- | --- | --- |
| `PRD.md` | 93 | 93 | 100% |
| `RFC.md` | 44 | 44 | 100% |
| `FDD.md` | 121 | 121 | 100% |
| `ADRs` | 25 | 25 | 100% |
| **Total** | 283 | 283 | 100% |

Linhas por fonte: TRANSCRICAO 264 (82.2%) · CODIGO 57 (17.8%)
