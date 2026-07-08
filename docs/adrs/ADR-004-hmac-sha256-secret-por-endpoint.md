# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

Esta feature é a primeira do sistema a expor dados de pedidos para fora da infraestrutura da empresa, via chamada HTTP para um endpoint controlado pelo cliente. O cliente precisa de um jeito de validar que a requisição realmente veio da plataforma e que o payload não foi adulterado no caminho (`[09:19] Sofia`).

Também era preciso decidir o escopo da credencial usada para essa validação: uma secret única e global da plataforma, ou uma por cliente/endpoint. O time já tinha um precedente de vazamento de secret em log de aplicação de um cliente no passado, o que pesou diretamente nessa decisão (`[09:22] Diego`).

## Decisão

Toda chamada de webhook ao cliente é assinada com HMAC-SHA256 sobre o corpo da requisição, enviada no header `X-Signature` (`[09:20] Sofia`). Cada endpoint de webhook cadastrado tem sua própria secret, gerada pela plataforma no momento da criação e nunca escolhida pelo cliente — não existe secret global compartilhada entre endpoints ou clientes (`[09:21] Sofia`).

A secret suporta rotação sob demanda pelo próprio cliente via API. Ao rotacionar, a secret anterior continua válida por 24 horas em paralelo com a nova, dando tempo para o cliente migrar seus sistemas sem downtime; depois desse prazo, a secret antiga deixa de ser aceita (`[09:21] Sofia`). Adicionalmente, URLs cadastradas precisam ser HTTPS — HTTP é recusado na validação (`[09:23] Sofia`).

## Alternativas Consideradas

**Secret global única da plataforma para todos os endpoints.** Mais simples de implementar e operar, mas descartada porque um único vazamento comprometeria a validação de assinatura de todos os clientes ao mesmo tempo. O precedente real de vazamento de secret em log de cliente tornou esse risco concreto, não hipotético (`[09:21]-[09:22] Sofia/Diego`).

**Rotação sem grace period (secret antiga invalidada imediatamente).** Mais simples e mais segura em teoria, mas descartada por gerar uma janela de interrupção real: o cliente precisaria trocar a secret do lado dele exatamente no mesmo instante da rotação, sem margem para propagação de configuração (`[09:21] Sofia`).

## Consequências

**Positivas:**
- Um vazamento de secret compromete apenas um endpoint específico, não a base inteira de clientes.
- Rotação com grace period de 24 horas permite ao cliente reagir a um vazamento suspeito sem downtime na integração.
- HTTPS obrigatório elimina a possibilidade de o payload (e a assinatura) trafegarem em texto plano na rede.

**Negativas:**
- A secret é armazenada em texto plano no banco (campo `secret`/`oldSecret` em `WebhookEndpoint`, ver [FDD](../FDD.md)), consistente com o padrão já usado para outros segredos no projeto, mas ainda assim um vetor de risco caso o banco seja comprometido — ponto que a revisão de segurança da Sofia deve validar explicitamente antes do deploy (`[09:46] Sofia`).
- A validação de HTTPS na URL cadastrada é apenas uma checagem de schema (Zod), não uma verificação ativa de certificado válido ou de que o endpoint realmente está servindo TLS corretamente configurado.
- Suportar duas secrets simultâneas (atual e anterior) por 24 horas adiciona uma ramificação a mais na lógica de verificação de assinatura, tanto na documentação quanto na implementação futura do lado do cliente.
