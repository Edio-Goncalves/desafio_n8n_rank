# Desafio Técnico — Automação e Orquestração (n8n)

## Visão Geral

Este projeto implementa a **camada de automação e inteligência** do desafio técnico, utilizando o n8n como orquestrador central para ingestão, validação, processamento paralelo e consolidação de dados enviados pelo frontend.

O foco da solução não é apenas executar fluxos, mas **transformar dados brutos e variáveis em uma base estruturada**, pronta para análises avançadas e geração de insights por IA.

---

## Papel do n8n na Arquitetura

O n8n atua como:
- Orquestrador de fluxos complexos
- Controlador de paralelismo (fan-out / fan-in)
- Camada de validação e normalização de dados
- Base para agentes analíticos e geração de insights

O desenho do fluxo prioriza **previsibilidade, escalabilidade e tolerância a falhas**.

## Papel do n8n na Arquitetura

O n8n atua como:
- Orquestrador de fluxos complexos
- Controlador de paralelismo (fan-out / fan-in)
- Camada de validação e normalização de dados
- Base para agentes analíticos e geração de insights

O desenho do fluxo prioriza **previsibilidade, escalabilidade e tolerância a falhas**.

Além disso, o n8n é responsável por **retornar um payload estruturado e consistente para o frontend**, independentemente da variação ou qualidade dos dados de entrada.

De forma geral, o payload de saída segue um contrato pensado para fácil consumo pelo painel, contendo:
- Identificador do lote (`batchId`)
- Resultados agrupados por tipo de visualização (`kind`)
- Metadados necessários para exibição (título, contexto, categoria)
- Indicadores de status, warnings ou ausência de dados
- Estrutura previsível, mesmo quando determinados tipos não estão presentes

Essa abordagem reduz a complexidade no frontend e garante que a interface possa se concentrar apenas em **exibição e tomada de decisão**, sem necessidade de lógica defensiva adicional.

---

## Fluxo 1 — Orquestração Inicial e Preparação dos Dados

### Entrada de Dados

O fluxo inicia a partir de um **Webhook**, que recebe um ou múltiplos arquivos JSON enviados pelo painel.  
Todos os arquivos de uma submissão são tratados como um **lote único de processamento**, identificado por um `batchId`.

---

### Identificação de Lote (Batch ID)

Logo no início do fluxo é criado um `batchId`, que acompanha todos os arquivos durante a execução.

Esse identificador permite:
- rastreabilidade ponta a ponta
- consolidação segura no fan-in
- execução simultânea de múltiplos uploads sem conflito

---

### Fan-out (Split de Arquivos)

Os arquivos recebidos são separados em itens individuais para permitir:
- processamento paralelo
- isolamento de falhas
- escalabilidade para múltiplos arquivos

Cada arquivo passa a ser tratado como uma unidade independente dentro do mesmo lote.

---

### Normalização de Contrato

Cada item é normalizado para um contrato comum, contendo:
- identificadores e metadados do arquivo
- tipo de visualização (`kind`)
- payload bruto de dados

Caso o campo `kind` esteja ausente, ele é explicitamente definido como `unknown`, garantindo que **todo item seja roteável**, mesmo em cenários de dados incompletos.

---

### Validação de Payload

Após a normalização, cada item passa por validação de conteúdo.

O fluxo:
- identifica se há dados processáveis
- classifica o item como `ready` ou `no_data`
- adiciona warnings estruturados quando necessário

Arquivos sem dados **não interrompem o fluxo** e seguem adiante de forma controlada, preservando o paralelismo.

---

### Agrupamento por Tipo (`kind`)

Os itens são agrupados por tipo de visualização (`kind`), como:
- pie
- table
- bar
- area
- big_number
- unknown

Além do agrupamento genérico, o fluxo cria listas explícitas por tipo para facilitar:
- leitura do fluxo
- uso de condicionais
- execução modular por sub-workflows

---

### Controle de Fluxo e Fan-in Consistente

Para cada tipo de `kind`:
- se houver itens, eles são enviados ao sub-workflow correspondente
- se não houver, o fluxo gera um **stub controlado**

Essa estratégia garante:
- simetria entre fan-out e fan-in
- consolidação previsível
- redução de risco de timeout
- estrutura de saída consistente, independentemente dos dados enviados

---

### Modularização com Sub-workflows

Cada tipo de dado é processado em um **sub-workflow especializado**, mantendo o fluxo principal focado apenas em orquestração.

Essa abordagem:
- facilita manutenção
- permite evolução independente dos módulos
- simula uma arquitetura próxima de produção

---

## Intenção Arquitetural

Este fluxo foi desenhado para lidar com:
- múltiplos arquivos em paralelo
- variações de estrutura e ausência de dados
- necessidade de consolidação confiável
- base sólida para análises automatizadas e IA

A prioridade foi criar uma automação **robusta, extensível e observável**, e não apenas um fluxo funcional.

---

## Fluxo 2 — Subworkflows de Processamento, Análise e IA

Os subworkflows são responsáveis por **processar cada grupo de dados (`kind`) de forma independente**, transformando dados normalizados em informações estruturadas, analisáveis e prontas para geração de insights.

Todos os subworkflows seguem **o mesmo padrão base**, garantindo consistência, paralelismo e fan-in previsível no fluxo principal.

---

### Entrada dos Subworkflows

Cada subworkflow recebe um payload já agrupado por `kind`, contendo:
- `batchId` do lote
- Lista de itens do tipo correspondente (ex.: `areaItems`)
- Itens previamente normalizados e validados no fluxo principal para nao repetir validações globais

---

### Normalização e Split por Item

O grupo recebido é normalizado e, em seguida, dividido para que **cada arquivo/widget seja tratado individualmente**.

