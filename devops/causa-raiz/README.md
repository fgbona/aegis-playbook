---
nome: Causa-raiz de degradação de busca
descricao: Análise de causa-raiz de degradação num cluster Elasticsearch cruzando configuração, métricas e logs, separando causa de consequência e dizendo o que os dados não provam
versao: 1.0.0
tags: [elasticsearch, causa-raiz, incidentes, sre, diagnostico]
inputs:
  - nome: config
    descricao: YAML de configuração do cluster como está no repositório de infra (shards, réplicas, heap, jobs agendados, cache)
  - nome: metricas
    descricao: Tabela de métricas da janela do incidente, um ponto por intervalo (latência p99, docs indexados/s, heap usado %, cache hit %)
  - nome: logs
    descricao: Trecho dos logs nativos do Elasticsearch de um nó cobrindo a mesma janela das métricas
---

# Causa-raiz de degradação de busca

## Objetivo

Levar o modelo da lista de sintomas até a causa-raiz de uma degradação no Cerebro (Elasticsearch), cruzando três artefatos de fontes diferentes: a configuração versionada do cluster, as métricas da janela do incidente e os logs de um nó. A saída separa causa de consequência numa cadeia justificada por evidência literal, declara o que os dados não provam e propõe ações proporcionais, divididas em contenção e correção de fundo.

## Quando usar

- Busca lenta ou com resultados parciais no Cerebro, depois que o plantão já coletou config, métricas e logs.
- Pós-incidente, para montar a primeira versão da análise antes do postmortem.
- Não usar para triagem rápida: este prompt é caro e lento por desenho (modelo de raciocínio, 45 a 60 s). Para um primeiro olhar, use `nota-de-triagem`.

## Exemplo de uso

Entradas: [`exemplos/cerebro-config.yaml`](./exemplos/cerebro-config.yaml) em `{{config}}`, [`exemplos/cerebro-metricas.txt`](./exemplos/cerebro-metricas.txt) em `{{metricas}}` e [`exemplos/cerebro-logs.txt`](./exemplos/cerebro-logs.txt) em `{{logs}}`.

Saída real (Claude Sonnet 4.6), trecho das seções centrais:

```
CADEIA CAUSAL:
CAUSA: job reindex atrasado → CONSEQUÊNCIA: pressão de heap crescente → CONSEQUÊNCIA: GC old prolongado → CONSEQUÊNCIA: evicção de query cache → CONSEQUÊNCIA: latência de busca elevada → CONSEQUÊNCIA: circuit breaker + timeouts + resultados incompletos

- job reindex atrasado → pressão de heap crescente: "reindex task [88123] (scheduled 02:00) progress, created [3.8M]/[10M] docs (38%)" às 08:02, com heap_used_pct = 61% às 08:00 subindo para 79% às 09:00 enquanto indexed_docs_per_s dobra de 4100 para 9800 (inferência: aceleração do job aumentou carga de merge e indexing buffer).
- pressão de heap crescente → GC old prolongado: heap_used_pct = 88% às 09:30; log às 09:12 "gc][old] duration [1.1s], collections [2] in [60s], heap [6.3gb]->[5.9gb]/[8gb]"; às 09:58 "gc][old] duration [1.8s], collections [4] in [60s]" — GC old só aparece quando young não consegue liberar espaço suficiente, comportamento padrão JVM.
- GC old prolongado → evicção de query cache: "query cache eviction rate spiking, hit_ratio dropped to 0.29" às 09:58:47, precedido por cache_hit_pct = 41% às 09:30 e 29% às 10:00; GC old compacta heap e o Elasticsearch descarta entradas de cache para liberar memória, comportamento documentado do IndicesQueryCache.
- evicção de query cache → latência de busca elevada: cache_hit_pct cai antes de search_p99 atingir pico; sem cache, cada query executa fase de query completa no shard, confirmado por "slow query on shard [logs-2026.05][7] took [2380ms]" às 09:44 e "search took [5031ms] (timeout [5000ms])" às 09:58:46.
- latência de busca elevada → circuit breaker + timeouts + resultados incompletos: "CircuitBreakingException: [parent] Data too large, would be [7.9gb/8gb]" às 10:01:12; "search returned partial results: 11/12 shards succeeded" às 09:58:46; "all shards failed for index [logs-2026.05] on 3 of last 20 queries" às 10:05:50.

CAUSA-RAIZ:
O job de reindexação agendado para 02:00 (duração média esperada: 90 min, término previsto ~03:30) ainda estava em execução às 08:02, com apenas 38% de progresso, e continuou consumindo recursos até pelo menos 09:58. Essa sobreposição com o horário de pico de ingestão (indexed_docs_per_s dobrou de ~4100 para ~12400 entre 08:00 e 10:00) criou pressão contínua sobre heap, indexing buffer e thread pool de escrita, desencadeando a cadeia causal acima.
Confiança: alta para o atraso do job como gatilho; média para a aceleração do throughput de escrita como fator agravante (os dados mostram correlação temporal, mas a causa do aumento de indexed_docs_per_s não está nos artefatos).
Hipótese alternativa: pico orgânico de ingestão independente do job, que sozinho teria saturado o heap. Isso seria mais provável se os logs mostrassem o job concluído antes de 08:00 e o throughput de escrita subindo mesmo assim — o que os dados não mostram; o job ainda corria às 09:58:44 com apenas 41% de progresso, enfraquecendo essa hipótese.

```

A saída completa e a comparação entre cinco modelos estão em [`docs/checkpoints/03-causa-raiz.md`](../../docs/checkpoints/03-causa-raiz.md).

## Limitações conhecidas

- Logs de um único nó: o prompt instrui a não extrapolar para o cluster, mas a análise fica limitada ao que esse nó viu.
- Modelos pequenos erram fatos neste prompt (o Haiku 4.5 inventou o horário de início do job e um mecanismo de "24 segmentos"). Use modelo de raciocínio; a qualidade é verificada pelo juiz do Checkpoint 09.
- Modelos com raciocínio interno não limitado (Sonnet 5.5 no padrão, Gemini 3.8 Flash em `thinkingLevel: HIGH`) estouram o teto de tokens ou vazam o rascunho na saída. Ver curadoria.
- O limite de 30 a 45 linhas é respeitado pelo Gemini 3.1 Pro e levemente excedido pelo Sonnet 4.6 (49 linhas).
- Antes de enviar artefatos reais a um provedor externo, remova credenciais da config e corpos de query ou identificadores de cliente dos logs; o prompt não faz essa limpeza.
