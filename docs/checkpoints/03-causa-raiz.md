# Checkpoint 03 — Causa-raiz da degradação no Cerebro

## 1. Prompt parametrizável

Está em [`devops/causa-raiz/prompt.md`](../../devops/causa-raiz/prompt.md). Três parâmetros, cada um numa tag XML própria: `{{config}}` (YAML do cluster), `{{metricas}}` (tabela da janela) e `{{logs}}` (trecho de um nó). O prompt fixa o método de análise (logs → métricas → config → cruzamento → hipótese alternativa), as regras de evidência literal e precedência temporal, e seis seções de saída em ordem fixa, terminando em ações separadas por contenção e correção de fundo.

## 2. Execução

**Modelo:** Claude Sonnet 4.6 (`claude-sonnet-4-6`, Anthropic), sem raciocínio estendido.

**Por que esse modelo:** aqui a tarefa é raciocínio causal sobre três fontes, e o erro que o enunciado quer evitar é o modelo listar sintomas sem dizer o que causou o quê. Testei cinco configurações (tabela na curadoria). O Sonnet 4.6 foi o único que fez o cruzamento config × log de forma explícita ("config declara avg_duration_min: 90; às 08:02 o job estava em 38% após ~6 h"), dividiu a confiança entre o elo forte e o elo fraco, e listou cinco lacunas reais nos dados. Custou US$ 0,050 e levou 62 s; caro para um prompt de triagem, aceitável para um prompt que roda uma vez por incidente. Não há limite de latência ou custo para este prompt no Checkpoint 08, porque ele é de saída aberta e vai para o juiz do Checkpoint 09.

**Resultado real** (promptfoo, `--no-cache`, 2026-10-01, v2 do prompt, 62 s, US$ 0,050):

