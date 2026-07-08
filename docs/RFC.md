# RFC — Sistema de Webhooks de Notificação de Pedidos

**Autor:** Diego (Eng. Sênior, time de Plataforma)
**Status:** Em revisão
**Data:** 2026-07-08
**Revisores:** Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, time de Pedidos), Sofia (Eng. Segurança)
**Documentos relacionados:** [PRD](./PRD.md) · [FDD](./FDD.md) · [ADRs](./adrs/)

## TL;DR

Vamos adicionar notificações de saída (webhooks) ao OMS. Quando o status de um pedido mudar, o sistema grava um evento numa tabela de outbox dentro da própria transação que já muda o status, e um worker separado, rodando em polling, lê essa tabela e entrega o evento via HTTP ao endpoint que o cliente cadastrou. Entrega é assinada com HMAC-SHA256, garantida at-least-once, com retry em backoff exponencial e dead-letter para falhas permanentes. Nada disso exige infraestrutura nova: tudo roda em cima do MySQL e da stack já existente no projeto.

Esta proposta nasce da reunião técnica do time (`TRANSCRICAO.md`, 09:00, ~55min) e está aberta para revisão antes de virar trabalho de sprint.

## Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para serem avisados quando o status de um pedido muda, em vez de ficarem consultando `GET /orders` de tempos em tempos (`[09:00] Marcos`). Esse polling deixa a integração deles lenta e cara. A Atlas colocou um prazo real em cima disso: se não sair até o fim do trimestre, considera migrar de plataforma (`[09:00] Marcos`).

O requisito de latência não é "instantâneo": os próprios clientes definiram que qualquer coisa abaixo de 10 segundos já conta como tempo real (`[09:02] Marcos`). Isso importa porque abre espaço para uma solução assíncrona em vez de uma chamada síncrona no meio da transação de pedido, que seria bem mais arriscada.

O ponto de partida técnico foi descartar o caminho mais óbvio primeiro: dispensar uma chamada HTTP síncrona dentro do `changeStatus`. A transação de mudança de status já faz update em `orders`, insere em `order_status_history` e mexe em `stock_quantity` (`[09:04] Bruno`). Colocar uma chamada de rede no meio disso significa que um cliente lento ou fora do ar passa a travar a mudança de status de pedidos de outros clientes, e não existe rollback razoável para "o webhook falhou" (`[09:04]-[09:06] Bruno/Larissa/Diego`). Essa restrição definiu o formato de toda a solução daqui para frente.

## Proposta técnica

A proposta tem quatro peças que se encaixam:

**1. Outbox transacional.** Toda vez que `OrderService.changeStatus` muda o status de um pedido, dentro da mesma transação SQL ele grava uma linha numa tabela `webhook_outbox`, com o payload do evento já montado. Se a transação principal der commit, o evento existe. Se der rollback, o evento nunca existiu. Não tem cenário de inconsistência entre "status mudou" e "evento foi gerado" (`[09:06] Diego`). Isso resolve o problema descrito acima sem introduzir nenhuma peça de infraestrutura nova: o outbox mora no MySQL que já está em produção.

**2. Worker dedicado, em polling.** Um processo Node separado (`src/worker.ts`), com seu próprio `PrismaClient`, roda em loop a cada 2 segundos, busca os eventos pendentes mais antigos e dispara a chamada HTTP para o endpoint do cliente (`[09:09] Diego`). É processo separado da API de propósito: se a API reiniciar, o worker continua entregando o que já está na fila (`[09:11] Diego`). Dois segundos de intervalo cobre com folga o requisito de 10 segundos combinado com os clientes.

