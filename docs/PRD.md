# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Status | Rascunho |
| Data | 2026-10-30 |
| Responsável de produto | Marcos (Product Manager) |
| Stakeholders | Larissa (Tech Lead), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança), Atlas Comercial, MaxDistribuição e Nova Cargo (clientes B2B) |
| Relacionados | [RFC](RFC.md), [Design](FDD.md), [ADRs](adrs/README.md) |

## 1. Resumo e Contexto

Hoje os clientes B2B descobrem que um pedido mudou de status consultando a API periodicamente. A feature notifica o cliente por webhook, um HTTP de saída assinado, em menos de 10 segundos depois de cada mudança de status que ele assinou. O cliente cadastra os próprios endpoints, escolhe os status que quer receber, valida a assinatura e consulta o histórico de entregas. A feature toca a mudança de status de pedidos (`src/modules/orders/order.service.ts`), cria um módulo de webhooks e um processo de entrega separado da API ([RFC](RFC.md)).

## 2. Problema e Motivação

- Atlas Comercial, MaxDistribuição e Nova Cargo fizeram um pedido formal para serem notificados "em tempo real" quando o status dos pedidos muda ([09:00] Marcos).
- Hoje eles fazem polling em `GET /orders`, o que deixa a integração "lenta e cara" para eles ([09:00] Marcos). Não existe nenhuma notificação de saída no sistema ([mapeamento do código](research/mapeamento-codigo.md), Ausências).
- Risco comercial: a Atlas sinalizou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre ([09:00] Marcos). O prazo pedido depois foi "fim de novembro" ([09:45] Marcos).
- O pedido é só de saída: os clientes "querem receber, não mandar" ([09:02] Marcos).

## 3. Público-alvo e Cenários de Uso

**Público:**
- Clientes B2B que integram com a plataforma pela API, começando pelos três do pedido formal ([09:00] Marcos). Eles cadastram webhooks com o JWT do sistema, por meio de "usuários que representam o cliente" ([09:32] Marcos). Essa premissa não tem apoio no código (seção 10).
- ADMIN interno, que reprocessa eventos que esgotaram as tentativas ([09:36] Sofia).

**Cenários:**
- **CEN-01 — Cadastrar um endpoint:** o cliente cadastra uma URL https e os status que quer ouvir, e recebe a secret de assinatura uma única vez, na criação ([09:31] Marcos).
- **CEN-02 — Receber a mudança de status:** um pedido do cliente vira `SHIPPED`; em menos de 10 segundos o endpoint dele recebe o evento assinado e não precisa mais consultar a API ([09:02] Marcos; [09:33] Marcos).
- **CEN-03 — Cliente fora do ar:** o endpoint do cliente cai por duas horas em manutenção planejada; a plataforma retenta e entrega quando ele volta ([09:16] Diego).
- **CEN-04 — Evento duplicado:** o cliente recebe o mesmo evento duas vezes e descarta a cópia pelo identificador do evento ([09:25] Diego).
- **CEN-05 — Secret vazada:** o cliente vaza a secret num log, pede uma nova e tem 24 horas para trocar nos sistemas dele, com a antiga ainda válida ([09:21] Sofia; [09:22] Diego).
- **CEN-06 — Diagnóstico de entrega:** o cliente consulta as últimas entregas, com sucesso ou falha, payload, resposta e tempo de resposta ([09:34] Marcos).
- **CEN-07 — Reprocessamento:** um ADMIN recoloca na fila um evento que esgotou as tentativas, e o sistema registra quem fez ([09:18] Diego; [09:36] Sofia).

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta | Fonte |
| --- | --- | --- | --- |
| Notificar em "tempo real" na definição do cliente | Tempo entre a mudança de status e o primeiro envio ao cliente ([FDD](FDD.md) §11) | < 10 segundos | [09:02] Marcos |
| Não perder a Atlas | Feature em produção dentro do prazo pedido | Até o fim de novembro | [09:45] Marcos |
| Absorver indisponibilidades do cliente sem ação humana | Janela coberta pelas retentativas antes de o evento ir para a DLQ | ~15 horas | [09:17] Diego; [09:17] Marcos |

## 5. Escopo

