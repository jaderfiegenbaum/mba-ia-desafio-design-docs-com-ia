# PRD — Sistema de Webhooks de Notificação de Pedidos

**Autor:** Larissa (Tech Lead)
**Status:** Aprovado para desenvolvimento
**Data:** reunião de definição em 09:00, quinta-feira
**Stakeholders:** Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança)

## Resumo e contexto

Hoje o único jeito de um cliente saber que o status de um pedido mudou é ficar chamando `GET /orders` repetidamente. Isso já foi levantado formalmente por três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — como um problema real de integração, não uma preferência. Esta feature adiciona um sistema de webhooks de saída: sempre que o status de um pedido muda, o sistema notifica automaticamente os endpoints HTTP que cada cliente cadastrou, com garantias de entrega, assinatura criptográfica e histórico auditável.

O fluxo é só de saída. A plataforma envia; o cliente recebe. Nenhum webhook entra no sistema.

## Problema e motivação

Os clientes B2B fazem polling manual em `GET /orders` para detectar mudanças de status, o que deixa a integração deles lenta e cara em termos de chamadas desnecessárias. A Atlas Comercial já sinalizou que, se a notificação em tempo real não sair até o fim do trimestre, considera migrar para um concorrente. Esse é o gatilho comercial direto da feature.

## Público-alvo e cenários de uso

- **Clientes B2B com integração própria** (perfil principal: Atlas Comercial, MaxDistribuição, Nova Cargo): sistemas que consomem a API da plataforma via usuários autenticados e querem ser avisados assim que o pedido muda de status, sem precisar fazer polling.
- **Operadores internos**: usuários que cadastram e mantêm as configurações de webhook de um cliente através da própria API (não existe painel visual para o cliente final nesta fase).
- **Administradores (role ADMIN)**: responsáveis por reprocessar manualmente entregas que caíram definitivamente em falha (dead-letter).

Cenário principal: o pedido de um cliente muda de `PAID` para `PROCESSING`. O sistema publica o evento, o worker dispara uma chamada HTTP assinada para o endpoint cadastrado pelo cliente, e o cliente atualiza seu próprio sistema sem precisar consultar a API novamente.

## Objetivos e métricas de sucesso

- Eliminar a necessidade de polling manual de status pelos clientes B2B integrados.
- Entregar notificações dentro da janela definida como "tempo real" pelos próprios clientes: **latência de entrega abaixo de 10 segundos** desde a mudança de status até a tentativa de chamada ao endpoint do cliente, medida do commit da transação até o primeiro disparo HTTP.
- Reduzir o risco comercial de perda da conta Atlas Comercial, atendendo ao prazo combinado (fim de novembro, estimado em três sprints de desenvolvimento).
- Manter confiabilidade de entrega mesmo em cenários de indisponibilidade do cliente, cobrindo uma janela de retry de até ~15 horas antes de considerar falha permanente.

## Escopo

### Incluso

- Cadastro, edição, remoção e listagem de configurações de webhook por cliente (URL, secret, lista de status de interesse, estado ativo).
- Disparo automático de notificação HTTP quando o status de um pedido muda, integrado à transação existente de mudança de status.
- Garantia de entrega at-least-once, com retry automático em backoff exponencial e fila de dead-letter para falhas permanentes.
- Endpoint administrativo para reprocessar manualmente uma entrega em dead-letter.
- Consulta de histórico de entregas de um webhook (últimas 100, com status, payload, resposta e tempo de resposta).
- Autenticação HMAC-SHA256 do payload, com secret exclusiva por endpoint cadastrado e suporte a rotação de secret.
- Requisito de TLS (HTTPS obrigatório) na URL cadastrada.

### Fora de escopo

- **Notificação por e-mail em caso de falhas recorrentes de entrega.** Foi sugerido pelo Marcos e recusado pela Larissa nesta fase; fica como possível item de uma fase futura, após medir o impacto real da feature (`[09:37]` a `[09:38]`).
- **Painel visual (dashboard) para o cliente acompanhar seus webhooks.** Descartado explicitamente; é projeto separado do time de frontend, e por ora a interação é só via API (`[09:39]` a `[09:40]`).
- **Rate limiting de disparo por cliente.** Levantado pelo Diego como preocupação real (cliente com muitos pedidos mudando de status em pouco tempo recebendo várias chamadas seguidas), mas decidido observar o comportamento em produção antes de implementar qualquer controle (`[09:38]` a `[09:39]`).
- **Garantia de ordering global entre eventos.** A ordem só é garantida por `order_id` e apenas enquanto o sistema rodar com um único worker; escalar para múltiplos workers em paralelo é problema futuro e não faz parte desta entrega (`[09:12]` a `[09:14]`).
- **Arquivamento automático de eventos entregues na tabela de outbox.** Mencionado como necessário eventualmente (após ~30 dias), mas fora do escopo desta feature (`[09:08]`).

