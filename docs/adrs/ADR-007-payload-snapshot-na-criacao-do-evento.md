# ADR-007: Payload como snapshot na criação do evento

## Status

Aceito.

## Contexto

Entre a mudança de status e uma tentativa posterior de entrega, o pedido pode sofrer novas mudanças. Gerar o payload apenas no momento da entrega poderia fazer um evento antigo representar o estado atual, e não a transição que o originou.

O conteúdo precisa permanecer leve para respeitar o teto de 64 KB e não expor detalhes desnecessários.

## Decisão

Renderizar e persistir o JSON do evento na entrada da outbox, dentro da transação que registra a alteração. O payload terá `event_id`, `event_type` igual a `order.status_changed`, timestamp ISO 8601, identificadores e número do pedido, status anterior e novo, `customer_id` e `total_cents`. Itens do pedido não serão enviados; o consumidor poderá consultar `GET /orders/:id` quando precisar de detalhes.

Eventos acima de 64 KB serão tratados como erro, sem truncamento.

## Alternativas Consideradas

- Armazenar somente `order_id` e montar o payload no worker: rejeitado porque o conteúdo poderia refletir uma alteração posterior e perder o contexto histórico.
- Incluir todos os itens do pedido em toda notificação: rejeitado por aumentar payload, acoplamento e custo de entrega.
- Truncar payloads que excedam o limite: rejeitado porque geraria mensagens semanticamente incompletas e difíceis de detectar pelo cliente.

## Consequências

**Benefícios:** cada notificação é uma representação imutável e auditável da transição; retries enviam o mesmo conteúdo; payloads permanecem pequenos.

**Desvantagens:** a outbox armazena dados duplicados e qualquer evolução de contrato precisa considerar eventos já persistidos.

**Trade-offs:** usa-se mais armazenamento em troca de fidelidade temporal do evento e de comportamento consistente entre tentativas.
