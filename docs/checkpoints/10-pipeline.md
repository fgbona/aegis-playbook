# Checkpoint 10 — O playbook em produção contínua

## Entrega

- **Workflow:** [`.github/workflows/promptfoo.yml`](../../.github/workflows/promptfoo.yml). Roda a cada push na `main` e a cada pull request; um job por prompt testado (matriz), com um rerun automático em caso de falha, resumo por job e, em PR, um comentário com a tabela de resultados.
- **Cobertura:** os quatro prompts com teste (`nota-de-triagem`, `triagem-de-pods`, `networkpolicy-sentinel` com asserts determinísticos; `causa-raiz` com juiz LLM). Os outros cinco prompts (decisão de backpressure e a cadeia de migração) são de saída aberta e ainda não têm juiz; o job da matriz pula a pasta sem `promptfooconfig.yaml` e isso aparece no log. Ver "O que ficou de fora" abaixo.
- **Secrets:** `GOOGLE_API_KEY` e `ANTHROPIC_API_KEY` como secrets do repositório no GitHub, lidos pelo workflow via `${{ secrets.* }}` e expostos só como variáveis de ambiente do job. Nunca entram em arquivo versionado.

## Evidência de execução

### Run 1 — primeiro push do workflow (três configs; `causa-raiz` ainda sem config)

