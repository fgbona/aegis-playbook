# Checkpoint 02 — Padronizando as notas de triagem

## 1. Prompt parametrizável

Está em [`devops/nota-de-triagem/prompt.md`](../../devops/nota-de-triagem/prompt.md). Um único parâmetro, `{{alerta_cru}}`. O prompt carrega o contexto da plataforma (o que cada sistema faz e o handle do time dono), as regras por campo, as restrições de "não invente" e os três exemplos aprovados pelo time como referência de forma.

## 2. Execução

**Modelo:** Gemini 3.8 Flash (`gemini-3.8-flash`, Google) com `thinkingLevel: LOW`.

**Por que esse modelo:** é uma tarefa de reformatação com padrão fixo e saída de cinco linhas. Não há raciocínio longo a fazer, então o critério é custo, latência e obediência ao formato. O Gemini 3.8 Flash custou US$ 0,0012 a 0,0015 por nota, respondeu em 1,4 a 2,9 s e, na comparação com o Haiku 4.5, foi mais fiel às regras (não vazou detalhes dos exemplos e manteve o nome do sistema como está no alerta). Também é o segundo fornecedor do playbook, que o desafio exige.

**Resultados reais** (promptfoo, `--no-cache`, 2026-10-01). Os três passaram nas checagens de formato que viram asserts no Checkpoint 08 (cinco rótulos, handle `@time` no escalonamento, no máximo 8 linhas).

### Alerta 1

Entrada:

```
2026-05-12 14:02:09 UTC [Sentinel] autoscaler hit max replicas (60/60) on sentinel-api,
queue depth on Relay growing 2k/min, CPU avg 88%, tenant stark-industries
sending 4x baseline volume after onboarding new region
```

Saída (2020 ms · US$ 0.00149):

```
ALERTA: Sentinel – autoscaler atingiu réplicas máximas (60/60) no sentinel-api com CPU média de 88% e fila no Relay crescendo 2k/min
IMPACTO: processamento e ingestão de telemetry atrasados ou sob risco de degradação, afetando stark-industries e potencialmente outros tenants
HIPÓTESE INICIAL: possível sobrecarga causada pelo onboarding de nova região do tenant stark-industries que elevou o tráfego a 4x o volume base
AÇÃO IMEDIATA: aplicar rate limiting temporário ou throttling na ingestão do tenant stark-industries
ESCALAR PARA: @sentinel-core se a CPU ou o crescimento da fila no Relay não estabilizarem em 15min
```

### Alerta 2

Entrada:

```
2026-05-13 03:11:00 UTC [Relay] ingest reject rate 6% for 8min, tenant wakanda-systems,
buffer saturated after deploy 02:55
```

Saída (2855 ms · US$ 0.00124):

```
ALERTA: Relay – taxa de rejeição de ingestão em 6% por 8min
IMPACTO: ingestão de telemetry afetada para o tenant wakanda-systems
HIPÓTESE INICIAL: possível saturação de buffer causada pelo deploy das 02:55
AÇÃO IMEDIATA: rollback do deploy das 02:55 no Relay
ESCALAR PARA: @relay-core se a taxa de rejeição não cair em 15min
```

### Alerta 3

Entrada:

```
2026-05-13 11:40:22 UTC [Forge] consumer lag 9min and climbing, batch forge-batch-ingest
delayed after previous job failure, downstream Sentinel starting to lag
```

Saída (1367 ms · US$ 0.0013):

```
ALERTA: Forge – consumer lag em 9min e subindo no batch forge-batch-ingest
IMPACTO: downstream Sentinel começando a apresentar atraso; tenants afetados a confirmar
HIPÓTESE INICIAL: possível acúmulo de dados causado pelo atraso do batch forge-batch-ingest após falha de job anterior
AÇÃO IMEDIATA: reiniciar ou reexecutar o job forge-batch-ingest para drenar a fila
ESCALAR PARA: @data-platform se o lag não estabilizar em 15min
```

