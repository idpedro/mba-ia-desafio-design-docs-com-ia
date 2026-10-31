# Mapeamento do Código

## Visão Geral
API REST de gestão de pedidos (`order-management-api`, `package.json:2-4`) em Node/TypeScript com Express e Prisma sobre MySQL. Cobre autenticação JWT de usuários internos (ADMIN/OPERATOR), cadastro de clientes e produtos, e o ciclo de vida de pedidos com máquina de status, histórico de transições e controle de estoque. Não há hoje nenhuma comunicação de saída com sistemas externos.

## Stack
| Camada | Tecnologia | Evidência |
| --- | --- | --- |
| Runtime | Node >= 20, ESM (`"type": "module"`), `--env-file=.env` | `package.json:6-9`, `package.json:11`, `package.json:13` |
| Linguagem | TypeScript 5.6.3 (strict, `noUncheckedIndexedAccess`, alias `@/*`) | `package.json:51`, `tsconfig.json:9-10`, `tsconfig.json:24-26` |
| Build / dev | `tsc -p tsconfig.build.json` (rootDir `src`, saída `dist`); `tsx` 4.19.2 em watch | `package.json:11-12`, `tsconfig.build.json:4-5`, `package.json:50` |
| HTTP | Express 4.21.1 | `package.json:28`, `src/app.ts:56` |
| ORM | Prisma / @prisma/client 5.22.0 | `package.json:26`, `package.json:48`, `prisma/schema.prisma:1-9` |
| Banco | MySQL 8.0 (docker-compose, utf8mb4) | `docker-compose.yml:3`, `docker-compose.yml:21-24`, `prisma/schema.prisma:6` |
| Validação | Zod 3.23.8 (requests e env) | `package.json:33`, `src/config/env.ts:3-10` |
| Autenticação | jsonwebtoken 9.0.2 + bcrypt 5.1.1 | `package.json:27`, `package.json:29`, `src/modules/auth/auth.service.ts:47-50` |
| Logging | Pino 9.5.0 (+ pino-pretty 11.3.0 em dev). `pino-http` 10.3.0 está declarado mas não é usado em `src/` | `package.json:30-31`, `package.json:46`, `src/shared/logger/index.ts:13-30` |
| IDs auxiliares | `uuid` 11.0.3 (só para request id) | `package.json:32`, `src/middlewares/request-logger.middleware.ts:2` |
| Testes | Vitest 2.1.4 + Supertest 7.0.0, contra MySQL real | `package.json:49`, `package.json:52`, `vitest.config.ts:3-19` |
| Lint / format | ESLint 8.57.1, Prettier 3.3.3 | `package.json:19-20`, `package.json:44`, `package.json:47` |
| Scripts | `dev`, `build`, `start`, `db:migrate`, `db:reset`, `db:seed`, `test`, `test:watch`, `lint`, `format` (não existe `worker`) | `package.json:10-21` |
| Configuração | `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN` validados; `SHADOW_DATABASE_URL` só no Prisma | `src/config/env.ts:3-10`, `.env.example:1-14`, `prisma/schema.prisma:8` |