## Requisitos funcionais

1. Um cliente autenticado pode cadastrar um webhook informando URL de destino e a lista de status de pedido que deseja receber; a secret de assinatura é gerada pela plataforma e devolvida na criação (`[09:31]`).
2. Um cliente autenticado pode editar (PATCH) um webhook já cadastrado (`[09:33]`).
3. Um cliente autenticado pode remover (DELETE) um webhook cadastrado (`[09:33]`).
4. Um cliente autenticado pode listar (GET) os webhooks cadastrados para um customer (`[09:33]`).
5. O sistema só publica um evento de notificação se existir ao menos um webhook ativo do customer inscrito naquele status de destino; o filtro é aplicado no momento da inserção na fila de saída, não no momento do envio (`[09:33]` a `[09:34]`).
6. Um cliente autenticado pode consultar o histórico das últimas 100 entregas de um webhook, incluindo status de sucesso/falha, payload enviado, resposta recebida e tempo de resposta (`[09:34]`).
7. Um administrador (role ADMIN) pode reprocessar manualmente uma entrega que caiu em dead-letter, recolocando o evento na fila de envio como pendente (`[09:18]`, `[09:35]`, `[09:36]`).
8. Toda requisição de webhook enviada ao cliente é assinada com HMAC-SHA256 do corpo da mensagem, usando uma secret exclusiva daquele endpoint (`[09:20]`, `[09:21]`).
9. Um cliente pode solicitar rotação da própria secret pela API; a secret anterior continua válida por 24 horas em paralelo com a nova, para permitir migração sem downtime (`[09:21]`).
10. O sistema tenta reentregar um evento que falhou seguindo backoff exponencial de 5 tentativas (1min, 5min, 30min, 2h, 12h) antes de movê-lo definitivamente para dead-letter (`[09:15]` a `[09:17]`).
11. Cada evento de notificação carrega um identificador único (`event_id`) para que o cliente possa deduplicar recebimentos repetidos, já que a garantia de entrega é at-least-once (`[09:24]` a `[09:26]`).
12. O sistema recusa o cadastro de um webhook cuja URL não seja HTTPS (`[09:23]`).

## Requisitos não funcionais

- **Latência de entrega:** abaixo de 10 segundos entre a mudança de status e a tentativa de envio, atendida com folga por um worker com polling de 2 segundos (`[09:02]`, `[09:09]`, `[09:10]`).
- **Consistência:** a inserção do evento de notificação faz parte da mesma transação SQL que a mudança de status do pedido; se a transação principal falhar, o evento não pode existir (`[09:06]`, `[09:40]`, `[09:41]`).
- **Disponibilidade do worker:** o processo que envia as notificações roda separado da API, para não perder a capacidade de envio quando a API reinicia (`[09:11]`).
- **Limite de tamanho de payload:** eventos que ultrapassem 64KB são rejeitados no envio (erro), nunca truncados (`[09:23]`, `[09:24]`).
- **Timeout de chamada:** toda chamada HTTP ao endpoint do cliente tem timeout de 10 segundos; passado esse tempo, é tratada como falha e entra no fluxo de retry (`[09:42]`).
- **Auditoria:** toda ação de reprocessamento manual de dead-letter via endpoint admin deve ficar registrada, identificando quem executou (`[09:36]`).
- **Reuso de padrões existentes:** o módulo deve seguir a estrutura já usada pelos demais domínios do sistema (controller, service, repository, routes, schemas), reaproveitando `AppError`, o logger Pino e o middleware de erro centralizado já existentes na aplicação (`[09:27]` a `[09:30]`).

## Decisões e trade-offs principais

- **Outbox em MySQL em vez de fila dedicada (Redis Streams ou similar):** garante atomicidade com a transação de mudança de status sem exigir infraestrutura nova, ao custo de não ter um mecanismo nativo de notificação reativa do banco (`[09:06]` a `[09:07]`).
- **Worker em polling de 2 segundos em vez de trigger de banco reativo:** o MySQL não tem um equivalente ao `LISTEN/NOTIFY` do Postgres, então polling foi a opção mais simples que ainda atende ao requisito de latência (`[09:09]`).
- **5 tentativas de retry em vez de 3:** 3 tentativas matariam o evento rápido demais em uma indisponibilidade real do cliente (já houve caso de cliente fora do ar por 2 horas em manutenção planejada); 5 tentativas cobrem até ~15 horas (`[09:15]` a `[09:17]`).
- **At-least-once com deduplicação no cliente, em vez de exactly-once:** garantir exactly-once exigiria coordenação entre os dois lados e aumentaria muito a complexidade; é o mesmo padrão adotado por Stripe e GitHub (`[09:24]` a `[09:26]`).
- **Ordering só por `order_id`, não global:** aceito porque nenhum cliente pediu ordering global — eles só querem saber quando o próprio pedido muda (`[09:12]` a `[09:14]`).
- **Snapshot do payload no momento da inserção do evento, não renderização tardia:** evita que uma mudança posterior no pedido afete o conteúdo de um evento já gerado (`[09:51]` a `[09:52]`).

