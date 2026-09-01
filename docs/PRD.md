# PRD — Webhooks de notificação de status de pedidos

## Resumo e contexto da feature

O sistema de gestão de pedidos expõe uma API REST autenticada para clientes, usuários, produtos e pedidos. Hoje, clientes B2B consultam repetidamente `GET /orders` para descobrir mudanças no status de seus pedidos.

Esta feature introduz **webhooks outbound**: quando o status de um pedido muda, a plataforma notifica os endpoints HTTPS cadastrados pelo cliente. A entrega será assíncrona e persistente, para que a alteração do pedido não dependa da disponibilidade do endpoint externo.

O pedido foi formalizado por Atlas Comercial, MaxDistribuição e Nova Cargo. A Atlas sinalizou risco de migração para um concorrente se a necessidade não for atendida até o fim do trimestre. A estimativa discutida é de três sprints, incluindo a revisão de segurança antes do deploy.

## Problema e motivação

O polling de `GET /orders` torna a integração dos clientes lenta e cara e obriga atualizações manuais periódicas. Os clientes precisam reagir a mudanças de status sem consultar continuamente a API.

Enviar a notificação de forma síncrona dentro da mudança de status não é aceitável: a transação atual já atualiza o pedido, registra o histórico e pode ajustar o estoque. Um endpoint externo lento ou indisponível não pode travar ou reverter essa operação de negócio.

## Público-alvo e cenários de uso

- **Clientes B2B integradores** — cadastram endpoints pela API, escolhem quais status desejam receber e atualizam seus sistemas quando um pedido muda, por exemplo, para `SHIPPED` ou `DELIVERED`.
- **Administradores da plataforma** — investigam falhas definitivas e reprocessam manualmente eventos na fila de mensagens mortas (DLQ), com trilha de auditoria.
- **Times de operações e suporte** — consultam o histórico das últimas 100 entregas de cada webhook, incluindo resultado, payload, resposta e tempo de resposta, para diagnóstico.

## Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Entregar mudanças de status em tempo real para o cliente | Latência entre a mudança de status e a entrega do webhook | Menor que 10 segundos |
| Evitar perda de eventos quando a mudança de status é confirmada | Registro do evento na outbox na mesma transação da alteração do pedido | 100% das mudanças confirmadas que tenham ao menos um webhook elegível registradas atomicamente |

## Escopo

### Incluso

- Webhooks exclusivamente outbound para mudanças de status de pedidos.
- Cadastro, edição, remoção e listagem de configurações de webhook por cliente, via API autenticada.
- Seleção, por endpoint, dos status de pedido que devem gerar notificação.
- Registro de evento em outbox MySQL na mesma transação SQL que altera o pedido e registra seu histórico.
- Worker separado da API, com polling a cada 2 segundos, para entrega HTTP dos eventos pendentes.
- Retentativas, DLQ, histórico de entregas e replay administrativo.
- Assinatura HMAC-SHA256, secret exclusiva por endpoint e rotação de secret.

### Fora de escopo

- Notificações por e-mail quando um webhook falhar repetidamente; foram adiadas para uma próxima fase.
- Rate limiting de envio por cliente; será observado antes de uma decisão futura.
- Dashboard visual para o cliente gerenciar ou visualizar webhooks; nesta fase serão disponibilizados somente endpoints de API.
- Arquivamento de eventos já entregues após 30 dias.
- Escalabilidade com múltiplos workers e garantia de ordenação nesse cenário.
- Webhooks de entrada (clientes enviando eventos para a plataforma).

## Requisitos funcionais

