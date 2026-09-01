# ADR-005: Entrega at-least-once com identificador de evento

## Status

Aceito.

## Contexto

Após enviar uma requisição, o worker pode não receber a resposta por falha de rede, embora o cliente tenha processado o evento. Retentar nesse cenário é necessário para não perder notificações, mas pode gerar uma entrega duplicada.

Garantir exatamente uma vez exigiria coordenação durável entre nossa plataforma e o sistema de cada cliente, o que não é viável para webhooks externos.

## Decisão

Oferecer semântica de entrega *at-least-once*. Cada evento receberá um UUID estável na criação da outbox e será enviado no header `X-Event-Id` — além de constar no payload. Clientes deverão deduplicar eventos usando esse identificador.

## Alternativas Consideradas

- Garantia exactly-once: rejeitada pela coordenação distribuída, armazenamento de estado e integração bilateral que exigiria.
- Entrega at-most-once, sem retry depois de envio incerto: rejeitada porque poderia perder mudanças de status importantes.

## Consequências

**Benefícios:** evita perda de eventos diante de falhas de comunicação; apresenta um mecanismo de deduplicação simples e comum a integrações por webhook.

**Desvantagens:** o consumidor precisa implementar idempotência e armazenar eventos já processados; duplicatas continuam possíveis.

**Trade-offs:** transfere-se parte da responsabilidade de consistência para o consumidor em troca de uma garantia de entrega prática e implementável entre sistemas independentes.