## Dependências

- Máquina de estados de status de pedido e a transação de `changeStatus` no serviço de pedidos existente, que precisa ser estendida para inserir o evento de notificação na mesma transação (`[09:40]` a `[09:41]`).
- Middleware de autenticação (`requireRole`) já existente, reaproveitado para restringir o endpoint de reprocessamento de dead-letter à role ADMIN (`[09:35]` a `[09:36]`).
- Padrão de classes de erro (`AppError` e subclasses) e o middleware de erro centralizado, que já tratam esse tipo de erro sem exigir mudança (`[09:28]` a `[09:29]`).
- Logger Pino já usado em toda a aplicação, sem necessidade de nova ferramenta de log (`[09:29]`).
- Revisão de segurança da Sofia sobre a geração de secret e a implementação de HMAC, obrigatória antes do deploy (`[09:46]`).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente indisponível por longos períodos causa acúmulo de eventos em retry ou perda de notificações | Média | Alto (cliente não recebe atualização de pedido) | Backoff exponencial de 5 tentativas cobrindo ~15h, dead-letter persistida com payload e motivo da falha, reprocessamento manual disponível via endpoint admin (`[09:15]` a `[09:18]`) |
| Vazamento de secret de assinatura em log do cliente, como já ocorreu antes | Média | Alto (permite falsificação de webhooks) | Secret exclusiva por endpoint (não global), suporte a rotação com grace period de 24h para migração sem downtime (`[09:21]` a `[09:22]`) |
| Chamada HTTP ao cliente lento trava a transação de mudança de status de outros pedidos | Baixa (mitigado por design) | Alto (afeta operação geral do sistema) | Notificação desacoplada da transação via padrão outbox; nenhum HTTP call acontece de forma síncrona dentro do `changeStatus` (`[09:03]` a `[09:06]`) |
| Volume alto de eventos em rajada (muitos pedidos mudando de status junto) satura o cliente com chamadas simultâneas | Baixa | Médio (cliente pode rejeitar ou limitar chamadas) | Fora de escopo nesta fase; será observado em produção antes de decidir sobre rate limiting de saída (`[09:38]` a `[09:39]`) |

## Critérios de aceitação

- Cliente consegue cadastrar, editar, remover e listar webhooks via API autenticada.
- Mudança de status de um pedido gera, quando aplicável, um evento de notificação na mesma transação da mudança — nunca fora dela.
- Evento é entregue ao endpoint do cliente em até 10 segundos na ausência de falhas.
- Falha de entrega aciona retry conforme o backoff definido, e após 5 tentativas o evento é movido para dead-letter.
- Toda chamada de webhook enviada carrega assinatura HMAC-SHA256 válida e identificador único de evento.
- Reprocessamento manual de dead-letter só é possível com role ADMIN e fica registrado para auditoria.
- Histórico de entregas de um webhook é consultável e mostra sucesso/falha, payload, resposta e tempo de resposta.
- Cadastro de URL sem HTTPS é recusado.

## Estratégia de testes e validação

- Testes automatizados de unidade para as regras de negócio do módulo de webhooks (filtragem por status, geração e validação de HMAC, cálculo de backoff).
- Teste de integração cobrindo o fluxo completo: mudança de status → inserção na outbox → processamento pelo worker → entrega ao endpoint de destino (mock).
- Teste de integração garantindo que uma falha na inserção do evento de notificação provoca rollback da mudança de status (a transação não pode ficar inconsistente).
- Teste dedicado ao fluxo de retry e dead-letter, validando o número de tentativas e o conteúdo persistido na fila de dead-letter.
- Revisão de segurança dedicada (Sofia), com pelo menos dois dias úteis reservados antes do deploy, focada especificamente em geração de secret e implementação de HMAC (`[09:46]`).
- Validação manual do fluxo de reprocessamento via endpoint admin antes de liberar para os clientes piloto (Atlas, MaxDistribuição, Nova Cargo).