## Estrutura e Módulos
| Módulo | Caminho | Responsabilidade |
| --- | --- | --- |
| Bootstrap | `src/server.ts` | Cria o app com o `prisma` singleton, sobe o HTTP, faz shutdown em SIGINT/SIGTERM (`src/server.ts:6-22`) |
| Composição do app | `src/app.ts` | Composition root manual: instancia repository, service e controller de cada módulo (`src/app.ts:26-53`), monta middlewares, `/health`, `/api/v1`, 404 e error handler (`src/app.ts:55-76`) |
| Roteador da API | `src/routes/index.ts` | Agrega os routers por módulo sob `/auth`, `/users`, `/customers`, `/products`, `/orders` (`src/routes/index.ts:21-31`) |
| Config | `src/config/env.ts`, `src/config/database.ts` | Validação do env com Zod e fábrica/singleton do `PrismaClient` (`src/config/database.ts:4-10`) |
| auth | `src/modules/auth/` | Registro, login (emite JWT) e `/me`. Não tem repository próprio, usa o de users |
| users | `src/modules/users/` | Leitura de usuário por id (só ADMIN) e criação de usuário (usada pelo register) |
| customers | `src/modules/customers/` | CRUD de clientes (paginação e busca) |
| products | `src/modules/products/` | CRUD de produtos (SKU único, estoque, ativo) |
| orders | `src/modules/orders/` | Criação, listagem, detalhe, troca de status e exclusão de pedidos; máquina de status em `order.status.ts` |
| Middlewares | `src/middlewares/` | `authenticate`/`requireRole`, `validate` (Zod), `requestLogger`, `errorMiddleware` |
| Shared | `src/shared/errors/`, `src/shared/http/`, `src/shared/logger/` | Hierarquia `AppError`, helpers de paginação, logger Pino |
| Banco | `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `prisma/seed.ts` | Schema, migration única inicial e seed (2 usuários, 10 clientes, 20 produtos, 26 pedidos) |
| Testes | `tests/` | Testes de integração HTTP (`auth.test.ts`, `orders.test.ts`), setup de limpeza e factories |

## Padrões
| Padrão | Como funciona | Arquivo de referência |
| --- | --- | --- |
| Organização de módulo | Cada módulo tem `*.routes.ts` (fábrica `buildXRouter(controller)`), `*.controller.ts` (classe com handlers arrow `RequestHandler`, `try/catch` + `next(err)`), `*.service.ts` (regras de negócio), `*.repository.ts` (acesso Prisma) e `*.schemas.ts` (Zod + tipos inferidos). Injeção por construtor, montada à mão em `buildControllers`. O padrão se repete em customers, products e users | `src/modules/orders/order.routes.ts:12-27`, `src/modules/orders/order.controller.ts:6-56`, `src/app.ts:42-44` |
| Validação | Middleware `validate({ body, query, params })` faz `parse` com Zod, substitui `req.body`/`req.params` e faz `Object.assign` em `req.query`; `ZodError` vira `ValidationError` com `details: [{ path, message }]`. IDs de rota validados com `z.string().uuid()` | `src/middlewares/validate.middleware.ts:11-37`, `src/modules/orders/order.schemas.ts:4`, `src/modules/orders/order.schemas.ts:18-21` |
| Erros | `AppError(message, statusCode, errorCode, details?)` (`src/shared/errors/app-error.ts:3-16`). Subclasses em `src/shared/errors/http-errors.ts`: `BadRequestError` 400 `BAD_REQUEST` (:3), `ValidationError` 400 `VALIDATION_ERROR` (:9), `UnauthorizedError` 401 `UNAUTHORIZED` (:15), `ForbiddenError` 403 `FORBIDDEN` (:21), `NotFoundError` 404 `NOT_FOUND` (:27), `ConflictError` 409 código customizável (:33), `UnprocessableEntityError` 422 código customizável (:39), `InvalidStatusTransitionError` 409 `INVALID_STATUS_TRANSITION` com `{from,to}` (:45), `InsufficientStockError` 422 `INSUFFICIENT_STOCK` com `{unavailable}` (:55). Barrel em `src/shared/errors/index.ts:1-13` | `src/shared/errors/http-errors.ts:3-63` |
| Códigos de erro em uso | Além dos acima: `INACTIVE_PRODUCT` (`src/modules/orders/order.service.ts:70`), `INVALID_ORDER_STATE_FOR_DELETE` (`src/modules/orders/order.service.ts:187`), `EMAIL_ALREADY_USED` (`src/modules/users/user.service.ts:24`, `src/modules/customers/customer.service.ts:32`), `SKU_ALREADY_USED` (`src/modules/products/product.service.ts:33`), `CONFLICT`/`NOT_FOUND` para Prisma e `INTERNAL_SERVER_ERROR` no handler | `src/middlewares/error.middleware.ts:37-64` |
| Tratamento de erros | `errorMiddleware`: `AppError` vira `{ error: { code, message, details? } }` com `statusCode` (:15-24); `ZodError` vira 400 `VALIDATION_ERROR` (:26-35); Prisma `P2002` vira 409 `CONFLICT` com o target (:38-47); `P2025` vira 404 `NOT_FOUND` (:48-53); qualquer outro erro (inclusive outros códigos Prisma como FK `P2003`) é logado com `requestId` e vira 500 (:56-64) | `src/middlewares/error.middleware.ts:14-65` |
| Autenticação | `authenticate` lê `Authorization: Bearer`, `jwt.verify` com `env.JWT_SECRET`, popula `req.user = { id: sub, email, role }`. Payload assinado: `{ sub, email, role }` com `expiresIn = JWT_EXPIRES_IN` (padrão `8h`). `req.user`/`req.id` declarados por module augmentation | `src/middlewares/auth.middleware.ts:6-47`, `src/modules/auth/auth.service.ts:47-50`, `src/config/env.ts:9` |
| Roles | Enum `UserRole { ADMIN, OPERATOR }`, default `OPERATOR`. `requireRole(...roles)` dá 401 sem `req.user` e 403 `FORBIDDEN` fora da lista. Hoje só é usado em `GET /users/:id` com `requireRole('ADMIN')`; orders, customers e products só exigem `authenticate` | `prisma/schema.prisma:11-14`, `src/middlewares/auth.middleware.ts:49-61`, `src/modules/users/user.routes.ts:12-18` |
| Logging | Logger Pino singleton com `level` do env, `redact` de `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`, `base: { service, env }`, timestamp ISO e `pino-pretty` em development. Uso: evento como mensagem snake_case + objeto de contexto (`server_started`, `http_request`). `requestLogger` gera/propaga `X-Request-Id` e loga ao `finish`. Services não logam nada | `src/shared/logger/index.ts:4-32`, `src/middlewares/request-logger.middleware.ts:5-28`, `src/server.ts:10` |
| Acesso a dados | Repositories recebem `PrismaClient` no construtor. Listagens usam `$transaction([findMany, count])` em array (batch). `OrderService` recebe também o `PrismaClient` e, dentro de transações, usa o `tx` direto em vez do repository | `src/modules/orders/order.repository.ts:33-45`, `src/app.ts:43`, `src/modules/orders/order.service.ts:26-30` |
| Transações | `$transaction` interativo (`async (tx) => ...`), sem opções de isolamento, timeout ou lock explícito; helpers privados recebem `tx: Prisma.TransactionClient` (alias `TxClient`) | `src/modules/orders/order.service.ts:24`, `src/modules/orders/order.service.ts:58`, `src/modules/orders/order.service.ts:131` |
| Banco: IDs e nomes | IDs `String @id @default(uuid()) @db.Char(36)` gerados pelo Prisma (exceção: `OrderNumberSequence.id Int`). Tabelas em snake_case via `@@map` (`users`, `customers`, `products`, `orders`, `order_items`, `order_status_history`, `order_number_sequence`). Colunas não têm `@map`: no MySQL ficam em camelCase (`orderNumber`, `customerId`, `totalCents`, `stockQuantity`, `createdAt`). Valores monetários em centavos `Int`. `orderNumber` no formato `ORD-000001` | `prisma/schema.prisma:74-97`, `prisma/migrations/20260519182739_init/migration.sql:50-69`, `src/modules/orders/order.service.ts:245-254` |
| Respostas HTTP | Recurso único retornado cru (`res.status(200).json(entity)`), sem envelope; criação 201; exclusão 204 sem corpo. Listagens usam `paginated()` com `{ data, pagination: { page, pageSize, total, totalPages } }`. Erros em `{ error: { code, message, details? } }` | `src/shared/http/response.ts:8-24`, `src/modules/orders/order.controller.ts:13`, `src/modules/orders/order.controller.ts:32`, `src/modules/orders/order.controller.ts:51` |
| Testes | Integração HTTP via Supertest sobre `buildApp({ prisma })` (app cacheado) com MySQL real; `fileParallelism: false` e `singleFork`; `beforeEach` apaga todas as tabelas em ordem de FK; factories criam usuário, cliente e produto direto no Prisma e obtêm token via `POST /auth/login`. Asserções em status, `body.error.code` e estado do banco | `vitest.config.ts:5-17`, `tests/setup.ts:8-16`, `tests/helpers/factories.ts:10-15`, `tests/helpers/factories.ts:76-83`, `tests/orders.test.ts:59-87` |

## Fluxos Relevantes para a Feature

### 1. Troca de status de pedido (`PATCH /api/v1/orders/:id/status`)
1. Rota montada em `/api/v1` (`src/app.ts:67`) e `/orders` (`src/routes/index.ts:28`); o router aplica `authenticate` a todas as rotas (`src/modules/orders/order.routes.ts:14`), sem `requireRole`.
2. `validate` com `orderIdParamSchema` (uuid) e `updateOrderStatusSchema` `{ toStatus: OrderStatus, reason?: string<=500 }` (`src/modules/orders/order.routes.ts:19-23`, `src/modules/orders/order.schemas.ts:18-21`).
3. `OrderController.changeStatus` exige `req.user` e chama `orders.changeStatus(id, body, req.user.id)` (`src/modules/orders/order.controller.ts:38-46`).
4. `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) abre um `$transaction` interativo (`:131`) e, com o `tx`:
   1. `tx.order.findUnique({ where: { id }, include: { items: true } })` (`:132-135`); sem pedido, `NotFoundError('Order')` (`:136`). A leitura é simples, sem `SELECT ... FOR UPDATE`.
   2. Se `from === to`, lança `ConflictError` com código `INVALID_STATUS_TRANSITION` (`:140-146`).
   3. Se `!canTransition(from, to)`, lança `InvalidStatusTransitionError` (`:147-149`).
   4. Se `shouldDebitStock` (PENDING para PAID), chama `debitStock` (`:151-153`): lê os produtos, junta os sem saldo e lança `InsufficientStockError`, ou faz `stockQuantity: { decrement }` item a item (`:204-231`).
   5. Se `shouldReplenishStock` (PAID ou PROCESSING para CANCELLED), chama `replenishStock` com `increment` (`:154-156`, `:233-243`).
   6. `tx.order.update({ data: { status: to } })` (`:158`).
   7. `tx.orderStatusHistory.create({ orderId, fromStatus, toStatus, changedById: userId, reason })` (`:159-167`).
   8. Relê o pedido com `items.product`, `history` e `customer { id, name, email }` e o retorna (`:169-177`). Hoje é o último passo dentro da transação; um insert em outbox caberia entre `:167` e `:169`.
