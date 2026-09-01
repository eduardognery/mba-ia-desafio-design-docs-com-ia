# ADR-003: Ordem de entrega por pedido com worker único

## Status

Aceito.

## Contexto

Um pedido pode mudar rapidamente, por exemplo, de `PAID` para `PROCESSING` e depois para `SHIPPED`. Os clientes precisam receber as transições daquele pedido em sequência compreensível, mas não solicitaram ordenação global entre pedidos distintos.

O paralelismo de vários workers poderia fazer tentativas de eventos diferentes terminarem em ordem diferente.

## Decisão

Operar inicialmente um único worker, consumindo a outbox por `created_at` ascendente. A garantia assumida é de ordenação por `order_id` enquanto houver uma única instância de worker; não será oferecida garantia de ordenação global como contrato público.

## Alternativas Consideradas

- Executar vários workers em paralelo desde o início: rejeitado porque sacrifica a ordenação por pedido e não há demanda atual para essa escala.
- Implementar particionamento por `order_id` ou locks pessimistas agora: postergado porque adiciona complexidade operacional e de concorrência antes de haver evidência de necessidade.

## Consequências

**Benefícios:** comportamento simples e previsível para cada pedido; implementação inicial menor; satisfaz a expectativa atual dos clientes.

**Desvantagens:** a taxa de processamento é limitada à capacidade de um worker e a garantia deixa de valer caso a arquitetura seja escalada sem particionamento.

**Trade-offs:** privilegia ordenação por entidade de negócio sobre throughput máximo. Uma evolução futura deverá particionar consistentemente por `order_id` antes de introduzir múltiplos consumidores.