1. O sistema deve disponibilizar um endpoint autenticado para cadastrar um webhook com URL, cliente e lista de status de pedido de interesse; a secret deve ser gerada pela plataforma e devolvida na criação.
2. O sistema deve disponibilizar endpoints autenticados para editar, remover e listar os webhooks de um cliente; o `customer_id` deve ser informado no corpo ou no caminho da requisição, e não inferido do JWT.
3. Cada webhook deve manter URL, secret, `customer_id`, estado ativo e a lista de status filtrados.
4. Quando ocorrer uma transição válida de status em `OrderService.changeStatus`, o sistema deve identificar os webhooks ativos daquele cliente que escutam o novo status e criar eventos somente para eles.
5. Se nenhum webhook do cliente escutar o status, o sistema não deve inserir evento na outbox.
6. A criação do evento elegível na `webhook_outbox` deve ocorrer na mesma transação SQL que atualiza `orders` e cria o registro em `order_status_history`; uma falha ao inserir na outbox deve causar rollback da mudança de status.
7. O evento deve guardar um snapshot do payload no momento de sua criação, para refletir o estado da mudança mesmo se o pedido for alterado depois.
8. Um worker executado como processo separado da API deve buscar os eventos pendentes mais antigos em pequenos lotes por polling de 2 segundos, enviá-los ao endpoint e registrar o resultado da entrega.
9. A entrega deve usar semântica *at-least-once* e incluir um UUID único por evento no header `X-Event-Id`, permitindo que o cliente faça deduplicação.
10. O payload JSON deve conter `event_id`, `event_type` (`order.status_changed`), timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, incluindo `total_cents`; não deve incluir itens do pedido.
11. Cada chamada deve enviar `Content-Type: application/json`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
12. Após falha de entrega, o sistema deve tentar novamente até cinco vezes, com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
13. Ao esgotar as cinco tentativas, o sistema deve mover o evento para a tabela separada `webhook_dead_letter`, preservando payload, motivo da falha e timestamp.
14. O sistema deve expor `POST /admin/webhooks/dead-letter/:id/replay` para recolocar uma mensagem da DLQ como pendente na outbox. O acesso deve exigir a role `ADMIN` e registrar quem realizou o replay para auditoria.
15. O sistema deve expor `GET /webhooks/:id/deliveries` para consultar as últimas 100 entregas de um webhook, com sucesso/falha, payload, resposta e tempo de resposta.
16. Cada endpoint deve possuir secret exclusiva, usada para assinar o corpo da requisição com HMAC-SHA256 em `X-Signature`.
17. O sistema deve permitir a rotação de secret pela API: a secret anterior deve continuar válida por 24 horas após a rotação e tornar-se inválida ao final desse período.

## Requisitos não funcionais

- A URL cadastrada deve usar HTTPS; URLs HTTP devem ser rejeitadas por validação.
- O corpo do payload não pode exceder 64 KB. Caso exceda o limite, a entrega deve falhar; o payload não deve ser truncado.
- Cada chamada HTTP do worker deve ter timeout de 10 segundos; exceder esse limite é falha elegível para retry.
- A outbox deve possuir índices para status do evento e `created_at`, permitindo leitura eficiente dos pendentes mais antigos.
- Com um único worker, a ordem de entrega deve ser preservada por `order_id`; não há garantia de ordenação global.
- O worker deve usar o mesmo banco e `DATABASE_URL` da API, mas seu próprio `PrismaClient`, pois será outro processo Node.
- A feature deve reutilizar os padrões existentes: módulo em `src/modules`, schemas Zod, `AppError`, middleware central de erros, logger Pino e códigos de erro prefixados por `WEBHOOK_`.

## Decisões e trade-offs principais

| Decisão | Trade-off aceito |
| --- | --- |
| Usar outbox no MySQL existente em vez de Redis Streams | Menos infraestrutura e complexidade para um time pequeno, em troca de polling no banco. |
| Processar com polling a cada 2 segundos | Menos reatividade que um mecanismo de eventos, mas atende à meta de menos de 10 segundos e é compatível com MySQL. |
| Manter worker separado e único | Isola a entrega dos reinícios da API e preserva a ordem por pedido, mas não oferece escalabilidade paralela nesta fase. |
| Garantir *at-least-once* com `X-Event-Id` | Pode haver duplicidade; o cliente deve deduplicar. Evita a complexidade de uma garantia *exactly-once*. |
| Limitar retry a cinco tentativas e usar DLQ | Evita eventos pendentes indefinidamente, mas eventos de clientes indisponíveis por mais de aproximadamente 15 horas exigem ação administrativa. |
| Salvar snapshot renderizado do payload na outbox | Preserva o estado histórico correto, ao custo de persistir o payload no evento. |

## Dependências