**Incluso:**
- Cadastro, edição, remoção e listagem de webhooks por customer, com filtro de status ([09:31] Marcos; [09:33] Bruno).
- Notificação assinada de cada mudança de status assinada pelo cliente ([09:33] Marcos; [09:20] Sofia).
- Rotação de secret com 24 horas de convivência ([09:21] Sofia).
- Retentativa automática e fila de falhas definitivas (DLQ) ([09:15] Diego; [09:18] Diego).
- Histórico de entregas por webhook ([09:34] Marcos).
- Reprocessamento manual da DLQ por ADMIN, com auditoria ([09:18] Diego; [09:36] Sofia).
- Documentação de integração no portal de desenvolvedor, com a deduplicação em destaque ([09:26] Marcos; [09:40] Marcos).

**Fora de escopo:**

- Webhooks de entrada (clientes enviando para a plataforma) — descartado, os clientes "querem receber, não mandar", [09:02] Marcos.
- Aviso por email quando o webhook do cliente falha — adiado para a próxima fase, "depois que a gente medir o impacto", [09:37] Larissa; [09:38] Marcos.
- Dashboard visual para o cliente ver os webhooks — fora de escopo, "Painel é projeto separado do time de frontend", [09:40] Larissa.
- Rate limiting de envio por cliente — em observação, "A gente observa e implementa se virar problema", [09:39] Diego; [09:39] Larissa.
- Arquivamento das entregas antigas ("depois de 30 dias ou assim") — fora do escopo da feature, [09:08] Diego.
- Restringir por role o CRUD de configuração de webhooks — adiado, "Mais pra frente a gente pode endurecer", [09:37] Sofia.
- Garantia de ordem global entre pedidos — não pedida, os clientes "só querem saber se cada pedido deles mudou", [09:14] Marcos.
- Entrega exactly-once — descartada, "exigiria coordenação dos dois lados e fica muito mais complexo", [09:25] Diego.
- Items do pedido no payload — descartado "pra não inflar"; o cliente busca os detalhes em `GET /orders/:id`, [09:43] Diego.
- Truncar eventos acima do limite de tamanho — descartado, "Se chegou nesse tamanho, tem algo errado", [09:23] Sofia.

As alternativas técnicas descartadas ou adiadas estão no [RFC](RFC.md) (Alternativas Consideradas) e nos [ADRs](adrs/README.md).

## 6. Requisitos Funcionais

### RF-001 — Cadastrar webhook

- **Descrição:** o cliente cadastra um endpoint de webhook para um customer, informando a URL e os status que quer receber, e recebe a secret gerada pela plataforma.
- **Fluxo principal:** informa customer, URL https e lista de status → a plataforma gera a secret → devolve o webhook criado com a secret.
- **Alternativas e erros:** URL `http` é recusada com erro de validação ([09:23] Sofia). Customer inexistente é recusado. A secret só aparece nesta resposta e na rotação ([FDD](FDD.md) §10).
- **Prioridade:** Alta
- **Fonte:** [09:31] Marcos; [09:32] Larissa

### RF-002 — Editar, remover e listar webhooks

- **Descrição:** o cliente edita a URL, os status e o estado ativo de um webhook, remove um webhook e lista os webhooks de um customer.
- **Fluxo principal:** edição parcial dos campos; remoção pelo id; listagem filtrada por customer.
- **Alternativas e erros:** webhook inexistente responde "não encontrado". A edição segue as mesmas regras de URL do cadastro. A listagem nunca devolve a secret ([FDD](FDD.md) §10).
- **Prioridade:** Alta
- **Fonte:** [09:33] Bruno

### RF-003 — Filtrar eventos por status

- **Descrição:** cada webhook recebe só as mudanças para os status que ele assinou (ex.: só `SHIPPED` e `DELIVERED`).
- **Fluxo principal:** mudança de status → a plataforma verifica os webhooks ativos do customer → só registra o evento para os que assinam o novo status.
- **Alternativas e erros:** se nenhum webhook do customer assina o status, nenhum evento é gerado ([09:34] Bruno).
- **Prioridade:** Alta
- **Fonte:** [09:33] Marcos; [09:34] Bruno

### RF-004 — Notificar a mudança de status

- **Descrição:** a cada mudança de status assinada, o endpoint do cliente recebe um evento `order.status_changed` com os dados básicos do pedido, o status anterior e o novo.
- **Fluxo principal:** status muda → evento registrado junto com a mudança → envio ao endpoint com identificador do evento, identificador do webhook, timestamp e assinatura. Payload e headers ficam no [FDD](FDD.md) §6.9.
- **Alternativas e erros:** o payload é um retrato do pedido no momento da mudança, mesmo que o pedido mude antes do envio ([ADR-008](adrs/ADR-008-gravar-payload-como-snapshot-na-insercao.md)). Os items não vão no evento; o cliente consulta `GET /orders/:id` ([09:43] Diego).
- **Prioridade:** Alta
- **Fonte:** [09:00] Marcos; [09:43] Diego; [09:44] Diego; [09:44] Sofia

