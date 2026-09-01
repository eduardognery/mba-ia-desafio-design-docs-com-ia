# ADR-004: Retry limitado com dead letter persistida

## Status

Aceito.

## Contexto

Endpoints de clientes podem ficar temporariamente indisponíveis. Abandonar o evento na primeira falha viola a expectativa de entrega; retentar indefinidamente mantém eventos pendentes sem limite quando um endpoint foi removido ou permanece defeituoso.

Também é necessário ter evidência de falhas e um caminho controlado para reprocessá-las.

## Decisão

Realizar no máximo cinco tentativas de entrega, usando backoff exponencial de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas após falhas. Uma chamada terá timeout de 10 segundos. Ao esgotar as tentativas, mover o registro para a tabela separada `webhook_dead_letter`, com payload, motivo e data da falha.

Permitir replay manual por `POST /admin/webhooks/dead-letter/:id/replay`, que recria o evento pendente na outbox. O endpoint exigirá a role `ADMIN` e registrará quem executou a ação para auditoria.

## Alternativas Consideradas

- Retentar indefinidamente: rejeitado porque eventos de endpoints abandonados ficariam pendurados para sempre.
- Limitar a três tentativas: rejeitado por não cobrir indisponibilidades planejadas de algumas horas já observadas em clientes.
- Manter falhas na própria outbox: rejeitado porque mistura a fila operacional com registros que requerem investigação e replay.

## Consequências

**Benefícios:** cobre falhas transitórias durante quase 15 horas; impede crescimento infinito de tentativas; preserva material para diagnóstico e recuperação manual; protege o replay com autorização forte.

**Desvantagens:** a recuperação final depende de intervenção administrativa; tabelas e estados adicionais exigem monitoramento; a última entrega pode ocorrer muitas horas após o evento.

**Trade-offs:** escolhe-se previsibilidade operacional e capacidade de auditoria em vez de retenção ilimitada e entrega automática sem fim.