```
SINTOMAS OBSERVADOS:
- Latência p99 de busca saiu de 850 ms às 08:00 e chegou a 6700 ms às 10:00, aumento de ~8×.
- Cache hit caiu de 74% às 08:00 para 29% às 10:00, com log confirmando: "query cache eviction rate spiking, hit_ratio dropped to 0.29" às 09:58:47.
- Resultados incompletos registrados às 09:58:46: "search returned partial results: 11/12 shards succeeded", evoluindo para "all shards failed for index [logs-2026.05] on 3 of last 20 queries" às 10:05:50.
- Heap subiu de 61% às 08:00 para 94% às 10:00, com circuit breaker disparado às 09:58:46: "usage [7.7gb/8gb] (96%) over limit".

LINHA DO TEMPO:
- 02:00 — reindex job 88123 agendado para iniciar (config: schedule "0 2 * * *", avg_duration_min 90; término esperado ~03:30) — config
- 08:02 — job ainda em execução, apenas 38% concluído (3,8 M/10 M docs), ~6 h após início — log
- 08:14 — primeiro GC young longo: 620 ms, heap 4,9 gb → 3,1 gb; heap_used_pct = 61% às 08:00 — log + métrica
- 08:41 — throttling de indexação no shard [logs-2026.05][7]: "segment writing can't keep up" — log
- 09:00 — indexed_docs_per_s salta de ~4100 para 9800 (primeiro desvio expressivo de throughput de escrita); heap_used_pct = 79%; search_p99 = 2300 ms — métrica
- 09:12 — primeiro GC old: 1,1 s, 2 coletas em 60 s; às 09:31 circuit breaker parent em 86% — log
- 09:58 — write thread pool full (200/200), bulk rejections, GC old 1,8 s (4 coletas/60 s), circuit breaker disparado (96%), shard timeout 5031 ms, resultados parciais — log
- 10:01–10:05 — CircuitBreakingException em buscas, 3/20 queries com todos os shards falhando — log

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

O QUE OS DADOS NÃO PERMITEM CONCLUIR:
- Por que o job levou mais de 6 h em vez de ~90 min → logs de progresso entre 03:30 e 08:02 (ausentes) e métricas de I/O de disco e merge rate do período noturno fechariam essa lacuna.
- Se o aumento de indexed_docs_per_s a partir de 09:00 é carga orgânica ou aceleração do próprio job → breakdown de throughput separando bulk do reindex e bulk de produção, não disponível nos artefatos.
- Estado dos demais nós do cluster → os logs cobrem apenas cerebro-node-3; não é possível afirmar se outros nós sofreram pressão similar ou absorveram shards.
- Se o shard que falhou (1/12 ausente em 09:58:46) é réplica ou primário → informação de alocação de shards não está nos artefatos; isso determina se houve perda de dados ou apenas degradação de disponibilidade.
- Se o circuit breaker foi acionado por memória de busca, de indexação ou de field data → o log indica "parent" breaker, mas não detalha qual sub-breaker filho disparou primeiro.

AÇÕES:

Contenção agora:
- Cancelar ou pausar o job reindex 88123 via API (_tasks/88123/_cancel ou _reindex com wait_for_completion=false seguido de cancel): libera indexing buffer, reduz pressão de heap e permite que GC old recupere espaço; reverter reiniciando o job em janela de baixa carga. Risco: progresso de 41% será perdido se o job não suportar retomada — verificar se a implementação usa slice e ponto de retomada antes de cancelar.
- Reduzir temporariamente o refresh_interval de "1s" para "30s" no índice logs-2026.05 via PUT /logs-2026.05/_settings: diminui frequência de flush de segmentos, alivia merge pressure e libera heap de indexing buffer; reverter com PUT de volta a "1s" após estabilização. Ação reversível.
- Forçar roteamento de buscas para réplicas saudáveis com preference=_replica nas queries críticas enquanto o nó-3 está sob pressão: reduz carga de busca no nó afetado sem alterar configuração de cluster; reverter removendo o parâmetro. Ação reversível, sem efeito colateral estrutural.

Correção de fundo:
- Mover o schedule do reindex para uma janela que garanta término antes do pico de ingestão, considerando a duração real observada (mais de 8 h) e não a média configurada (90 min): ataca o elo "job atrasado → sobreposição com pico". Investigar a causa do atraso (I/O, throttling, volume de dados) antes de redefinir o horário.
- Configurar throttling explícito no reindex via parâmetro requests_per_second (ex.: iniciar com 500 req/s e ajustar) para limitar a taxa de ingestão do job e evitar que ele compita com tráfego de produção pelo heap e pelo thread pool de escrita: ataca o elo "pressão de heap crescente → GC old".
- Aumentar o heap JVM de 8 g para o máximo recomendado (26 g, limite antes de compressed oops deixar de funcionar) se o host comportar, ou distribuir shards primários entre mais nós para reduzir heap por nó: ataca o elo "GC old prolongado → evicção de cache". Atenção: aumentar heap além de 32 g desativa compressed oops e pode piorar GC — manter abaixo de 26 g é regra documentada do Elasticsearch; ação requer reinício do nó, planejar com rolling restart.
- Adicionar alerta em heap_used_pct maior que 75% por mais de 10 min e em indexed_docs_per_s acima de 2× a linha de base das 08:00, com acionamento automático de throttling do reindex: fecha a lacuna de detecção precoce, pois o primeiro sinal claro (heap 79% às 09:00) ocorreu ~1 h antes do circuit breaker, tempo suficiente para intervenção manual ou automática.
```

Checagem humana contra o que os artefatos mostram: a causa apontada é o job de reindexação das 02:00 que deveria ter terminado às 03:30 e ainda estava em 41% às 09:58, saturando o heap; circuit breaker, timeouts e queda do cache aparecem como consequência, na ordem certa. O modelo marcou como inferência o único elo que os dados não fecham (se o salto de docs/s é o job acelerando ou carga orgânica) e repetiu isso na seção de lacunas. As ações atacam o job (cancelar ou throttlar) e não só o efeito (limpar cache).

## 3. Curadoria