### RF-005 — Validar origem e integridade

- **Descrição:** o cliente consegue confirmar que a requisição veio da plataforma e que o corpo não foi adulterado.
- **Fluxo principal:** a plataforma assina o corpo com a secret do endpoint → o cliente recalcula a assinatura com a secret dele e compara.
- **Alternativas e erros:** cada endpoint tem secret própria; um vazamento afeta só aquele endpoint ([09:21] Sofia).
- **Prioridade:** Alta
- **Fonte:** [09:19] Sofia; [09:20] Sofia

### RF-006 — Rotacionar a secret

- **Descrição:** o cliente pede uma nova secret pela API, e a antiga continua válida por 24 horas em paralelo.
- **Fluxo principal:** pedido de rotação → a plataforma devolve a nova secret → durante 24h os envios são verificáveis com a antiga e com a nova → depois, só com a nova.
- **Alternativas e erros:** o comportamento de uma segunda rotação dentro das 24h está no [FDD](FDD.md) §7.
- **Prioridade:** Alta
- **Fonte:** [09:21] Sofia; [09:22] Sofia

### RF-007 — Retentar e mover falhas para a DLQ

- **Descrição:** quando o endpoint do cliente falha ou não responde, a plataforma retenta com intervalos crescentes e, esgotadas as tentativas, guarda o evento numa fila de falhas definitivas com o motivo.
- **Fluxo principal:** envio inicial falha → retentativas após 1m, 5m, 30m, 2h e 12h → sucesso em qualquer uma encerra o evento → falha na última move o evento para a DLQ.
- **Alternativas e erros:** evento acima do limite de tamanho não é enviado e vai direto para a DLQ (RNF-006; [FDD](FDD.md) §9).
- **Prioridade:** Alta
- **Fonte:** [09:15] Diego; [09:17] Diego; [09:18] Diego

### RF-008 — Consultar o histórico de entregas

- **Descrição:** o cliente vê as últimas 100 entregas de um webhook, com sucesso ou falha, payload, resposta e tempo de resposta.
- **Fluxo principal:** consulta pelo id do webhook → lista das entregas mais recentes.
- **Alternativas e erros:** webhook inexistente responde "não encontrado".
- **Prioridade:** Média
- **Fonte:** [09:34] Marcos; [09:35] Larissa

### RF-009 — Reprocessar evento da DLQ

- **Descrição:** um ADMIN recoloca um evento da DLQ na fila de envio, e o sistema registra quem fez o replay.
- **Fluxo principal:** ADMIN pede o replay de um item da DLQ → o evento volta como pendente → o worker envia com o mesmo identificador do evento.
- **Alternativas e erros:** usuário sem role ADMIN é recusado ([09:36] Sofia). Os casos de item já reprocessado e de webhook inativo estão no [FDD](FDD.md) §7.
- **Prioridade:** Média
- **Fonte:** [09:18] Diego; [09:36] Sofia; [09:36] Larissa

### RF-010 — Permitir deduplicação no cliente

- **Descrição:** cada evento tem um identificador único e estável, que se repete nas retentativas e no replay, para o cliente descartar duplicatas.
- **Fluxo principal:** o evento recebe um UUID ao ser registrado → todos os envios do evento levam o mesmo identificador.
- **Alternativas e erros:** o portal de desenvolvedor explica, em destaque, que o mesmo evento pode chegar mais de uma vez ([09:26] Marcos).
- **Prioridade:** Alta
- **Fonte:** [09:24] Diego; [09:25] Diego

## 7. Requisitos Não Funcionais

