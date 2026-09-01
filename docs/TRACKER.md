# Tracker de Rastreabilidade — Webhooks de notificação de pedidos

Este tracker relaciona os requisitos, decisões, restrições e trade-offs registrados na documentação à evidência primária na reunião ou à base efetivamente existente no código. Itens de webhooks são planejados/documentados; não representam implementação já presente no código.

| ID | Documento | Tipo | Conteúdo | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B usam polling em `GET /orders`, tornando a integração lenta e cara. | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Entregar mudanças de status com latência menor que 10 segundos. | TRANSCRICAO | [09:02] Marcos |
| PRD-SCP-01 | docs/PRD.md | Restrição de escopo | A feature é somente de webhooks outbound, da plataforma para o cliente. | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook autenticado com URL, cliente, status de interesse e secret gerada pela plataforma. | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar, remover e listar webhooks de um cliente; `customer_id` vem de body ou path, não do JWT. | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Manter URL, secret, `customer_id`, estado ativo e filtro de status por endpoint. | TRANSCRICAO | [09:21] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Gerar eventos apenas para endpoints ativos inscritos no novo status do pedido. | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Não inserir na outbox quando nenhum webhook do cliente escuta o status. | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Inserir o evento na outbox na mesma transação da mudança de status e fazer rollback se a inserção falhar. | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Persistir snapshot do payload na criação do evento. | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Worker separado busca pendências em polling de 2 segundos e as entrega por HTTP. | TRANSCRICAO | [09:09] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Entregar com semântica *at-least-once* e UUID em `X-Event-Id` para deduplicação do cliente. | TRANSCRICAO | [09:25] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Enviar payload `order.status_changed` com dados básicos do pedido, sem itens. | TRANSCRICAO | [09:43] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Enviar `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`. | TRANSCRICAO | [09:44] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Retentar até cinco vezes nos intervalos 1m, 5m, 30m, 2h e 12h. | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Mover a falha definitiva para `webhook_dead_letter` com payload, motivo e timestamp. | TRANSCRICAO | [09:18] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Permitir replay em `POST /admin/webhooks/dead-letter/:id/replay`, restrito a `ADMIN` e auditado. | TRANSCRICAO | [09:36] Larissa |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Expor as últimas 100 entregas, com resultado, payload, resposta e duração. | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Assinar o corpo com HMAC-SHA256 usando secret exclusiva por endpoint. | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-17 | docs/PRD.md | Requisito Funcional | Rotacionar secret e manter a anterior válida por 24 horas. | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Aceitar apenas URL HTTPS. | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Rejeitar payload acima de 64 KB, sem truncamento. | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Tratar como falha a chamada HTTP que exceder 10 segundos. | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Restrição | Indexar a outbox por status e `created_at` e processar pequenos lotes. | TRANSCRICAO | [09:08] Diego |
| PRD-NFR-05 | docs/PRD.md | Restrição | Garantir ordenação por pedido somente com worker único; não há ordenação global. | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | E-mail de aviso por falhas consecutivas fica para fase futura. | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Rate limiting por cliente será apenas observado nesta fase. | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Dashboard visual é projeto separado; esta fase oferece somente endpoints. | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Arquivamento de entregas após cerca de 30 dias não integra a feature. | TRANSCRICAO | [09:08] Diego |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Rejeitar HTTP síncrono porque endpoint lento bloquearia a transação crítica de pedidos. | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Usar outbox no MySQL em vez de Redis Streams para evitar infraestrutura adicional. | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Rejeitar retry indefinido para impedir eventos pendentes para sempre. | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Rejeitar *exactly-once* pela coordenação complexa entre plataforma e consumidores. | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Escala com múltiplos workers exigirá particionamento por `order_id` ou locking no futuro. | TRANSCRICAO | [09:13] Diego |
| FDD-DATA-01 | docs/FDD.md | Decisão | Usar UUID como identificador dos registros da outbox, conforme padrão do projeto. | TRANSCRICAO | [09:51] Larissa |
| FDD-FLOW-01 | docs/FDD.md | Restrição | A API confirma a alteração normalmente após o commit e não faz HTTP ao cliente no fluxo de status. | TRANSCRICAO | [09:04] Bruno |
| FDD-CONTRACT-01 | docs/FDD.md | Contrato | O consumidor pode consultar `GET /orders/:id` quando precisar dos itens do pedido. | TRANSCRICAO | [09:43] Diego |
| FDD-SEC-01 | docs/FDD.md | Segurança | `X-Timestamp` permite que o consumidor aplique proteção contra replay. | TRANSCRICAO | [09:44] Diego |
| FDD-DEP-01 | docs/FDD.md | Dependência | Reservar pelo menos dois dias úteis para revisão de HMAC e geração de secret antes do deploy. | TRANSCRICAO | [09:46] Sofia |
| ADR-001 | docs/adrs/ADR-001-outbox-atomica-no-mysql.md | Decisão | Usar outbox atômica no MySQL na transação de mudança de status. | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Executar worker em processo separado e consultar pendências a cada 2 segundos. | TRANSCRICAO | [09:11] Diego |
| ADR-003 | docs/adrs/ADR-003-ordem-por-pedido-com-worker-unico.md | Decisão | Iniciar com worker único para preservar ordem por `order_id`. | TRANSCRICAO | [09:12] Diego |
| ADR-004 | docs/adrs/ADR-004-retry-limitado-e-dead-letter.md | Decisão | Limitar retries, persistir DLQ separada e permitir replay administrativo. | TRANSCRICAO | [09:17] Larissa |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md | Decisão | Oferecer *at-least-once* com `X-Event-Id` estável para deduplicação. | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-assinatura-hmac-e-rotacao-por-endpoint.md | Decisão | Usar HMAC-SHA256, secret por endpoint e rotação com janela de 24 horas. | TRANSCRICAO | [09:22] Sofia |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-criacao-do-evento.md | Decisão | Salvar payload renderizado na inserção para preservar o estado da transição. | TRANSCRICAO | [09:52] Larissa |
| ADR-008 | docs/adrs/ADR-008-modulo-webhooks-e-reuso-de-padroes.md | Decisão | Criar módulo de webhooks e reutilizar padrões consolidados da aplicação. | TRANSCRICAO | [09:30] Larissa |
| CODE-ORD-01 | docs/PRD.md | Evidência de Código | `changeStatus` já executa atualização do pedido e histórico dentro de `prisma.$transaction`. | CODIGO | src/modules/orders/order.service.ts |
| CODE-ORD-02 | docs/RFC.md | Evidência de Código | A mudança de status já preserva regras de transição e ajustes de estoque no serviço de pedidos. | CODIGO | src/modules/orders/order.service.ts |
| CODE-DB-01 | docs/adrs/ADR-001-outbox-atomica-no-mysql.md | Evidência de Código | O schema Prisma atual usa MySQL e IDs UUID nas entidades de domínio. | CODIGO | prisma/schema.prisma |
| CODE-AUTH-01 | docs/FDD.md | Evidência de Código | A autenticação JWT expõe as roles `ADMIN` e `OPERATOR` e disponibiliza `requireRole`. | CODIGO | src/middlewares/auth.middleware.ts |
| CODE-ROUTE-01 | docs/FDD.md | Evidência de Código | As rotas de pedidos existentes são autenticadas e incluem `PATCH /:id/status`. | CODIGO | src/modules/orders/order.routes.ts |
| CODE-MOD-01 | docs/adrs/ADR-008-modulo-webhooks-e-reuso-de-padroes.md | Evidência de Código | O domínio de pedidos segue a estrutura controller, service, repository, routes e schemas sob `src/modules`. | CODIGO | src/modules/orders |
| CODE-APP-01 | docs/FDD.md | Evidência de Código | A composição de controllers e o prefixo `/api/v1` estão centralizados na aplicação. | CODIGO | src/app.ts |
| CODE-LOG-01 | docs/adrs/ADR-008-modulo-webhooks-e-reuso-de-padroes.md | Evidência de Código | O logger compartilhado usa Pino e já redige credenciais sensíveis configuradas. | CODIGO | src/shared/logger/index.ts |
| CODE-ERR-01 | docs/adrs/ADR-008-modulo-webhooks-e-reuso-de-padroes.md | Evidência de Código | A aplicação possui `AppError` e middleware centralizado de erros para reutilização pelo novo módulo. | CODIGO | src/shared/errors/app-error.ts |

## Referências entre documentos

| ID | Documento | Tipo | Conteúdo | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| REF-RFC-ADR-01 | docs/RFC.md | Referência | A seção “Decisões relacionadas” consolida as decisões da reunião em ADR-001 a ADR-008. | TRANSCRICAO | [09:48] Larissa |
| REF-FDD-CODE-01 | docs/FDD.md | Referência | A seção de integração liga a feature aos padrões e componentes já existentes. | TRANSCRICAO | [09:30] Larissa |
| REF-PRD-CODE-01 | docs/PRD.md | Dependência | O PRD consolida as dependências de transação de pedidos, banco, autenticação e worker. | TRANSCRICAO | [09:40] Bruno |
