# Checkpoint 01 — O primeiro prompt do playbook: triagem de pods

## 1. Prompt parametrizável

Está em [`devops/triagem-de-pods/prompt.md`](../../devops/triagem-de-pods/prompt.md). Um único parâmetro, `{{snapshot}}`, recebe a saída colada de `kubectl get pods`, `describe` e `logs`. O prompt fixa papel, ordem de raciocínio (get pods → describe → logs → cruzamento), regras de "não invente" e um formato de saída rígido que funciona tanto com pods problemáticos quanto com cluster saudável.

## 2. Execução

**Modelo:** Claude Haiku 4.5 (`claude-haiku-4-5-20251001`, Anthropic).

**Por que esse modelo:** a tarefa é classificação com evidência, não raciocínio aberto. Precisa ser barata e rápida porque roda a cada alerta e, no Checkpoint 08, passa a ter limite de 5 s e US$ 0,01 por chamada. O Haiku 4.5 custa US$ 1/5 por milhão de tokens (entrada/saída) e respondeu em 2 a 4 s. Comparei com o Gemini 3.8 Flash (Google) no mesmo prompt; a tabela está na curadoria.

**Resultados reais** (promptfoo, `--no-cache`, 2026-10-01):

### Entrada 1 — pod reiniciando

Latência e custo: 3663ms · US$ 0.00324

```
RESUMO: CRÍTICO — 1 pod problemático de 4 analisados

PODS PROBLEMÁTICOS:
- `sentinel-api-7d9c8b6f4-h4m2t` | STATUS=CrashLoopBackOff | READY=0/1 | RESTARTS=14 (90s ago)
Causa provável: Pod excedendo limite de memória (512Mi) durante carregamento de cache de 6h em heap, causando OOMKilled e restart contínuo.
Evidência: "Reason: OOMKilled"; "Exit Code: 137"; "[cache] loading 6h alert cache (heap 410Mi/512Mi)"; "[runtime] out of memory, shutting down process"; "Limits: memory: 512Mi"
Confiança: alta
Próxima ação: Aumentar `limits.memory` de 512Mi para 1Gi (e `requests.memory` para 512Mi) e fazer rollout restart do deployment `sentinel-api`.

SEM PROBLEMA / OBSERVAÇÕES:
`sentinel-api-7d9c8b6f4-2xk9p`, `sentinel-worker-5b8f9c8d-jq7rs`, `sentinel-scheduler-6c4d7f-nn8vx` — todos Running com READY completo e sem restarts recentes.

LACUNAS DE EVIDÊNCIA:
Nenhuma.
```

### Entrada 2 — pods que não sobem

Latência e custo: 4377ms · US$ 0.00382

```
RESUMO: CRÍTICO — 2 pods problemáticos de 4 analisados

PODS PROBLEMÁTICOS:

- `sentinel-ingest-8f7a9c5b-4dkqm` | STATUS=Pending | READY=0/1 | RESTARTS=0
Causa provável: Pod não consegue ser agendado por falta de CPU disponível no cluster.
Evidência: "0/6 nodes are available: 6 Insufficient cpu"; Requests: cpu: 2
Confiança: alta
Próxima ação: Aumentar a capacidade de CPU do cluster (adicionar nodes ou reduzir requests de 2 para valor viável) e fazer rollout do deployment.

- `sentinel-api-7d9c8b6f4-zzp10` | STATUS=ImagePullBackOff | READY=0/1 | RESTARTS=0
Causa provável: Imagem `registry.aegis.io/sentinel-api:2.9.2` não existe ou não é acessível no registry.
Evidência: "Failed to pull image \"registry.aegis.io/sentinel-api:2.9.2\": manifest unknown"; "Error: ErrImagePull"
Confiança: alta
Próxima ação: Verificar se a tag 2.9.2 foi publicada no registry e se as credenciais de pull estão corretas; fazer rollout para uma tag conhecida ou corrigir a imagem.

SEM PROBLEMA / OBSERVAÇÕES:
`sentinel-api-7d9c8b6f4-2xk9p` (Running, READY 1/1, 0 restarts) e `sentinel-scheduler-6c4d7f-nn8vx` (Running, READY 1/1, 0 restarts) estão saudáveis.

LACUNAS DE EVIDÊNCIA:
Nenhuma.
```

### Entrada 3 — tudo saudável

Latência e custo: 2242ms · US$ 0.00237

```
RESUMO: SAUDÁVEL — 0 pods problemáticos de 4 analisados

PODS PROBLEMÁTICOS:
Nenhum.

SEM PROBLEMA / OBSERVAÇÕES:
`sentinel-api-7d9c8b6f4-2xk9p`, `sentinel-api-7d9c8b6f4-h4m2t`, `sentinel-scheduler-6c4d7f-nn8vx`: Running, READY 1/1, 0 restarts — saudáveis. `sentinel-worker-5b8f9c8d-jq7rs`: Running, READY 1/1, 1 restart ocorrido há 3 dias — restart isolado e antigo, pod estável há 3d, sem evidência de problema recorrente.

LACUNAS DE EVIDÊNCIA:
Nenhuma.
```

Checagem humana dos três outputs: na Entrada 1 cruzou `OOMKilled`/`Exit Code 137` com o log de heap 498Mi/512Mi e com o limite de 512Mi, em vez de só repetir `CrashLoopBackOff`. Na Entrada 2 separou os dois problemas (tag `2.9.2` inexistente no registry e `Insufficient cpu` no scheduler) com ações distintas. Na Entrada 3 reconheceu o cluster saudável e explicou por que o restart de 3 dias do `sentinel-worker` não é problema.

