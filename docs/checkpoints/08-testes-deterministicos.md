# Checkpoint 08 — Testes determinísticos com promptfoo

## Entrega

Três configs, um por prompt de saída estruturada, cada um ao lado do seu `prompt.md`:

| Prompt | Config | Casos | Asserts específicos |
|---|---|---|---|
| nota-de-triagem | [`devops/nota-de-triagem/promptfooconfig.yaml`](../../devops/nota-de-triagem/promptfooconfig.yaml) | os 3 alertas crus do CP02 | cinco rótulos (`contains-all`), handle `@time` (`regex`), ≤ 8 linhas (`javascript`) |
| triagem-de-pods | [`devops/triagem-de-pods/promptfooconfig.yaml`](../../devops/triagem-de-pods/promptfooconfig.yaml) | as 3 entradas do CP01 | por entrada: pod e causa certos (`contains`, `contains-all`, `regex`); na saudável, "Nenhum" e nenhum pod classificado (`regex`, `not-contains`, `javascript`) |
| networkpolicy-sentinel | [`devops/networkpolicy-sentinel/promptfooconfig.yaml`](../../devops/networkpolicy-sentinel/promptfooconfig.yaml) | manifesto + regras + mapa do CP06 | começa em `apiVersion` e tem `kind: NetworkPolicy` (`regex`, `contains`), `policyTypes` com Ingress e Egress (`regex`), sem `- {}` (`not-contains`), 5432, 9200 e `app: relay` (`contains-all`), todo `from`/`to` com comentário na linha de cima (`javascript`) |

Todos os três carregam os dois limites operacionais do playbook em `defaultTest`: `latency` ≤ 5000 ms e `cost` ≤ US$ 0,01 por chamada. Providers: Gemini 3.8 Flash (Google) e Claude Haiku 4.5 (Anthropic), os dois fornecedores do playbook; o esqueleto do enunciado usava `openai:gpt-4o-mini`, trocado porque as chaves com billing são Google e Anthropic. Rodar: `cd devops/<prompt> && promptfoo eval -c promptfooconfig.yaml`.

## Execução real (`promptfoo eval --no-cache`, 2026-10-01)

### nota-de-triagem — 6/6

```
Total Tokens: 8,263
  ✓ 6 passed (100%)
  0 failed (0%)
  0 errors (0%)
Duration: 4s (concurrency: 4)
```

| Provider | Caso | Resultado | Latência | Custo | Asserts que falharam |
|---|---|---|---|---|---|
| gemini-3.8-flash | alerta-1 Sentinel autoscaler no teto | ✅ pass | 1786 ms | US$ 0.0015 | — |
| claude-haiku-4-5 | alerta-1 Sentinel autoscaler no teto | ✅ pass | 2522 ms | US$ 0.0022 | — |
| claude-haiku-4-5 | alerta-2 Relay rejeitando ingestão | ✅ pass | 1429 ms | US$ 0.002 | — |
| gemini-3.8-flash | alerta-2 Relay rejeitando ingestão | ✅ pass | 1544 ms | US$ 0.0012 | — |
| gemini-3.8-flash | alerta-3 Forge com lag | ✅ pass | 1296 ms | US$ 0.0013 | — |
| claude-haiku-4-5 | alerta-3 Forge com lag | ✅ pass | 2427 ms | US$ 0.0021 | — |

### triagem-de-pods — primeira rodada 3/6, depois 12/12

Primeira rodada:

```
  ✓ 3 passed (50.00%)
  ✗ 3 failed (50.00%)
  0 errors (0%)
```

