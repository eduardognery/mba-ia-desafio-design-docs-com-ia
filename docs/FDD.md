# FDD — Webhooks de notificação de status de pedidos

## Contexto e motivação técnica

Clientes B2B consultam repetidamente a API de pedidos para identificar alterações de status. O módulo de webhooks disponibilizará notificações *outbound* para que cada cliente receba, em até 10 segundos, os eventos de seus pedidos.

O envio não pode acontecer de forma síncrona em `OrderService.changeStatus`: esse método já altera pedido, histórico e estoque em uma transação. Uma chamada HTTP externa acrescentaria latência e faria a mudança de estado depender da disponibilidade do cliente. A solução adotará **Transactional Outbox** no MySQL: a mudança de pedido e a criação do evento serão atômicas; um processo worker separado fará a entrega assíncrona.

As entregas têm semântica *at-least-once*. Portanto, o consumidor deve deduplicar o cabeçalho `X-Event-Id` (também presente no corpo). Cada chamada será autenticada com HMAC-SHA256 do corpo bruto e segredo exclusivo por endpoint.

## Objetivos técnicos

- Criar, alterar, excluir e listar configurações de webhook por cliente autenticado.
- Persistir um snapshot imutável de `order.status_changed` na mesma transação da mudança de status.
- Entregar eventos por worker independente, com polling de 2 segundos e ordenação por pedido enquanto houver uma única instância do worker.
- Aplicar timeout de 10 s, até cinco tentativas e backoff de 1 min, 5 min, 30 min, 2 h e 12 h.
- Preservar falhas definitivas em DLQ e permitir replay auditável apenas a `ADMIN`.
- Expor histórico de entregas e manter rastreabilidade por evento, endpoint, pedido e correlação de requisição.

## Escopo e exclusões

**Incluído:** CRUD de configuração; filtro de status; rotação de segredo; outbox; worker; assinaturas; histórico de entregas; DLQ e replay administrativo.

**Excluído nesta fase:** webhooks de entrada; dashboard web; e-mail de alerta após falhas; rate limit por destino; broker externo (Redis/Kafka); retenção/arquivamento de entregas concluídas além da política futura de 30 dias; garantia de ordenação global; e garantia *exactly-once*.

## Modelo de dados e estados

Adicionar ao Prisma quatro modelos (UUIDs `Char(36)`, nomes físicos abaixo) e a migração correspondente:

| Tabela | Campos essenciais | Índices e regras |
|---|---|---|
| `webhooks` | `id`, `customerId`, `url`, `secretCurrent`, `secretPrevious?`, `previousSecretExpiresAt?`, `subscribedStatuses` (JSON), `active`, timestamps | índice `customerId`; `url` HTTPS; segredo nunca retornado novamente, exceto na criação/rotação |
| `webhook_outbox` | `id` (event id), `webhookId`, `orderId`, `eventType`, `payload` (JSON), `status`, `attemptCount`, `nextAttemptAt`, `processingStartedAt?`, `lastError?`, timestamps | índice composto `(status, nextAttemptAt, createdAt)` e `(webhookId, createdAt)`; payload ≤ 65.536 bytes serializados UTF-8 |
| `webhook_deliveries` | `id`, `eventId`, `webhookId`, `attemptNumber`, `requestTimestamp`, `responseStatus?`, `responseBody?`, `durationMs?`, `outcome`, `errorCode?`, timestamps | índice `(webhookId, createdAt)`; armazenar resposta limitada/redigida para diagnóstico |
| `webhook_dead_letter` | `id`, `originalEventId`, `webhookId`, `orderId`, `eventType`, `payload`, `attemptCount`, `finalError`, `failedAt`, `replayedAt?`, `replayedById?` | índice `(webhookId, failedAt)`; conservar evidência e auditoria de replay |

Estados da outbox: `PENDING`, `PROCESSING`, `DELIVERED`, `FAILED`. `FAILED` só é transitório durante a movimentação transacional para DLQ; o registro operacional é removido da outbox após inserir a DLQ. Uma instância única seleciona os registros elegíveis por `nextAttemptAt ASC, createdAt ASC`. Ao recuperar um registro `PROCESSING` cujo lease expirou (por exemplo, 30 s), devolvê-lo a `PENDING` para tolerar queda do worker.

## Fluxos detalhados

### Criação do evento na outbox

