# ADR-001 — Padrão outbox no MySQL

## Status

Aceito

## Contexto

O sistema precisa notificar clientes externos quando o status de um pedido muda, sem colocar uma chamada HTTP dentro da transação síncrona de `changeStatus`. Essa transação já faz update em `orders`, insere em `order_status_history` e ajusta `stock_quantity`; acrescentar uma chamada de rede no meio dela significa que um cliente lento ou fora do ar passa a travar a mudança de status de pedidos de outros clientes, sem um caminho razoável de rollback (`[09:04]-[09:06] Bruno/Larissa/Diego`, `TRANSCRICAO.md`).

O time precisava de um mecanismo que garantisse que a notificação só existisse se a mudança de status também existisse — e vice-versa — sem introduzir uma peça de infraestrutura nova, já que o time é pequeno (`[09:07] Diego`).

## Decisão

Adotar o padrão outbox transacional em cima do MySQL já provisionado. Dentro da mesma transação SQL que muda o status do pedido, o sistema insere uma linha numa tabela `webhook_outbox` com o evento já montado. Um worker separado lê essa tabela de forma assíncrona e dispara as chamadas HTTP. Se a transação principal der commit, o evento existe; se der rollback, o evento nunca existiu (`[09:06] Diego`).

A tabela tem índice composto em `status` e `nextAttemptAt`/`created_at`, para que o worker leia apenas os eventos pendentes prontos para tentativa, em lotes pequenos, sem degradar performance à medida que a tabela cresce (`[09:07]-[09:08] Diego`).

## Alternativas Consideradas

**Fila dedicada (Redis Streams ou equivalente).** Resolveria o mesmo problema de desacoplamento, mas exigiria provisionar, operar e monitorar uma peça de infraestrutura nova. Descartada por ser overengineering para o tamanho do time e da carga atual (`[09:07] Diego/Larissa`).

**Chamada HTTP síncrona dentro do `changeStatus`.** Foi a primeira ideia colocada na mesa e descartada de imediato: acoplaria a disponibilidade de clientes externos à disponibilidade da própria mudança de status, sem caminho de rollback aceitável (`[09:04] Bruno`).

## Consequências

**Positivas:**
- Nenhuma infraestrutura nova é necessária; o outbox convive no mesmo banco e no mesmo ciclo de deploy da aplicação.
- Atomicidade garantida por construção: a consistência entre "status mudou" e "evento existe" vem de graça da transação SQL, sem lógica extra de compensação.
- Reaproveita o padrão de transação já usado em `OrderService` (o mesmo `tx` que hoje serve `debitStock`/`replenishStock` passa a servir também `publishWebhookEvent`), conforme detalhado em [`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts).

**Negativas:**
- A tabela de outbox cresce continuamente e precisa de rotina de arquivamento (explicitamente fora do escopo desta feature, conforme `[09:08] Diego`), ou eventualmente pode pesar em espaço e em índice.
- Não há mecanismo reativo do MySQL para acordar o worker quando um evento novo é inserido — essa lacuna motiva a decisão registrada em [ADR-002](./ADR-002-worker-processo-separado-polling.md).
- Se o volume de eventos crescer muito além do estimado hoje, pode ser necessário revisar o tamanho de lote e a frequência de polling antes que a arquitetura outbox comece a introduzir latência perceptível.
