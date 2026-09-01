# ADR-002: Worker separado com polling da outbox

## Status

Aceito.

## Contexto

Os eventos persistidos na outbox precisam ser entregues sem manter recursos HTTP da API ocupados. O requisito de negócio aceita uma latência inferior a 10 segundos. MySQL não oferece um mecanismo nativo equivalente a `LISTEN/NOTIFY` para acordar um processo externo quando uma linha é inserida.

## Decisão

Executar a entrega em um processo Node separado da API, com uma entrada própria em `src/worker.ts`. O worker terá seu próprio `PrismaClient`, apontando para o mesmo banco e `DATABASE_URL`, e consultará os eventos pendentes mais antigos a cada 2 segundos, em lotes pequenos.

## Alternativas Consideradas

- Executar o loop dentro da instância da API: rejeitado porque reinícios, escala e ciclo de vida da API passariam a afetar o processamento de entregas.
- Usar trigger de banco para notificar o processo: rejeitado porque triggers MySQL executam SQL, não fornecem notificação confiável para processos externos.
- Usar um broker reativo: postergado por acrescentar infraestrutura sem necessidade para a latência requerida.

## Consequências

**Benefícios:** isola carga e falhas de entrega da API; atende confortavelmente ao SLA de menos de 10 segundos; mantém implantação e operação simples.

**Desvantagens:** polling gera consultas periódicas mesmo sem eventos e acrescenta até aproximadamente 2 segundos de espera antes do processamento.

**Trade-offs:** consome pequenas leituras recorrentes no banco em troca de evitar um broker e de manter a entrega resiliente a reinícios da API.
