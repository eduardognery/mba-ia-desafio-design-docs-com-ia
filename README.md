# Desafio Técnico - Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

O desafio consiste em orientar um agente de IA a analisar o código-fonte de um sistema e a transcrição de uma reunião técnica em que foi discutida uma nova feture para o sistema. A partir desta análise, o agente de IA deve gerar um pacote completo de Design Docs, de forma a documentar formalmente a feature discutida na reunião.

O sistema em questão trata-se de um Order Management System (OMS). A reunião técnica transcrita discute um Sistema de Webhooks de Notificação de Pedidos. A documentação gerada pela IA inclui PRD, RFC, FDD e ADRs, bem como um arquivo de rastreabilidade entre documentos e transcrição/código.

## Ferramentas de IA utilizadas

O agente de IA utilizado foi o Codex, da OpenAI, utilizado com sua extensão do VSCode.

## Workflow adotado

Os prompts enviados ao agente orientavam a, primeiro, analisar o código como um todo, e depois, ler a transcrição da reunião. Após essa leitura, foi gerada a documentação. Para cada documento, foi enviado um prompt separado, listando os requisitos exigidos para cada um, bem como restrições ao agente.

Primeiro foram gerados os ADRs, de forma a formar o esqueleto do "como implementar". Após, foi gerado o RFC, para consolidar a proposta técnica em cima das decisões. Como os ADRS devem ser citados no RFC, este teve que ser gerado após os ADRs. Depois, foi gerado o FDD, para construir o desenho técnico em cida das decisões arquiteturais e a proposta de implementação já documentadas. Após, foi produzido o PRD, por este ser o de mais alto nível. Por fim, foi gerado um Tracker de rasterabilidade, de forma a mapear os documentos com o código e a transcrição.

## Iterações e ajustes

Foram necessárias 3 iterações de todos os documentos, sempre seguindo o fluxo descrito acima. Em cada iteração, foram adicionadas mais orientações explícitas para que a documentação gerada pelo agente cumprisse com os requisitos do desafio. Também foram adicionadas restrições para evitar que o agente gerasse mais conteúdo que o necessário.

Nas primeiras iterações, algumas seções não estavam padronizadas, e alguns requisitos não foram cumpridos. Além disso, o FDD gerado mencionava arquivos ainda inexistentes, que deveriam ser criados durante a implementação da feature. Também no FDD, alguns endpoints não apresentavam exemplos de payload. O primeiro PRD gerado não tinha acentuação e cedilhas, mesmo escrito em português.

Seções não-obrigatórias não eram consideradas como erros ou pontos a melhorar, desde que não mencionassem algo inexistente no código ou na transcrição.

## Prompts customizados

A seguir são apresentados os prompts da última iteração da documentação. Cada prompt tinha como objetivo gerar um item da documentação.

### Prompt de geração das ADRs:

```markdown
Você é um Arquiteto de Software Sênior responsável por registrar Architecture Decision Records (ADR).

Sua missão é:
- Analisar o código existente neste diretório
- Analisar o arquivo `TRANSCRICAO.md`, contendo a transcrição de uma reunião técnica acerca de uma nova feature do sistema deste diretório
- A partir desta análise, gerar entre 5 e 8 ADRs e salvá-las na pasta `docs/adrs/`, nomeados no formato `ADR-NNN-titulo-em-kebab-case.md`
  - ex: `ADR-001-outbox-no-mysql.md`

Cada ADR deve registrar uma decisão arquitetural isolada, com contexto e consequências.

De forma geral, um ADR deve responder: "Por que decidimos exatamente assim?"

Os ADRs gerados devem seguir o padrão MADR, com as seguites seções:
- Status
- Contexto
- Decisão
- Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível)
- Consequências (benefícios, desvantagens, e trade-offs)

Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.

O conjunto de ADRs deve cobrir, no mínimo, 5 decisões principais discutidas na reunião transcrita em `TRANSCRICAO.md`.

Restrições obrigatórias:
- NÃO implementar código
- NÃO alterar arquivos existentes
- NÃO fazer fork
- NÃO executar scripts
```

### Prompt de geração do RFC:

