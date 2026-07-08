# ADR-003 — Retry com backoff exponencial e dead-letter

## Status

Aceito

## Contexto

A entrega de um webhook pode falhar por motivos temporários (cliente fora do ar, timeout de rede, resposta de erro). O time precisava decidir quantas vezes tentar reentregar, com que espaçamento entre tentativas, e o que fazer quando as tentativas se esgotam (`[09:14]-[09:18] Larissa/Diego/Bruno`).

Já havia um caso real observado de cliente indisponível por cerca de 2 horas durante uma manutenção planejada, o que serviu de referência concreta para calibrar a janela de retry (`[09:16] Diego`).

## Decisão

Retry com backoff exponencial de 5 tentativas, na progressão 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas — cobrindo uma janela total de aproximadamente 15 horas entre a primeira falha e a última tentativa (`[09:15]-[09:17] Diego`). Esgotadas as 5 tentativas, o evento é removido da outbox e persistido em uma tabela separada, `webhook_dead_letter`, com o payload, o motivo da última falha e o total de tentativas (`[09:18] Diego`).

Reprocessamento de um evento em dead-letter é manual, via endpoint administrativo (`POST /admin/webhooks/dead-letter/:id/replay`), restrito a usuários com role `ADMIN`, que recoloca o evento na outbox como pendente (`[09:18]-[09:19] Diego/Larissa`, `[09:35]-[09:36] Sofia`).

## Alternativas Consideradas

**3 tentativas em vez de 5.** Mais agressivo em liberar o evento para dead-letter mais rápido. Descartada porque mataria o evento de forma prematura diante de uma indisponibilidade real e não incomum — o caso já observado de 2 horas de manutenção planejada não seria coberto por 3 tentativas em um intervalo curto (`[09:15]-[09:16] Diego/Bruno`).

**Retry indefinido, sem teto de tentativas.** Considerada implicitamente ao se discutir alternativas a um número fixo. Descartada porque deixaria eventos pendurados indefinidamente caso o cliente tenha desaparecido de vez, sem sinalizar isso como falha permanente em lugar nenhum (`[09:15] Diego`).

**Marcar falha permanente como um status na própria tabela de outbox, sem tabela separada.** Mais simples de implementar à primeira vista, mas descartada porque misturaria eventos ativos (aguardando processamento) com eventos mortos na mesma tabela consultada pelo worker a cada ciclo, dificultando a leitura e a auditoria de falhas (`[09:18] Diego`).

## Consequências

**Positivas:**
- Cobre com folga o pior cenário de indisponibilidade de cliente já observado na operação (~2 horas), com margem considerável (~15 horas de janela total).
- Tabela `webhook_dead_letter` separada mantém a leitura da outbox principal limpa e serve como evidência para debug, sem exigir instrumentação adicional.
- Reprocessamento manual e restrito a `ADMIN` evita que qualquer operador recoloque eventos na fila sem critério, e fica registrado para auditoria (`[09:36] Sofia`).

**Negativas:**
- Um cliente fora do ar por mais de ~15 horas perde definitivamente a notificação automática daquele evento específico; a única recuperação é o replay manual, que depende de alguém identificar e agir sobre a entrada em dead-letter.
- Não existe hoje uma listagem administrativa de entradas em dead-letter — o endpoint de replay opera por `id` direto, o que pressupõe que a descoberta de falhas aconteça por outro canal (métricas ou log), documentado no [FDD](../FDD.md).
- O reprocessamento manual não escala para volumes grandes de falha simultânea (por exemplo, uma indisponibilidade generalizada de um cliente grande gerando centenas de entradas em dead-letter).