**Técnica de criação:** meta-prompting. O meta-prompt (no final) descreveu os três artefatos, as sete garantias de conteúdo (linha do tempo, cadeia causal com evidência por seta, uma causa-raiz com confiança, cruzamento config × log, honestidade epistêmica, ações proporcionais, não inventar) e o formato de seis seções. O Sonnet 5.5 gerou a v1 em uma rodada.

**Framework do prompt gerado:** papel + método em passos numerados + regras + formato fixo. É a estrutura que o desafio chama de R-I-S-E, com o "S" (steps) carregando o peso: a ordem logs → métricas → config → cruzamento é o que impede o modelo de pular direto para "heap alto".

**O que refinei (v1 → v2):**

1. Placeholders `{{CONFIGURACAO_YAML}}`, `{{METRICAS_TABELA}}` e `{{LOGS_ELASTICSEARCH}}` viraram `{{config}}`, `{{metricas}}` e `{{logs}}`, convenção do template.
2. Na v1, o Sonnet 4.6 respondeu com cabeçalhos markdown, negrito e um bloco de código, e passou de 100 linhas apesar do limite de 30 a 45. O Gemini 3.1 Pro respeitou. A v2 ganhou duas regras de forma: "sem markdown, rótulo em maiúsculas seguido de dois pontos" e "linha do tempo com no máximo 8 bullets, eventos do mesmo minuto agrupados". Na v2 o Sonnet 4.6 caiu para 49 linhas, sem markdown.
3. Sonnet 5.5 não serve para este prompt via API sem controle do raciocínio: no padrão gastou os 8.000 tokens inteiros raciocinando e não respondeu; com `budget_tokens: 4000` raciocinou 11.719 tokens e truncou de novo; `thinking: disabled` é rejeitado pelo modelo. Registrei e segui com o 4.6.
4. Gemini 3.8 Flash com `thinkingLevel: HIGH` vazou o rascunho de raciocínio (em inglês, com "Drafting Line Count Check") dentro da resposta e truncou em 17 linhas. Modo descartado para prompts de saída longa.
5. Haiku 4.5 entregou texto convincente com erros de fato: afirmou que o job começou às 08:02 (o log diz `scheduled 02:00`), calculou "116 min depois", disse que não havia bulk de clientes quando o log mostra `rejecting bulk`, e inventou um mecanismo de "24 segmentos sendo reescritos". É o caso clássico de modelo pequeno em tarefa de raciocínio: formato certo, conteúdo errado. É exatamente o que o juiz do Checkpoint 09 precisa pegar.

**Comparação de modelos** (mesmos artefatos; v1 e v2 do prompt):

| Modelo | Versão | Saída | Latência | Custo | Veredito |
|---|---|---|---|---|---|
| Gemini 3.1 Pro | v1 | 30 linhas, formato ok | 36 s | US$ 0,059 | Correto, causa dupla (job + pico) |
| Claude Sonnet 4.6 | v1 | >100 linhas, markdown | 84 s | US$ 0,067 | Correto, ignorou o limite de tamanho |
| Claude Sonnet 5.5 | v1 | vazio (só raciocínio) | 67 a 95 s | n/d | Inutilizável sem controle de thinking |
| Gemini 3.1 Pro | v2 | 34 linhas | 46 s | US$ 0,076 | Correto, respeita o formato |
| **Claude Sonnet 4.6** | **v2** | **49 linhas** | **62 s** | **US$ 0,050** | **Escolhido: melhor cruzamento config × log e lacunas** |
| Gemini 3.8 Flash (thinking HIGH) | v2 | 17 linhas, rascunho vazado | 29 s | US$ 0,032 | Inutilizável |
| Claude Haiku 4.5 | v2 | 61 linhas | 30 s | US$ 0,016 | Erros de fato |

**Sanitização:** os artefatos trazem nome de nó (`cerebro-node-3`), nome de índice (`logs-2026.05`) e id de task. Mantive os três porque a análise precisa citá-los (o shard 7 do índice aparece em todos os eventos críticos). O que eu removeria antes de mandar artefatos reais para um provedor externo: credenciais ou endpoints no YAML (aqui não há), corpos de query e identificadores de tenant dentro dos logs (aqui não há), e IPs de nós. A regra que fica no README: o prompt não limpa nada; a limpeza é passo do plantonista antes de colar.

