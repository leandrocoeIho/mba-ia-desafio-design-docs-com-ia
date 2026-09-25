# ADR-006: Reuso dos padrões arquiteturais já existentes no projeto

## Status

Aceito

## Contexto

O módulo de webhooks é uma feature nova dentro de uma base de código já estabelecida, com convenções consistentes entre os módulos de `users`, `customers`, `products` e `orders` (o módulo `auth` segue a mesma linha, mas sem `repository` próprio — reaproveita o `UserRepository`). Bruno abriu essa discussão diretamente: "A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual" ([09:27] Bruno).

Era preciso decidir, para cada preocupação transversal (estrutura de módulo, erros, logging, tratamento de erro HTTP, geração de IDs), se o módulo de webhooks introduziria algo novo ou reaproveitaria o que já existe.

## Decisão

O módulo de webhooks segue **estritamente os mesmos padrões já em uso no restante do projeto**, sem introduzir novas convenções:

- **Estrutura de módulo:** `src/modules/webhooks/` (novo, a ser criado) com `controller`, `service`, `repository`, `routes` e `schemas`, no mesmo formato de [src/modules/orders/](../../src/modules/orders/) (`order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts`) ([09:27]-[09:28] Bruno/Diego).
- **Erros:** subclasses de [`AppError`](../../src/shared/errors/app-error.ts), seguindo o padrão de [`src/shared/errors/http-errors.ts`](../../src/shared/errors/http-errors.ts) (ex.: `InsufficientStockError`, `InvalidStatusTransitionError`), com códigos de erro prefixados por **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`) ([09:28]-[09:29] Bruno/Larissa). Bruno citou dois exemplos nesse momento da reunião que não sobreviveram a decisões posteriores ([09:28]): `WEBHOOK_SECRET_REQUIRED`, inaplicável porque a secret é sempre gerada pelo próprio OMS e nunca enviada pelo cliente ([09:31] Marcos); e `WEBHOOK_INVALID_URL`, que o [FDD](../FDD.md) decide tratar como `VALIDATION_ERROR` genérico, já que Sofia descreveu essa checagem como "só uma validação no schema Zod" ([09:23] Sofia). Ambos os códigos foram deliberadamente excluídos da matriz final — ver [FDD](../FDD.md), seção 6.
- **Validação de entrada:** segue o padrão de schemas Zod já usado nos demais módulos, validados pelo middleware [`src/middlewares/validate.middleware.ts`](../../src/middlewares/validate.middleware.ts) ([09:30] Larissa: "padrão de schemas Zod, padrão de códigos de erro").
- **Tratamento de erro HTTP:** o middleware centralizado [`src/middlewares/error.middleware.ts`](../../src/middlewares/error.middleware.ts) já trata qualquer instância de `AppError` e `ZodError` sem alteração; para erros do Prisma, hoje ele só trata especificamente `P2002` (violação de unicidade → 409) e `P2025` (registro não encontrado → 404) — qualquer outro erro do Prisma cai no handler genérico de 500 ([09:29] Bruno).
- **Logging:** reaproveita o logger Pino centralizado em [`src/shared/logger/index.ts`](../../src/shared/logger/index.ts); nenhuma nova biblioteca ou padrão de log é introduzido ([09:29] Bruno: "Não vamos botar nada novo"). Vale notar que a lista de redação de campos sensíveis desse logger (`redactPaths`) hoje cobre `password`, `passwordHash`, `token` e `accessToken`, mas não um campo `secret` — algo a revisar quando a secret do webhook passar a trafegar em logs de aplicação.
- **Identificadores:** a tabela `webhook_outbox` usa UUID como chave primária, seguindo o padrão já usado nas tabelas de entidade do projeto (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` usam `String @id @default(uuid())`) — com a exceção da tabela auxiliar `OrderNumberSequence`, que usa um inteiro fixo (`id Int @id @default(1)`) por ser um contador singleton, não uma entidade. Diego perguntou diretamente sobre isso e Larissa confirmou: "UUID, segue o padrão do resto do projeto. Tudo é uuid" ([09:51] Diego/Larissa).
- **Autorização:** o endpoint administrativo de replay de DLQ reaproveita a função [`requireRole`](../../src/middlewares/auth.middleware.ts) já existente, exigindo role `ADMIN` ([09:36] Larissa: "a gente reaproveita o requireRole que já existe").
- **Infraestrutura de dados:** o worker usa o mesmo `DATABASE_URL` e o mesmo Prisma Client gerado a partir de [`prisma/schema.prisma`](../../prisma/schema.prisma), apenas instanciado em um processo separado (ver [ADR-002](ADR-002-worker-processo-separado-em-polling.md)).

## Alternativas Consideradas

1. **ID auto-incremental para a tabela `webhook_outbox`, em vez de UUID.** Levantada por Diego ao final da reunião — "prefere id auto incremental ou UUID?" — e descartada por Larissa em favor de manter o padrão UUID do restante do projeto ([09:51] Diego/Larissa).
2. **Introduzir um novo logger ou biblioteca de log dedicada ao módulo de webhooks.** Descartada explicitamente por Bruno ao propor a estrutura do módulo: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" ([09:29] Bruno).

## Consequências

**Positivas:**
- Qualquer desenvolvedor já familiarizado com os módulos existentes (orders, products, customers) entende a estrutura do módulo de webhooks sem curva de aprendizado adicional.
- Zero mudança necessária no middleware de erro ou na autenticação/autorização — reduz superfície de regressão em código compartilhado. O logger é reaproveitado sem mudança estrutural, com uma única ressalva pontual (ver Consequências Negativas).
- Consistência de auditoria e observabilidade: os mesmos mecanismos de log e tratamento de erro já usados em produção cobrem o novo módulo desde o primeiro dia.

**Negativas:**
- Qualquer limitação já existente nos padrões atuais é herdada pelo módulo de webhooks sem oportunidade de melhoria isolada — por exemplo, a lista de redação de logs do Pino (`redactPaths` em `src/shared/logger/index.ts`) não cobre um campo `secret`, então logar acidentalmente o objeto de configuração de um webhook vazaria a secret em texto claro até que essa lista seja atualizada.
- Decisões de estrutura de módulo ficam acopladas à evolução dos padrões do restante do projeto — mudar a convenção de módulos no futuro exigiria alterar `webhooks` junto com os demais.
