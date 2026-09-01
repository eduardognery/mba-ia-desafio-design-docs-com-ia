# RFC — Webhooks de notificação de status de pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa, Tech Lead |
| Status | Proposto para revisão |
| Data | 07/08/2026 |
| Revisores | Marcos (Product Manager), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Sofia (Segurança) |

## Resumo executivo

Propõe-se disponibilizar webhooks **outbound** para que clientes B2B recebam mudanças de status dos seus pedidos com latência inferior a 10 segundos. A entrega será assíncrona: a mudança do pedido registrará atomically um evento no MySQL, e um worker independente o enviará ao endpoint configurado pelo cliente.

O desenho privilegia confiabilidade e simplicidade operacional: entrega *at-least-once*, retry limitado, dead letter persistida, idempotência no consumidor e autenticação da mensagem por HMAC. A proposta reutiliza a arquitetura modular, Prisma, Pino, `AppError` e autorização já presentes na API.

## Contexto e problema

Clientes B2B hoje consultam `GET /orders` repetidamente para detectar mudanças, o que aumenta custo e atrasa integrações. O requisito acordado é notificação em menos de 10 segundos, apenas no sentido plataforma → cliente.

O fluxo atual de `changeStatus` já atualiza pedido, histórico e estoque em uma transação MySQL. Não pode depender da disponibilidade de endpoints externos, nem confirmar uma mudança de status sem deixar um evento de entrega recuperável.

## Proposta técnica

### Visão geral da solução

Será criado o domínio de webhooks, responsável por configurações de endpoint, entregas e consulta de histórico. Cada endpoint será associado a um cliente, poderá selecionar os status de interesse e terá URL HTTPS e segredo próprios.

Ao ocorrer uma transição de status, somente endpoints ativos e inscritos naquele status gerarão eventos. O evento será persistido como um snapshot imutável na outbox, na mesma transação que atualiza o pedido, o histórico e o estoque. Assim, o commit de negócio e o registro para entrega são indivisíveis.

Um processo worker, separado da API, consultará periodicamente a outbox e realizará os envios HTTP. A primeira versão executará um único worker e consumirá os eventos em ordem de criação, preservando a sequência por pedido enquanto esse modelo estiver em uso. Falhas terão até cinco tentativas com backoff de 1 min, 5 min, 30 min, 2 h e 12 h; depois seguirão para uma dead letter persistida, sujeita a replay manual por administrador.

As requisições incluirão um payload JSON enxuto de `order.status_changed` e os cabeçalhos `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`. O corpo será assinado com HMAC-SHA256; cada endpoint terá segredo individual, com rotação e janela de convivência de 24 horas. A plataforma oferecerá entrega *at-least-once*, cabendo ao consumidor deduplicar pelo `X-Event-Id`.

O CRUD de configurações e a consulta de entregas integrarão a API autenticada existente. O replay de dead letter será administrativo, usando a autorização `ADMIN` existente e registrando auditoria. O worker terá ciclo de vida e cliente Prisma próprios, embora compartilhe o mesmo banco MySQL.

### Limites de escopo

Não fazem parte desta fase: painel visual, envio de e-mail como fallback, broker externo e garantia de ordenação global. O payload não incluirá itens do pedido e terá limite de 64 KB; detalhes poderão ser obtidos pela API de pedidos.

## Alternativas consideradas

| Alternativa | Motivo do descarte |
| --- | --- |
| Disparar HTTP de modo síncrono no fluxo de alteração de status | Acoplaria a transação crítica à latência e disponibilidade do cliente, podendo bloquear pedidos ou criar um dilema incorreto de rollback. |
| Publicar diretamente em Redis Streams ou outro broker | Introduz infraestrutura e operação adicionais para um time pequeno, além de não resolver por si só a consistência entre banco e broker. A outbox no MySQL existente atende ao requisito. |
| Executar a entrega no mesmo processo da API | Reinícios e escala da API afetariam o processamento de eventos; um processo separado isola falhas e carga. |
| Retentar indefinidamente | Eventos destinados a endpoints abandonados permaneceriam pendentes sem limite. O limite de cinco tentativas preserva uma janela razoável de recuperação e dá destino operacional à falha. |
| Garantir entrega *exactly-once* | Exigiria coordenação durável com cada sistema consumidor. *At-least-once* com identificador estável é viável e é o contrato adequado para webhooks. |

## Questões em aberto

- **Rate limiting por cliente/endpoint:** observar o volume real antes de definir limites ou política de enfileiramento para picos de notificações.
- **Escala do worker:** múltiplos consumidores exigirão particionamento consistente por `order_id` ou outro mecanismo que preserve a ordenação por pedido; não é necessário na primeira versão.
- **Notificação de falhas ao cliente:** e-mail após falhas consecutivas foi adiado para uma fase posterior, após medir impacto e necessidade.
- **Política de retenção e arquivamento:** a reunião sugeriu arquivar entregas concluídas após cerca de 30 dias, mas prazo, destino e operação ainda não foram definidos.

## Impacto e riscos

O schema MySQL ganhará entidades de configuração, outbox, tentativas/entregas e dead letter; o fluxo transacional de mudança de status passará a publicar eventos, e haverá um novo processo operacional para o worker. A API também ganhará rotas autenticadas para gestão e visibilidade das entregas.

Os principais riscos são acúmulo de backlog, indisponibilidade de endpoints de clientes, entregas duplicadas e exposição indevida de segredos ou payloads. Eles são mitigados, respectivamente, por monitoramento operacional e índices da outbox; timeout e retry com DLQ; `X-Event-Id`; e HTTPS, HMAC por endpoint, rotação de segredo, limite de payload e revisão de segurança antes do deploy. A capacidade inicial é deliberadamente limitada a um worker, devendo ser revisitada conforme o volume crescer.

## Decisões relacionadas

- [ADR-001 — Outbox atômica no MySQL](adrs/ADR-001-outbox-atomica-no-mysql.md)
- [ADR-002 — Worker separado com polling](adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Ordem de entrega por pedido](adrs/ADR-003-ordem-por-pedido-com-worker-unico.md)
- [ADR-004 — Retry limitado e dead letter](adrs/ADR-004-retry-limitado-e-dead-letter.md)
- [ADR-005 — Entrega at-least-once com event ID](adrs/ADR-005-entrega-at-least-once-com-event-id.md)
- [ADR-006 — Assinatura HMAC e rotação por endpoint](adrs/ADR-006-assinatura-hmac-e-rotacao-por-endpoint.md)
- [ADR-007 — Payload snapshot na criação do evento](adrs/ADR-007-payload-snapshot-na-criacao-do-evento.md)
- [ADR-008 — Módulo de webhooks e reuso de padrões](adrs/ADR-008-modulo-webhooks-e-reuso-de-padroes.md)