**3. Entrega confiável.** Cada chamada carrega um `event_id` único (header `X-Event-Id`), uma assinatura HMAC-SHA256 do corpo (header `X-Signature`) e o id do endpoint cadastrado (`X-Webhook-Id`), além do timestamp do envio. A garantia de entrega é at-least-once: o cliente pode receber o mesmo evento mais de uma vez e é responsabilidade dele deduplicar pelo `event_id` (`[09:24]-[09:26] Diego`). Falhas de entrega entram num fluxo de retry com backoff exponencial (1min, 5min, 30min, 2h, 12h — 5 tentativas, cobrindo cerca de 15 horas de indisponibilidade do cliente) e, esgotadas as tentativas, o evento vai para uma tabela de dead-letter, com reprocessamento manual disponível via endpoint administrativo (`[09:15]-[09:18] Diego`).

**4. Configuração por cliente.** Cada cliente cadastra um ou mais endpoints de webhook via API autenticada, escolhendo a URL e quais status de pedido quer receber. A plataforma gera e devolve uma secret exclusiva daquele endpoint na criação, usada para assinar as chamadas, com suporte a rotação e grace period de 24 horas (`[09:21] Sofia`).

O módulo novo (`src/modules/webhooks`) segue a mesma estrutura de controller, service, repository, routes e schemas já usada pelos outros domínios do projeto, reaproveitando `AppError`, o logger Pino e o middleware de erro central sem modificação (`[09:27]-[09:30] Bruno/Larissa`). O detalhamento de contratos, modelagem de dados, matriz de erros e fluxo completo está no [FDD](./FDD.md); aqui o objetivo é deixar claro o desenho geral e por que ele foi escolhido.

## Alternativas consideradas

**Fila dedicada (Redis Streams ou similar) em vez de outbox no MySQL.** Foi a primeira alternativa discutida ao padrão outbox. Resolveria o mesmo problema de desacoplar o envio da transação, mas exigiria subir e operar uma peça de infraestrutura nova. Descartada porque o time é pequeno e o MySQL já provisionado resolve o mesmo problema sem esse custo operacional adicional (`[09:07] Diego/Larissa`).

**Trigger de banco para acordar o worker de forma reativa, em vez de polling fixo.** Levantada como forma de reduzir a latência mínima de entrega. Descartada porque o MySQL não tem um mecanismo equivalente ao `LISTEN/NOTIFY` do Postgres — um trigger consegue executar SQL, mas não consegue notificar um processo externo. Fazer isso funcionar exigiria um workaround (escrever em arquivo, chamar um endpoint) considerado mais complicado do que o ganho justificaria, especialmente porque polling de 2 segundos já atende o requisito de 10 segundos com folga (`[09:09] Diego`).

**Retry com 3 tentativas em vez de 5.** Mais agressivo em liberar o slot de dead-letter mais rápido, mas descartado porque mataria eventos de forma prematura em uma indisponibilidade real e não incomum: o time já tinha visto um cliente ficar fora do ar por até 2 horas em manutenção planejada, e 3 tentativas em um intervalo curto não cobririam nem essa janela (`[09:15]-[09:16] Diego/Bruno`).

**Garantia de exactly-once em vez de at-least-once.** Levantada implicitamente ao se discutir deduplicação. Descartada porque exigiria coordenação entre os dois lados (idempotência distribuída) e aumentaria muito a complexidade da entrega, para um ganho que o mercado já resolve de outro jeito — Stripe e GitHub, por exemplo, adotam at-least-once com deduplicação por identificador único do lado do consumidor, e essa foi a escolha replicada aqui (`[09:24]-[09:26] Diego`).

## Questões em aberto

