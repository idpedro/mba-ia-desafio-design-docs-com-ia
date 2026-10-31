# RFC: Notificar clientes B2B sobre mudanças de status de pedido por webhooks, com outbox no MySQL e worker dedicado

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | 2026-10-30 |
| Revisores | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| Impacto | Alto — altera a transação de `changeStatus`, cria um segundo processo em produção e um contrato externo com clientes B2B |
| Relacionados | [PRD](PRD.md), [Design](FDD.md), [ADRs](adrs/README.md) |

## Resumo (TL;DR)

Três clientes B2B consultam `GET /orders` periodicamente para saber se um pedido mudou de status, e um deles ameaça migrar para o concorrente se não houver notificação em tempo real ([09:00] Marcos). Propomos webhooks de saída: a mudança de status grava um evento numa outbox no MySQL, na mesma transação, e um worker separado entrega o evento por HTTP assinado com HMAC. O principal trade-off aceito é a entrega at-least-once com até 2 segundos de atraso do polling, o que exige que o cliente dedupe pelo `X-Event-Id`, em troca de não subir nenhuma infraestrutura nova ([09:07] Diego, [09:10] Larissa, [09:25] Diego).

## Contexto e Problema

Hoje `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) muda o status numa transação que atualiza `orders`, insere em `order_status_history` e ajusta o estoque. Não há notificação: o cliente só descobre a mudança consultando `GET /orders` ou `GET /orders/:id` (`src/modules/orders/order.routes.ts`).

Atlas Comercial, MaxDistribuição e Nova Cargo pediram notificação em tempo real. O polling deixa a integração deles "lenta e cara", e não fazer nada arrisca perder a Atlas, que falou em migrar para o concorrente se a entrega não sair até o fim do trimestre ([09:00] Marcos), prazo depois citado como "fim de novembro" ([09:45] Marcos). Para os clientes, "tempo real" é menos de 10 segundos ([09:02] Marcos).

Ficam fora desta proposta: webhooks de entrada ([09:02] Marcos), arquivamento das linhas entregues ([09:08] Diego), aviso por email quando o webhook falha ([09:37] Larissa) e dashboard visual para o cliente ([09:40] Larissa).

## Premissas

| Premissa | Fonte | Gatilho de invalidação |
| --- | --- | --- |
| Menos de 10 segundos de atraso conta como "tempo real" para os clientes | [09:02] Marcos | Um cliente exigir latência menor que o ciclo de polling |
| Os clientes não precisam de ordering global, só saber se cada pedido mudou | [09:14] Marcos | Um cliente depender da ordem entre pedidos diferentes |
| Os clientes conseguem deduplicar eventos pelo `X-Event-Id` | [09:25] Diego, [09:26] Marcos | Reclamações de efeito duplicado no lado do cliente |
| Uma indisponibilidade de cliente acima de ~15 horas é problema do cliente | [09:17] Marcos | Volume recorrente na DLQ vindo de clientes ativos |
| Um único worker dá conta do volume de eventos | [09:12] Diego, [09:13] Diego | Fila de pendentes crescendo além do ciclo de 2 segundos |

## Critérios de Decisão

| Critério | Peso | Fonte |
| --- | --- | --- |
| Entregar a notificação em menos de 10 segundos | Obrigatório | [09:02] Marcos |
| Um cliente lento ou fora do ar não pode travar nem desfazer a mudança de status | Obrigatório | [09:04] Bruno |
| Se o status mudou, o evento tem de existir, sem caso de status mudar e evento não sair | Obrigatório | [09:40] Bruno, [09:41] Diego |
| O cliente consegue validar origem e integridade da requisição | Obrigatório | [09:19] Sofia |
| Não subir infraestrutura nova, porque o time é pequeno | Alto | [09:07] Larissa, [09:07] Diego |
| Cobrir indisponibilidades de cliente de algumas horas | Alto | [09:16] Diego |
| Caber no prazo da Atlas | Alto | [09:45] Marcos |
| Reusar os padrões que já existem no projeto | Médio | [09:30] Larissa |

## Proposta Técnica

São três peças. Um módulo novo, `src/modules/webhooks` (a criar), guarda a configuração dos endpoints de cada customer e expõe o CRUD, o histórico de entregas e o replay da DLQ ([09:27] Bruno, [09:33] Bruno, [09:34] Marcos). Uma tabela de outbox no MySQL recebe os eventos dentro da transação de `changeStatus`, por uma função que recebe o `tx` ([09:41] Bruno). Um worker roda como processo separado da API, com entry-point própria e PrismaClient próprio, lê a outbox e faz as chamadas HTTP ([09:11] Larissa, [09:30] Bruno).

Fluxo de alto nível:

1. `changeStatus` muda o status e, na mesma transação, grava o evento na outbox se algum webhook do customer assina o novo status. Se nenhum assina, nada é gravado ([09:34] Bruno, [09:34] Diego).
2. O worker busca os eventos pendentes mais antigos a cada 2 segundos.
3. Para cada evento, o worker assina o corpo e faz o POST para a URL https cadastrada.
4. Uma resposta de sucesso marca o evento como entregue. Uma falha agenda nova tentativa com backoff, e, ao esgotar as tentativas, o evento vai para a DLQ.
5. Um ADMIN pode recolocar um evento da DLQ na outbox pelo endpoint de replay.
6. O cliente valida a assinatura e descarta duplicatas pelo `X-Event-Id`.

Payload, headers, códigos de erro, timeout, limite de tamanho e o modelo das tabelas ficam no [documento de design](FDD.md).

Decisões estruturais:
- Outbox no MySQL, gravada na mesma transação da mudança de status — [ADR-001](adrs/ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md)
- Worker em processo separado, por polling a cada 2 segundos — [ADR-002](adrs/ADR-002-processar-outbox-com-worker-separado-em-polling.md)
- Worker único, com ordering só por `order_id` como limitação conhecida — [ADR-003](adrs/ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md)
- Retry com backoff exponencial em 5 tentativas e DLQ em tabela separada, com replay por ADMIN — [ADR-004](adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md)
- HMAC-SHA256 com secret por endpoint, rotacionável com grace period de 24h — [ADR-005](adrs/ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md)
- Entrega at-least-once com `X-Event-Id` para deduplicação no cliente — [ADR-006](adrs/ADR-006-garantir-entrega-at-least-once-com-x-event-id.md)
- Reuso de AppError, Pino, error middleware e do padrão de módulos — [ADR-007](adrs/ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md)
- Payload gravado como snapshot na inserção da outbox — [ADR-008](adrs/ADR-008-gravar-payload-como-snapshot-na-insercao.md)

## Alternativas Consideradas

### Disparar o webhook de forma síncrona no service de orders

Fazer a chamada HTTP ao cliente dentro de `changeStatus`, no momento da mudança de status ([09:03] Larissa).

**Por que foi descartada:** a transação "hoje já é pesada", e um cliente lento travaria a mudança de status de outros pedidos. Com o cliente fora do ar, não dá para desfazer a mudança de status ([09:04] Bruno). "Síncrono está fora de questão" ([09:06] Diego).

### Redis Streams

Usar Redis Streams "ou alguma coisa parecida" como fila de eventos ([09:07] Larissa).

**Por que foi descartada:** exigiria "subir mais infra" ([09:07] Larissa), e para um time pequeno isso é "overengineering" ([09:07] Diego). A verificação externa aponta que Streams roda num Redis standalone, sem o Cluster citado: o motivo que se sustenta é o custo operacional de um Redis a mais ([verificação externa](research/verificacao-externa.md), VER-002).

### Trigger do banco para acordar o worker

Usar uma trigger no MySQL para o worker reagir à inserção em vez de fazer polling ([09:09] Bruno).

**Por que foi descartada:** o MySQL não tem listener nativo como o NOTIFY/LISTEN do Postgres, e a trigger "só executa SQL". Avisar o worker exigiria improvisar escrita em arquivo ou chamada a endpoint. Polling de 2 segundos já atende o requisito de menos de 10 segundos ([09:09] Diego).

### Retry indefinido com backoff

Retentar sem teto, só aumentando o intervalo ([09:15] Diego).

**Por que foi descartada:** um evento ficaria "pendurado pra sempre se o cliente sumiu" ([09:15] Diego). O teto de 5 tentativas cobre indisponibilidades de algumas horas e dá a cada evento um fim definido ([09:16] Diego, [09:17] Larissa).

### DLQ como status "failed" na própria outbox

Marcar o evento morto com um status na outbox, sem tabela à parte ([09:17] Larissa).

**Por que foi descartada:** uma tabela separada deixa "mais limpa a leitura da outbox principal" e guarda a evidência para debug e reprocessamento ([09:18] Diego).

## Questões em Aberto

| Questão | Estado | Gatilho para revisitar | Fonte |
| --- | --- | --- | --- |
| Rate limiting de envio por cliente | Em observação | Um cliente reclamar de volume ou responder com erros de sobrecarga | [09:38] Diego, [09:39] Larissa |
| Escalar para múltiplos workers, particionando por `order_id` ou com lock pessimista | Adiado | O worker único não esvaziar os pendentes dentro do ciclo de 2 segundos | [09:13] Diego |
| Restringir por role o CRUD de configuração de webhooks, hoje aberto a qualquer role autenticada | Adiado | A revisão de segurança antes do deploy ([09:46] Sofia), ou a criação de um vínculo entre `User` e `Customer` | [09:37] Sofia |
| O `X-Timestamp` fica fora do conteúdo assinado, porque o HMAC cobre só o corpo. Assim ele não serve para detectar replay, como foi proposto. Stripe e Standard Webhooks incluem o timestamp na assinatura | Não decidido | Revisão de segurança da Sofia, antes de publicar a documentação do contrato para os clientes | [09:44] Diego × [09:22] Sofia; [verificação externa](research/verificacao-externa.md), VER-013 |
| Mesmo com um único worker, um evento em retry pode ser ultrapassado pelo evento seguinte do mesmo pedido, e eventos com o mesmo `created_at` não têm ordem garantida. A limitação de ordering do ADR-003 promete mais do que o desenho entrega | Não decidido | Antes de documentar a garantia de ordem no portal de desenvolvedor ([09:26] Marcos) | [09:12] Diego; [verificação externa](research/verificacao-externa.md), VER-005 |

## Impacto e Riscos

**Impacto:** o time de Pedidos altera `changeStatus` (`src/modules/orders/order.service.ts`). O time de Plataforma passa a operar um segundo processo, que usa o mesmo banco ([09:11] Diego). Os três clientes B2B precisam implementar validação de assinatura e deduplicação ([09:25] Sofia). A documentação do portal de desenvolvedor fica com o Marcos ([09:26] Marcos).

| Risco | Probabilidade | Impacto | Mitigação | Fonte |
| --- | --- | --- | --- | --- |
| Qualquer usuário autenticado pode cadastrar, ler ou apagar webhooks de qualquer customer, inclusive ver a secret devolvida na criação e os payloads do histórico de entregas. O `customer_id` vem no body ou no path, e o CRUD aceita qualquer role | Alta | Alto | Decidir na revisão de segurança se o endurecimento adiado entra nesta fase | [09:32] Larissa, [09:37] Sofia, [09:46] Sofia; `src/middlewares/auth.middleware.ts` |
| A premissa de que existem "usuários que representam o cliente" não tem apoio no código. `User` não tem relação com `Customer`, e as roles são só `ADMIN` e `OPERATOR`. Não há como checar se quem chama tem direito sobre o `customer_id` informado | Alta | Alto | Decidir com o Marcos e a Sofia se o vínculo `User`↔`Customer` entra no escopo. Até lá, registrar a limitação no PRD | [09:32] Marcos; `prisma/schema.prisma`; `src/middlewares/auth.middleware.ts` |
| A restrição de ADMIN no replay da DLQ não protege sozinha. `POST /auth/register` é público e aceita `role: 'ADMIN'` | Média | Alto | Incluir o registro público no escopo da revisão de segurança | [09:36] Sofia; `src/modules/auth/auth.routes.ts`; `src/modules/auth/auth.schemas.ts` |
| Não se sabe se as três sprints estimadas cabem até "fim de novembro": a reunião não informa a data de início nem a duração da sprint | Média | Alto | Marcos confirma o prazo com a Atlas, mantendo os dois dias de revisão de segurança no fim | [09:45] Marcos, [09:46] Larissa, [09:46] Sofia, [09:47] Marcos |

## Decisões Relacionadas

- [ADR-001: Usar padrão outbox no MySQL](adrs/ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md) — evento gravado na mesma transação da mudança de status.
- [ADR-002: Worker separado em polling](adrs/ADR-002-processar-outbox-com-worker-separado-em-polling.md) — processo próprio, ciclo de 2 segundos.
- [ADR-003: Worker único e ordering por order_id](adrs/ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md) — limitação conhecida de ordering.
- [ADR-004: Retry com backoff e DLQ em tabela separada](adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md) — 5 tentativas, depois DLQ com replay por ADMIN.
- [ADR-005: HMAC-SHA256 com secret por endpoint](adrs/ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md) — assinatura e rotação com grace period de 24h.
- [ADR-006: At-least-once com X-Event-Id](adrs/ADR-006-garantir-entrega-at-least-once-com-x-event-id.md) — deduplicação do lado do cliente.
- [ADR-007: Reusar padrões do projeto](adrs/ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md) — módulo, erros, logger e middleware existentes.
- [ADR-008: Payload como snapshot na inserção](adrs/ADR-008-gravar-payload-como-snapshot-na-insercao.md) — o evento reflete o pedido no momento da mudança.

## Próximos Passos

- Estimativa de três sprints, incluindo a revisão de segurança ([09:46] Larissa, [09:47] Larissa).
- Pelo menos dois dias úteis no fim para a revisão de segurança, com foco em HMAC e geração de secret ([09:46] Sofia).
- Sessão de revisão do documento de design com Bruno e Diego antes de começar a codar ([09:50] Larissa).
- Marcos confirma o prazo com a Atlas e atualiza os clientes ([09:47] Marcos, [09:49] Marcos).
