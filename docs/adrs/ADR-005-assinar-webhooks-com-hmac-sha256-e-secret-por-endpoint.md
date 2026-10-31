---
date: 2026-10-30
decision-makers: [Sofia]
consulted: [Bruno, Diego]
informed: [Larissa, Marcos]
---

# ADR-005: Assinar webhooks com HMAC-SHA256 usando uma secret por endpoint, rotacionável com grace period de 24h

## Status

Aceito — reunião técnica de webhooks, fechado em [09:22] Sofia ("Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h") e confirmado em [09:48] Larissa.

## Contexto e Problema

Os eventos levam dados de pedidos para um endpoint fora da infraestrutura da plataforma. "O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio" ([09:19] Sofia). Como o cliente prova origem e integridade, e como limitar o estrago de um vazamento?

## Fatores de Decisão

- Autenticidade e integridade verificáveis pelo cliente ([09:19] Sofia).
- Usar uma técnica que "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
- Já houve cliente "que vazou secret em log de aplicação dele" ([09:22] Diego).

## Alternativas Consideradas

- HMAC-SHA256 com secret por endpoint, rotacionável
- HMAC com uma secret global da plataforma
- Confiar só no TLS, sem assinatura (alternativa plausível, não discutida na reunião)

## Decisão

Opção escolhida: "HMAC-SHA256 com secret por endpoint, rotacionável", porque "HMAC-SHA256 é o padrão de mercado" ([09:20] Sofia) e a secret por endpoint isola o vazamento. Vence a secret global porque "se vaza uma, vaza tudo" ([09:21] Sofia). Vence o TLS sozinho porque ele protege o transporte, mas não prova a origem. O TLS continua obrigatório ([09:23] Sofia).

Parâmetros:
- Assinatura do corpo do request no header `X-Signature` ([09:20] Sofia).
- A secret é gerada pela plataforma e devolvida na criação do webhook ([09:31] Marcos).
- Endpoint de rotação: a secret antiga fica válida por 24h em paralelo e depois "morre" ([09:21] Sofia).
- A URL precisa ser `https`, e `http` é recusado no schema Zod ([09:23] Sofia).

O armazenamento da secret e o formato da assinatura ficam no [FDD](../FDD.md).

## Consequências

- Positiva: o cliente valida origem e integridade com bibliotecas comuns ([09:20] Sofia).
- Positiva: um vazamento afeta um endpoint só, e a rotação dá tempo para migrar ([09:21] Sofia).
- Negativa: a secret precisa ficar recuperável para assinar, e não basta um hash como o da senha.
- Negativa: durante as 24h, duas secrets estão válidas.
- Trade-off: aceita-se guardar secrets recuperáveis e manter duas válidas por 24h em troca de verificação simples e de vazamentos isolados.

### Confirmação

Revisão de segurança da Sofia, de pelo menos dois dias úteis, antes do deploy ([09:46] Sofia). Um teste verifica a assinatura com uma secret conhecida e a recusa de URL `http`.

## Prós e Contras das Alternativas

### Secret global

Uma secret única para todos os clientes ([09:21] Sofia).

- Bom, porque a gestão é mais simples.
- Ruim, porque "se vaza uma, vaza tudo" ([09:21] Sofia). ← motivo do descarte

### Só TLS

Confiar no `https` sem assinar o payload. Alternativa plausível, não discutida na reunião.

- Bom, porque não exige secret nem verificação no cliente.
- Ruim, porque o cliente não consegue "validar que a requisição veio realmente da gente" ([09:19] Sofia). ← motivo do descarte

## Mais Informações

- Fontes: [09:19] Sofia a [09:23] Sofia; [09:31] Marcos; [09:46] Sofia.
- Código relacionado: `src/modules/auth/auth.service.ts` (único hash hoje, bcrypt de senha).
- Referências externas:
  - [RFC 2104](https://www.rfc-editor.org/rfc/rfc2104), [GitHub — Validating webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries) e [Standard Webhooks](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md) — respaldam o HMAC-SHA256 como padrão, Confirmada.
  - [Zod v3.23.8 README](https://raw.githubusercontent.com/colinhacks/zod/v3.23.8/README.md) — Confirmada, com a nota de que `z.string().url()` aceita qualquer esquema.
- Relacionados: [ADR-006](ADR-006-garantir-entrega-at-least-once-com-x-event-id.md).
- Questões em aberto:
  - Veredito Parcial: o `X-Timestamp` ([09:44] Diego) só ajuda contra replay se entrar no conteúdo assinado, mas a decisão é assinar só o corpo.
  - Não foi dito se, durante a rotação, o envio leva uma ou duas assinaturas.
