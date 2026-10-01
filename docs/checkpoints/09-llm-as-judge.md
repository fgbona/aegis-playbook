# Checkpoint 09 — Gate de qualidade com LLM-as-judge

## Rubrica

Arquivo: [`devops/causa-raiz/rubrica-juiz.txt`](../../devops/causa-raiz/rubrica-juiz.txt). Quatro critérios, cada um de 0 a 2, total de 0 a 8:

| # | Critério | 2 | 1 | 0 |
|---|---|---|---|---|
| 1 | Causa-raiz correta | nomeia o job de reindexação travado como causa principal e cruza com a config (duração esperada × observada) | cita o job mas como um fator entre vários, ou acerta com erro de fato relevante | outra causa, só sintomas, ou análise truncada |
| 2 | Correlação × causa | cadeia explícita em que cache hit, timeouts e circuit breaker são efeitos, com evidência e precedência temporal por elo | separa em parte, ou um efeito classificado como causa, ou cadeia sem evidência | sintomas tratados como causa ou sem cadeia |
| 3 | Ação proporcional | contenção ataca o job (cancelar/throttlar) e correção de fundo evita recorrência, com reversibilidade | ações razoáveis mas desproporcionais em parte (só limpar cache, trocar o cluster, heap sem base) | ações desconectadas ou ausentes |
| 4 | Honestidade epistêmica | seção explícita de lacunas com ao menos duas das três reais, inferências marcadas | lacunas genéricas ou só uma | certeza sobre o que os dados não mostram, ou dados inventados |

**Corte:** total ≥ 6 e nenhum critério com 0. A rubrica traz os fatos de referência dos artefatos (o que o job deveria ter feito, a ordem real dos eventos, as três lacunas) para que o juiz confira horários e números em vez de confiar no texto avaliado, e regras de julgamento que limitam a nota quando há erro de fato, ordem causal invertida, ação que afrouxa proteção ou mecanismo inventado.

## Config com o juiz como gate

[`devops/causa-raiz/promptfooconfig.yaml`](../../devops/causa-raiz/promptfooconfig.yaml). O modelo sob teste é o Claude Sonnet 4.6 (referência do CP03). O juiz é o Gemini 3.8 Flash via `llm-rubric`, com `defaultTest.options.provider` apontando para o Google de propósito: fornecedor diferente do modelo sob teste, para o modelo não corrigir a própria prova. O juiz devolve JSON com `reason` (nota por critério e total), `pass` (o corte) e `score` (total/8); o `pass` do juiz é o que aprova ou reprova o caso. Não há limite de latência ou custo neste config: é um prompt de saída aberta, lento e caro por desenho, e o custo do gate aparece na curadoria.

## Calibração do juiz contra a minha pontuação

Usei as quatro saídas reais do CP03, pontuei cada uma à mão e depois rodei a rubrica em dois modelos candidatos a juiz, usando o provider `echo` do promptfoo para que a "saída" avaliada fosse exatamente o texto da amostra. Minha pontuação (C1 C2 C3 C4 = total):

| Amostra | Saída | Minha nota | Por quê |
|---|---|---|---|
| A | Claude Sonnet 4.6, v2 (a escolhida no CP03) | 2 2 2 2 = 8 | cruza config × log, cadeia na ordem certa, ações no job, três lacunas reais |
| B | Gemini 3.1 Pro, v2 | 1 2 2 2 = 7 | trata "job + pico de indexação" como causa dupla; o resto está certo |
| C | Claude Haiku 4.5, v2 | 1 1 1 1 = 4 | diz que o job começou às 08:02 e rodou 116 min; coloca o circuit breaker (09:58:46) como causa da rejeição da fila (09:58:41); propõe elevar o limite do breaker e reduzir cache; inventa "24 segmentos" e tamanho "padrão" de buffer |
| D | Gemini 3.8 Flash thinking HIGH (rascunho vazado, truncada) | 0 0 0 0 = 0 | ilegível |

**Rubrica v1** (critérios e corte, sem as regras de conferência):

| Juiz | A | B | C | D | Desvio máximo por critério |
|---|---|---|---|---|---|
| Gemini 3.8 Flash | 2 2 2 2 = 8 | 1 2 2 2 = 7 | 1 2 2 2 = **7** | 0 0 0 0 = 0 | 1 em C2, C3 e C4 da amostra C |
| Claude Sonnet 4.6 | 2 2 2 2 = 8 | 1 2 2 2 = 7 | 2 2 2 2 = **8** | 0 0 0 0 = 0 | 1 em todos os critérios da amostra C |

