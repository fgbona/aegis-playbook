---
nome: Plano executável de etapa da migração
descricao: Elo 3 da cadeia de migração batch → event-driven. Recebe o roteiro do elo 2 e o ID de uma etapa e devolve o plano executável e reversível dessa etapa (pré-condições, passos, observação, rollback, pronto)
versao: 1.0.0
tags: [migracao, pipeline-de-dados, plano, rollback, cadeia-de-prompts]
inputs:
  - nome: roteiro
    descricao: Saída integral do prompt `roteiro-migracao-event-driven` (elo 2), com as etapas E*
  - nome: etapa
    descricao: Identificador da etapa a detalhar, exatamente como aparece no roteiro (ex.: E2)
---

# Plano executável de etapa da migração

## Objetivo

Terceiro elo da cadeia. Recebe o roteiro do elo 2 e o ID de uma etapa e devolve o plano que um operador executa: pré-condições, passos com verificação, o que observar, gatilhos objetivos de rollback, procedimento de rollback com efeitos colaterais irreversíveis explicitados, definição de pronto e pendências. Roda uma vez por etapa.

## Cadeia

1. [`diagnostico-pipeline-batch`](../diagnostico-pipeline-batch/) — descrição → diagnóstico.
2. [`roteiro-migracao-event-driven`](../roteiro-migracao-event-driven/) — diagnóstico + requisitos → etapas.
3. **`plano-executavel-de-etapa`** (este) — roteiro + etapa → plano executável e reversível.

## Quando usar

- Na véspera de executar uma etapa do roteiro, para transformar "E5 | Parallel Run" em checklist operacional.
- Para revisar se uma etapa tem rollback de verdade antes de aprovar a janela.

## Exemplo de uso

Entradas: [`exemplos/forge-roteiro.txt`](./exemplos/forge-roteiro.txt) em `{{roteiro}}` (saída real do elo 2) e `E5` em `{{etapa}}`.

Saída real (Claude Sonnet 4.6), gatilhos e procedimento de rollback:

```
GATILHO DE ROLLBACK
G1 | diferença de contagem de registros supera A CONFIRMAR: limiar de tolerância em qualquer ciclo dentro do período de observação | acionar imediatamente ao detectar o ciclo infrator
G2 | diferença de valores agregados supera A CONFIRMAR: limiar de tolerância em qualquer ciclo dentro do período de observação | acionar imediatamente ao detectar o ciclo infrator
G3 | novo pipeline acumula atraso crescente por A CONFIRMAR: número de ciclos consecutivos sem recuperação | acionar ao confirmar a tendência no log de comparação
G4 | qualquer contrato C1, C2 ou C3 violado durante o período de observação, independentemente da causa | acionar imediatamente

PROCEDIMENTO DE ROLLBACK
1. Desligar o novo pipeline | verificação: confirmar que nenhum micro-lote novo é iniciado após o desligamento
2. Confirmar que o batch continua gerando partições horárias normalmente | verificação: aguardar o próximo ciclo do cron e consultar a contagem da partição gerada
3. Verificar que Sentinel, Cerebro e Pepper seguem lendo das fontes anteriores sem erro | verificação: ausência de alertas de contrato C1, C2 e C3 no ciclo seguinte ao rollback
4. Registrar formalmente o ciclo e o motivo do rollback no log de comparação | verificação: entrada de rollback presente no log com carimbo de tempo e causa
Efeito colateral: eventos já consumidos do Relay pelo novo pipeline não são desfeitos; dados eventualmente gravados em tabelas intermediárias do novo pipeline devem ser identificados e descartados ou isolados antes de reiniciar E5; confirmar com o time se há tabelas intermediárias com escrita ativa durante E5.

```

Saída completa em [`docs/checkpoints/05-migracao-forge.md`](../../docs/checkpoints/05-migracao-forge.md).

## Limitações conhecidas

- O plano herda os "A CONFIRMAR" do roteiro; limiares não resolvidos nos elos anteriores viram pendências bloqueantes aqui, de propósito.
- Ações são descritas de forma genérica ("consultar a contagem de linhas da partição") porque o prompt proíbe supor ferramentas; o operador traduz para o stack real.
- O Gemini 3.1 Pro propôs apagar as tabelas paralelas no rollback; o Sonnet 4.6 manda preservá-las para análise de causa. Use o Sonnet.
