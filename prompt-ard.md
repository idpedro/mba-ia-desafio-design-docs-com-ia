/adr

analise o documento @docs/research/decisoes-candidatas.md e sintetize em 5 a no maximo 8 adrs seguindo as regas:

As Adrs devem conter as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 2 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

O conjunto de ADRs deve cobrir todas as 6 decições pincipais. 

- Padrão Outbox no MySQL
- Política de retry com backoff e DLQ
- Autenticação HMAC-SHA256 com secret por endpoint
- Garantia at-least-once com X-Event-Id
- Worker em processo separado em polling
- Reuso dos padrões existentes do projeto

O reuso dos padrões aparece no research como "detalhe", mas é uma decisão do time ([09:30] Larissa) e deve virar ADR.

Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais ou ficar apenas no FDD, conforme você considerar mais adequado.

Justifique cada escolha.
