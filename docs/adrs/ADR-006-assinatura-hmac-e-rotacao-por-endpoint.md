# ADR-006: Assinatura HMAC e rotação de segredo por endpoint

## Status

Aceito.

## Contexto

Webhooks transportam dados de pedidos para fora da infraestrutura. O cliente precisa verificar a origem e integridade do corpo recebido. Um segredo único para toda a plataforma ampliaria o impacto de um vazamento.

## Decisão

Exigir URLs HTTPS e assinar o corpo bruto de cada requisição com HMAC-SHA256. A assinatura será enviada em `X-Signature`; `X-Timestamp` permitirá que clientes apliquem sua própria proteção contra replay; e `X-Webhook-Id` identificará o cadastro de destino.

Cada endpoint terá segredo próprio. A rotação emitirá um novo segredo e manterá o anterior válido por 24 horas para permitir a migração do consumidor. A geração e o uso dos segredos serão revisados pela área de segurança antes do deploy.

## Alternativas Consideradas

- Segredo global compartilhado: rejeitado porque um único vazamento comprometeria todos os clientes.
- Não assinar o payload e confiar somente em TLS: rejeitado porque TLS não fornece ao consumidor uma prova aplicável de que a mensagem foi emitida pela nossa plataforma.
- Revogar imediatamente o segredo anterior: rejeitado porque interromperia integrações durante a atualização do cliente.

## Consequências

**Benefícios:** autentica a origem e protege a integridade do conteúdo; limita o impacto de credenciais vazadas; permite rotação sem indisponibilidade planejada.

**Desvantagens:** acrescenta gestão segura de segredos e documentação de verificação para consumidores; durante 24 horas existem duas chaves válidas por endpoint.

**Trade-offs:** aceita-se a janela temporária de duas credenciais para obter uma rotação operacionalmente segura e reduzir risco de interrupções.
