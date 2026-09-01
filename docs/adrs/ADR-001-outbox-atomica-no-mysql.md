# ADR-001: Outbox atômica no MySQL

## Status

Aceito.

## Contexto

Clientes B2B precisam ser notificados sobre mudanças de status de pedidos em menos de 10 segundos. Uma chamada HTTP síncrona durante a alteração de status aumentaria a duração da transação e tornaria o fluxo de pedidos dependente da disponibilidade e latência de sistemas externos.

O projeto já usa MySQL, configurado em `prisma/schema.prisma`, e o método `changeStatus` em `src/modules/orders/order.service.ts` atualiza pedido, histórico e estoque dentro de `prisma.$transaction`. A notificação não pode se perder quando essa alteração for confirmada, nem deve existir quando ela sofrer rollback.

## Decisão

Persistir um evento de webhook em uma tabela `webhook_outbox` no mesmo MySQL e na mesma transação que altera o status do pedido. A inserção será feita por uma função de publicação que receba o `tx` da transação atual. Se a gravação do evento falhar, toda a mudança de status será revertida.

Os registros da outbox usarão UUID, coerente com as entidades existentes no schema.

## Alternativas Consideradas

- Disparar HTTP de forma síncrona no `OrderService`: rejeitado porque um endpoint lento ou indisponível bloquearia a mudança de status e criaria uma decisão inadequada de rollback.
- Publicar diretamente em Redis Streams ou outro broker: rejeitado nesta fase por introduzir infraestrutura operacional adicional e ainda exigir tratamento da consistência entre banco e broker.

## Consequências

**Benefícios:** preserva atomicidade entre a alteração de negócio e o registro para entrega; desacopla o tempo de resposta da API da disponibilidade dos clientes; reutiliza a infraestrutura MySQL existente.

**Desvantagens:** acrescenta uma tabela, índices e um processo de processamento; a entrega deixa de ser imediata.

**Trade-offs:** aceita-se entrega assíncrona em troca de não perder eventos confirmados e de manter o caminho crítico de pedidos independente de chamadas externas.