## 3. Curadoria

**Técnica:** meta-prompting. Em vez de redigir o prompt na mão, escrevi um meta-prompt descrevendo o problema, as seis garantias que a resposta precisa ter (apontar pods, causa cruzando evidências, próxima ação concreta, saída legível, reconhecer o caso saudável, não inventar) e os requisitos de forma (pt-BR, um único placeholder, ~40 linhas). Rodei no Claude Sonnet 5.5 e usei a saída como v1. O meta-prompt está no final deste documento; ele não faz parte da biblioteca.

**Framework do prompt gerado:** o Sonnet entregou algo próximo de R-I-S-E (papel, input, passos de raciocínio, expectativa de saída), que é o que eu teria escolhido para uma tarefa de diagnóstico com formato fixo: o valor está na ordem dos passos e na saída rígida, não em persona elaborada.

**O que refinei:**

1. O placeholder veio como `{{SNAPSHOT}}`; a convenção do template é minúscula, então virou `{{snapshot}}`. Foi a única mudança no texto do prompt. A v1 já cobria os três cenários, então não forcei uma v2 por obrigação.
2. A primeira rodada do meta-prompt saiu truncada: Sonnet 5.5 gasta tokens de raciocínio por padrão (1.264 na chamada) e o teto de 2.000 tokens cortou o prompt no meio. Subi para 8.000. Lição que vale para o playbook inteiro: em modelos com raciocínio interno, o `max_tokens` precisa contar o raciocínio.
3. O mesmo problema apareceu na execução com Gemini 3.8 Flash: no modo padrão ele cortou a saída nas Entradas 1 e 2 e levou 6,6 a 6,8 s. Com `thinkingLevel: LOW` acertou as três em 1,9 a 3,8 s. Para uma tarefa de triagem com formato fixo, raciocínio interno só adiciona custo e latência.

**Comparação de modelos no mesmo prompt** (3 entradas cada):

| Modelo | Acertou as 3? | Latência | Custo por chamada |
|---|---|---|---|
| Claude Haiku 4.5 | sim | 2,2 a 4,4 s | US$ 0,0024 a 0,0038 |
| Gemini 3.8 Flash (padrão, com thinking) | não, truncou 2 de 3 | 4,9 a 6,8 s | US$ 0,0050 a 0,0066 |
| Gemini 3.8 Flash (`thinkingLevel: LOW`) | sim | 1,9 a 3,8 s | US$ 0,0016 a 0,0024 |

Fiquei com o Haiku como modelo de referência pela consistência sem ajuste de parâmetro; o Gemini com thinking baixo é a alternativa de segundo fornecedor e entra como provider no Checkpoint 08.

**Sanitização:** o snapshot traz hostname de registry interno (`registry.aegis.io`) e nomes de pods. Para este checkpoint mantive como está, porque nome de pod é justamente o que o output precisa citar e o registry é necessário para diagnosticar `manifest unknown`. Num ambiente real, o que eu removeria antes de enviar a um modelo externo seriam tokens de pull secret e dados de cliente dentro dos logs, que não aparecem aqui.

---

### Meta-prompt usado (registro do caminho, não entra na biblioteca)

```
Você é um engenheiro de prompts sênior que escreve prompts para um playbook de operações de SRE. Os prompts do playbook são reutilizados por qualquer plantonista, então precisam ser parametrizáveis, autocontidos e previsíveis.

Preciso que você escreva o PROMPT FINAL (não execute a tarefa, escreva o prompt) para o caso abaixo.

## O problema que o prompt resolve
O time de SRE precisa de uma triagem rápida e confiável da saúde dos pods de um namespace Kubernetes. O plantonista coleta um snapshot do cluster (saída de `kubectl get pods`, `kubectl describe pod` dos pods suspeitos e `kubectl logs` relevantes) e cola esse snapshot no prompt. O modelo não tem acesso ao cluster: só enxerga o que está no snapshot.

## O que o prompt precisa garantir na resposta do modelo
1. Apontar quais pods estão em estado problemático (CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled, Error, restarts anormais etc.).
2. Para cada pod problemático, chegar à CAUSA PROVÁVEL cruzando o STATUS com os eventos do describe e com as linhas de log. Não basta repetir o STATUS; a causa precisa citar a evidência (ex.: "OOMKilled com heap 498Mi/512Mi no log").
3. Recomendar a PRÓXIMA AÇÃO concreta do plantão para cada pod (um comando ou uma decisão, não uma lista genérica).
4. Devolver uma saída legível e curta, boa para colar num canal de incidente, com estrutura fixa.
5. Reconhecer explicitamente quando NÃO há nada problemático (por exemplo, um restart antigo e isolado num pod Running 1/1 não é problema) em vez de forçar um diagnóstico.
6. Nunca inventar informação que não esteja no snapshot. Se faltar evidência para a causa, dizer que falta e indicar qual comando coletaria.

## Requisitos de forma do prompt
- Em português do Brasil.
- Exatamente um parâmetro, o placeholder `{{snapshot}}`, onde o snapshot é colado. Nenhum outro placeholder.
- Deve definir papel, instruções de raciocínio (que evidências cruzar e em que ordem), regras de "não invente", e um formato de saída fixo que o modelo deve seguir à risca.
- Formato de saída deve funcionar tanto para o caso com pods problemáticos quanto para o caso saudável.
- Conciso: o prompt inteiro deve caber em cerca de 40 linhas.

Responda apenas com o texto do prompt, sem comentários antes ou depois, sem cercas de código.
```