[actions/runs/36868186790](https://github.com/fgbona/aegis-playbook/actions/runs/36868186790) — **success**. `nota-de-triagem`, `triagem-de-pods` e `networkpolicy-sentinel` passaram na primeira rodada; `causa-raiz` foi pulado em 4 s por não ter `promptfooconfig.yaml` ainda; `comenta no PR` pulado por ser push.

### Run 2 — push do gate com juiz (quatro configs)

[actions/runs/36868918937](https://github.com/fgbona/aegis-playbook/actions/runs/36868918937) — **success**, 2 min 35 s de parede.

| Job | Resultado | Duração |
|---|---|---|
| devops/nota-de-triagem | success | 1 min 18 s |
| devops/triagem-de-pods | success | 1 min 36 s |
| devops/networkpolicy-sentinel | success | 1 min 19 s |
| devops/causa-raiz (Sonnet 4.6 + juiz Gemini) | success | 2 min 33 s |

### Run 3 — falha provocada por prompt regredido (pull request #1)

[pull/1](https://github.com/fgbona/aegis-playbook/pull/1) · [actions/runs/36869265081](https://github.com/fgbona/aegis-playbook/actions/runs/36869265081) — **failure**, fechado sem merge.

A regressão: um commit "refactor(devops): simplifica nota de triagem removendo a linha de escalonamento" que troca "cinco linhas" por "quatro" no prompt e apaga a regra do `ESCALAR PARA:`. É o tipo de simplificação que alguém faria de boa fé.

| Job | Resultado | O que aconteceu |
|---|---|---|
| devops/nota-de-triagem | **failure** | Gemini 3.8 Flash falhou os 3 alertas em `contains-all` (faltou o rótulo) e `regex` (faltou o handle), nas duas rodadas. Build reprovado. |
| devops/triagem-de-pods | success | Na primeira rodada o Gemini estourou a latência na entrada 3 (5.193 ms); o rerun automático passou (4.151 ms). Exatamente o caso para o qual o rerun existe. |
| devops/networkpolicy-sentinel | success | 4.962 ms, na borda, passou de primeira. |
| devops/causa-raiz | success | 7/8 no juiz. |
| comenta no PR | success | Publicou a tabela abaixo no PR. |

Trecho do comentário publicado automaticamente no PR (as duas rodadas de cada job aparecem):

```
## promptfoo — failure

| Prompt | Provider | Caso | Resultado | Latência | Custo | Falhou |
|---|---|---|---|---|---|---|
| causa-raiz | claude-sonnet-4-6 | incidente do Cerebro (config + métricas + logs do CP03) | ✅ | 69286 ms | US$ 0.0547 |  |
| nota-de-triagem | gemini-3.8-flash | alerta-1 Sentinel autoscaler no teto | ❌ | 1291 ms | US$ 0.0013 | contains-all, regex |
| nota-de-triagem | claude-haiku-4-5 | alerta-1 Sentinel autoscaler no teto | ✅ | 1878 ms | US$ 0.0021 |  |
| nota-de-triagem | gemini-3.8-flash | alerta-2 Relay rejeitando ingestão | ❌ | 1501 ms | US$ 0.0011 | contains-all, regex |
| nota-de-triagem | claude-haiku-4-5 | alerta-2 Relay rejeitando ingestão | ✅ | 1197 ms | US$ 0.0018 |  |
| nota-de-triagem | gemini-3.8-flash | alerta-3 Forge com lag | ❌ | 1778 ms | US$ 0.0011 | contains-all, regex |
| nota-de-triagem | claude-haiku-4-5 | alerta-3 Forge com lag | ✅ | 1771 ms | US$ 0.0019 |  |
| triagem-de-pods | gemini-3.8-flash | entrada-3 tudo saudável | ❌ | 5193 ms | US$ 0.0017 | latency |
| triagem-de-pods | gemini-3.8-flash | entrada-3 tudo saudável | ✅ | 4151 ms | US$ 0.0016 |  |
```

**Um achado que não estava no plano:** o Claude Haiku 4.5 continuou escrevendo `ESCALAR PARA: @relay-core …` mesmo com a regra apagada e o prompt pedindo quatro linhas. Ele copiou o formato dos três exemplos few-shot que ficaram no prompt. O Gemini seguiu a instrução nova e, por isso, foi ele que pegou a regressão. Dois providers no mesmo config não são redundância: são dois leitores diferentes do mesmo prompt, e o gate falha se qualquer um deles mostrar o problema. É também um lembrete do CP02: exemplo few-shot pesa mais que regra escrita para alguns modelos.


## Estratégia de gate: o que falha o build e por quê

**O build falha quando qualquer caso de qualquer config falha depois de duas rodadas.** Isso inclui os asserts determinísticos (formato, conteúdo, latência ≤ 5 s, custo ≤ US$ 0,01) e o veredito do juiz no `causa-raiz` (total ≥ 6 e nenhum critério zerado). Não há threshold separado para o juiz no pipeline: o corte já está na rubrica, calibrado no Checkpoint 09, e o `pass` do juiz é tratado como qualquer outro assert.

**Rerun uma vez antes de reprovar.** Falha de conteúdo (rótulo faltando, pod errado, `- {}` no YAML, nota do juiz abaixo do corte por mérito) falha nas duas rodadas e derruba o build. Falha por flutuação (latência da API num dia ruim, juiz oscilando 1 ponto numa saída na borda) costuma passar na segunda. O custo é no máximo dobrar a rodada que falhou, nunca a suíte inteira.

## Decisões de desenho, cada uma comparada com pelo menos duas alternativas

### 1. O que faz o build falhar: determinísticos + juiz, ou só determinísticos?

| Opção | Ganha | Perde |
|---|---|---|
| Só asserts determinísticos | zero flutuação, barato, rápido | o prompt de causa-raiz fica sem gate nenhum; é justamente o prompt em que uma regressão é silenciosa (formato igual, raciocínio pior) |
| Determinísticos + juiz como aviso (não bloqueia) | nunca derruba o build por flutuação | aviso que não bloqueia vira ruído em uma semana; a regressão passa |
| **Determinísticos + juiz bloqueante com rerun (escolhida)** | regressão de qualidade barra o merge; calibração do CP09 mostrou desvio ≤ 1 ponto | pode reprovar por flutuação; mitigado pelo rerun e pelo corte em 6 (uma saída boa tira 8, tem 2 pontos de folga) |

Threshold: o corte do enunciado (≥ 6 de 8, nenhum zero). Não subi para 7 porque a amostra B da calibração, que eu considero aprovável, tira 7 ou 8 conforme o juiz lê a causa dupla; com corte 7 ela flutuaria entre aprovada e reprovada. Não desci para 5 porque a amostra C (nota 4) é o que o gate existe para barrar, e um corte em 5 a deixaria passar com 1 ponto de flutuação.

### 2. Suíte inteira ou só os prompts alterados?

| Opção | Ganha | Perde |
|---|---|---|
| Só prompts alterados (como faz a action oficial do promptfoo, que filtra por arquivos modificados) | mais barato e rápido em PRs pequenos | mudança no `prompt.md` de um elo da cadeia, num arquivo de `exemplos/` ou na rubrica do juiz não dispara o teste certo; a regressão entra sem passar pelos testes, que é exatamente o que o checkpoint pede para impedir |
| **Suíte inteira a cada push e PR (escolhida)** | cobertura total, lógica simples, sem heurística de "o que mudou" | custa a suíte inteira por execução |
| Suíte inteira só na `main`, alterados em PR | economiza em PR | a regressão aparece depois do merge, quando já é tarde |

O custo da suíte inteira é o que decide: 13 chamadas determinísticas (US$ 0,035, 25 s) mais 1 chamada do Sonnet e 1 do juiz (US$ 0,06, 60 a 90 s). Com o rerun no pior caso, menos de US$ 0,20 e 4 minutos por execução, rodando em paralelo na matriz. A um push por hora de trabalho, são centavos por dia. Se a biblioteca crescer para dezenas de prompts com juiz, a conta muda e a opção "alterados em PR, inteira na main" passa a valer; a matriz já está preparada para isso (basta filtrar a lista por `git diff`).

### 3. Onde guardar as chaves e como limitar o gasto

| Opção | Ganha | Perde |
|---|---|---|
| **Secrets do repositório (escolhida)** | padrão do GitHub, mascaradas no log, rotacionáveis sem commit | qualquer workflow do repositório as enxerga; em PR de fork não estão disponíveis (o GitHub não injeta secrets em PR de fork), então contribuição externa não roda a suíte |
| Secrets de environment com aprovação manual | gasto só depois de alguém aprovar | adiciona um clique humano em cada PR; para um playbook interno com meia dúzia de autores, atrito sem ganho |
| Chave num serviço de secrets externo (Vault, AWS Secrets Manager) via OIDC | rotação centralizada, auditoria | infraestrutura que a Aegis já teria para outros fins, mas que este repositório não precisa carregar |

Controle de gasto fora do secret: `concurrency` com `cancel-in-progress` cancela a suíte de um push antigo quando chega outro no mesmo branch, e `timeout-minutes: 15` impede um job preso de ficar chamando modelo.

### 4. Como rodar o promptfoo: action oficial ou CLI na matriz?

| Opção | Ganha | Perde |
|---|---|---|
| `promptfoo/promptfoo-action` (ponto de partida sugerido) | comentário no PR pronto, cache, pouca configuração | aceita um único `config`; só roda quando os arquivos do glob `prompts` mudam (volta à decisão 2); o comentário é opaco ao formato que eu quero |
| **CLI `promptfoo eval` numa matriz por prompt, comentário montado com `gh` (escolhida)** | um config por prompt como o template exige, controle do rerun, da tabela e do artefato JSON, mesma versão fixada (`0.123.1`) que roda localmente | mais YAML para manter; o comentário é feito à mão |
| Um único config raiz agregando todos os prompts | um job só | o promptfoo cruza prompts × testes de um config; prompts com parâmetros diferentes não cabem num config só sem gambiarra |

### 5. Versão do promptfoo

Fixada em `0.123.1`, a mesma usada localmente em todos os checkpoints, via `npx promptfoo@0.123.1`. A alternativa (`latest`) ganha correções de graça e perde reprodutibilidade: um assert que muda de semântica entre versões reprovaria o build sem mudança no prompt.

## O que ficou de fora, e por quê

- **Juiz para `decisao-backpressure` e para a cadeia de migração.** Cada um precisa de rubrica própria com fatos de referência, como o CP09 mostrou; sem calibração, um juiz genérico aprova texto bonito. Esses cinco prompts ficam sem gate até terem rubrica calibrada; o workflow já os cobre assim que um `promptfooconfig.yaml` aparecer na pasta.
- **Cache do promptfoo no CI.** `--no-cache` de propósito: cache mascararia regressão de modelo e flutuação de latência, que são parte do que o gate mede.
- **Comparação com a versão anterior do prompt.** O enunciado fala em "piorar em relação à versão anterior"; o gate implementa isso com limiares absolutos (asserts e corte do juiz), não com diff de notas entre commits. Comparação relativa exigiria guardar a nota do último run bom; é o próximo passo natural, usando o artefato JSON que o workflow já publica.