1. `PATCH /api/v1/orders/:id/status` autentica o usuário, valida a transição e inicia a transação já existente.
2. Após atualizar `orders`, gravar `order_status_history` e aplicar o estoque, buscar webhooks ativos de `order.customerId` cujo `subscribedStatuses` contenha o novo status.
3. Para cada destino elegível, gerar UUID `event_id` e serializar o snapshot abaixo. Validar tamanho máximo de 64 KiB antes de persistir.
4. Inserir uma linha `webhook_outbox` por endpoint, com `PENDING`, `attemptCount=0` e `nextAttemptAt=now()` usando o mesmo `tx` Prisma.
5. Qualquer falha na consulta, serialização, validação ou insert da outbox faz rollback de pedido, histórico, estoque e eventos. Sem endpoint inscrito, não criar linha de outbox.
6. Após o commit, a API responde normalmente; ela não realiza HTTP para o cliente.

Payload persistido e reenviado sem mutação:

```json
{
  "event_id": "a82b4ba8-52a2-4d8d-8cb3-2fbdd7f4667e",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-09T12:00:00.000Z",
  "order_id": "f48e3a4a-bf7e-4ea7-aa56-132c368a4486",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "4ddb9174-cc57-44d1-81e9-d0c62d759631",
  "total_cents": 15990
}
```

### Processamento

1. `src/worker.ts` inicializa um `PrismaClient` próprio e inicia o processor em loop de 2 s.
2. Em cada ciclo, o repositório busca lote pequeno (sugestão: 20) de `PENDING` com `nextAttemptAt <= now`, ordenado por `createdAt`. A aquisição deve ser atômica e marcar cada item como `PROCESSING`, para não haver dupla posse caso a topologia mude.
3. Para cada item, gerar `X-Timestamp` ISO 8601 no envio, calcular `HMAC-SHA256(secret, rawPayload)` e fazer `POST` à URL com timeout total de 10 s.
4. Enviar os headers `Content-Type: application/json`, `User-Agent: order-management-webhooks/1.0`, `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`.
5. Resposta HTTP 2xx significa sucesso: inserir `webhook_deliveries` com `DELIVERED` e atualizar a outbox para `DELIVERED` na mesma transação. Respostas 3xx não serão seguidas automaticamente; qualquer não-2xx é falha.
6. Registrar cada tentativa, incluindo duração, status HTTP e erro classificado. Nunca registrar o segredo, assinatura ou corpo de resposta sem limites/redação.

Enquanto houver apenas um worker, o processamento por `createdAt` preserva a ordem por `order_id` na fila. Essa ordem não é garantia global e pode ser relaxada quando houver múltiplos workers; escalar exigirá particionamento determinístico por `orderId` ou lock por pedido.

### Retry

Considerar retentável: timeout, erro de rede/DNS/TLS transitório, HTTP 408, 425, 429 e 5xx. Considerar não retentável: URL inválida já bloqueada no cadastro, HTTP 400/401/403/404/410 e erro de payload. Para 429, usar `Retry-After` se válido e maior que o backoff calculado, limitado ao intervalo máximo de 12 h.

Após uma falha retentável, gravar a entrega e incrementar `attemptCount`. Se ainda houver tentativa disponível, retornar a outbox para `PENDING` e definir `nextAttemptAt` conforme a tentativa falha: 1ª→1 min, 2ª→5 min, 3ª→30 min, 4ª→2 h, 5ª→12 h. A sexta falha (ou uma falha não retentável) encerra o ciclo e segue para DLQ. Acrescentar jitter positivo de até 10% para reduzir sincronização de retries.

### DLQ