5. O controller responde 200 com o pedido completo (`src/modules/orders/order.controller.ts:42`). Erros seguem para o `errorMiddleware` (`src/middlewares/error.middleware.ts:14`).

### 2. Máquina de status
- Enum `OrderStatus { PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED }` (`prisma/schema.prisma:16-23`), replicado como ENUM do MySQL em `orders.status`, `order_status_history.fromStatus/toStatus` (`prisma/migrations/20260519182739_init/migration.sql:54`, `:90-91`).
- Transições (`src/modules/orders/order.status.ts:3-10`): PENDING para PAID ou CANCELLED; PAID para PROCESSING ou CANCELLED; PROCESSING para SHIPPED ou CANCELLED; SHIPPED para DELIVERED; DELIVERED e CANCELLED são terminais (`isTerminal`, `:20-22`).
- Helpers: `canTransition` (`:12-14`), `allowedTransitions` (`:16-18`, sem uso em `src/`), `shouldDebitStock` (`:29-31`), `shouldReplenishStock` (`:33-37`).
- Cada transição gera uma linha em `order_status_history`, e esse é o único registro de "evento" de mudança de status que existe hoje (`prisma/schema.prisma:116-131`).

### 3. Criação de pedido (`POST /api/v1/orders`)
- Também em `$transaction` interativo (`src/modules/orders/order.service.ts:58-123`): valida cliente e produtos, calcula totais, reserva o `orderNumber` por upsert em `order_number_sequence` (`:93`, `:245-254`) e cria o pedido com `history.create` inicial `fromStatus: null, toStatus: PENDING, reason: 'order created'` (`:95-120`). É a primeira "mudança de status" (nula para PENDING) e não passa por `changeStatus`.

