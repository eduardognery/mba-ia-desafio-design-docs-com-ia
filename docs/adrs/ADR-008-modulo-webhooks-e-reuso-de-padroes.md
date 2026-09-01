# ADR-008: Módulo de webhooks e reuso dos padrões existentes

## Status

Aceito.

## Contexto

A aplicação organiza cada domínio em `src/modules`, com controller, service, repository, rotas e schemas, como ocorre em `src/modules/orders`. A composição de dependências está centralizada em `src/app.ts`; os erros de domínio usam `AppError`; o logger compartilhado é Pino; e `src/middlewares/auth.middleware.ts` já fornece `authenticate` e `requireRole`.

A nova feature terá CRUD de configurações, consulta de entregas e replay administrativo. Criar convenções próprias para ela produziria uma exceção desnecessária à arquitetura atual.

## Decisão

Implementar a feature como `src/modules/webhooks`, com controller, service, repository, routes e schemas Zod. Reutilizar `AppError`, middleware central de erros, Pino e códigos de erro com prefixo `WEBHOOK_`.

As operações de CRUD serão autenticadas normalmente. O replay de dead letter exigirá `requireRole('ADMIN')`, reutilizando a autorização existente; a execução será logada para auditoria.

## Alternativas Consideradas

- Concentrar endpoints e processamento no módulo de pedidos: rejeitado porque configuração, entrega e segurança de webhook constituem uma responsabilidade própria.
- Criar um serviço separado ou novas bibliotecas de erros e logs: rejeitado porque aumenta superfície operacional e fragmenta padrões já consolidados.
- Permitir replay a qualquer usuário autenticado: rejeitado porque altera deliberadamente a fila de entrega e exige privilégio administrativo.

## Consequências

**Benefícios:** mantém previsibilidade para manutenção e testes; reduz código duplicado; aplica tratamento de erros, observabilidade e autorização já validados no projeto.

**Desvantagens:** a composição em `src/app.ts` e o roteamento precisarão conhecer mais um módulo; o worker terá dependências do domínio de webhooks mesmo sendo processo separado.

**Trade-offs:** prioriza consistência com a base de código atual sobre uma abstração independente prematura, mantendo uma separação clara entre API e processamento assíncrono.
