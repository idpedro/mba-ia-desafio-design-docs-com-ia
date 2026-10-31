# Decisões Candidatas a ADR

## Registrar eventos de webhook numa outbox MySQL dentro da transação de mudança de status

- **Classificação:** Documentar
- **Categoria:** mensageria / persistência
- **O que foi decidido:** padrão outbox no MySQL existente. Dentro da mesma transação SQL de `changeStatus` (update em `orders`, insert em `order_status_history`, ajuste de estoque), insere-se uma linha numa tabela `webhook_outbox`. Se a inserção na outbox falhar, a transação inteira dá rollback. A inserção é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o tx client da transação atual e é chamada pelo order service. A outbox tem índice em status (pendente, processando, falhou, entregue) e em `created_at`.
- **Evidências:** INV-010, INV-011, INV-013, INV-084, INV-085, INV-086, INV-102 ([09:06] Diego; [09:08] Larissa; [09:40] Bruno; [09:41] Diego; [09:48] Larissa). Código: `src/modules/orders/order.service.ts:126` (`changeStatus`), `:131` (`$transaction` interativo com `tx`), `:158` (`order.update`), `:159-167` (`orderStatusHistory.create`); `OrderService` em `order.service.ts:26`.
- **Verificação externa:** VER-001 — Parcial; VER-002 — Parcial.
- **Alternativas levantadas:**
  - Disparo síncrono no service de orders — um cliente lento trava a mudança de status de outros pedidos, e com o cliente fora do ar não dá para dar rollback (INV-009, [09:04] Bruno; [09:06] Diego).
  - Redis Streams — "a gente acabaria precisando subir mais infra"; "Subir Redis Cluster pra isso é overengineering" (INV-012, [09:07] Larissa; [09:07] Diego).
  - Injetar um repository de webhook no `OrderService` — "Não precisa injetar repository inteiro" (INV-088, [09:41] Diego).
- **Trade-off:** ganha atomicidade entre a mudança de status e o registro do evento, sem infraestrutura nova. Perde reatividade (exige polling, ver a próxima candidata), torna mais pesada uma transação que "hoje já é pesada" (INV-008) e cria uma tabela que cresce até haver arquivamento (INV-014, fora de escopo).
- **Perguntas em aberto:**
  - VER-001 (Parcial): "Não tem inconsistência possível" vale para o registro do evento. A entrega continua at-least-once e pode duplicar.
  - VER-002 (Parcial): Redis Streams roda em Redis standalone, e o Cluster é opcional. O argumento de "mais infra" se sustenta; o de "Redis Cluster" descarta uma topologia mais pesada que a necessária.
  - Conflito com o código: a criação do pedido grava o `PENDING` inicial sem passar por `changeStatus` (`order.service.ts:95-120`). A fonte não diz se a criação também gera evento.
  - Conflito com o código: `changeStatus` não trava a linha (sem `SELECT … FOR UPDATE` em `order.service.ts:132`). Duas mudanças concorrentes no mesmo pedido podem gerar eventos com o mesmo `from_status`.

## Processar a outbox com um worker em processo separado, por polling a cada 2 segundos

