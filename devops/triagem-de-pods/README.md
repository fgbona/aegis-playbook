---
nome: Triagem de pods
descricao: Triagem da saúde dos pods de um namespace Kubernetes a partir de um snapshot (get pods, describe, logs), com causa provável e próxima ação por pod
versao: 1.0.0
tags: [kubernetes, sre, triagem, plantao, incidentes]
inputs:
  - nome: snapshot
    descricao: Saída colada de `kubectl get pods`, `kubectl describe pod` dos pods suspeitos e `kubectl logs` relevantes, coletada por quem tem acesso ao cluster
---

# Triagem de pods

## Objetivo

Transformar um snapshot colado do cluster (`kubectl get pods`, `kubectl describe pod`, `kubectl logs`) numa triagem curta e confiável: quais pods estão problemáticos, a causa provável de cada um apoiada em evidência literal do snapshot, a próxima ação do plantão e, quando não há nada errado, dizer isso explicitamente.

## Quando usar

- Início de plantão ou ao receber um alerta de pod reiniciando, não subindo ou preso em Pending.
- Para gerar um resumo colável no canal do incidente a partir de um snapshot bruto.
- Quando o plantonista tem acesso ao cluster mas o modelo não (o snapshot é a única fonte).

## Exemplo de uso

Entrada (`{{snapshot}}`): o arquivo [`exemplos/entrada-1-pod-reiniciando.txt`](./exemplos/entrada-1-pod-reiniciando.txt).

Saída real (Claude Haiku 4.5):

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

Os outros dois exemplos (pods que não sobem e cluster saudável) estão em [`exemplos/`](./exemplos/) e as saídas correspondentes em [`docs/checkpoints/01-triagem-de-pods.md`](../../docs/checkpoints/01-triagem-de-pods.md).

## Limitações conhecidas

- O modelo só enxerga o snapshot. Se o `describe` ou o `logs` de um pod suspeito não foi colado, a causa vem como "Evidência insuficiente" e o prompt devolve o comando que coletaria o dado.
- A "próxima ação" é uma recomendação de triagem, não um runbook: valores como `limits.memory` sugeridos precisam de validação humana antes de aplicar.
- Snapshots muito grandes (dezenas de pods com logs longos) encarecem a chamada e podem passar do limite de latência de 5s definido para o playbook; recorte o snapshot aos pods suspeitos.
- Modelos com raciocínio interno ligado (ex.: Gemini 3.8 Flash no modo padrão) podem truncar a saída e estourar a latência; ver a curadoria do checkpoint.