### 4. Autenticação e autorização
- Login: `AuthService.login` busca por e-mail, compara com bcrypt e assina `{ sub, email, role }` (`src/modules/auth/auth.service.ts:31-50`); a resposta é `{ user, tokens: { accessToken, expiresIn, tokenType: 'Bearer' } }` (`:40-44`).
- Registro público `POST /auth/register` (`src/modules/auth/auth.routes.ts:10`) aceita `role: 'ADMIN' | 'OPERATOR'` no corpo (`src/modules/auth/auth.schemas.ts:7`), ou seja, qualquer pessoa pode se registrar como ADMIN.
- Autorização por role: `requireRole` (`src/middlewares/auth.middleware.ts:49-61`), usado só em `src/modules/users/user.routes.ts:15`.

### 5. Ciclo de vida do processo e do PrismaClient
- `prisma` singleton criado na importação de `src/config/database.ts:10` via `createPrismaClient()` (`:4-8`), com log Prisma `warn/error` em dev.
- `server.ts` injeta o singleton em `buildApp` (`src/server.ts:7`) e no shutdown fecha o HTTP e chama `prisma.$disconnect()` (`src/server.ts:13-21`).
- `env` e `logger` também são singletons carregados na importação (`src/config/env.ts:27`, `src/shared/logger/index.ts:32`); o `env` faz `process.exit(1)` se a configuração for inválida (`src/config/env.ts:16-23`).