| Provider | Caso | Resultado | Latência | Custo | Asserts que falharam |
|---|---|---|---|---|---|
| claude-haiku-4-5 | entrada-1 pod reiniciando por OOM | ✅ pass | 3773 ms | US$ 0.0035 | — |
| gemini-3.8-flash | entrada-1 pod reiniciando por OOM | ✅ pass | 3965 ms | US$ 0.0023 | — |
| gemini-3.8-flash | entrada-2 dois pods que não sobem | ❌ fail | 3481 ms | US$ 0.0024 | regex |
| claude-haiku-4-5 | entrada-2 dois pods que não sobem | ❌ fail | 4304 ms | US$ 0.0038 | regex |
| gemini-3.8-flash | entrada-3 tudo saudável | ❌ fail | 5285 ms | US$ 0.0015 | latency |
| claude-haiku-4-5 | entrada-3 tudo saudável | ✅ pass | 3083 ms | US$ 0.0026 | — |

As duas falhas de `regex` na entrada 2 eram **bug do teste**, não do prompt: escrevi `(?i)insufficient|cpu` e o promptfoo avalia regex com o motor JavaScript, que não aceita o flag inline `(?i)`. A expressão inválida falha sempre. Corrigi para `Insufficient|cpu` (as saídas trazem literalmente `Insufficient cpu`). A falha de `latency` do Gemini na entrada 3 (5.285 ms) foi o primeiro sinal de que esse modelo roda na borda do limite.

Depois da correção, duas rodadas seguidas:

| Rodada | Provider | Caso | Resultado | Latência | Custo | Asserts que falharam |
|---|---|---|---|---|---|---|
| 1 | gemini-3.8-flash | entrada-1 pod reiniciando por OOM | ✅ pass | 2865 ms | US$ 0.0022 | — |
| 1 | claude-haiku-4-5 | entrada-1 pod reiniciando por OOM | ✅ pass | 3899 ms | US$ 0.0033 | — |
| 1 | gemini-3.8-flash | entrada-2 dois pods que não sobem | ✅ pass | 4285 ms | US$ 0.0024 | — |
| 1 | claude-haiku-4-5 | entrada-2 dois pods que não sobem | ✅ pass | 4404 ms | US$ 0.0038 | — |
| 1 | claude-haiku-4-5 | entrada-3 tudo saudável | ✅ pass | 3047 ms | US$ 0.0026 | — |
| 1 | gemini-3.8-flash | entrada-3 tudo saudável | ✅ pass | 3882 ms | US$ 0.0016 | — |
| 2 | gemini-3.8-flash | entrada-1 pod reiniciando por OOM | ✅ pass | 2541 ms | US$ 0.0023 | — |
| 2 | claude-haiku-4-5 | entrada-1 pod reiniciando por OOM | ✅ pass | 3636 ms | US$ 0.0033 | — |
| 2 | claude-haiku-4-5 | entrada-2 dois pods que não sobem | ✅ pass | 4185 ms | US$ 0.0038 | — |
| 2 | gemini-3.8-flash | entrada-2 dois pods que não sobem | ✅ pass | 4755 ms | US$ 0.0024 | — |
| 2 | claude-haiku-4-5 | entrada-3 tudo saudável | ✅ pass | 3084 ms | US$ 0.0026 | — |
| 2 | gemini-3.8-flash | entrada-3 tudo saudável | ✅ pass | 4489 ms | US$ 0.0015 | — |

### networkpolicy-sentinel — Haiku excluído, Gemini 3/5 por latência

Primeira rodada com os dois providers (prompt v3.1):

| Rodada | Provider | Resultado | Latência | Custo | Asserts que falharam |
|---|---|---|---|---|---|
| 1 | gemini-3.8-flash | ❌ fail | 7191 ms | US$ 0.0054 | latency |
| 1 | claude-haiku-4-5 | ❌ fail | 9217 ms | US$ 0.0083 | regex, latency |
| 2 | claude-haiku-4-5 | ❌ fail | 9536 ms | US$ 0.0084 | regex, latency |
| 2 | gemini-3.8-flash | ✅ pass | 4931 ms | US$ 0.005 | — |

O Haiku 4.5 falhou nas duas rodadas por dois motivos que não mudam com rerun: 9,2 a 9,5 s de latência e o `regex` de `^apiVersion`, porque ele imprime a "preparação" antes do YAML apesar da instrução "passos 1 e 2 em silêncio" (o mesmo comportamento das três versões do CP06). Um provider que falha sempre não é teste, é ruído; removi o Haiku desse config e deixei o motivo em comentário no YAML. O requisito de dois fornecedores é cumprido nos outros dois configs.

