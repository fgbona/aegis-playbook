---
nome: Nota de triagem
descricao: Converte um alerta cru do Sentinel na nota de triagem padronizada de cinco linhas (alerta, impacto, hipótese inicial, ação imediata, escalonamento)
versao: 1.0.0
tags: [triagem, alertas, plantao, incidentes, padronizacao]
inputs:
  - nome: alerta_cru
    descricao: Texto livre do alerta como disparado pelo Sentinel (uma a três linhas), com sistema, métrica, tenant e contexto que vierem no alerta
---

# Nota de triagem

## Objetivo

Converter o texto livre de um alerta do Sentinel na nota de triagem padrão da Aegis: cinco linhas fixas (`ALERTA`, `IMPACTO`, `HIPÓTESE INICIAL`, `AÇÃO IMEDIATA`, `ESCALAR PARA`) que qualquer plantonista escreve igual e quem assume o turno seguinte lê sem adivinhar.

## Quando usar

- No momento em que um alerta dispara e o plantonista abre a nota no canal de incidente.
- Para padronizar notas antigas escritas em formato livre.
- Não usar para análise de causa-raiz: a nota registra uma hipótese inicial, não um diagnóstico (para isso há o prompt `causa-raiz`).

## Exemplo de uso

Entrada (`{{alerta_cru}}`):

```
2026-05-13 03:11:00 UTC [Relay] ingest reject rate 6% for 8min, tenant wakanda-systems,
buffer saturated after deploy 02:55
```

Saída real (Gemini 3.8 Flash, `thinkingLevel: LOW`):

```
ALERTA: Relay – taxa de rejeição de ingestão em 6% por 8min
IMPACTO: ingestão de telemetry afetada para o tenant wakanda-systems
HIPÓTESE INICIAL: possível saturação de buffer causada pelo deploy das 02:55
AÇÃO IMEDIATA: rollback do deploy das 02:55 no Relay
ESCALAR PARA: @relay-core se a taxa de rejeição não cair em 15min
```

Os três alertas de exemplo estão em [`exemplos/`](./exemplos/) e as saídas completas em [`docs/checkpoints/02-nota-de-triagem.md`](../../docs/checkpoints/02-nota-de-triagem.md).

## Limitações conhecidas

- O handle de escalonamento vem de um mapa fixo de quatro times dentro do prompt (Relay, Forge, Cerebro, Sentinel). Time novo exige editar o prompt e subir a versão.
- O prazo em `ESCALAR PARA` é regra de processo (15 min quando o alerta não traz prazo), não dado do alerta.
- A hipótese inicial é inferida do próprio texto do alerta. Alerta pobre em contexto gera hipótese fraca; o prompt diz "sem gatilho identificado" em vez de inventar.
- Modelos tendem a reaproveitar detalhes dos exemplos few-shot (ex.: "via Argo CD") mesmo com a instrução contrária; o Gemini 3.8 Flash respeitou melhor que o Haiku 4.5 nos testes.