## Pontos de Extensão
| Ponto | Caminho | Como a feature pode se encaixar |
| --- | --- | --- |
| Transação de `changeStatus` | `src/modules/orders/order.service.ts:131-178` | Inserir a linha em `webhook_outbox` com o mesmo `tx`, depois do `orderStatusHistory.create` (`:159-167`), para que o evento seja atômico com a troca de status |
| Tipo do client transacional | `src/modules/orders/order.service.ts:24` | `TxClient = Prisma.TransactionClient` já é o tipo que um repository/serviço de outbox pode receber para gravar dentro da transação |
| Criação de pedido (status inicial) | `src/modules/orders/order.service.ts:95-120` | Se o PENDING inicial também precisar notificar, o insert no outbox entraria nesta transação |
| Schema Prisma | `prisma/schema.prisma` (após `:138`) | Novos models (config de webhook, outbox, dead letter) seguindo `@id @default(uuid()) @db.Char(36)`, `@@map` snake_case e `@@index`; relação com `Customer` (`:40-54`) |
| Migrations | `prisma/migrations/` | Nova migration via `npm run db:migrate` (`package.json:14`); hoje só existe `20260519182739_init` |
| Composition root | `src/app.ts:26-53` | Instanciar repository, service e controller de webhooks e incluí-los em `Controllers`; injetar o serviço de outbox no `OrderService` (`src/app.ts:43`) |
| Roteador da API | `src/routes/index.ts:13-31` | Adicionar `router.use('/webhooks', ...)` e um router admin (ex.: `/admin`), estendendo o tipo `Controllers` |
| Novo módulo | `src/modules/webhooks/` (não existe) | Replicar a estrutura de `src/modules/orders/` (routes, controller, service, repository, schemas) |
| Proteção admin | `src/middlewares/auth.middleware.ts:49-61` | `requireRole('ADMIN')` já existe e serve para o replay; exemplo de uso em `src/modules/users/user.routes.ts:12-18` |
| Erros de domínio | `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts` | Novas subclasses de `AppError` com códigos próprios, exportadas no barrel; o `errorMiddleware` já as serializa |
| Env | `src/config/env.ts:3-10`, `.env.example` | Variáveis novas (intervalo de polling, timeout HTTP, máximo de tentativas) entram no `envSchema` |
| Prisma do worker | `src/config/database.ts:4-8` | `createPrismaClient()` é exportado e permite ao worker criar o próprio client, sem reutilizar o singleton `prisma` (`:10`) |
| Bootstrap/shutdown | `src/server.ts:6-27` | Modelo de bootstrap, shutdown por sinal e `logger.fatal` para um `src/worker.ts` |
| Scripts | `package.json:10-21` | Adicionar `worker` (ex.: no estilo de `dev`/`start`, com `tsx --env-file=.env`) |
| Logger | `src/shared/logger/index.ts:4-11`, `:32` | Reusar o singleton no worker; incluir o header/secret de assinatura nos `redactPaths` |
| Testes | `tests/setup.ts:8-16`, `tests/helpers/factories.ts` | Incluir as novas tabelas na limpeza do `beforeEach` (antes de `orders`/`customers` por causa das FKs) e criar factories de config de webhook |