```markdown
Você é um Arquiteto de Software Sênior responsável por escrever um RFC (Request for Comments) descrevendo a proposta técnica de uma solução, submetida à equipe para revisão.

Sua missão é:
- Analisar o código existente neste diretório
- Analisar o arquivo `TRANSCRICAO.md`, contendo a transcrição de uma reunião técnica acerca de uma nova feature do sistema deste diretório
- A partir desta análise, gerar a RFC e salvá-la em `docs/RFC.md`

O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto. É um documento conciso, portanto, o detalhamento de implementação fica no FDD.

De forma geral, o RFC deve responder: "Como pretendemos resolver?"

O RFC deve incluir as seguintes seções:
- Metadados:
  - autor
  - status
  - data
  - revisores (participantes da reunião)
- Resumo executivo
- Contexto e problema
- Proposta técnica:
  - visão geral da solução
  - NÃO descer ao detalhe de implementação do FDD
- Alternativas consideradas:
  - pelo menos 2 alternativas reais discutidas e descartadas na reunião
  - cada uma com o trade-off que levou ao descarte
- Questões em aberto:
- pelo menos 2 pontos levantados na reunião e não decididos ou adiados
- Impacto e riscos
- Decisões relacionadas:
  - links para os ADRs correspondentes

O RFC deve referenciar os ADRs já escritos, salvos na pasta `docs/adrs/`.

Restrições obrigatórias:
- NÃO implementar código
- NÃO alterar arquivos existentes, com exceção de `docs/RFC.md`
- NÃO fazer fork
- NÃO executar scripts
```

### Prompt de geração do FDD:

```markdown
Você é um Arquiteto de Software Sênior responsável por escrever um FDD (Feature Design Document) descrevendo em detalhes como implementar uma solução.

Sua missão é:
- Analisar o código existente neste diretório
- Analisar o arquivo `TRANSCRICAO.md`, contendo a transcrição de uma reunião técnica acerca de uma nova feature do sistema deste diretório
- A partir desta análise, gerar a FDD e salvá-la em `docs/FDD.md`

O FDD é um documento mais técnico e precisa estar acionável o suficiente para um desenvolvedor pegar e começar a desenvolver.

De forma geral, o FDD deve responder: "Como construir, em detalhe?"

O FDD deve incluir as seguintes seções:
- Contexto e motivação técnica
- Objetivos técnicos
- Escopo e exclusões
- Fluxos detalhados:
  - criação do evento na outbox
  - processamento
  - retry
  - DLQ
- Contratos públicos:
  - mínimo de 4 endpoints HTTP.
  - Todos endpoints devem incluir:
    - exemplo de payload de request
    - exemplo de payload de response
    - headers
    - status codes
    - semântica
- Matriz de erros previstos com códigos no padrão `WEBHOOK_*`
- Estratégias de resiliência:
  - timeouts
  - retries
  - backoff
  - fallback
- Observabilidade:
  - métricas
  - logs
  - tracing
- Dependências e compatibilidade
- Critérios de aceite técnicos
- Riscos e mitigação
- Integração com o sistema existente:
  - mínimo 4 caminhos de arquivo reais do código base
  - descrever como o módulo de webhooks vai se integrar com cada um

Restrições obrigatórias:
- NÃO implementar código
- NÃO alterar arquivos existentes, com exceção de `docs/FDD.md`
- NÃO fazer fork
- NÃO executar scripts

Critérios de aceite do FDD:
- Arquivo existe e está em Markdown
- Contém todas as seções obrigatórias
- Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes
- Matriz de erros usa códigos com prefixo `WEBHOOK_`
- Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
- Seção "Observabilidade" cita métricas, logs e tracing
```

### Prompt de geração do PRD:

```markdown
Você é um Gestor de Produto experiente, responsável por escrever um PRD (Product Requirement Document).

Sua missão é:
- Analisar o código existente neste diretório
- Analisar o arquivo `TRANSCRICAO.md`, contendo a transcrição de uma reunião técnica acerca de uma nova feature do sistema deste diretório
- Identificar requisitos, restrições e decisões presentes EXCLUSIVAMENTE na transcrição e no código
- A partir desta análise, gerar a PRD e salvá-la em `docs/PRD.md`

O PRD descreve o problema, público, escopo e métricas de sucesso da feature discutida da reunião transcrita em `TRANSCRICAO.md`.

De forma geral, o PRD deve responder: "Por que e o quê?"

O PRD deve incluir as seguintes seções:
- Resumo e contexto da feature
- Problema e motivação
- Público-alvo e cenários de uso
- Objetivos e métricas de sucesso:
  - no mínimo 1 objetivo com métrica e meta quantitativa
- Escopo (incluso e fora de escopo)
  - A seção "Fora de escopo" deve listar explicitamente pelo menos 2 itens descartados ou adiados
- Requisitos funcionais (mínimo de 8)
- Requisitos não funcionais
- Decisões e trade-offs principais
- Dependências
- Riscos
  - no mínimo 2 riscos com probabilidade, impacto e mitigação
- Critérios de aceitação
- Estratégia de testes e validação

Restrições obrigatórias:
- NÃO implementar código
- NÃO alterar arquivos existentes, com exceção de `docs/PRD.md`
- NÃO fazer fork
- NÃO executar scripts

Critérios de aceite do PRD:
- Arquivo existe e está em Markdown
- Contém todas as seções obrigatórias
- Identifica no mínimo 8 requisitos funcionais discutidos na reunião
- Inclui pelo menos 1 objetivo com métrica e meta quantitativa
- Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
- Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação
```