Essa abordagem permite:
- processamento paralelo por item
- isolamento de falhas
- melhor controle de fluxo e observabilidade

---

### Controle de Fluxo (Itens Válidos vs. Stubs)

Cada item passa por uma verificação:
- Se o item for um **stub**, `kind=unknown` ou `status` diferente de `ready`, ele é encaminhado para um fluxo `warn_<kind>`, que apenas preserva o item para manter a simetria do fan-in.
- Se o item for válido, ele segue para o pipeline de processamento.

Essa estratégia garante que **todos os itens do lote retornem**, mesmo quando não há dados processáveis.

---

### Parse de Dados (`parse_<kind>`)

Nesta etapa, os dados brutos são transformados em um formato **data-first**, amigável ao frontend e à análise.

Exemplo (para `area`):
- Deduplicação por data (mantendo o maior valor do dia)
- Ordenação temporal
- Geração de séries (`x`, `y`)
- Cálculo de métricas iniciais (crescimento absoluto, percentual, média diária)

O payload original é substituído por uma estrutura parseada, mais leve e previsível.

---

### Construção da Análise (`build_analysis_<kind>`)

Após o parse, cada item é enriquecido com uma camada de análise estruturada, contendo:
- Métricas consolidadas (`features`)
- Detecção de padrões básicos (crescimento, estabilidade, aceleração)
- Preview reduzido da série para leitura rápida
- Fallbacks automáticos quando dados parciais estão disponíveis

Essa camada prepara os dados para **consumo direto pelo frontend** e para **análise por IA**, sem depender do dataset completo.

---

### Geração de Insights com IA

Com os dados já estruturados, um modelo de linguagem é utilizado para:
- interpretar métricas e padrões
- identificar tendências relevantes
- gerar insights textuais claros e acionáveis

A IA atua apenas sobre **dados preparados**, evitando análises superficiais ou baseadas em dados inconsistentes.

---

### Agregação e Encaminhamento para o Workflow Consolidador

Após o processamento de cada item (incluindo análise estruturada, geração de insights por IA ou warnings), os resultados são **agregados dentro do próprio subworkflow**.

Em seguida, cada subworkflow envia sua saída para um **workflow consolidador dedicado**, responsável por receber todas as execuções paralelas.

Esse workflow consolidador:
- sincroniza os resultados provenientes de múltiplos subworkflows
- executa o fan-in global das execuções simultâneas
- constrói um **payload único, estruturado e previsível**

Somente após essa consolidação final o payload é preparado e disponibilizado para consumo pelo frontend.

---

### Intenção Arquitetural

Os subworkflows foram desenhados para:
- modularizar a lógica por tipo de dado (`kind`)
- permitir evolução independente de cada módulo
- manter paralelismo com fan-in controlado
- preparar dados de forma consistente para análise por IA e visualização

Essa abordagem reflete uma arquitetura próxima de produção, com separação clara entre **orquestração**, **processamento**, **análise**, **consolidação** e **apresentação**.

---

## Fluxo 3 — Consolidação Final e Fan-In Global (`fan-in_parse_output`)

Este fluxo é responsável por **finalizar a execução do lote**, realizando o **fan-in global** de todos os subworkflows executados em paralelo e gerando **um único payload final**, pronto para consumo pelo frontend.

Ele representa o ponto de convergência de toda a arquitetura.

---

### Entrada de Dados

Cada subworkflow envia sua saída de forma independente para este fluxo, sempre associada ao mesmo `batchId`.

Os dados chegam **parcialmente consolidados**, persistidos em uma tabela intermediária, onde:
- cada coluna representa um tipo de visualização (`pie`, `bar`, `area`, `table`, `big_number`, `unknown`)
- os valores são armazenados como **strings JSON**, conforme exigência do banco de dados

O fan-in só é considerado completo quando **todos os tipos esperados para o lote estiverem preenchidos**.

---

### Desserialização e Preparação do Fan-In  
**Code node:** `fan_in_deserialize_db_row`

Nesta etapa, o fluxo:
- recupera a linha correspondente ao `batchId`
- converte cada coluna armazenada como string JSON em estruturas nativas (arrays/objetos)
- aplica tratamento defensivo para valores nulos, vazios ou inválidos
- consolida todos os resultados parciais em uma única estrutura intermediária

---

### Flatten e Normalização Global  
**Code node:** `fan_in_build_frontend_payload`

Com os dados já unificados, o fluxo executa uma série de transformações finais:
   - Suporte a múltiplos formatos intermediários (`pieItems`, `parse_<kind>`, widgets diretos)
   - Conversão de tudo para uma lista única de widgets
   - Remoção de itens com `status: "ignore"`
   - Manutenção explícita de widgets `ready` e `no_data` (com warnings)
   - Garantia de campos mínimos consistentes para todos os widgets:

Essa etapa garante que **todos os widgets sigam o mesmo contrato**, independentemente da origem ou do caminho percorrido nos fluxos anteriores.

---

### Construção do Payload Final

Após a normalização, o fluxo gera o payload final no formato:

- Identificador do lote (`batchId`)
- Tipo fixo (`kind: "fan_in"`)
- Contagem total de widgets processados
- Lista completa de widgets prontos para renderização e análise

Esse payload é **único, previsível e consistente**, permitindo que o frontend consuma os dados sem lógica defensiva adicional.

---

### Intenção Arquitetural

Este fluxo fecha o ciclo de processamento ao:

- sincronizar execuções paralelas
- centralizar a consolidação final dos dados
- desacoplar persistência intermediária da entrega ao frontend
- garantir previsibilidade mesmo em cenários de dados incompletos

A separação entre **fan-out distribuído** e **fan-in centralizado** simula um padrão real de arquiteturas orientadas a eventos, comum em ambientes de produção escaláveis.