Checagem humana: no Alerta 1 o modelo colocou o sintoma no Sentinel (autoscaler no teto) e a hipótese no onboarding do tenant `stark-industries`, com ação de contenção reversível (throttling do tenant). No Alerta 2 preservou o tenant `wakanda-systems`, ancorou a hipótese no deploy das 02:55 e escalou para `@relay-core`. No Alerta 3 reconheceu que o alerta não diz quais tenants são afetados e escreveu "a confirmar" em vez de inventar.

## 3. Curadoria

**Decisão de método.** O enunciado deixa a escolha de como ensinar o padrão. Havia três caminhos na mesa:

| Caminho | O que ganha | O que perde |
|---|---|---|
| Só regras (descrever cada campo) | Prompt curto, sem risco de vazar dados de exemplo | Perde tom e tamanho das notas reais; o modelo tende a escrever frases longas |
| Só few-shot (os três exemplos) | Tom e tamanho idênticos aos aprovados | Modelo reaproveita conteúdo dos exemplos como se fosse dado do alerta |
| Few-shot + regras por campo (escolhido) | Tom dos exemplos com regras que fecham as brechas (handle obrigatório, hipótese marcada como hipótese, "a confirmar" quando falta dado) | Prompt maior, ~50 linhas, custo de entrada um pouco maior |

Fiquei com o híbrido. A instrução explícita de que os exemplos são referência de forma e não dados do alerta atual foi necessária: mesmo com ela, o Haiku 4.5 escreveu "rollback via Argo CD" no Alerta 2, que não cita ferramenta nenhuma. O Gemini não fez isso.

**Técnica de criação:** meta-prompting. O meta-prompt (no final deste documento) descreveu o problema, os três exemplos como material de few-shot, o mapa de times da plataforma e oito garantias de conteúdo. O Sonnet 5.5 gerou a v1 em uma rodada.

**O que refinei:**

1. Placeholder `{{ALERTA_CRU}}` para `{{alerta_cru}}`, convenção do template. Foi a única edição no texto.
2. O mapa de times não existia no enunciado para o Sentinel (os exemplos só mostram `@relay-core`, `@data-platform` e `@search-infra`). Defini `@sentinel-core` no meta-prompt e a regra "escalar para o time da causa, não do sintoma", porque o Alerta 1 tem sintoma no Sentinel e causa num tenant que entra pelo Relay. Os dois modelos leram a regra de forma diferente (Gemini escalou para `@sentinel-core`, Haiku para `@relay-core`); ambas são defensáveis e isso é um sinal de que esse campo merece um juiz humano, não um assert de regex.
3. Rodei os dois modelos com asserts de formato já no primeiro teste. Os seis outputs passaram, então não houve v2 por falha; a escolha do Gemini foi por qualidade e custo, não por correção.

**Comparação de modelos no mesmo prompt** (3 alertas cada):

| Modelo | Formato ok nos 3? | Latência | Custo por chamada | Observação |
|---|---|---|---|---|
| Gemini 3.8 Flash (`thinkingLevel: LOW`) | sim | 1,4 a 2,9 s | US$ 0,0012 a 0,0015 | Nome do sistema correto, sem vazamento dos exemplos |
| Claude Haiku 4.5 | sim | 1,4 a 2,6 s | US$ 0,0020 a 0,0022 | "Sentinel-api" no Alerta 1; "via Argo CD" no Alerta 2 |

**Sanitização:** os alertas carregam nomes de tenant (`stark-industries`, `wakanda-systems`). A nota precisa citá-los, então não dá para remover. O que faria num ambiente real é trocar por um identificador opaco antes de enviar ao provedor e reverter na volta; aqui, com dados fictícios, mantive para que a execução fosse verificável.

---

### Meta-prompt usado (registro do caminho, não entra na biblioteca)