Mais três rodadas só com o Gemini 3.8 Flash:

| Rodada | Resultado | Latência | Custo | Asserts que falharam |
|---|---|---|---|---|
| 3 | ❌ fail | 11309 ms | US$ 0.0054 | latency |
| 4 | ✅ pass | 4672 ms | US$ 0.0054 | — |
| 5 | ✅ pass | 4820 ms | US$ 0.0055 | — |

Cinco rodadas, três passam. O custo é idêntico em todas (US$ 0,0050 a 0,0055, mesmo número de tokens), então a diferença entre 4,7 s e 11,3 s é variação do lado da API, não do prompt. O conteúdo passou em todos os asserts de conteúdo nas cinco rodadas.

Tentei também dois modelos Gemini mais leves no mesmo prompt para fugir da borda:

| Modelo | Latência (2 rodadas) | Custo | Conteúdo |
|---|---|---|---|
| Gemini 3.5 Flash (sem thinking) | 4,8 s / 4,8 s | US$ 0,012 | passa conteúdo, estoura o custo |
| Gemini 3.5 Flash Lite | 3,2 s / 3,5 s | US$ 0,003 | põe o comentário dentro do item (`- # Libera…` e `from:` na linha seguinte), omite o `A CONFIRMAR` da porta de ingress e o `(mesmo item = E lógico)` em parte das regras |

O Lite é rápido e barato, mas não obedece às regras que o CP06 acrescentou depois da revisão de segurança. Para uma NetworkPolicy, obediência vale mais que 1,5 s. Fiquei com o Gemini 3.8 Flash e com o limite de 5 s como está.

## Curadoria

**O que passou:** formato e conteúdo em 100% dos casos nos três prompts, nos dois providers onde os dois foram mantidos. Nenhum prompt precisou mudar por causa dos testes deste checkpoint: as mudanças de prompt já tinham acontecido nos checkpoints 01, 02 e 06, quando os mesmos asserts foram usados como verificação durante a curadoria.

**O que falhou e o que ajustei:**

1. **No teste:** o regex com `(?i)`. Lição: assert de regex no promptfoo é JavaScript; flags inline não existem, e uma regex inválida falha silenciosamente como "não casou". Vale rodar o config uma vez contra uma saída conhecida boa antes de confiar nele.
2. **No teste:** o assert `not-contains "- {}"` da NetworkPolicy estourou na v1 do prompt porque a checklist de autoverificação citava o literal. A correção foi no prompt (CP06), mas foi o teste que achou.
3. **No config:** Haiku removido da NetworkPolicy. Documentado em vez de "ajustado": subir o limite de latência para o Haiku passar seria mudar a régua para caber o modelo.
4. **Não ajustado, de propósito:** o limite de 5 s na NetworkPolicy, mesmo com 40% de falha por variação de API. Mudar o limite esconderia o problema; o lugar de tratar flutuação é o desenho do gate no Checkpoint 10 (rerun de assert de latência antes de reprovar o build).

**Latência e custo como parte da qualidade.** Os limites conversam diretamente com a escolha de modelo deste playbook: os três prompts testáveis rodam em Flash/Haiku a US$ 0,002 a 0,006 por chamada e 1,3 a 4,8 s, e é por isso que foram escolhidos no CP01, CP02 e CP06. O Sonnet 4.6, que ganhou nos prompts de raciocínio (CP03, CP04, CP05), custaria US$ 0,03 a 0,05 e 35 a 60 s aqui: reprovaria os dois limites em qualquer caso. Um modelo por tarefa, não um modelo por playbook.

**Custo da suíte:** os três configs somam 13 chamadas por rodada completa (6 + 6 + 1), cerca de US$ 0,035 e 25 s de parede com concorrência 4. Isso é o que o CI do Checkpoint 10 vai gastar por push, mais o juiz do Checkpoint 09.