## Ausências
- Não existe tabela, model ou migration de webhooks: nem configuração, nem `webhook_outbox`, nem dead letter (`prisma/schema.prisma` só tem os models das linhas 25-138).
- Não existe módulo `src/modules/webhooks` nem qualquer rota `/webhooks` ou `/admin` (`src/routes/index.ts:24-28`).
- Não existe worker, job, scheduler ou loop de polling: não há `src/worker.ts`, nem script `worker` em `package.json:10-21`, nem `setInterval` em `src/`.
- Não existe fila, event bus, `EventEmitter` de domínio, Redis ou cache; nenhuma dependência disso em `package.json:25-34`, e nada no `docker-compose.yml` além do MySQL.
- Não existe cliente HTTP de saída (axios, wrapper de `fetch`, timeouts, retry), nem nenhum lugar onde a API chama sistemas externos.
- Não existe utilitário de criptografia/HMAC nem geração de secrets; o único uso de hash é `bcrypt` para senha (`src/modules/users/user.service.ts:26`) e o JWT (`src/modules/auth/auth.service.ts:49`).
- Não existe lógica de retry/backoff nem de idempotência em lugar nenhum.
- Não existe vínculo entre `User` e `Customer`: `User` (`prisma/schema.prisma:25-38`) só se relaciona com `Order.createdBy` e `OrderStatusHistory.changedBy`, e `Customer` (`:40-54`) só com `orders`. Os usuários são operadores internos (ADMIN/OPERATOR); nenhum usuário representa um cliente, e não há role de cliente.
- Não existe escopo por cliente na autorização: qualquer usuário autenticado lê e altera pedidos de qualquer cliente (`src/modules/orders/order.routes.ts:14-24`).
- A role ADMIN existe (`prisma/schema.prisma:11-14`), mas `requireRole('ADMIN')` só protege `GET /users/:id` (`src/modules/users/user.routes.ts:15`), e o registro público permite criar ADMIN (`src/modules/auth/auth.schemas.ts:7`, `src/modules/auth/auth.routes.ts:10`).
- Não existe log de domínio nos services: o logger só é usado em `src/server.ts`, `src/middlewares/request-logger.middleware.ts` e `src/middlewares/error.middleware.ts`. Também não há métricas nem tracing.
- `pino-http` está nas dependências (`package.json:31`), mas não é importado; o log HTTP é o middleware próprio.
- O `errorMiddleware` não trata violação de FK do Prisma (`P2003`), que cai em 500 (`src/middlewares/error.middleware.ts:37-64`); isso vale para FKs novas apontando para `customers`/`orders`.
- Não há lock pessimista nem nível de isolamento explícito em `changeStatus` (`src/modules/orders/order.service.ts:131-135`): duas requisições concorrentes podem ler o mesmo `from`.
- Não há testes unitários nem mocks; só integração HTTP (`tests/auth.test.ts`, `tests/orders.test.ts`), e nenhum teste de customers, products ou users.
