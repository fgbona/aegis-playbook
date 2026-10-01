---
nome: Decisão de backpressure
descricao: Apoia a decisão de estratégia de backpressure num barramento de eventos comparando caminhos contra as restrições do time antes de recomendar, em formato de ADR
versao: 1.0.0
tags: [backpressure, arquitetura, filas, decisao, adr]
inputs:
  - nome: estado
    descricao: Estado atual do barramento (throughput sustentado, pico observado, retenção, consumidores)
  - nome: restricoes
    descricao: Restrições que a solução precisa respeitar (SLAs por consumidor, orçamento, regras não negociáveis)
  - nome: opcoes
    descricao: Caminhos que o time já colocou na mesa, um por linha
---

# Decisão de backpressure

## Objetivo

Apoiar uma decisão de arquitetura cara: como segurar a sobrecarga de um barramento de eventos quando a chegada passa da capacidade de entrega. O prompt obriga o modelo a classificar as restrições (eliminatórias e de peso), fazer as contas que o estado permite, comparar pelo menos três caminhos numa tabela com critérios fixos e só então recomendar, com o que foi descartado, riscos, mitigações e métricas de sucesso. A saída tem forma de ADR.

## Quando usar

- Decisão de backpressure, priorização de consumidores, filas de reprocessamento ou isolamento de tenant no Relay ou em qualquer barramento com consumidores de SLA diferente.
- Quando o time já tem opções na mesa e precisa de uma comparação honesta, não de uma resposta pronta.
- Não usar para incidente em andamento: é um prompt de decisão, lento e caro por desenho.

## Exemplo de uso

Entradas: [`exemplos/relay-cenario.txt`](./exemplos/relay-cenario.txt) em `{{estado}}`, [`exemplos/relay-restricoes.txt`](./exemplos/relay-restricoes.txt) em `{{restricoes}}` e [`exemplos/relay-opcoes.txt`](./exemplos/relay-opcoes.txt) em `{{opcoes}}`.

Saída real (Claude Sonnet 4.6), seção de recomendação:

```
RECOMENDAÇÃO

CAMINHO: Prioridade ao Sentinel + DLQ para Forge (ADIÇÃO — combinação das opções "dar prioridade ao Sentinel" e "guardar o que não foi processado numa fila separada").

JUSTIFICATIVA: É o único caminho que atende estruturalmente as duas restrições eliminatórias sem adicionar custo variável proporcional ao pico. O Sentinel recebe prioridade de consumo no Relay, protegendo o SLA de 60s sem novo componente de escalonamento. O Forge é roteado para DLQ durante o pico, eliminando competição por capacidade de consumo e preservando todas as mensagens (restrição de não-perda). O custo incremental é o armazenamento da DLQ, menor do que instâncias adicionais de consumidores sob pico, relevante dado o orçamento 8% acima do previsto. O acúmulo calculado de 210M mensagens em 25min precisa caber na DLQ; isso deve ser validado com o tamanho médio de mensagem antes do deploy.

VALE ENQUANTO: a taxa de drenagem da DLQ pelo Forge, após o pico, for suficiente para consumir o acumulado em menos de 15min. Se o tempo de drenagem medido em produção exceder 15min, reavaliar o throughput do consumidor Forge e considerar escalonamento automático exclusivo para ele.


```

A saída completa, com contas, tabela de comparação e a comparação entre quatro modelos, está em [`docs/checkpoints/04-decisao-backpressure.md`](../../docs/checkpoints/04-decisao-backpressure.md).

## Limitações conhecidas

- O modelo só calcula o que o estado permite; sem tamanho médio de mensagem e taxa de consumo por consumidor, "a retenção cobre o pico?" sai como indeterminado. Isso é desejado, mas significa que a decisão final ainda depende de medir esses dois dados.
- A saída costuma passar das 60 linhas pedidas com modelos Anthropic (85 na execução de referência) por causa da tabela larga.
- Modelos de raciocínio do Google (Gemini 3.1 Pro) truncaram a saída com `maxOutputTokens` de 8.000 e inventaram ferramentas e prazos ("lambdas", "1 sprint") com 20.000. Modelos pequenos interpretam mal opções como dead-letter queue. Use Claude Sonnet 4.6.
- As estimativas de custo e prazo são relativas (menor/maior); o prompt proíbe números sem premissa.