Os dois juízes aprovaram a amostra C, que eu reprovo. O Sonnet chegou a aceitar "116 min observado" como cruzamento correto com a config. O problema não era o critério, era a boa vontade: o juiz lia "cadeia causal com evidências" e dava 2 sem conferir se a ordem batia com os timestamps.

**Rubrica v2**, com as regras de julgamento que estão no arquivo: conferir cada horário e número contra os fatos de referência (e o lembrete de que 08:02 é o primeiro log, não o início do job); ordem invertida limita C2 a 1; ação que afrouxa proteção limita C3 a 1; mecanismo ou número inventado limita C4 a 1 e tira 1 de C1 se sustentar a causa.

| Juiz | A | B | C | D | Desvio máximo por critério |
|---|---|---|---|---|---|
| **Gemini 3.8 Flash** | 2 2 2 2 = 8 | 2 2 2 2 = 8 | **1 1 1 1 = 4** | 0 0 0 0 = 0 | **1** (só C1 da amostra B) |
| Claude Sonnet 4.6 | 2 2 2 2 = 8 | 1 2 2 2 = 7 | sem JSON (resposta cortada em 1.024 tokens) | 0 0 0 0 = 0 | n/d |

Com a v2 o Gemini bateu a minha nota na amostra C critério a critério, manteve A e D, e ficou 1 ponto acima em C1 da amostra B (ele aceitou a causa dupla; eu não). Está dentro da tolerância do enunciado (no máximo 1 ponto por critério) nas 16 notas. O Sonnet escreveu uma justificativa tão longa que estourou o limite de resposta do grader e o promptfoo não conseguiu extrair o JSON: numa pipeline isso vira falha espúria. Juiz escolhido: Gemini 3.8 Flash, pela calibração, pelo custo (centavos por julgamento) e por ser de outro fornecedor.

## Execução real do gate

Duas rodadas completas (`promptfoo eval --no-cache`, 2026-10-01): o Sonnet 4.6 gera a análise a partir dos artefatos do Cerebro e o juiz a avalia.

**Rodada 1:**

```
  ✓ 1 passed (100%)
  0 failed (0%)
modelo sob teste: 60954 ms, US$ 0.05 · juiz: 6439 tokens
veredito do juiz: pass=true score=0.88
C1=2 (identificou com precisão o reindex job atrasado cruzando config e métricas); C2=2 (cadeia causal explícita com ordem temporal correta e evidências); C3=1 (ações de contenção e correção adequadas, mas propôs valor de heap de até 30-31gb sem base na RAM do hardware); C4=2 (apresentou seção dedicada de lacunas cobrindo os limites reais dos dados); total=7
```

**Rodada 2:**

```
  ✓ 1 passed (100%)
  0 failed (0%)
modelo sob teste: 57588 ms, US$ 0.048 · juiz: 5519 tokens
veredito do juiz: pass=true score=0.88
C1=2 (identificou com precisão o job de reindexação travado cruzando a configuração de 90 min esperados com a execução prolongada); C2=2 (cadeia causal rigorosa com precedência temporal e evidências consistentes); C3=1 (ações de contenção excelentes, mas propor elevar o heap para 16 gb sem dados da memória total do host limita a nota); C4=2 (seção dedicada de lacunas cobrindo os limites reais dos dados e marcando inferências); total=7
```

Saída avaliada na rodada 1 (a análise que o juiz aprovou):

