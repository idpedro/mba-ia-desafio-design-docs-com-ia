# Verificação Externa

Data da verificação: 2026-10-30

Versões do repositório consideradas: MySQL `mysql:8.0` (`docker-compose.yml`), Prisma `5.22.0` e Zod `3.23.8` (`package.json`).

## VER-001 — Outbox na mesma transação elimina qualquer inconsistência

- **Afirmação:** "Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível." — INV-010, [09:06] Diego
- **Veredito:** Parcial
- **Evidência:** [Pattern: Transactional outbox — microservices.io (Chris Richardson)](https://microservices.io/patterns/data/transactional-outbox.html) (acesso em 2026-10-30) — "first store the message in the database as part of the transaction that updates the business entities"
- **Evidência:** [Pattern: Transactional outbox — microservices.io (Chris Richardson)](https://microservices.io/patterns/data/transactional-outbox.html) (acesso em 2026-10-30) — "The Message relay might publish a message more than once." / "a message consumer must be idempotent"
- **Evidência:** [MySQL 8.0 Reference Manual — 17.2 InnoDB and the ACID Model](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html) (acesso em 2026-10-30) — "The atomicity aspect of the ACID model mainly involves InnoDB transactions."
- **O que mudou:** A atomicidade entre a mudança de status e o registro do evento está confirmada: é a garantia de transação do InnoDB, e o padrão outbox se apoia nela. A ressalva é que o "não tem inconsistência possível" vale só para o registro. A entrega continua sujeita a duplicidade, porque o relay pode publicar a mesma mensagem mais de uma vez (por exemplo, se cair depois do envio HTTP e antes de marcar a linha como enviada). Além disso, a atomicidade só existe se o INSERT na outbox rodar na mesma transação e na mesma conexão que o UPDATE em `orders`/`order_status_history`. No Prisma 5.22 isso significa fazer tudo dentro de um mesmo `$transaction`. A microservices.io não é documentação de produto, mas é o site do autor que catalogou o padrão, então foi aceita como fonte primária do padrão. A parte transacional se apoia na doc do MySQL.
- **Impacto:** Sustenta a decisão "outbox em MySQL" (INV-010/INV-102). O risco está em ler a frase como garantia ponta a ponta. A outbox não impede envio duplicado nem evento registrado que nunca chega (DLQ). Por isso a decisão de at-least-once com `X-Event-Id` (INV-052) é complemento obrigatório, e não opcional. Também fica em risco qualquer implementação que chame `publishWebhookEvent` fora da transação do `order.service`.

## VER-002 — Redis Streams exigiria mais infra e um Redis Cluster

- **Afirmação:** "A alternativa seria botar Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra." — INV-012, [09:07] Larissa / "Subir Redis Cluster pra isso é overengineering." — INV-012, [09:07] Diego
- **Veredito:** Parcial
- **Evidência:** [Redis streams — Redis docs](https://redis.io/docs/latest/develop/data-types/streams/) (acesso em 2026-10-30) — "A Redis stream is a data structure that acts like an append-only log"
- **Evidência:** [XADD — Redis docs](https://redis.io/docs/latest/commands/xadd/) (acesso em 2026-10-30) — metadado do comando: "since": "5.0.0"
- **Evidência:** [Redis cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) (acesso em 2026-10-30) — "Redis Cluster implements all the single key commands available in the non-distributed version of Redis."
- **Evidência:** [Scale with Redis Cluster — Redis docs](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/) (acesso em 2026-10-30) — "Redis scales horizontally with a deployment topology called Redis Cluster."
- **O que mudou:** A primeira parte está confirmada: Redis Streams é um servidor a mais para operar, e o repositório hoje só tem MySQL no `docker-compose.yml`. A segunda parte troca o termo. Streams é um tipo de dado do Redis desde a 5.0 e funciona numa instância única ("non-distributed version"). Redis Cluster é uma topologia opcional de escala horizontal, não um requisito de Streams. A alternativa mínima era um Redis standalone, não um Cluster. Sobre licença: desde março de 2024 (7.4) o Redis passou a RSALv2/SSPLv1, e a partir do Redis 8 acrescentou a AGPLv3 como opção ([Redis licenses](https://redis.io/legal/licenses/), acesso em 2026-10-30 — "starting with Redis 8"). Para uso interno como fila, isso não pesa no argumento.
- **Impacto:** Sustenta a alternativa descartada INV-012. O motivo "mais infra" continua de pé. O que fica fraco é o reforço "Redis Cluster é overengineering": ele descarta uma configuração mais pesada do que a necessária. Se o ADR registrar o descarte, o motivo correto é o custo operacional de um servidor Redis a mais (mesmo standalone) para um time pequeno, não o custo de um Cluster.

## VER-003 — MySQL não tem listener nativo tipo NOTIFY/LISTEN

- **Afirmação:** "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres." — INV-016, [09:09] Diego
- **Veredito:** Confirmada
- **Evidência:** [PostgreSQL Documentation — NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html) (acesso em 2026-10-30) — "The NOTIFY command sends a notification event together with an optional “payload” string to each client application that has previously executed LISTEN channel"
- **Evidência:** [MySQL 8.0 Reference Manual — Chapter 15 SQL Statements](https://dev.mysql.com/doc/refman/8.0/en/sql-statements.html) (acesso em 2026-10-30) — o índice de statements (DDL, DML, transacionais, replicação, prepared statements, compound, administração, utilitários) não tem nenhum statement equivalente a LISTEN/NOTIFY
- **Evidência:** [MySQL 8.0 Reference Manual — 7.4.4 The Binary Log](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html) (acesso em 2026-10-30) — "The binary log contains “events” that describe database changes such as table creation operations or changes to table data."
- **O que mudou:** —
- **Impacto:** Sustenta o descarte do worker reativo por trigger (INV-016) e a escolha do polling de 2s. Não há mecanismo de notificação push para clientes SQL no MySQL. A única fonte nativa de mudanças em tempo quase real é o binary log, que serve à replicação. Consumi-lo como CDC exige um componente externo lendo o protocolo de replicação (ex.: Debezium), ou seja, mais infra, a mesma objeção de VER-002. A prova da ausência é negativa (nada no índice de statements), o que é o máximo que uma doc oficial costuma oferecer.

## VER-004 — Trigger do MySQL só executa SQL e não notifica processo externo

- **Afirmação:** "Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL." — INV-016, [09:09] Diego
- **Veredito:** Confirmada
- **Evidência:** [MySQL 8.0 Reference Manual — 15.1.22 CREATE TRIGGER Statement](https://dev.mysql.com/doc/refman/8.0/en/create-trigger.html) (acesso em 2026-10-30) — "trigger_body is the statement to execute when the trigger activates."
- **Evidência:** [MySQL 8.0 Reference Manual — 27.8 Restrictions on Stored Programs](https://dev.mysql.com/doc/refman/8.0/en/stored-program-restrictions.html) (acesso em 2026-10-30) — "stored functions and triggers cannot use dynamic SQL"
- **Evidência:** [Extending MySQL 8.0 — Adding a Loadable Function](https://dev.mysql.com/doc/extending-mysql/8.0/en/adding-loadable-function.html) (acesso em 2026-10-30) — "Files implementing loadable functions must be compiled and installed on the host where the server runs."
- **O que mudou:** —
- **Impacto:** Sustenta o descarte de INV-016. Uma ressalva que não muda o veredito: um trigger pode chamar uma *loadable function* (antiga UDF) que faça I/O externo, como os `sys_exec` da comunidade. Isso não é recurso nativo, porém. Exige compilar C++ e instalar uma biblioteca no host do servidor, e a doc avisa que esse código "becomes part of the running server". É justamente o "improviso esquisito" que o Diego descreveu.

## VER-005 — Um único worker entrega em ordem de created_at

- **Afirmação:** "Se a gente tem um único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em ordem. Se a gente escala pra múltiplos workers em paralelo no futuro, perde a garantia." — INV-021, [09:12] Diego
- **Veredito:** Parcial
- **Evidência:** [Prisma docs — MySQL connector, type mapping](https://www.prisma.io/docs/orm/overview/databases/mysql) (acesso em 2026-10-30) — "DateTime | DATETIME(3)"
- **Evidência:** [MySQL 8.0 Reference Manual — 13.2.6 Fractional Seconds in Time Values](https://dev.mysql.com/doc/refman/8.0/en/fractional-seconds.html) (acesso em 2026-10-30) — "with up to microseconds (6 digits) precision"
- **Evidência:** [MySQL 8.0 Reference Manual — 10.2.1.19 LIMIT Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/limit-optimization.html) (acesso em 2026-10-30) — "If multiple rows have identical values in the ORDER BY columns, the server is free to return those rows in any order"
- **Evidência:** [Stripe Docs — Receive Stripe events in your webhook endpoint](https://docs.stripe.com/webhooks?locale=en-US) (acesso em 2026-10-30) — "Stripe doesn't guarantee the delivery of events in the order that they're generated."
- **O que mudou:** A parte sobre múltiplos workers está correta. A parte sobre um worker único precisa de duas ressalvas. (1) Com retry e backoff (1m/5m/30m/2h/12h, INV-102), se o evento N de um pedido falha e o N+1 do mesmo pedido é entregue no ciclo seguinte, o cliente recebe N+1 antes de N. Isso acontece mesmo com um só worker, e só não acontece se o worker bloquear os eventos seguintes do mesmo pedido enquanto houver um pendente. (2) `created_at` mapeado pelo Prisma como `DateTime` vira `DATETIME(3)` (milissegundo) no MySQL. Duas mudanças de status no mesmo milissegundo empatam, e com empate o MySQL pode devolver as linhas em qualquer ordem. Como o id da outbox é UUID (INV-105), não há desempate natural. A ordem só é determinística com um segundo critério monotônico.
- **Impacto:** Afeta a limitação documentada de ordering (INV-021/INV-022) e o que se promete ao cliente B2B. Se o FDD ou a doc do cliente disserem "com um worker os eventos chegam em ordem", a promessa é mais forte do que o desenho entrega. Stripe, a referência citada na reunião, não promete ordem.

## VER-006 — Concorrência entre workers via partição por order_id ou lock pessimista

- **Afirmação:** "Aí dá pra particionar por order_id, ou usar lock pessimista." — INV-023, [09:13] Diego
- **Veredito:** Confirmada
- **Evidência:** [MySQL 8.0 Reference Manual — 17.7.2.4 Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html) (acesso em 2026-10-30) — "A locking read that uses SKIP LOCKED never waits to acquire a row lock. The query executes immediately, removing locked rows from the result set."
- **Evidência:** [MySQL 8.0 Reference Manual — 17.7.2.4 Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html) (acesso em 2026-10-30) — "SKIP LOCKED is therefore not suitable for general transactional work. However, it may be used to avoid lock contention when multiple sessions access the same queue-like table."
- **O que mudou:** —
- **Impacto:** Sustenta o caminho futuro de escala (INV-023). O MySQL 8.0 do repositório tem `SELECT ... FOR UPDATE` e `SKIP LOCKED`/`NOWAIT`. Ressalvas de uso, não de veracidade: `SKIP LOCKED` evita que dois workers peguem a mesma linha, mas sozinho não preserva a ordem por pedido, porque um worker pode pular o evento N travado e pegar o N+1 do mesmo pedido. Por isso a ordem depende da partição por `order_id`. A doc também avisa que esses statements "are unsafe for statement based replication", o que importa se houver réplica com binlog em formato STATEMENT.

## VER-007 — HMAC-SHA256 é o padrão de mercado para assinar webhooks

- **Afirmação:** "Padrão é HMAC." — INV-039, [09:20] Sofia / "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso." — INV-039, [09:20] Sofia
- **Veredito:** Confirmada
- **Evidência:** [RFC 2104 — HMAC: Keyed-Hashing for Message Authentication](https://www.rfc-editor.org/rfc/rfc2104) (acesso em 2026-10-30) — "This document describes HMAC, a mechanism for message authentication using cryptographic hash functions."
- **Evidência:** [NIST FIPS 198-1 — The Keyed-Hash Message Authentication Code (HMAC)](https://csrc.nist.gov/pubs/fips/198-1/final) (acesso em 2026-10-30) — "This Standard describes a keyed-hash message authentication code (HMAC)"
- **Evidência:** [Stripe Docs — Verify webhook signatures manually](https://docs.stripe.com/webhooks?locale=en-US) (acesso em 2026-10-30) — "Stripe generates signatures using a hash-based message authentication code (HMAC) with SHA-256."
- **Evidência:** [GitHub Docs — Validating webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries) (acesso em 2026-10-30) — "The hash signature will appear in each delivery as the value of the X-Hub-Signature-256 header."
- **Evidência:** [Shopify Dev — Deliver webhooks through HTTPS](https://shopify.dev/docs/apps/build/webhooks/subscribe/https) (acesso em 2026-10-30) — "Each HTTPS delivery includes a base64-encoded HMAC signature"
- **Evidência:** [Standard Webhooks specification](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md) (acesso em 2026-10-30) — "HMAC-SHA256 is fast and often hardware accelerated, and much faster than any asymmetric scheme."
- **O que mudou:** —
- **Impacto:** Sustenta a decisão "HMAC-SHA256, secret por endpoint, rotação com grace period de 24h" (INV-039). A rotação de 24h coincide com a da Stripe ("delay its expiration for up to 24 hours"). O "todo cliente tem biblioteca" é razoável: HMAC-SHA256 é primitiva padronizada (RFC/FIPS) presente nas bibliotecas criptográficas comuns. Mas o mercado não é uniforme no *que* se assina. Ver VER-013.

## VER-008 — Exigir https é só uma validação no schema Zod

- **Afirmação:** "Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod." — INV-045, [09:23] Sofia
- **Veredito:** Confirmada
- **Evidência:** [Zod v3.23.8 — README](https://raw.githubusercontent.com/colinhacks/zod/v3.23.8/README.md) (acesso em 2026-10-30) — "z.string().startsWith("https://", { message: "Must provide secure URL" });"
- **Evidência:** [Zod v3.23.8 — src/types.ts, check "url"](https://raw.githubusercontent.com/colinhacks/zod/v3.23.8/src/types.ts) (acesso em 2026-10-30) — "new URL(input.data);"
- **Evidência:** [Zod docs (v4) — URLs](https://zod.dev/api?id=urls) (acesso em 2026-10-30) — "const schema = z.url({ protocol: /^https$/ });"
- **O que mudou:** —
- **Impacto:** Sustenta o tratamento do requisito como validação de schema (INV-045/INV-046) e o código `WEBHOOK_INVALID_URL`. Ressalva de implementação: no Zod 3.23.8 do repositório, `z.string().url()` só testa se `new URL()` aceita a string, então qualquer esquema passa (`http:`, `ftp:`, `mailto:`). É preciso combinar com `.startsWith("https://")` ou com um `.refine` que cheque `new URL(v).protocol === "https:"`. A opção `z.url({ protocol })` só existe no Zod 4 (a versão mais recente no npm hoje é a 4.x), não na versão do repo. Exigir https não cobre outros riscos de URL cadastrada pelo cliente (ex.: hosts internos), mas isso não estava na afirmação.

## VER-009 — At-least-once com event id é o que Stripe e GitHub fazem

- **Afirmação:** "Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim." — INV-052, [09:25] Diego
- **Veredito:** Parcial
- **Evidência:** [Stripe Docs — Handle duplicate events](https://docs.stripe.com/webhooks?locale=en-US) (acesso em 2026-10-30) — "Webhook endpoints might occasionally receive the same event more than once. You can guard against duplicated event receipts by logging the event IDs you've processed"
- **Evidência:** [Stripe Docs — Automatic retries](https://docs.stripe.com/webhooks?locale=en-US) (acesso em 2026-10-30) — "Stripe attempts to deliver events to your destination for up to three days with an exponential back off in live mode."
- **Evidência:** [GitHub Docs — Handling failed webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries) (acesso em 2026-10-30) — "GitHub does not automatically redeliver failed deliveries."
- **Evidência:** [GitHub Docs — Best practices for using webhooks](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks) (acesso em 2026-10-30) — "If you request a redelivery, the X-GitHub-Delivery header will be the same as in the original delivery."
- **Evidência:** [Shopify Dev — Deliver webhooks through HTTPS](https://shopify.dev/docs/apps/build/webhooks/subscribe/https) (acesso em 2026-10-30) — "Shopify minimizes duplicate deliveries"
- **O que mudou:** A Stripe faz exatamente o desenho decidido: retry automático com backoff, duplicatas possíveis e dedup por event id. O GitHub não. Ele tem um id estável (`X-GitHub-Delivery`) que se repete na reentrega, mas não reenvia sozinho o que falhou. A reentrega é manual, pela UI ou API. O GitHub é um exemplo de "event id para o cliente dedupar", mas não de retry automático at-least-once. A Shopify (não citada na reunião) é um exemplo melhor do mesmo modelo da Stripe.
- **Impacto:** Sustenta a decisão de at-least-once com `X-Event-Id` e dedup no cliente (INV-051/INV-052/INV-102). O argumento de "padrão de mercado" continua válido com Stripe e Shopify. Se o ADR citar o GitHub como precedente do retry automático, cita errado.

## VER-010 — Exactly-once exigiria coordenação dos dois lados

- **Afirmação:** "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo." — INV-052, [09:25] Diego
- **Veredito:** Inconclusiva
- **Evidência:** [Pattern: Transactional outbox — microservices.io](https://microservices.io/patterns/data/transactional-outbox.html) (acesso em 2026-10-30) — "a message consumer must be idempotent"
- **O que mudou:** Não há especificação ou doc oficial que trate "exactly-once em webhooks HTTP" e sirva de fonte primária para a afirmação como foi dita. O que as fontes primárias mostram é indireto. A referência do padrão outbox admite publicação duplicada e transfere a correção para o consumidor idempotente. Nenhum provedor consultado (Stripe, GitHub, Shopify, Standard Webhooks) oferece exactly-once, e todos pedem dedup no receptor. Isso é coerente com a afirmação, mas não a prova.
- **Impacto:** Apoia o descarte de exactly-once (INV-052). Fica sem fonte o grau de complexidade ("muito mais complexo"). Se o ADR usar isso como motivo, o argumento é de engenharia do time, não fato verificado. O apoio verificável é "nenhum provedor de referência oferece exactly-once" (VER-009).

## VER-011 — At-least-once com event_id resolve 99% dos casos

- **Afirmação:** "At-least-once com event_id resolve 99% dos casos." — INV-052, [09:25] Diego
- **Veredito:** Inconclusiva
- **Evidência:** Nenhuma fonte primária encontrada para o número. As docs consultadas (Stripe, GitHub, Shopify, Standard Webhooks) recomendam dedup por id, mas nenhuma quantifica a cobertura.
- **O que mudou:** O "99%" é retórico, sem medição. Fica como estimativa informal.
- **Impacto:** Não deve aparecer em ADR/RFC como número. A decisão se sustenta pela prática de mercado (VER-009), não pelo percentual.

## VER-012 — PrismaClient é por processo

- **Afirmação:** "PrismaClient é por processo." — INV-063, [09:30] Bruno
- **Veredito:** Confirmada
- **Evidência:** [Prisma docs — Instantiating Prisma Client](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/instantiate-prisma-client) (acesso em 2026-10-30) — "Creating multiple instances of PrismaClient will create multiple connection pools"
- **Evidência:** [Prisma docs — Connection pool](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-pool) (acesso em 2026-10-30) — default do pool (até v6): "num_cpus::get_physical() * 2 + 1"
- **O que mudou:** —
- **Impacto:** Sustenta a decisão de um PrismaClient separado no worker (INV-063, que corrige INV-020). A rigor o pool é por instância: um processo pode ter várias e cada uma abre seu pool. Na prática, o processo `npm run worker` terá o próprio pool, e o MySQL passa a receber as conexões da API mais as do worker. No Prisma 5.22 do repo o default é `núcleos físicos × 2 + 1` por processo, ajustável via `connection_limit` na `DATABASE_URL`. Vale dimensionar contra o `max_connections` do MySQL. Observação de versão: a partir do Prisma 7 (nov/2025) os defaults de pool passam a vir do driver adapter ("connection pooling defaults (and configuration) now come from the driver itself"), e a linha 8 está em release candidate. Nada disso afeta o repo enquanto ele estiver no 5.22.

## VER-013 — X-Timestamp permite ao cliente detectar replay attack

- **Afirmação:** "X-Timestamp com o timestamp do envio (pra cliente conseguir detectar replay attack se quiser)" — INV-093, [09:44] Diego
- **Veredito:** Parcial
- **Evidência:** [Stripe Docs — Preventing replay attacks](https://docs.stripe.com/webhooks?locale=en-US) (acesso em 2026-10-30) — "Because this timestamp is part of the signed payload, it's also verified by the signature, so an attacker can't change the timestamp without invalidating the signature."
- **Evidência:** [Stripe Docs — Prepare the signed_payload string](https://docs.stripe.com/webhooks?locale=en-US) (acesso em 2026-10-30) — "The timestamp (as a string) / The character . / The actual JSON payload (that is, the request body)"
- **Evidência:** [Standard Webhooks specification](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md) (acesso em 2026-10-30) — "The content to be signed is therefore: msg_id.timestamp.payload"
- **Evidência:** [GitHub Docs — Best practices for using webhooks](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks) (acesso em 2026-10-30) — "you can use the X-GitHub-Delivery header to ensure that each delivery is unique per event"
- **O que mudou:** O timestamp só serve contra replay se estiver coberto pela assinatura. Nas duas referências (Stripe e Standard Webhooks) ele entra no conteúdo assinado. A decisão do time foi "HMAC-SHA256 sobre o corpo do request" ([09:22] Sofia; o resumo de [09:48] diz "sobre payload", INV-103), sem incluir o timestamp. Com `X-Timestamp` fora do HMAC, quem intercepta uma entrega pode reenviá-la com um `X-Timestamp` novo e a assinatura continua válida. O cabeçalho então não detecta replay. O que resta ao cliente é o dedup por `X-Event-Id`, como no GitHub, que exige guardar ids processados.
- **Impacto:** Afeta o formato dos headers (INV-093) e a decisão de assinatura (INV-039), ponto que a Sofia pediu para revisar (INV-098). Se a doc do cliente prometer proteção contra replay via `X-Timestamp` com a assinatura só sobre o corpo, a promessa não se cumpre. A verificação não decide o desenho, mas sinaliza a divergência entre o header como descrito e o conteúdo assinado como decidido.
