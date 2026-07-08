# ADR-006 — Reuso dos padrões existentes do projeto

## Status

Aceito

## Contexto

O OMS já tem convenções estabelecidas para módulos de domínio (`src/modules/*` com controller, service, repository, routes e schemas), tratamento de erro (`AppError` e subclasses com código próprio), logging (Pino) e autorização por role (`requireRole`). Ao desenhar o módulo de webhooks, o time discutiu se deveria seguir essas convenções ou introduzir padrões novos específicos para a feature (`[09:27]-[09:30] Bruno/Diego/Larissa`).

## Decisão

O módulo de webhooks segue integralmente os padrões já existentes no projeto, sem introduzir peça nova de infraestrutura ou convenção:

- Estrutura de módulo idêntica aos demais domínios: `src/modules/webhooks` com controller, service, repository, routes e schemas (`[09:27]-[09:28] Bruno`).
- Classes de erro seguem o padrão de `AppError`/`InsufficientStockError`/`InvalidStatusTransitionError`, com códigos prefixados `WEBHOOK_` (por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`), reaproveitando as classes-base já existentes em [`src/shared/errors/http-errors.ts`](../../src/shared/errors/http-errors.ts) (`[09:28]-[09:29] Bruno/Larissa`).
- Logging usa a instância Pino global já configurada em [`src/shared/logger/index.ts`](../../src/shared/logger/index.ts), sem transporte ou sink novo (`[09:29] Bruno`).
- O middleware de erro central, [`src/middlewares/error.middleware.ts`](../../src/middlewares/error.middleware.ts), já trata `AppError`, `ZodError` e erros do Prisma, então os erros do módulo de webhooks fluem por ele sem exigir nenhuma alteração.
- O endpoint de replay administrativo reaproveita `requireRole('ADMIN')`, já exportado por [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts), sem modificação no middleware (`[09:36] Sofia`).
- O worker usa uma instância própria de `PrismaClient` (por ser processo separado), mas conectada à mesma `DATABASE_URL` e ao mesmo schema já usado pela API (`[09:29]-[09:30] Diego/Bruno`).

## Alternativas Consideradas

**Criar convenções específicas para o módulo de webhooks** (por exemplo, um formato de erro próprio, ou um logger dedicado para o worker). Não chegou a ser proposta com força durante a reunião, mas foi implicitamente descartada a cada ponto de decisão — em todos os casos discutidos (erros, logging, autorização), a resposta do time foi reaproveitar o que já existe em vez de desenhar algo novo (`[09:28]-[09:30]`).

## Consequências

**Positivas:**
- Qualquer desenvolvedor já familiarizado com os outros módulos do OMS consegue navegar o módulo de webhooks sem aprender convenções novas.
- Reduz superfície de manutenção: nenhum middleware, formato de erro ou mecanismo de log novo para testar e sustentar isoladamente.
- O middleware de erro central e o `requireRole` já testados e em produção continuam cobrindo o módulo novo sem mudança, reduzindo risco de regressão nos módulos existentes.

**Negativas:**
- Qualquer limitação já existente nesses padrões (por exemplo, ausência de redação automática de campos sensíveis por nome customizado no logger) se propaga também para o módulo de webhooks, exigindo atenção manual em code review — como o cuidado explícito de nunca logar a secret do webhook, apontado no [FDD](../FDD.md).
- Acoplar-se totalmente às convenções existentes significa que qualquer evolução futura dessas convenções (por exemplo, uma reformulação do padrão de erros) afeta também o módulo de webhooks, exigindo atualização em conjunto.