| ID | Categoria | Requisito | Fonte |
| --- | --- | --- | --- |
| RNF-001 | Desempenho | O primeiro envio ao cliente sai em menos de 10 segundos após a mudança de status | [09:02] Marcos; [09:09] Diego |
| RNF-002 | Confiabilidade | Se o status mudou, o evento existe; se a mudança foi desfeita, o evento não existe. Falha ao registrar o evento desfaz a mudança de status | [09:40] Bruno; [09:41] Diego |
| RNF-003 | Disponibilidade | Um cliente lento ou fora do ar não atrasa nem desfaz a mudança de status de nenhum pedido | [09:04] Bruno |
| RNF-004 | Segurança | A URL do webhook é obrigatoriamente `https`; `http` é recusado com erro de validação | [09:23] Sofia |
| RNF-005 | Segurança | Toda requisição de saída é assinada com HMAC-SHA256 sobre o corpo, com secret única por endpoint | [09:20] Sofia; [09:21] Sofia |
| RNF-006 | Limite | Evento acima de 64KB não é enviado e gera erro, sem truncar | [09:23] Sofia; [09:24] Larissa |
| RNF-007 | Desempenho | Resposta do cliente acima de 10 segundos conta como falha e entra em retentativa | [09:42] Diego |
| RNF-008 | Confiabilidade | Envio inicial mais 5 retentativas, com intervalos de 1m, 5m, 30m, 2h e 12h (quase 15h entre a primeira falha e a última tentativa) | [09:17] Diego; [09:17] Larissa |
| RNF-009 | Confiabilidade | Entrega at-least-once: nenhum evento registrado é descartado sem ir para a DLQ | [09:24] Diego; [09:26] Larissa |
| RNF-010 | Consistência | Eventos do mesmo pedido chegam na ordem das mudanças, enquanto houver um único worker; não há ordem global entre pedidos | [09:12] Diego; [09:13] Larissa |
| RNF-011 | Segurança | O replay da DLQ exige role ADMIN; o CRUD de webhooks aceita qualquer usuário autenticado nesta fase | [09:36] Sofia; [09:37] Sofia |
| RNF-012 | Auditoria | Todo replay da DLQ registra quem o executou | [09:36] Sofia |
| RNF-013 | Segurança | A secret só é exibida na criação e na rotação, e nunca aparece em log | [09:31] Marcos; [09:21] Sofia; `src/shared/logger/index.ts` |