Na falha terminal, uma transação deve inserir em `webhook_dead_letter` o snapshot, total de tentativas e causa final, registrar a última entrega e remover/marcar definitivamente o evento da outbox. O operador consulta a evidência pelas tabelas de entrega e DLQ.

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` exige JWT de role `ADMIN`. Ele deve: bloquear/reler a DLQ; rejeitar replay já efetuado; validar que o webhook ainda existe e está ativo; criar novo evento com **novo** `event_id` e o mesmo payload de negócio, atualizando o `event_id` também no snapshot; inserir na outbox como `PENDING`; marcar `replayedAt` e `replayedById`. Logar usuário, DLQ e novo evento para auditoria. O novo id evita colisão de deduplicação com uma tentativa passada.

## Contratos públicos

Base URL: `/api/v1`. Todos os contratos abaixo usam JSON e `Authorization: Bearer <JWT>`. Erros seguem `{ "error": { "code", "message", "details?" } }`.

### 1. Criar configuração

`POST /webhooks`

Headers: `Authorization: Bearer <JWT>`, `Content-Type: application/json`, `Accept: application/json`.

Request:

```json
{
  "customerId": "4ddb9174-cc57-44d1-81e9-d0c62d759631",
  "url": "https://cliente.example.com/hooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201` (o único momento em que o segredo é retornado):

```json
{
  "id": "64dbb72c-2432-4f17-b86b-c5f0bc1e2ba8",
  "customerId": "4ddb9174-cc57-44d1-81e9-d0c62d759631",
  "url": "https://cliente.example.com/hooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_...",
  "createdAt": "2026-08-09T12:00:00.000Z"
}
```

Semântica: cria endpoint ativo e gera segredo criptograficamente seguro por endpoint; persisti-lo protegido (criptografado em repouso, se o gerenciador de segredos estiver disponível). Status: `201`, `400` (`WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_SUBSCRIBED_STATUS`), `401`, `409` (`WEBHOOK_ALREADY_EXISTS`) e `422` (`WEBHOOK_PAYLOAD_TOO_LARGE`, quando aplicável à configuração).

### 2. Listar configurações de um cliente

`GET /webhooks?customerId=4ddb9174-cc57-44d1-81e9-d0c62d759631&page=1&pageSize=20`

Headers: `Authorization: Bearer <JWT>`, `Accept: application/json`. Não possui corpo de request.

Response `200`:

```json
{
  "data": [{
    "id": "64dbb72c-2432-4f17-b86b-c5f0bc1e2ba8",
    "customerId": "4ddb9174-cc57-44d1-81e9-d0c62d759631",
    "url": "https://cliente.example.com/hooks/orders",
    "subscribedStatuses": ["SHIPPED", "DELIVERED"],
    "active": true,
    "previousSecretExpiresAt": null,
    "createdAt": "2026-08-09T12:00:00.000Z"
  }],
  "meta": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Semântica: lista somente metadados; nunca expõe segredo atual ou anterior. Status: `200`, `400` (query inválida) e `401`.

### 3. Atualizar configuração

`PATCH /webhooks/:id`

Headers: `Authorization: Bearer <JWT>`, `Content-Type: application/json`, `Accept: application/json`.

Request:

```json
{
  "url": "https://cliente.example.com/integrations/orders",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `200`:

```json
{
  "id": "64dbb72c-2432-4f17-b86b-c5f0bc1e2ba8",
  "customerId": "4ddb9174-cc57-44d1-81e9-d0c62d759631",
  "url": "https://cliente.example.com/integrations/orders",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-08-09T12:05:00.000Z"
}
```

Semântica: altera apenas campos enviados; desativar impede novos eventos, mas não apaga entregas/DLQ existentes. Status: `200`, `400` (`WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_SUBSCRIBED_STATUS`), `401`, `404` (`WEBHOOK_NOT_FOUND`) e `409`.

### 4. Rotacionar segredo

`POST /webhooks/:id/secret/rotate`

Headers: `Authorization: Bearer <JWT>`, `Content-Type: application/json`, `Accept: application/json`.

Request: `{}`.

Response `200`:

```json
{
  "webhookId": "64dbb72c-2432-4f17-b86b-c5f0bc1e2ba8",
  "secret": "whsec_novo...",
  "previousSecretExpiresAt": "2026-08-10T12:06:00.000Z"
}
```

Semântica: promove um novo segredo e mantém o anterior válido por 24 h. Durante a janela, o worker deve assinar uma entrega com o segredo atual; consumidores devem aceitar o atual e anterior durante a migração. Status: `200`, `401`, `404` (`WEBHOOK_NOT_FOUND`) e `409` (`WEBHOOK_SECRET_ROTATION_IN_PROGRESS`).

### 5. Consultar entregas

`GET /webhooks/:id/deliveries?page=1&pageSize=100&outcome=FAILED`

Headers: `Authorization: Bearer <JWT>`, `Accept: application/json`. Não possui corpo de request.

Response `200`:

```json
{
  "data": [{
    "id": "df60bc51-73c5-4cb3-9427-6d2da4f25d88",
    "eventId": "a82b4ba8-52a2-4d8d-8cb3-2fbdd7f4667e",
    "attemptNumber": 2,
    "outcome": "FAILED",
    "responseStatus": 503,
    "durationMs": 1002,
    "errorCode": "WEBHOOK_DELIVERY_HTTP_5XX",
    "createdAt": "2026-08-09T12:10:00.000Z"
  }],
  "meta": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Semântica: retorna histórico paginado do endpoint, inclusive falhas; payload e resposta são disponibilizados apenas se respeitarem política de mascaramento/limite. Status: `200`, `400`, `401` e `404` (`WEBHOOK_NOT_FOUND`).

### 6. Excluir configuração

`DELETE /webhooks/:id`

Headers: `Authorization: Bearer <JWT>`, `Accept: application/json`. Não possui corpo de request. Response `204` sem corpo.

Semântica: desativa e remove a configuração para novos eventos; preservar deliveries e DLQ para auditoria, com referência histórica imutável. Status: `204`, `401`, `404` (`WEBHOOK_NOT_FOUND`) e `409` (`WEBHOOK_HAS_PENDING_DELIVERIES`, se a política optar por exigir desativação antes da remoção).

### 7. Replay administrativo de DLQ

`POST /admin/webhooks/dead-letter/:id/replay`

Headers: `Authorization: Bearer <JWT de ADMIN>`, `Content-Type: application/json`, `Accept: application/json`.

Request: `{}`.

Response `202`:

```json
{
  "deadLetterId": "4d2943c3-fd12-4834-94c3-880ae87a037f",
  "eventId": "bb326c0e-b204-4d45-a249-d3eaee28714d",
  "status": "PENDING",
  "replayedAt": "2026-08-09T13:00:00.000Z"
}
```

Semântica: agenda nova entrega a partir do registro DLQ e não executa chamada HTTP no request. Status: `202`, `401`, `403`, `404` (`WEBHOOK_DEAD_LETTER_NOT_FOUND`, `WEBHOOK_NOT_FOUND`), `409` (`WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`, `WEBHOOK_INACTIVE`) e `422` (`WEBHOOK_REPLAY_PAYLOAD_INVALID`).

### Contrato de entrega ao consumidor

O worker envia `POST` à URL cadastrada, com o payload da seção de criação da outbox e os headers:

- `Content-Type: application/json`
- `X-Event-Id: <UUID>`
- `X-Webhook-Id: <UUID>`
- `X-Timestamp: <ISO-8601 UTC>`
- `X-Signature: sha256=<hex(HMAC-SHA256(secret, raw request body))>`

O consumidor deve validar HTTPS, recomputar o HMAC sobre os bytes exatos do corpo, comparar em tempo constante, validar a defasagem de `X-Timestamp` conforme sua política e deduplicar `X-Event-Id`. Qualquer resposta 2xx confirma processamento; duplicatas devem também responder 2xx.

## Matriz de erros previstos

| Código | HTTP | Situação e ação do cliente/operador |
|---|---:|---|
| `WEBHOOK_INVALID_URL` | 400 | URL ausente, malformada ou não HTTPS; corrigir cadastro. |
| `WEBHOOK_INVALID_SUBSCRIBED_STATUS` | 400 | Status não pertence a `OrderStatus` ou lista vazia; corrigir request. |
| `WEBHOOK_NOT_FOUND` | 404 | Configuração não existe; conferir id. |
| `WEBHOOK_ALREADY_EXISTS` | 409 | Endpoint duplicado para mesmo cliente/URL; reutilizar ou atualizar. |
| `WEBHOOK_INACTIVE` | 409 | Operação/replay para endpoint inativo; reativar ou criar outro. |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | Rotação já em curso; aguardar/janela de 24 h. |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Snapshot excede 64 KiB; investigar contrato/dados, sem truncar. |
| `WEBHOOK_DELIVERY_TIMEOUT` | interno/entrega | Timeout de 10 s; registrar e retentar. |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | interno/entrega | Erro de DNS/TLS/rede; registrar e retentar conforme política. |
| `WEBHOOK_DELIVERY_HTTP_4XX` | interno/entrega | Falha não retentável (exceto 408/425/429); mover para DLQ. |
| `WEBHOOK_DELIVERY_HTTP_5XX` | interno/entrega | Falha retentável; aplicar backoff. |
| `WEBHOOK_DELIVERY_RETRY_EXHAUSTED` | interno/DLQ | Limite atingido; mover para DLQ. |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | DLQ inexistente; conferir id. |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Replay duplicado; consultar evento resultante. |
| `WEBHOOK_REPLAY_PAYLOAD_INVALID` | 422 | Snapshot da DLQ não pode ser reenfileirado; investigar e corrigir dados. |
| `WEBHOOK_HAS_PENDING_DELIVERIES` | 409 | Exclusão conflita com eventos pendentes, quando essa regra estiver habilitada. |

## Estratégias de resiliência

- **Timeouts:** 10 s para conexão+resposta HTTP; lease de 30 s para `PROCESSING`; a API de mudança de status não espera o worker.
- **Retries:** máximo de cinco para falhas transitórias; retry de 429 respeita `Retry-After`; 4xx definitivos seguem diretamente para DLQ.
- **Backoff:** 1 m, 5 m, 30 m, 2 h, 12 h, com jitter de até 10%; `nextAttemptAt` persiste para sobreviver a restart.
- **Fallback:** DLQ persistente e replay manual por `ADMIN`; em falha de worker, leases vencidos retornam à fila. Não há e-mail nem envio alternativo nesta fase.
- **Segurança operacional:** validar URL HTTPS e bloquear IPs/hosts privados, loopback e link-local para reduzir SSRF; não seguir redirects; limitar corpo a 64 KiB e resposta registrada a tamanho seguro; manter segredos e assinaturas fora de logs.

## Observabilidade

**Métricas:** `webhook_outbox_pending_total`, `webhook_delivery_attempts_total{outcome,http_status}`, `webhook_delivery_duration_ms` (histograma), `webhook_retry_scheduled_total`, `webhook_dlq_total`, `webhook_processing_lag_seconds`, `webhook_worker_poll_duration_ms`, `webhook_processing_lease_recovered_total` e `webhook_payload_rejected_total`.

**Logs estruturados Pino:** emitir `webhook_enqueued`, `webhook_delivery_started`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_retry_scheduled`, `webhook_moved_to_dlq`, `webhook_dlq_replayed` e eventos de lifecycle do worker. Incluir `eventId`, `webhookId`, `orderId`, `customerId`, `attemptCount`, `statusCode`, `durationMs`, `errorCode` e `requestId`/`actorUserId` quando disponíveis. Aplicar redação a `secretCurrent`, `secretPrevious`, `X-Signature`, `Authorization`, payload sensível e respostas externas.

**Tracing:** criar spans `webhook.outbox.enqueue`, `webhook.worker.poll`, `webhook.delivery` e `webhook.dlq.replay`, propagando `traceId`/`requestId` da alteração de pedido para o registro de outbox e para logs do worker. A chamada externa deve ter atributos de destino sanitizado, tentativa, `eventId`, resultado e duração; nunca incluir segredo, assinatura ou payload integral como atributo.

Alertas sugeridos: backlog com idade acima de 10 s; aumento de DLQ; taxa de falha por endpoint acima do limiar acordado; e worker sem polls/heartbeat.

## Dependências e compatibilidade

- Manter Node.js `>=20`, TypeScript, Express 4, Prisma `5.22`, MySQL e Zod já presentes.
- Não introduzir broker. Para HTTP de saída, usar a capacidade nativa disponível no Node 20 ou adicionar cliente somente após decisão explícita; o contrato de timeout e não-redirecionamento é obrigatório em ambos os casos.
- Gerar a migration Prisma para os novos modelos, enums, relações e índices; não alterar semântica das tabelas existentes.
- Criar script separado `worker` e build que inclua `src/worker.ts`; API e worker usam a mesma `DATABASE_URL`, mas clientes Prisma distintos por processo.
- Preservar compatibilidade da API atual sob `/api/v1`; adicionar o novo router sem alterar contratos de pedidos. Consumidores recebem apenas `order.status_changed` v1 e devem tolerar campos adicionais futuros.
- Antes de produção, revisar geração, armazenamento e rotação de segredo com Segurança.

## Integração com o sistema existente

| Caminho real | Integração proposta |
|---|---|
| `prisma/schema.prisma` | Declarar modelos/enum de webhook, relações com `Customer` e `Order`, UUIDs e os índices da fila, delivery e DLQ. A migration cria as tabelas no MySQL já configurado. |
| `src/modules/orders/order.service.ts` | No método `changeStatus`, depois de atualizar pedido/histórico/estoque e antes do commit, chamar `publishWebhookEvent(tx, order, from, to)`. A função consulta inscrições e grava snapshots no mesmo `tx`, garantindo atomicidade. |
| `src/app.ts` | Instanciar `WebhookRepository`, `WebhookService` e `WebhookController`; adicionar o controller à composição. Não iniciar worker neste arquivo, pois é outro processo. |
| `src/routes/index.ts` | Estender `Controllers`, importar `buildWebhookRouter` e montar `/webhooks` e `/admin/webhooks` sob o prefixo existente `/api/v1`. |
| `src/middlewares/auth.middleware.ts` | Reutilizar `authenticate` para CRUD/consulta; aplicar `requireRole('ADMIN')` exclusivamente no router de replay da DLQ. |
| `src/shared/errors/app-error.ts` e `src/middlewares/error.middleware.ts` | Criar/usar erros derivados de `AppError` com códigos `WEBHOOK_*`; o formato JSON e o middleware central permanecem a fonte única de respostas. |
| `src/shared/logger/index.ts` | Reutilizar Pino e ampliar a lista de campos redigidos para segredos e assinatura de webhook; o worker importa a mesma instância/configuração. |
| `src/server.ts` | Manter exclusivamente o bootstrap HTTP. Criar `src/worker.ts` paralelo, com seu próprio bootstrap, sinais de shutdown, `PrismaClient` e loop de polling. |

Estrutura nova esperada: `src/modules/webhooks/{webhook.routes.ts,webhook.controller.ts,webhook.service.ts,webhook.repository.ts,webhook.schemas.ts,webhook.processor.ts,webhook.errors.ts}` e `src/worker.ts`.

## Critérios de aceite técnicos

- `docs/FDD.md` descreve o módulo e todos os contratos acima; a implementação posterior mantém a API existente sem quebra.
- Uma transição de status com endpoint inscrito cria outbox na mesma transação; induzir falha no insert impede o commit do pedido.
- Sem inscrição para o status, não há evento; com múltiplos endpoints inscritos, há um evento por endpoint, cada qual com `event_id` único e snapshot ≤64 KiB.
- Worker separado faz polling de 2 s, não bloqueia a API, assina corpo bruto com HMAC-SHA256 e trata apenas 2xx como sucesso.
- Timeout e erros transitórios seguem a agenda de cinco tentativas; falha final/não retentável preserva DLQ e delivery auditável.
- Replay exige `ADMIN`, é idempotente contra duplicação, registra o ator e gera novo `event_id`.
- CRUD, rotação e entregas aplicam JWT, validações Zod e códigos `WEBHOOK_*`; nenhum endpoint de leitura expõe segredos.
- Métricas, logs redigidos e tracing permitem correlacionar mudança de pedido, outbox, tentativas e DLQ.
- Testes a serem implementados cobrem atomicidade, filtros, HMAC, payload snapshot, timeout/retry, 4xx/5xx, recuperação de lease, DLQ/replay, autorização ADMIN e idempotência do consumidor via `X-Event-Id`.

## Riscos e mitigação

| Risco | Impacto | Mitigação |
|---|---|---|
| Endpoint lento/indisponível | backlog e atraso | timeout de 10 s, backoff persistido, DLQ, alertas de lag. |
| Entrega duplicada | efeito duplicado no cliente | *at-least-once* documentado; `X-Event-Id` estável e orientação de deduplicação. |
| Perda entre update e publicação | cliente não é avisado | outbox criada dentro da mesma transação Prisma/MySQL. |
| Vazamento de segredo | falsificação de chamadas | segredo por endpoint, rotação de 24 h, redação de logs e revisão de Segurança. |
| SSRF por URL cadastrada | acesso à rede interna | HTTPS obrigatório, validação DNS/IP contra faixas privadas e redirects desabilitados. |
| Queda do worker após enviar | tentativa repetida | lease e recuperação de `PROCESSING`; consumidor idempotente. |
| Escala com vários workers | inversão de ordem por pedido | manter single worker inicialmente; antes de escalar, particionar por `orderId` ou introduzir lock. |
| Crescimento de tabelas | custo e consultas lentas | índices definidos, paginação e política futura de arquivamento após 30 dias. |
