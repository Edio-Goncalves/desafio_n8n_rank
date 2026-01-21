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