```
SINTOMAS OBSERVADOS:
- Latência p99 de busca saiu de 850 ms às 08:00 e chegou a 6700 ms às 10:00, aumento de ~8×.
- Cache hit caiu de 74% às 08:00 para 29% às 10:00, com log confirmando: "query cache eviction rate spiking, hit_ratio dropped to 0.29" às 09:58:47.
- Resultados incompletos registrados às 09:58:46: "search returned partial results: 11/12 shards succeeded", evoluindo para "all shards failed for index [logs-2026.05] on 3 of last 20 queries" às 10:05:50.
- Heap subiu de 61% às 08:00 para 94% às 10:00, com circuit breaker disparado às 09:58:46: "usage [7.7gb/8gb] (96%) over limit".

LINHA DO TEMPO:
- 02:00 — reindex job 88123 agendado para iniciar (config: schedule "0 2 * * *", avg_duration_min 90; término esperado ~03:30) — config
- 08:02 — job ainda em execução, 38% concluído (3,8M/10M docs), ~6h após início — log
- 08:14 — GC young de 620 ms, heap 4,9 gb → 3,1 gb; heap_used_pct = 61% às 08:00 — log/métrica
- 08:41 — throttling de indexação no shard [logs-2026.05][7]: "segment writing can't keep up" — log
- 09:00 — indexed_docs_per_s salta de ~4100 para 9800 (+139%); heap_used_pct = 79%; search_p99 = 2300 ms — métrica
- 09:12/09:31 — GC old de 1,1 s; circuit breaker parent em 86% (6,9 gb/8 gb) — log
- 09:58 — write queue cheia (200/200), bulk rejeitado; GC old 1,8 s (4 coletas/60 s); circuit breaker disparado (96%); shard [7] timeout (5031 ms > 5000 ms); resultados parciais — log
- 10:01/10:05 — CircuitBreakingException "Data too large, would be [7.9gb/8gb]"; 1284 bulks rejeitados em 5 min; falha total em 3/20 queries — log

CADEIA CAUSAL:
CAUSA: reindex job atrasado → CONSEQUÊNCIA: pico de indexação → CONSEQUÊNCIA: pressão de heap/GC → CONSEQUÊNCIA: evicção de cache → CONSEQUÊNCIA: timeouts e resultados incompletos

- reindex job atrasado → pico de indexação: "reindex task [88123] (scheduled 02:00) progress, created [3.8M]/[10M] docs (38%)" às 08:02, com indexed_docs_per_s saltando de 4100 para 9800 às 09:00; job deveria ter terminado ~03:30 (config avg_duration_min 90) mas ainda corria às 09:58:44 com 41% — log/config/métrica
- pico de indexação → pressão de heap/GC: throttling de shard às 08:41 ("segment writing can't keep up"), GC old com 4 coletas em 60 s às 09:58:43, heap_used_pct = 88% às 09:30 e 94% às 10:00 — log/métrica
- pressão de heap/GC → evicção de cache: "query cache eviction rate spiking, hit_ratio dropped to 0.29" às 09:58:47, precedido por heap em 94% e circuit breaker em 96%; cache_hit_pct caiu de 74% para 29% — log/métrica
- evicção de cache → timeouts e resultados incompletos: shard [7] levou 5031 ms (timeout 5000 ms) às 09:58:46, "search returned partial results: 11/12 shards succeeded", CircuitBreakingException às 10:01:12 — log

CAUSA-RAIZ:
Causa única: reindex job 88123 iniciado às 02:00 não concluiu no tempo esperado (~90 min, término previsto ~03:30 conforme config) e ainda executava às 09:58, sobrepondo-se ao horário de pico de ingestão operacional. Essa sobreposição gerou carga de indexação anormalmente alta (9800–12400 docs/s contra linha de base de ~4100), que esgotou o buffer de indexação, forçou GC old generation frequente e consumiu heap até o limite do circuit breaker, colapsando o query cache e bloqueando threads de busca.
Confiança: alta — há cruzamento direto entre config (duração esperada 90 min), logs (job ainda em 41% às 09:58, ~8h após início) e métricas (pico de indexação e heap correlacionados temporalmente com degradação de busca).
Hipótese alternativa: aumento orgânico de tráfego de ingestão sem relação com o reindex. Enfraquecida porque: (a) indexed_docs_per_s na linha de base (08:00–08:30) era estável em ~4100, compatível com operação normal; (b) o salto para 9800 às 09:00 coincide com o período em que o job ainda corria e o throttling já havia sido registrado; (c) o log de 09:20 confirma o job ativo. Tornaria a hipótese alternativa mais provável: métricas de ingestão de outros índices ou de outros nós mostrando crescimento independente do reindex — dado não disponível.

O QUE OS DADOS NÃO PERMITEM CONCLUIR:
- Motivo do atraso do job (duração real ~8h+ vs. esperada 90 min) → logs de execução do job desde 02:00, métricas de I/O de disco e de merge durante a madrugada fechariam essa lacuna.
- Estado dos demais 11 nós do cluster → logs e métricas dos outros nós; os dados cobrem apenas cerebro-node-3, e a falha de 1/12 shards pode ter origem em outro nó.
- Se o shard [logs-2026.05][7] é primário ou réplica em cerebro-node-3 → configuração de alocação de shards ou API cat/shards; isso afeta o impacto real da rejeição de bulk nesse nó.
- Se houve aumento real de tráfego de busca externo no período → métricas de requisições por segundo no load balancer ou no coordinating layer; ausentes nos artefatos.
- Se o circuit breaker disparou em outros nós → logs dos demais nós; resultados parciais (11/12 shards) sugerem que sim, mas não há evidência direta.

AÇÕES:

Contenção agora:
- Cancelar o reindex job 88123 via API POST _tasks/88123/_cancel: remove imediatamente a carga de indexação extra, aliviando heap e GC; reverter reiniciando o job fora do horário de pico (ex.: 23:00) após confirmar que o índice de destino está íntegro. Risco: dados parcialmente reindexados (41%) serão descartados; avaliar se o índice de destino pode ser usado no estado atual antes de cancelar.
- Reduzir temporariamente o refresh_interval de "1s" para "30s" no índice logs-2026.05 via PUT logs-2026.05/_settings: diminui a frequência de flush de segmentos, reduzindo pressão de heap e GC enquanto a carga de indexação ainda estiver alta; reverter para "1s" após normalização do heap (abaixo de 70%). Sem risco de perda de dados; afeta apenas a visibilidade de documentos novos na busca.
- Ativar throttling explícito de ingestão reduzindo indices.store.throttle.max_bytes_per_sec ou limitando o cliente de ingestão para no máximo 4000 docs/s (linha de base): dá ao GC tempo para recuperar heap sem rejeitar bulks; reverter elevando o limite gradualmente após heap estabilizar. Atenção: se o cliente não suportar backpressure, bulks continuarão sendo rejeitados pelo thread pool.
- Se heap permanecer acima de 90% após as ações acima, forçar um GC via POST _nodes/cerebro-node-3/hot_threads seguido de análise, e considerar reinício controlado do nó com rolling restart (um nó por vez para não perder réplicas): libera heap imediatamente; irreversível no sentido de que reiniciar descarta caches em memória — avisar equipe antes de executar.

Correção de fundo:
- Ajustar o horário do reindex job para fora do horário de pico e adicionar um mecanismo de auto-cancelamento: se o job ultrapassar avg_duration_min × 2 (180 min), cancelar automaticamente e alertar; ataca o elo "job atrasado → sobreposição com pico". Implementar via script de wrapper no agendador (cron ou scheduler do ES).
- Investigar e corrigir a causa do atraso do job (I/O de disco, volume de dados, ausência de throttle no reindex): configurar requests_per_second no reindex API para limitar a taxa (ex.: 500 docs/s) e evitar que o job consuma heap desproporcionalmente; ataca o elo "pico de indexação → pressão de heap".
- Aumentar o heap JVM de 8 gb para o máximo recomendado para o hardware disponível (até 30–31 gb para evitar compressed oops overhead, ou o valor adequado ao nó), respeitando a regra de não ultrapassar 50% da RAM física: amplia a margem antes de o circuit breaker disparar; ataca o elo "pressão de heap → evicção de cache e circuit breaker". Atenção: alteração requer reinício do nó; planejar rolling restart.
- Configurar alertas proativos em heap_used_pct > 75% e GC old duration > 500 ms, e monitorar o progresso do reindex job a cada 15 min com alerta se progresso < 10% por hora: permite intervenção antes de o circuit breaker disparar; ataca todos os elos da cadeia ao reduzir o tempo de detecção.
```