### Prompt de geração do Tracker de rastreabilidade:

```markdown
Você é um Analista de Rastreabilidade Sênior, responsável por criar um Tracker de Rastreabilidade.

Neste contexto, um "Tracker de Rastreabilidade", ou apenas "Tracker", é uma tabela markdown que mapeia cada item registrado em documentos (requisitos, decisões, restrições ou trade-offs) à sua origem na transcrição, no código ou em outros documentos.

Sua missão é:
- Analisar o código existente neste diretório
- Analisar o arquivo `TRANSCRICAO.md`, contendo a transcrição de uma reunião técnica acerca de uma nova feature do sistema deste diretório
- Analisar os documentos na pasta `/docs`, e identificar referenciamento entre estes e o código e o arquivo `TRANSCRICAO.md`
- A partir desta análise, gerar um Tracker de Rastreabilidade e salvá-lo no arquivo `docs/TRACKER.md`

O Tracker funciona como uma referência cruzada: permite que qualquer leitor entenda de onde veio cada decisão, requisito ou restrição, e garante que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

De forma geral, o Tracker deve responder: "De onde veio cada coisa?"

Formato obrigatório da tabela:

| ID | Documento | Tipo | Conteúdo | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |

Onde:
- ID: identificador único do item (ex: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-002)
- Documento: arquivo onde o item aparece (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-...md`)
- Tipo: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, entre outros
- Conteúdo: descrição de uma linha do item
- Fonte: `TRANSCRICAO` ou `CODIGO`
- Localização:
  - para `TRANSCRICAO`, timestamp + nome do falante (ex: `[09:17] Diego`)
  - para `CODIGO`, caminho do arquivo (ex: `src/modules/orders/order.service.ts`)

Restrições obrigatórias:
- NÃO implementar código
- NÃO alterar arquivos existentes
- NÃO fazer fork
- NÃO executar scripts

Critérios de aceite do documento gerado:
- Arquivo existe e segue o formato de tabela definido:
  - ID
  - Documento
  - Tipo
  - Conteúdo (resumo)
  - Fonte
  - Localização
- Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- Pelo menos 70% das linhas têm Fonte = `TRANSCRICAO` com timestamp válido no formato `[hh:mm] Nome`
- Pelo menos 5 linhas têm Fonte = `CODIGO` com caminho de arquivo real
```

## Como navegar a entrega

A documentação gerada está salva na pasta `docs/`. O desenvolvedor (ou o agente de IA) que vai implementar a feature deve ler os documentos na seguinte ordem:

1. `PRD.md`: descreve o problema que a feature visa resolver, o escopo onde se encontra, e métricas de sucesso;
2. `RFC.md`: proposta técnica da solução do problema: abordagem geral, alternativas e questões em aberto;
3. `ADRs.md`: registro de cada decisão arquitetural isolada, com contexto e consequências;
4. `FDD.md`: especificação de implementação: fluxos, contratos, erros, integração com o código;

O `TRACKER.md` funciona como uma lista de referências dos documentos. Este arquivo é um guia de onde cada item mencionado nos documentos se localiza no código ou na transcrição da reunião.

A arquitetura final do sistema ficou como segue:

```
.
├── README.md                              (descrição do desafio)
├── TRANSCRICAO.md                         (transcrição da reunião técnica, não alterada)
├── docs/                                  (documentação gerada pelo agente de IA)
│   ├── PRD.md                             
│   ├── RFC.md                             
│   ├── FDD.md                             
│   ├── TRACKER.md                         
│   └── adrs/
│       ├── ADR-001-outbox-atomica-no-mysql.md
│       ├── ADR-002-worker-separado-com-polling.md
│       ├── ADR-003-ordem-por-pedido-com-worker-unico.md
│       ├── ADR-004-retry-limitado-e-dead-letter.md
│       ├── ADR-005-entrega-at-least-once-com-event-id.md
│       ├── ADR-006-assinatura-hmac-e-rotacao-por-endpoint.md
│       ├── ADR-007-payload-snapshot-na-criacao-do-evento.md
│       └── ADR-008-modulo-webhooks-e-reuso-de-padroes.md
├── src/                                   (arquivos originais do sistema, não alterados)
├── prisma/                                (arquivos originais do sistema, não alterados)
├── tests/                                 (arquivos originais do sistema, não alterados)
└── ... (demais arquivos do boilerplate, não alterados)
```