- **Classificação:** Documentar
- **Categoria:** mensageria / arquitetura de processos
- **O que foi decidido:** um worker em loop busca os eventos pendentes mais antigos a cada 2 segundos, em batch pequeno, processa e marca. Roda como processo separado da API ("Senão se a API reinicia, perde o worker"), com entry-point `src/worker.ts`, script `npm run worker`, mesmo banco, mesma `DATABASE_URL` e um PrismaClient próprio. A lógica fica dentro do módulo (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`). O timeout do HTTP call é de 10s, e estourar o timeout conta como falha e vai para retry.
- **Evidências:** INV-015, INV-018, INV-019, INV-056, INV-063 (corrige INV-020), INV-089 ([09:09] Diego; [09:10] Larissa; [09:11] Diego; [09:11] Larissa; [09:30] Bruno; [09:42] Diego). Código: `src/server.ts` como modelo de entry-point; `src/config/database.ts:4` (`createPrismaClient()`) e `:10` (singleton `prisma`); `package.json:10-21` (não há script `worker`).
- **Verificação externa:** VER-003 — Confirmada; VER-004 — Confirmada; VER-012 — Confirmada.
- **Alternativas levantadas:**
  - Trigger do banco para acordar o worker — "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres", e a trigger "só executa SQL" (INV-016, [09:09] Diego).
  - Worker dentro do processo da API — um restart da API derruba o worker (INV-018, [09:11] Diego).
  - Reusar o mesmo PrismaClient da API — descartado porque "PrismaClient é por processo" (INV-063, [09:30] Bruno).
- **Trade-off:** ganha simplicidade e isolamento sem infraestrutura nova, cumprindo o requisito de menos de 10s (INV-005). Perde até 2s de latência por ciclo de polling, faz consultas periódicas mesmo sem eventos e exige um segundo processo para operar e monitorar.
- **Perguntas em aberto:**
  - INV-017: Larissa disse "latência mínima vai ser 2 segundos no pior caso". Falta definir qual limite vale para o requisito.
  - VER-012 (Confirmada, com nota): o pool do worker soma conexões às da API (`núcleos×2+1` por padrão no Prisma 5.x) e precisa ser dimensionado contra `max_connections`.
  - INV-057: o nome do arquivo ficou entre `webhook.worker.ts` e `webhook.processor.ts`.

## Operar com um único worker e garantir ordering só por order_id

- **Classificação:** Considerar
- **Categoria:** mensageria / consistência
- **O que foi decidido:** single-worker, com ordering implícito por `order_id` na ordem de `created_at` da outbox. Fica como limitação conhecida a documentar: "Não é garantia de ordering global, só por order_id e enquanto for single-worker."
- **Evidências:** INV-021, INV-022, INV-024 ([09:12] Diego; [09:13] Larissa; [09:14] Marcos).
- **Verificação externa:** VER-005 — Parcial; VER-006 — Confirmada.
- **Alternativas levantadas:** múltiplos workers com partição por `order_id` ou lock pessimista — "isso é problema do futuro, não agora" (INV-023, Adiado, [09:13] Diego).
- **Trade-off:** ganha simplicidade, sem coordenação entre workers. Perde a capacidade de escalar horizontalmente a entrega; o worker único é gargalo e ponto único de falha.
- **Perguntas em aberto:**
  - VER-005 (Parcial): mesmo com um único worker, o retry com backoff (1m…12h) permite que o evento N+1 de um pedido seja entregue antes do evento N que falhou. Além disso, empates em `created_at` (`DATETIME(3)`) não têm ordem garantida no MySQL e o UUID não desempata. A garantia "por order_id" depende de o worker segurar os eventos seguintes do mesmo pedido, o que não foi discutido.
  - VER-006 (Confirmada, com nota): `SKIP LOCKED` sozinho não preserva ordem por pedido, o que vale para a escala futura.

## Fazer retry com backoff exponencial de 5 tentativas (1m/5m/30m/2h/12h)

- **Classificação:** Documentar
- **Categoria:** resiliência / integração
- **O que foi decidido:** backoff exponencial com teto de 5 tentativas e progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas ("quase 15 horas entre primeira falha e última tentativa"). Ao estourar o teto, é falha permanente e o evento vai para a DLQ.
- **Evidências:** INV-025, INV-026, INV-030, INV-031, INV-029 ([09:15] Diego; [09:16] Larissa; [09:17] Diego; [09:17] Larissa; [09:17] Marcos).
- **Verificação externa:** nenhuma afirmação externa associada. VER-009 (Parcial) cita a janela de retry de 3 dias da Stripe só como comparação.
- **Alternativas levantadas:**
  - Retry indefinido — "evento ficar pendurado pra sempre se o cliente sumiu" (INV-027, [09:15] Diego).
  - 3 tentativas — "retentaria três vezes em 30 minutos e mataria" (INV-028, [09:16] Diego).
- **Trade-off:** ganha cobertura de indisponibilidades de algumas horas (já houve cliente com 2h de manutenção, INV-029) e um fim definido para cada evento. Perde eventos que ficam até ~15h em trânsito e é bem mais curto que a referência de mercado (VER-009).
- **Perguntas em aberto:**
  - INV-032: a janela "12 ou 24 horas" [09:15] não bate com o "quase 15 horas" [09:17].
  - INV-033: não ficou dito se o envio inicial conta entre as 5 tentativas (5 intervalos para "5 tentativas").

## Persistir a DLQ em tabela separada, com replay manual por ADMIN

- **Classificação:** Documentar
- **Categoria:** persistência / operação
- **O que foi decidido:** uma tabela `webhook_dead_letter` separada, com payload, motivo da falha e timestamp. O reprocessamento é manual por `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. O replay exige role ADMIN via o `requireRole` existente e registra em log quem fez o replay, para auditoria.
- **Evidências:** INV-034, INV-036, INV-037, INV-075, INV-076, INV-077 ([09:18] Diego; [09:19] Larissa; [09:35] Diego; [09:36] Sofia; [09:36] Larissa). Código: `requireRole` em `src/middlewares/auth.middleware.ts:49`, usado hoje só em `src/modules/users/user.routes.ts:15`; roles `ADMIN | OPERATOR` em `auth.middleware.ts:6`; não existe nenhuma rota `/admin`.
- **Verificação externa:** nenhuma afirmação externa associada.
- **Alternativas levantadas:** marcar como "failed" na própria outbox — a tabela separada deixa "mais limpa a leitura da outbox principal" (INV-035, [09:18] Diego).
- **Trade-off:** ganha uma outbox enxuta e evidência preservada para debug e reprocessamento. Perde ao mover dados entre duas tabelas (outbox → DLQ → outbox) e exige intervenção humana para cada evento morto.
- **Perguntas em aberto:**
  - Conflito com o código: `POST /auth/register` é público e aceita `role: 'ADMIN'` (`src/modules/auth/auth.schemas.ts:7`), então a restrição ADMIN do replay não protege por si só.
  - Conflito com o código: o logger não é usado em nenhum service hoje (INV-060). A auditoria do replay será o primeiro log de domínio.

## Assinar o corpo com HMAC-SHA256, com secret por endpoint e rotação com grace period de 24h

- **Classificação:** Documentar
- **Categoria:** segurança
- **O que foi decidido:** HMAC-SHA256 sobre o corpo do request, com a assinatura no header `X-Signature`. Cada endpoint de webhook tem secret única, gerada pela plataforma e devolvida na criação. A secret é rotacionável por endpoint da API, e a antiga continua válida por 24h em paralelo. A configuração guarda url + secret + customer_id + estado ativo. A URL tem que ser https, e http é recusado com erro de validação no schema Zod.
- **Evidências:** INV-038, INV-039, INV-040, INV-042, INV-043, INV-045, INV-065, INV-098 ([09:19] Sofia; [09:22] Sofia; [09:23] Sofia; [09:31] Marcos; [09:46] Sofia). Código: não existe util de HMAC/crypto nem geração de secret; o único hash é bcrypt de senha (`docs/research/mapeamento-codigo.md#Ausências`).
- **Verificação externa:** VER-007 — Confirmada; VER-008 — Confirmada; VER-013 — Parcial.
- **Alternativas levantadas:** secret global da plataforma — "Senão se vaza uma, vaza tudo" (INV-041, [09:21] Sofia).
- **Trade-off:** ganha autenticidade e integridade verificáveis pelo cliente com bibliotecas comuns, e isola o vazamento a um endpoint. Perde por ter de guardar a secret de forma recuperável (para assinar, a secret não pode ficar só em hash) e por manter duas secrets válidas durante a rotação.
- **Perguntas em aberto:**
  - VER-013 (Parcial): o `X-Timestamp` só permite detectar replay se fizer parte do conteúdo assinado (Stripe assina `t.payload`; Standard Webhooks assina `id.timestamp.payload`). Com o HMAC só sobre o corpo, o timestamp pode ser trocado sem invalidar a assinatura.
  - VER-008 (Confirmada, com nota): no Zod 3.23.8, `z.string().url()` aceita qualquer esquema, então o https precisa de `.startsWith('https://')` ou `.refine`.
  - Durante o grace period, não foi dito se o envio leva duas assinaturas ou só a da nova secret.
  - Revisão de segurança de pelo menos 2 dias úteis antes do deploy (INV-098).

## Garantir entrega at-least-once com X-Event-Id para deduplicação no cliente

- **Classificação:** Documentar
- **Categoria:** integração / contrato externo
- **O que foi decidido:** at-least-once. O cliente pode receber o mesmo evento duas vezes e dedupica pelo `X-Event-Id`, um UUID gerado quando o evento entra na outbox e único por evento. Os headers são `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. Marcos documenta isso no portal de desenvolvedor.
- **Evidências:** INV-049, INV-050, INV-051, INV-053, INV-093, INV-094 ([09:24] Diego; [09:25] Diego; [09:26] Larissa; [09:26] Marcos; [09:44] Diego; [09:44] Sofia).
- **Verificação externa:** VER-009 — Parcial; VER-010 — Inconclusiva; VER-011 — Inconclusiva.
- **Alternativas levantadas:** exactly-once — "exigiria coordenação dos dois lados e fica muito mais complexo" (INV-052, [09:25] Diego).
- **Trade-off:** ganha simplicidade do lado da plataforma e segue o padrão de provedores conhecidos. Perde ao transferir ao cliente a responsabilidade de deduplicar ("Isso joga responsabilidade pro cliente", INV-051, [09:25] Sofia).
- **Perguntas em aberto:**
  - VER-009 (Parcial): a Stripe confirma o modelo. O GitHub tem id estável, mas não faz retry automático de falhas; a comparação com o GitHub vale para o id, não para o retry.
  - VER-010 (Inconclusiva): não há fonte primária sobre a complexidade de exactly-once.
  - VER-011 (Inconclusiva): o "99% dos casos" não tem fonte.
  - INV-108: não foi dito se o id da linha da outbox é o mesmo valor do `X-Event-Id`. No replay da DLQ, também não foi dito se o evento mantém o mesmo `X-Event-Id`.

## Gravar o payload como snapshot na inserção da outbox

- **Classificação:** Documentar
- **Categoria:** persistência / contrato externo
- **O que foi decidido:** o payload é renderizado e guardado no momento da inserção na outbox, e não reconstruído no envio. Formato: JSON com `event_id`, `event_type` (ex.: `"order.status_changed"`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e os campos básicos da order, como `total_cents`. Os items não vão no payload. O limite é de 64KB, com erro se ultrapassar.
- **Evidências:** INV-109, INV-090, INV-091, INV-047 ([09:52] Larissa; [09:52] Bruno; [09:43] Diego; [09:24] Larissa). Código: as colunas no banco são camelCase (`totalCents`, `orderNumber`, `customerId`; `prisma/migrations/20260519182739_init/migration.sql:52,57`), e o snake_case do payload é decisão de formato.
- **Verificação externa:** nenhuma afirmação externa associada.
- **Alternativas levantadas:**
  - Guardar só o `order_id` e renderizar no envio — "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" (INV-110, [09:52] Larissa).
  - Incluir os items — "pra não inflar"; o cliente usa `GET /orders/:id` (INV-091, [09:43] Diego).
  - Truncar acima de 64KB — "Se chegou nesse tamanho, tem algo errado" (INV-048, [09:23] Sofia).
- **Trade-off:** ganha fidelidade ao estado no momento da mudança e um worker que não depende de reler o pedido. Perde mais armazenamento por linha, e o formato fica congelado no evento já gravado.
- **Perguntas em aberto:**
  - Decidido depois do resumo, sem Marcos e Sofia na call (INV-109). A falta de Sofia importa porque o payload é o que é assinado.
  - A lista de "campos básicos" além de `total_cents` não foi enumerada (INV-090).

## Filtrar por status assinado na inserção da outbox

- **Classificação:** Considerar
- **Categoria:** persistência / modelo de dados
- **O que foi decidido:** cada webhook tem uma lista dos status que quer ouvir. O filtro é aplicado na inserção da outbox: "Se nenhum webhook do customer quer aquele status, nem insere."
- **Evidências:** INV-071, INV-072 ([09:33] Marcos; [09:34] Bruno; [09:34] Diego). Código: status `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` e `CANCELLED` em `src/modules/orders/order.status.ts:3-10`.
- **Verificação externa:** nenhuma afirmação externa associada.
- **Alternativas levantadas:** filtrar na hora do envio — filtrar na inserção "Economiza linha na tabela" (INV-073, [09:34] Bruno).
- **Trade-off:** ganha menos linhas na outbox. Perde porque a inscrição passa a ser avaliada no momento da mudança: mudar o filtro depois não afeta eventos já gravados, e a transação de `changeStatus` passa a ler a configuração de webhooks.
- **Perguntas em aberto:**
  - Com vários webhooks por customer e filtros diferentes, a fonte não diz se a outbox guarda uma linha por evento ou uma linha por webhook × evento. Isso afeta o `X-Webhook-Id` e o retry por endpoint.
  - Pode ser só um detalhe do FDD se o time não o tratar como decisão própria.

## Identificar o customer pelo body ou path, não pelo JWT, com CRUD aberto a qualquer role autenticada

- **Classificação:** Considerar
- **Categoria:** segurança / autorização
- **O que foi decidido:** os endpoints de configuração são autenticados normalmente com o JWT do sistema. O `customer_id` vai no body ou no path e não vem do JWT. Por enquanto, POST, PATCH, DELETE e GET de configuração e `GET /webhooks/:id/deliveries` podem ser chamados por qualquer role autenticada.
- **Evidências:** INV-066 (corrigido por INV-068), INV-067, INV-068, INV-070, INV-074, INV-078 ([09:32] Larissa; [09:33] Bruno; [09:37] Sofia). Código: `AuthUser` com `role: 'ADMIN' | 'OPERATOR'` em `src/middlewares/auth.middleware.ts:6`; `User` (`prisma/schema.prisma:25`) sem relação com `Customer` (`prisma/schema.prisma:40`).
- **Verificação externa:** nenhuma afirmação externa associada.
- **Alternativas levantadas:** `customer_id` implícito do JWT — "o JWT atual é do usuário operador, não do cliente" (INV-066 → INV-068, [09:31] Marcos; [09:32] Larissa). O endurecimento de roles ficou adiado: "Mais pra frente a gente pode endurecer" (INV-079, [09:37] Sofia).
- **Trade-off:** ganha entrega sem mexer no modelo de autenticação. Perde porque qualquer usuário autenticado pode cadastrar, ler (inclusive os payloads em deliveries) ou apagar webhooks de qualquer customer.
- **Perguntas em aberto:**
  - Conflito com o código: Marcos diz "A gente tem usuários que representam o cliente" (INV-067), mas não existe vínculo `User` ↔ `Customer`, e as roles são só `ADMIN` e `OPERATOR`.
  - INV-069: body ou path não foi escolhido.
  - Como a secret é devolvida na criação e as deliveries contêm payload, a abertura a qualquer role é relevante para a revisão da Sofia (INV-098).

## Itens `Decisão` tratados como detalhe

| Item | Onde entra |
| --- | --- |
| INV-026, INV-030 (5 tentativas; progressão) | Detalhe da candidata de retry |
| INV-040 (secret por endpoint) | Detalhe da candidata de HMAC |
| INV-054 (módulo `src/modules/webhooks`), INV-059 (prefixo `WEBHOOK_`), INV-064 (reuso de AppError, Pino, error middleware, schemas Zod) | Padrões do FDD; sem alternativas discutidas. Conflitos: Pino não é usado nos services (INV-060) e o error middleware só trata os erros Prisma `P2002`/`P2025` (INV-061, `src/middlewares/error.middleware.ts:37-48`) |
| INV-063 (PrismaClient separado) | Detalhe da candidata do worker |
| INV-075 (ADMIN no replay) | Detalhe da candidata de DLQ |
| INV-086 (`publishWebhookEvent(tx, …)`) | Detalhe da candidata de outbox |
| INV-102 (resumo de confirmação) | Fecha as demais; as omissões e divergências estão em INV-103 |
| INV-105 (id UUID na outbox) | Detalhe do modelo de dados; segue `@default(uuid()) @db.Char(36)` (`prisma/schema.prisma:26`) |

## Descartados, adiados e fora de escopo (sem ADR)

- Webhooks de entrada — Fora de Escopo (INV-007, [09:02] Marcos).
- Arquivamento das linhas entregues após ~30 dias — Fora de Escopo (INV-014, [09:08] Diego).
- Múltiplos workers (partição por `order_id` ou lock pessimista) — Adiado (INV-023, [09:13] Diego).
- Endurecer as roles do CRUD — Adiado (INV-079, [09:37] Sofia).
- Email de aviso de falha — Adiado para a próxima fase (INV-080, [09:37] Larissa).
- Rate limiting de saída — Aberto, "observar e decidir depois" (INV-081, [09:39] Larissa).
- Dashboard visual — Fora de Escopo, projeto do time de frontend (INV-082, [09:40] Larissa).