## 8. Decisões e Trade-offs

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Outbox no MySQL, gravada na mesma transação da mudança de status | Mais um insert numa transação já pesada, em troca de atomicidade sem infraestrutura nova | [ADR-001](adrs/ADR-001-usar-outbox-no-mysql-para-eventos-de-webhook.md) |
| Worker em processo separado, polling a cada 2 segundos | Até 2s de espera e um segundo processo para operar, em troca de não depender de infraestrutura nova | [ADR-002](adrs/ADR-002-processar-outbox-com-worker-separado-em-polling.md) |
| Worker único, com ordem garantida só por pedido | Teto de vazão e ponto único de falha, em troca de ordem por pedido sem coordenação | [ADR-003](adrs/ADR-003-operar-com-worker-unico-e-ordering-por-order-id.md) |
| Backoff exponencial em 5 retentativas e DLQ em tabela separada, com replay por ADMIN | Até ~15h de atraso e reprocessamento manual, em troca de um fim definido para cada evento | [ADR-004](adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md) |
| HMAC-SHA256 com secret por endpoint, rotacionável com 24h de convivência | Secret guardada de forma recuperável e duas secrets válidas por 24h, em troca de vazamentos isolados | [ADR-005](adrs/ADR-005-assinar-webhooks-com-hmac-sha256-e-secret-por-endpoint.md) |
| Entrega at-least-once com identificador de evento para deduplicação | O cliente passa a deduplicar, em troca de não perder eventos | [ADR-006](adrs/ADR-006-garantir-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto (módulos, erros, logger, middleware) | Herda os limites desses padrões, em troca de consistência e nenhuma dependência nova | [ADR-007](adrs/ADR-007-reusar-padroes-existentes-do-projeto-no-modulo-de-webhooks.md) |
| Payload gravado como snapshot no momento da mudança | Mais armazenamento e formato congelado nos eventos já gravados, em troca de fidelidade ao momento da mudança | [ADR-008](adrs/ADR-008-gravar-payload-como-snapshot-na-insercao.md) |

## 9. Dependências

- Fluxo de mudança de status do time de Pedidos — o evento nasce dentro de `changeStatus`, que precisa ser alterado, [09:40] Bruno; `src/modules/orders/order.service.ts`.
- Segundo processo em produção — o worker roda separado da API, no mesmo banco, [09:11] Diego. Quem opera esse processo não foi definido (seção 13).
- MySQL existente — é a única infraestrutura da feature, [09:07] Diego; `docker-compose.yml`.
- Revisão de segurança da Sofia — pelo menos dois dias úteis antes do deploy, com foco em HMAC e geração de secret, [09:46] Sofia; [09:49] Sofia.
- Sessão de revisão do design com Bruno e Diego — antes de começar a codar, [09:50] Larissa.
- Portal de desenvolvedor — Marcos documenta a integração e a deduplicação, [09:26] Marcos; [09:40] Marcos.
- Clientes B2B — precisam implementar a validação da assinatura e a deduplicação do lado deles, [09:25] Sofia.
- Role ADMIN e `requireRole` existentes — base da restrição do replay, [09:36] Larissa; `src/middlewares/auth.middleware.ts`.

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação | Fonte |
| --- | --- | --- | --- | --- |
| O `customer_id` não vem do JWT, e sim do body ou do path. O código não tem vínculo entre `User` e `Customer`, então nada garante que quem chama representa aquele customer: qualquer usuário autenticado pode cadastrar, ler, rotacionar ou apagar webhooks de qualquer customer, ver a secret na criação e os payloads no histórico. A premissa de "usuários que representam o cliente" não existe no código | Alta | Alto | Decidir com Marcos e Sofia, na revisão de segurança, se o vínculo `User`↔`Customer` e o endurecimento do CRUD entram nesta fase. Até lá, a limitação fica registrada e não é liberado acesso a usuários externos | [09:32] Larissa; [09:32] Marcos; [09:37] Sofia; `prisma/schema.prisma`; `src/middlewares/auth.middleware.ts` |
| O registro público aceita `role: 'ADMIN'`, o que anula a restrição do replay da DLQ | Média | Alto | Incluir o registro público no escopo da revisão de segurança | [09:36] Sofia; `src/modules/auth/auth.schemas.ts`; `src/modules/auth/auth.routes.ts` |
| Três sprints podem não caber até "fim de novembro"; a data de início e a duração da sprint não foram ditas, e a Atlas ameaça migrar | Média | Alto | Marcos confirma o prazo com a Atlas, mantendo os dois dias de revisão de segurança no fim | [09:00] Marcos; [09:45] Marcos; [09:46] Larissa; [09:47] Marcos |
| Clientes que não deduplicam processam o mesmo evento duas vezes | Média | Médio | Destacar a deduplicação no portal de desenvolvedor | [09:25] Sofia; [09:26] Marcos |
| Clientes lentos no mesmo lote atrasam o envio dos demais além dos 10 segundos | Média | Médio | Acompanhar a métrica de latência do primeiro envio ([FDD](FDD.md) §11) e revisitar o paralelismo se a meta for violada | [09:42] Diego; [09:02] Marcos; [FDD](FDD.md) R-05 |
| Um evento em retentativa pode ser ultrapassado pelo seguinte do mesmo pedido, contra a ordem prometida | Média | Médio | Não prometer ordem estrita no portal até a decisão do time | [09:12] Diego; [RFC](RFC.md) Questões em Aberto |
| O timestamp enviado não entra na assinatura, então não protege contra replay, ao contrário do que foi dito | Média | Médio | Levar à revisão da Sofia; o portal não promete proteção contra replay até lá | [09:44] Diego; [09:22] Sofia; [verificação externa](research/verificacao-externa.md) VER-013 |
| Sem rate limiting, um cliente com muitos pedidos mudando ao mesmo tempo recebe uma rajada de chamadas | Baixa | Médio | Observar e decidir depois, como acordado | [09:38] Diego; [09:39] Larissa |
| Secrets guardadas de forma recuperável no banco: um vazamento de leitura expõe todas | Baixa | Alto | Levar à revisão de segurança | [09:20] Sofia; `src/modules/auth/auth.service.ts` |

## 11. Critérios de Aceitação

- [ ] **CA-01** (RF-001, RNF-004): cadastrar um webhook com URL `https` devolve a secret; com URL `http`, a plataforma recusa com erro de validação.
- [ ] **CA-02** (RF-002, RNF-013): editar, remover e listar webhooks funcionam por customer, e nenhuma resposta além da criação e da rotação contém a secret.
- [ ] **CA-03** (RF-003): um webhook que assina só `DELIVERED` não recebe evento quando o pedido vira `SHIPPED`.
- [ ] **CA-04** (RF-004, RNF-001): com o worker rodando, uma mudança de status assinada chega ao endpoint de teste em menos de 10 segundos, com os dados do momento da mudança e sem items.
- [ ] **CA-05** (RF-005, RNF-005): a assinatura recebida confere com o HMAC-SHA256 do corpo calculado com a secret do endpoint.
- [ ] **CA-06** (RF-006): depois de uma rotação, os envios das 24h seguintes são verificáveis com a secret antiga e com a nova; depois disso, só com a nova.
- [ ] **CA-07** (RF-007, RNF-007, RNF-008): um endpoint que falha ou não responde em 10 segundos recebe o envio inicial e 5 retentativas nos intervalos 1m/5m/30m/2h/12h, e depois o evento aparece na DLQ com o motivo.
- [ ] **CA-08** (RNF-006): um evento acima de 64KB não é enviado e vai para a DLQ com o motivo.
- [ ] **CA-09** (RF-008): o histórico de um webhook mostra até 100 entregas, com sucesso ou falha, payload, resposta e tempo de resposta.
- [ ] **CA-10** (RF-009, RNF-011, RNF-012): o replay por um OPERATOR é recusado; por um ADMIN, o evento volta à fila e o log registra quem fez.
- [ ] **CA-11** (RF-010, RNF-009): o identificador do evento é o mesmo no envio inicial, nas retentativas e no envio após o replay.
- [ ] **CA-12** (RNF-002): se o registro do evento falha, a mudança de status é desfeita.
- [ ] **CA-13** (RNF-003): com o endpoint do cliente fora do ar, a mudança de status responde normalmente.
- [ ] **CA-14** (RNF-010): três mudanças seguidas do mesmo pedido chegam ao cliente na ordem em que aconteceram.

Os critérios técnicos detalhados estão no [FDD](FDD.md) §14.

## 12. Estratégia de Testes e Validação

- **Automatizados:** testes de integração HTTP no padrão existente (Vitest + Supertest contra MySQL real, `tests/`) para o CRUD, a rotação, o histórico e o replay (RF-001, RF-002, RF-006, RF-008, RF-009). Um servidor de teste recebe os webhooks para validar latência, assinatura e payload (RF-004, RF-005, RNF-001). Testes com relógio controlado cobrem retentativas e DLQ (RF-007, RNF-008), e um teste força a falha no registro do evento (RNF-002).
- **Validação com usuários:** revisão de segurança da Sofia antes do deploy ([09:46] Sofia); sessão de revisão do design com Bruno e Diego ([09:50] Larissa); Marcos confirma o prazo e atualiza os três clientes ([09:47] Marcos; [09:49] Marcos).
- **Pós-lançamento:** latência do primeiro envio, taxa de sucesso por tentativa, volume na DLQ e fila pendente acompanhados pelos logs estruturados do worker ([FDD](FDD.md) §11).

## 13. Considerações Gerais

- A meta de 10 segundos não tem percentil definido (todo envio, p95, p99). A fonte só diz "abaixo de 10 segundos" ([09:02] Marcos).
- Larissa disse "A latência mínima vai ser 2 segundos no pior caso" ([09:10] Larissa). "Mínima" e "pior caso" não combinam; o ciclo de 2s é o atraso máximo de leitura, não o de entrega.
- A janela de retentativa foi descrita como "12 ou 24 horas" ([09:15] Diego) e depois como "quase 15 horas" ([09:17] Diego). Vale a progressão fechada, com envio inicial e 5 retentativas ([ADR-004](adrs/ADR-004-retry-com-backoff-exponencial-e-dlq-em-tabela-separada.md)).
- O prazo da Atlas aparece como "fim do trimestre" ([09:00] Marcos) e como "fim de novembro" ([09:45] Marcos). A fonte não diz se é a mesma data.
- A fonte não traz linha de base nem meta para a redução do polling em `GET /orders`. Sem esse número, a redução do polling não entra como métrica.
- A fonte não define um piloto com os clientes antes da liberação geral.
- A fonte não define meta de adoção: o pedido formal vem dos três clientes ([09:00] Marcos), mas não há número de clientes com webhook ativo como objetivo.
- A fonte não define quem opera o worker em produção. Só fica dito que ele roda como processo separado, no mesmo banco ([09:11] Diego).
- A criação do pedido (status inicial `PENDING`) não passa pela mudança de status e não gera evento nesta fase ([FDD](FDD.md) Q-07). A reunião não tratou desse caso.
- O snapshot do payload e o id UUID da outbox foram decididos depois do resumo, sem Marcos e Sofia na call ([09:51] Larissa; [09:52] Bruno).
- Não foi escolhido se o `customer_id` vai no body ou no path ([09:32] Larissa). O FDD propõe body na criação e query na listagem ([FDD](FDD.md) §6).
