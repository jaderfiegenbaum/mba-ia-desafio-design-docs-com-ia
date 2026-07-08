# ADR-005 — Garantia at-least-once com X-Event-Id

## Status

Aceito

## Contexto

Com retry automático em caso de falha ([ADR-003](./ADR-003-retry-backoff-dead-letter.md)), existe a possibilidade real de o cliente receber o mesmo evento mais de uma vez — por exemplo, se a entrega teve sucesso do lado do cliente, mas a resposta HTTP se perdeu no caminho de volta e o worker registrou como falha, disparando uma nova tentativa. O time precisava decidir se a garantia de entrega seria at-least-once (podendo duplicar) ou exactly-once (garantindo entrega única), e como o cliente diferenciaria eventos repetidos (`[09:24]-[09:25] Diego/Bruno`).

## Decisão

A garantia de entrega adotada é at-least-once: o cliente pode receber o mesmo evento mais de uma vez, e é responsabilidade dele deduplicar. Todo evento carrega um identificador único (`event_id`, UUID gerado no momento em que o evento entra na outbox), enviado no header `X-Event-Id` em toda chamada relacionada àquele evento — inclusive em retentativas (`[09:25] Diego`). O cliente usa esse identificador para reconhecer e descartar entregas duplicadas do lado dele.

Essa responsabilidade é documentada de forma explícita no portal do desenvolvedor voltado aos clientes (`[09:26] Marcos`).

## Alternativas Consideradas

**Garantia exactly-once.** Eliminaria a necessidade de deduplicação do lado do cliente, mas foi descartada por exigir coordenação distribuída entre os dois lados (confirmação transacional de recebimento, idempotência ponta a ponta), aumentando muito a complexidade de implementação e operação para um ganho que o mercado já resolve de outra forma — Stripe e GitHub, por exemplo, adotam o mesmo modelo de at-least-once com deduplicação por identificador do lado do consumidor (`[09:24]-[09:25] Diego`).

## Consequências

**Positivas:**
- Implementação simples do lado da plataforma: não é preciso rastrear confirmação de processamento do cliente, apenas garantir que o evento foi enviado.
- Alinhado ao padrão adotado por integrações de mercado já conhecidas dos clientes B2B (Stripe, GitHub), reduzindo a curva de adaptação deles.
- O mesmo `event_id` aparece em toda linha de log e de `webhook_deliveries` relacionada ao evento, permitindo reconstruir o histórico completo de tentativas sem infraestrutura de tracing dedicada.

**Negativas:**
- Joga a responsabilidade de deduplicação para o cliente (`[09:25] Sofia`); um cliente que não implementar isso corretamente pode processar a mesma mudança de status mais de uma vez do lado dele.
- Depende inteiramente da qualidade da documentação do contrato no portal do desenvolvedor para que os clientes entendam e implementem a deduplicação — não há nenhum mecanismo técnico que force isso.
- Não protege contra ataques de replay por si só; a mitigação nesse ponto vem do header `X-Timestamp` combinado com a assinatura HMAC (ver [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md)), mas a decisão de rejeitar timestamps antigos fica a critério da implementação do cliente.