- **Rate limiting de disparo por cliente.** Se um cliente tiver muitos pedidos mudando de status no mesmo minuto, o worker pode acabar disparando muitas chamadas simultâneas para o mesmo endpoint. Diego levantou o ponto, mas o time decidiu não implementar nada agora e observar o comportamento real em produção antes de desenhar um controle (`[09:38]-[09:39] Diego/Larissa`). Precisa de uma decisão futura sobre se isso vira throttling no worker, agrupamento de eventos, ou fica como está.
- **Notificação de fallback por e-mail quando o webhook do cliente falha repetidamente.** Marcos sugeriu avisar o cliente por e-mail se o webhook dele falhar várias vezes seguidas. Larissa recusou para esta fase, deixando como possível item de uma fase futura, condicionado a medir o impacto real da feature em produção primeiro (`[09:37]-[09:38] Marcos/Larissa`). Não há decisão sobre quando essa fase futura aconteceria nem qual seria o gatilho exato (quantidade de falhas, tempo em dead-letter).
- **Recuperação de eventos presos em `processing`.** Não foi discutido na reunião, mas é uma lacuna que aparece ao desenhar o worker: se o processo cair no meio de um ciclo, uma linha marcada como `processing` pode ficar presa indefinidamente, sem voltar a ser tentada nem cair em dead-letter. Fica registrado aqui como algo que precisa de uma decisão antes ou logo depois do primeiro deploy em produção.
- **Estratégia de escala para múltiplos workers.** O desenho atual assume um único worker, e a ordenação de eventos por `order_id` depende disso (`[09:12]-[09:13] Diego`). Se o volume crescer a ponto de precisar de mais de um worker, será necessário decidir entre particionamento por `order_id` ou lock pessimista — nenhuma das duas foi aprofundada, ficou registrado apenas como problema de uma fase futura.

## Impacto e riscos

**Impacto no código existente:** o ponto de maior atenção é `OrderService.changeStatus`, que passa a chamar uma função de publicação de evento (`publishWebhookEvent`) dentro da transação já existente, recebendo o cliente de transação (`tx`) em andamento — o mesmo padrão que operações como débito de estoque já usam hoje (`[09:41] Diego`). Fora esse ponto, a feature é aditiva: módulo novo, tabelas novas, entry point novo (`src/worker.ts`), sem alterar comportamento de nenhum fluxo hoje existente.

**Impacto operacional:** passa a existir um processo novo para monitorar em produção (o worker), com seu próprio ciclo de vida, métricas e logs. A ausência desse processo rodando significa que webhooks param de ser entregues silenciosamente até alguém notar — não há hoje, nesta proposta, um alerta automático para "worker parado".

**Risco de segurança:** o módulo introduz o primeiro ponto do sistema que expõe dados de pedidos para fora da infraestrutura da empresa. Daí a exigência de HMAC-SHA256, secret exclusiva por endpoint e HTTPS obrigatório (`[09:19]-[09:23] Sofia`). Por ser uma superfície nova, a proposta já reserva revisão de segurança dedicada da Sofia antes do deploy, com pelo menos dois dias úteis focados especificamente em geração de secret e implementação de HMAC (`[09:46] Sofia`).

**Risco de confiabilidade:** a garantia de entrega depende inteiramente do worker estar de pé e da tabela de outbox não acumular além do que o polling consegue processar. Ambos os pontos têm mitigação desenhada (dead-letter com reprocessamento manual, índice dedicado para leitura eficiente), mas nenhum dos dois é testado sob carga real antes do primeiro deploy — isso é um risco aceito conscientemente nesta fase, dado o prazo comercial com a Atlas.

**Estimativa de esforço:** três sprints, incluindo a modelagem de outbox e DLQ, o worker com retry, o CRUD de configuração e o histórico de entregas, a integração no `changeStatus`, e a revisão de segurança da Sofia ao final (`[09:45]-[09:46] Larissa`).

## Decisões relacionadas

As decisões arquiteturais fechadas nesta reunião estão registradas individualmente, com contexto e alternativas, nos ADRs abaixo:

- [ADR-001 — Padrão outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker dedicado em processo separado, com polling](./adrs/ADR-002-worker-processo-separado-polling.md)
- [ADR-003 — Retry com backoff exponencial e dead-letter](./adrs/ADR-003-retry-backoff-dead-letter.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — Garantia at-least-once com X-Event-Id](./adrs/ADR-005-garantia-at-least-once-com-id-de-evento.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./adrs/ADR-006-reuso-padroes-existentes.md)

Este RFC não repete o detalhamento de cada decisão nem o "como construir" ponto a ponto — isso está no [FDD](./FDD.md), que assume as decisões abaixo como fechadas e desce ao nível de contrato, modelagem de dados e matriz de erros.