```
Você é um engenheiro de prompts sênior que escreve prompts para um playbook de operações de SRE. Os prompts são reutilizados por qualquer plantonista, então precisam ser parametrizáveis, autocontidos e previsíveis.

Escreva o PROMPT FINAL (não execute a tarefa) para o caso abaixo.

## O problema que o prompt resolve
Toda vez que o sistema de alerting dispara um alerta, o plantonista abre uma nota de triagem. Hoje cada um escreve do seu jeito e isso atrapalha quem assume o turno seguinte. Queremos um padrão único: o prompt recebe o ALERTA CRU (texto livre do alerta, uma a três linhas) e devolve a NOTA PADRONIZADA.

## O padrão de nota (três exemplos aprovados pelo time; são referência de FORMATO e devem entrar no prompt como exemplos)
```
ALERTA: Relay – taxa de rejeição de ingestão acima de 2% por 5min
IMPACTO: ingestão de telemetry degradada para ~12% dos tenants
HIPÓTESE INICIAL: deploy do Relay às 09:14 reduziu o buffer de ingestão
AÇÃO IMEDIATA: rollback iniciado via Argo CD
ESCALAR PARA: @relay-core se a rejeição não cair em 10min
```
```
ALERTA: Forge – lag de ingestão acima de 15min
IMPACTO: dashboards do Sentinel atrasados para todos os tenants
HIPÓTESE INICIAL: pico de volume do tenant acme-corp saturou o consumer
AÇÃO IMEDIATA: aumento manual de partições do consumer do Relay
ESCALAR PARA: @data-platform se lag não estabilizar em 20min
```
```
ALERTA: Cerebro – latência de busca p99 acima de 4s
IMPACTO: investigação de incidentes lenta para o time interno
HIPÓTESE INICIAL: reindexação noturna não concluiu antes do horário comercial
AÇÃO IMEDIATA: pausar reindexação e priorizar shard quente
ESCALAR PARA: @search-infra se p99 não cair em 15min
```

## Contexto da plataforma (para o prompt saber o que cada sistema faz e para quem escalar)
- Relay: barramento de eventos e borda de ingestão de telemetry. Time: @relay-core.
- Forge: pipeline de dados e data warehouse. Time: @data-platform.
- Cerebro: indexação e busca. Time: @search-infra.
- Sentinel: produto de observabilidade e alerting que o cliente usa. Time: @sentinel-core.
- Quando o alerta envolver mais de um sistema, escalar para o time do sistema onde está a causa provável, não onde está o sintoma.

## O que o prompt precisa garantir na resposta
1. Exatamente as cinco linhas, nesta ordem e com estes rótulos literais: `ALERTA:`, `IMPACTO:`, `HIPÓTESE INICIAL:`, `AÇÃO IMEDIATA:`, `ESCALAR PARA:`. Nada antes, nada depois, sem markdown, sem linhas em branco. No máximo 8 linhas no total.
2. `ALERTA:` começa com o nome do sistema seguido de " – " e um resumo do sintoma com a métrica e o limiar ou valor observado.
3. `IMPACTO:` descreve quem é afetado e como, inferido do alerta (tenants, time interno, dashboards, ingestão). Se o alerta não permite inferir o impacto, dizer "a confirmar" e o que confirmar.
4. `HIPÓTESE INICIAL:` é uma hipótese, não um fato: deve se apoiar em algo citado no alerta (deploy, tenant, job que falhou, volume) e deixar claro que é hipótese.
5. `AÇÃO IMEDIATA:` é uma única ação concreta de contenção, coerente com a hipótese e reversível quando possível.
6. `ESCALAR PARA:` tem obrigatoriamente um handle no formato `@nome-do-time` seguido de uma condição com tempo ("se X não Y em Nmin").
7. Nunca inventar números, horários ou nomes que não estejam no alerta. Timestamps do alerta podem ser citados.
8. Preservar nomes de tenant exatamente como aparecem no alerta.

## Requisitos de forma do prompt
- Em português do Brasil.
- Exatamente um parâmetro, o placeholder `{{alerta_cru}}`. Nenhum outro placeholder.
- Os três exemplos aprovados entram no prompt como exemplos de formato (few-shot), deixando explícito que são referência de forma, não dados do alerta atual.
- Conciso: cerca de 40 a 50 linhas no total, incluindo os exemplos.

Responda apenas com o texto do prompt, sem comentários antes ou depois, sem cercas de código ao redor do prompt inteiro.
```
