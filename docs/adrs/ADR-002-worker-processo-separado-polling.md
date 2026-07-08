# ADR-002 — Worker em processo separado, com polling

## Status

Aceito

## Contexto

Com o padrão outbox decidido ([ADR-001](./ADR-001-outbox-no-mysql.md)), era preciso definir quem lê a tabela `webhook_outbox` e dispara as chamadas HTTP, e onde esse processo roda. Se o processamento vivesse dentro da mesma instância da API, um reinício da API (deploy, crash, escala) interromperia a entrega de webhooks pendentes junto (`[09:11] Diego`).

Também era preciso decidir como o worker descobre que há trabalho novo. O MySQL não tem um mecanismo equivalente ao `LISTEN/NOTIFY` do PostgreSQL: um trigger de banco consegue executar SQL, mas não consegue notificar um processo externo diretamente (`[09:09] Diego`).

O requisito de latência combinado com os clientes é abaixo de 10 segundos entre a mudança de status e a tentativa de entrega (`[09:02] Marcos`).

## Decisão

O processamento da outbox roda em um processo Node dedicado (`src/worker.ts`), separado da API, com sua própria instância de `PrismaClient` conectada à mesma `DATABASE_URL` (`[09:11] Diego`, `[09:29]-[09:30] Bruno`). O worker opera em loop de polling, verificando a cada 2 segundos se há eventos pendentes prontos para tentativa (`[09:09]-[09:10] Diego`).

Um único worker roda por vez nesta fase. Isso implica ordenação de entrega apenas por `order_id` (via ordem de `created_at` na leitura), sem garantia de ordering global entre pedidos ou clientes diferentes — limitação aceita porque nenhum cliente pediu ordering global (`[09:12]-[09:14] Diego/Marcos`).

## Alternativas Consideradas

**Trigger de banco reativo.** Levantada para reduzir a latência mínima de entrega abaixo dos 2 segundos do polling. Descartada porque o MySQL não notifica processos externos a partir de um trigger — apenas executa SQL —, e qualquer workaround (escrever em arquivo, chamar um endpoint a partir do trigger) foi considerado mais complexo do que o ganho justificaria, dado que o polling de 2 segundos já atende com folga o requisito de 10 segundos (`[09:09] Diego`).

**Processamento dentro do mesmo processo da API.** Mais simples de implantar inicialmente, mas descartada porque acoplaria a disponibilidade de entrega de webhooks ao ciclo de vida da API — um reinício de API perderia a capacidade de processar a fila até subir de novo (`[09:11] Diego`).

## Consequências

**Positivas:**
- Entrega de webhooks continua funcionando mesmo durante deploy ou reinício da API.
- Latência de pior caso previsível e simples de raciocinar: no máximo um ciclo de polling (2 segundos) mais o tempo da própria chamada HTTP.
- Reaproveita o padrão de bootstrap e shutdown gracioso já usado em [`src/server.ts`](../../src/server.ts) como modelo direto para `src/worker.ts`.

**Negativas:**
- Novo processo para operar e monitorar em produção: se o worker parar sem que ninguém perceba, webhooks deixam de ser entregues silenciosamente (nenhum alerta automático foi desenhado nesta fase — registrado como risco no [RFC](../RFC.md)).
- Escalar para múltiplos workers no futuro exigirá particionamento por `order_id` ou lock pessimista para não quebrar a ordenação implícita atual; isso é problema explicitamente adiado (`[09:13] Diego`).
- Latência mínima de entrega tem um piso de 2 segundos por construção, mesmo em cenário ideal sem backlog.