- A transação de alteração de status já existente em `OrderService.changeStatus`, que atualiza pedido, histórico e estoque.
- MySQL e Prisma já usados pela aplicação, incluindo migração para as tabelas de configuração, outbox, entregas e DLQ.
- Autenticação JWT e autorização existente por roles `ADMIN` e `OPERATOR`; o replay reutilizará `requireRole`.
- Infraestrutura para executar o processo do worker separadamente da API, usando a mesma `DATABASE_URL`.
- Revisão de segurança de HMAC e geração/rotação de secret pela responsável de segurança antes do deploy, com pelo menos dois dias úteis reservados.
- Portal de desenvolvedor para orientar integradores sobre assinatura e deduplicação por `X-Event-Id`.

## Riscos

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Endpoint do cliente lento ou indisponível impede a entrega imediata | Alta | Alto | Timeout de 10 segundos, cinco retentativas com backoff e persistência em DLQ para replay administrativo. |
| Falha ao criar o evento causa divergência entre status do pedido e notificação | Média | Alto | Inserir a outbox na mesma transação SQL da mudança de status, com rollback se a inserção falhar. |
| Cliente processa duas vezes o mesmo evento | Média | Médio | Documentar a semântica *at-least-once* e fornecer UUID estável em `X-Event-Id` para deduplicação no cliente. |
| Vazamento de uma secret permite assinatura indevida para um endpoint | Média | Alto | Secret exclusiva por endpoint, HMAC-SHA256, rotação com janela de 24 horas e revisão de segurança antes do deploy. |
| Crescimento da outbox reduz a eficiência do worker | Média | Médio | Índices por status e `created_at`, processamento em lotes pequenos; arquivamento de entregues será tratado em fase posterior. |

## Critérios de aceitação

1. Um usuário autenticado consegue criar, editar, excluir e listar configurações de webhook de um cliente, com filtros de status e secret gerada na criação.
2. URLs sem HTTPS são rejeitadas; cada endpoint tem secret própria e é possível rotacioná-la mantendo a anterior válida por 24 horas.
3. Para uma mudança de status com webhook elegível, pedido, histórico e evento da outbox são confirmados juntos; se o registro da outbox falhar, a mudança de status não é confirmada.
4. Para uma mudança sem endpoint interessado naquele status, nenhum evento é criado na outbox.
5. O worker separado coleta pendências a cada 2 segundos e entrega o JSON e os cinco headers definidos para o endpoint configurado.
6. O payload entregue é o snapshot do momento da transição, contém os campos definidos e não contém itens do pedido.
7. Cada entrega é assinada com HMAC-SHA256 e possui `X-Event-Id` único e persistente entre as tentativas do mesmo evento.
8. Falhas ou timeout de 10 segundos seguem exatamente os cinco intervalos definidos; após a última tentativa, o evento está na DLQ com payload, motivo e timestamp preservados.
9. Somente um usuário `ADMIN` consegue executar o replay, que deixa registro de auditoria da identidade do solicitante e devolve o evento à outbox como pendente.
10. A consulta de entregas devolve no máximo as últimas 100 tentativas com resultado, payload, resposta e duração.
11. O payload acima de 64 KB falha sem truncamento.
12. Com o worker único, transições consecutivas do mesmo pedido são entregues na ordem de criação da outbox.

## Estratégia de testes e validação

- Testar schemas e endpoints de configuração: autenticação, CRUD, validação de HTTPS, filtros de status, geração de secret e rotação com janela de 24 horas.
- Testar a transação de mudança de status com e sem webhooks elegíveis, incluindo o rollback quando a inserção da outbox falhar e a manutenção das regras atuais de estoque e transição.
- Testar o worker com endpoint HTTP controlado para verificar polling, payload snapshot, headers, assinatura HMAC-SHA256, timeout de 10 segundos e registro do histórico de entrega.
- Simular falhas consecutivas para validar a sequência completa de retries, a transferência para a DLQ e o replay por usuário `ADMIN`, incluindo auditoria e bloqueio de roles não administrativas.
- Validar duplicidade planejada: repetir uma entrega e confirmar que o mesmo `X-Event-Id` é mantido.
- Executar testes ponta a ponta da mudança de status até a entrega, medindo a latência contra a meta de menos de 10 segundos.
- Submeter a implementação à revisão de segurança de HMAC e geração/rotação de secrets antes do deploy.