---

### Meta-prompt usado (registro do caminho, não entra na biblioteca)

```
Você é um engenheiro de prompts sênior que escreve prompts para um playbook de operações de SRE. Os prompts são reutilizados pelo time toda vez que uma degradação parecida aparece, trocando só o pacote de entrada, então precisam ser parametrizáveis, autocontidos e previsíveis.

Escreva o PROMPT FINAL (não execute a tarefa) para o caso abaixo.

## O problema que o prompt resolve
Um sistema de busca (serviço Java sobre Elasticsearch) começou a devolver buscas lentas e resultados incompletos. O plantonista junta três artefatos de fontes diferentes e cola no prompt:
1. a CONFIGURAÇÃO do cluster (YAML versionado no repositório de infra: shards, réplicas, heap da JVM, job de reindexação agendado, cache de query);
2. as MÉTRICAS das últimas horas em formato tabular (latência p99 de busca, docs indexados por segundo, percentual de heap usado, taxa de acerto do cache), um ponto a cada 30 minutos;
3. um trecho de LOGS nativos do Elasticsearch de um nó, cobrindo a mesma janela das métricas.

O prompt precisa levar o modelo a raciocinar até a CAUSA-RAIZ cruzando os três artefatos, e não só listar sintomas. É exatamente o erro que queremos evitar: um modelo que descreve "heap alto, cache caindo, buscas lentas" sem dizer o que causou o quê.

## O que o prompt precisa garantir na resposta
1. Reconstruir a linha do tempo a partir dos logs e das métricas, com horários, mostrando a degradação se formando.
2. Separar CAUSA de CONSEQUÊNCIA de forma explícita: uma cadeia causal "X → Y → Z" em que cada seta é justificada por uma evidência citada literalmente (linha de log ou valor de métrica com horário). Efeitos colaterais (ex.: cache hit caindo, timeouts) devem ser classificados como consequência, não como causa, quando a evidência apontar isso.
3. Apontar UMA causa-raiz principal, com nível de confiança (alta/média/baixa) e os fatos que a sustentam. Se houver hipótese alternativa plausível, listá-la com o que a tornaria mais provável.
4. Cruzar a configuração com os logs: por exemplo, se um job agendado deveria ter terminado num horário segundo a config e os logs mostram que ainda roda, isso é evidência central.
5. Honestidade epistêmica: uma seção explícita do que os dados NÃO permitem concluir (ex.: por que o job está lento, se outros nós estão no mesmo estado) e quais dados fechariam a lacuna.
6. Ações proporcionais ao diagnóstico, separadas em "contenção agora" (reversível, para parar a degradação) e "correção de fundo" (para não repetir). Sem ação desproporcional (ex.: trocar o cluster inteiro) nem subdimensionada (ex.: só limpar cache).
7. Nunca inventar números, horários ou linhas de log que não estejam nos artefatos. Nunca presumir comportamento do sistema além do que o Elasticsearch faz por padrão.

## Requisitos de forma do prompt
- Em português do Brasil.
- Exatamente três parâmetros: `{{config}}`, `{{metricas}}` e `{{logs}}`, cada um dentro de uma tag XML própria. Nenhum outro placeholder.
- Deve definir papel, método de análise em passos (que artefato ler primeiro, o que cruzar com o quê), as regras de "não invente" e um formato de saída fixo com seções nomeadas, nesta ordem: SINTOMAS OBSERVADOS, LINHA DO TEMPO, CADEIA CAUSAL, CAUSA-RAIZ, O QUE OS DADOS NÃO PERMITEM CONCLUIR, AÇÕES (contenção agora / correção de fundo).
- A saída deve ser objetiva: cerca de 30 a 45 linhas, sem introdução nem conclusão genérica.
- Conciso: o prompt inteiro em cerca de 45 linhas.

Responda apenas com o texto do prompt, sem comentários antes ou depois, sem cercas de código ao redor do prompt inteiro.
```