## Curadoria

**O que ajustei na rubrica:** só as regras de julgamento. Os quatro critérios e o corte são exatamente os do enunciado. A calibração mostrou que critério bem descrito não basta: sem instrução de conferir fatos e precedência, o juiz premia texto bem estruturado, e a saída do Haiku é bem estruturada e errada. Isso é o risco central de LLM-as-judge e a razão de o enunciado pedir calibração antes de ligar o gate.

**Como o gate roda sozinho:** o config fica ao lado do prompt; o workflow do Checkpoint 10 o executa a cada push e pull request. Se o prompt de causa-raiz mudar e a análise cair abaixo de 6, ou zerar um critério, o build falha.

**Custo e não-determinismo:** cada execução do gate custa cerca de US$ 0,05 no modelo sob teste mais menos de US$ 0,01 no juiz, e leva 60 a 90 s. O juiz é não-determinístico; nas duas rodadas e nas calibrações as notas da saída de referência não variaram, mas a amostra B oscilou 1 ponto em C1 entre as versões. O desenho do Checkpoint 10 trata isso com um rerun antes de reprovar o build. O que eu não faria: baixar o corte para 5 para "dar folga", porque a amostra C (nota 4) é exatamente o tipo de saída que o gate existe para barrar, e com corte 5 uma flutuação de 1 ponto a deixaria passar.

**Limite conhecido:** a rubrica carrega os fatos de referência do incidente do Cerebro, então ela só vale para esse caso de teste. Um segundo incidente exigiria uma segunda rubrica com seus próprios fatos. Trocar os fatos por "confira contra os artefatos" tornaria o juiz genérico, mas a calibração mostrou que é a conferência contra fatos explícitos que o mantém honesto.
