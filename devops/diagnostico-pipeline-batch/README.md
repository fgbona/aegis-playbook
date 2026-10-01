---
nome: Diagnóstico de pipeline em lote
descricao: Elo 1 da cadeia de migração batch → event-driven. Transforma a descrição do pipeline atual num diagnóstico estruturado (inventário, acoplamentos, pontos frágeis, contratos, perguntas) que o roteiro consome
versao: 1.0.0
tags: [migracao, pipeline-de-dados, diagnostico, cadeia-de-prompts, event-driven]
inputs:
  - nome: estado_atual
    descricao: Descrição em texto livre do pipeline em lote como está hoje (ingestão, transformação, destino, pontos frágeis, quem depende dele)
---

# Diagnóstico de pipeline em lote

## Objetivo

Primeiro elo da cadeia de migração batch → event-driven. Recebe a descrição em texto livre do pipeline atual e devolve um diagnóstico estruturado com IDs (inventário I*, acoplamentos A*, pontos frágeis F*, contratos a preservar C*, perguntas em aberto Q*) que o elo seguinte referencia. Não propõe solução.

## Cadeia

1. **`diagnostico-pipeline-batch`** (este) — descrição → diagnóstico.
2. [`roteiro-migracao-event-driven`](../roteiro-migracao-event-driven/) — diagnóstico + requisitos → etapas.
3. [`plano-executavel-de-etapa`](../plano-executavel-de-etapa/) — roteiro + etapa → plano executável e reversível.

## Quando usar

- Antes de planejar qualquer migração de pipeline: o roteiro precisa de um inventário com IDs, não de prosa.
- Quando a descrição do sistema vem de pessoas e tem lacunas; o prompt transforma lacuna em pergunta Q* em vez de supor.

## Exemplo de uso

Entrada (`{{estado_atual}}`): [`exemplos/forge-estado-atual.txt`](./exemplos/forge-estado-atual.txt).

Saída real (Claude Sonnet 4.6), seções de pontos frágeis e contratos:

```
PONTOS FRÁGEIS
F1 | pipeline completo (ingestão + 14 etapas) | acúmulo de volume por falha em cascata: quando um ciclo de 60 min falha, os dados desse ciclo não são descartados nem reprocessados imediatamente; no ciclo seguinte o job processa o volume normal mais o volume retido do ciclo anterior, dobrando a carga sobre o Spark e o destino; se o ciclo seguinte também atrasar ou falhar, o acúmulo pode crescer além do dobro (inferido); consumidores que dependem de partições horárias recebem dados atrasados ou ausentes | evidência: "se um lote falha, o próximo acumula o dobro de volume"
F2 | janela de tempo entre fim do pipeline e início dos relatórios da Pepper | mecanismo: o pipeline leva ~40 min de processamento após o disparo do cron; se o cron disparar no início da hora e o pipeline durar 40 min, a partição fica disponível ~40 min depois; qualquer atraso (falha, reprocessamento, acúmulo F1) pode ultrapassar o horário-limite dos relatórios noturnos da Pepper, causando execução sobre dados incompletos ou ausentes | evidência: duração declarada de ~40 min + relatórios de madrugada declarados; margem exata A CONFIRMAR
F3 | cadeia de 14 etapas Spark | mecanismo: falha em qualquer etapa intermediária interrompe toda a cadeia (inferido, dado "encadeadas"); não há declaração de ponto de retomada parcial (checkpoint), logo uma falha na etapa 13 exige reprocessamento desde o início (inferido) | evidência: "14 etapas de processamento encadeadas"; ausência de menção a checkpoints

CONTRATOS A PRESERVAR
C1 | Sentinel | recebe tabelas agregadas particionadas por hora produzidas pelo Forge | A CONFIRMAR: horário-limite de disponibilidade, esquema, granularidade da agregação; quebra se partições atrasarem ou o esquema mudar
C2 | Cerebro | recebe eventos transformados do Forge (canal exato A CONFIRMAR: tabela, fila ou outro) | A CONFIRMAR: contrato de esquema, horário-limite e frequência de indexação; quebra se eventos chegarem fora de ordem, atrasados ou com esquema alterado
C3 | Pepper (billing) | recebe tabelas do DW completas e disponíveis antes do início dos relatórios noturnos | A CONFIRMAR: horário-limite exato, quais tabelas, esquema; quebra se dados estiverem incompletos ou indisponíveis no horário dos relatórios

```

Saída completa e comparação de modelos em [`docs/checkpoints/05-migracao-forge.md`](../../docs/checkpoints/05-migracao-forge.md).

## Limitações conhecidas

- O diagnóstico é tão bom quanto a descrição: com três linhas de entrada, saem sete perguntas em aberto. Isso é o comportamento desejado, mas o roteiro vai carregar muitos "A CONFIRMAR".
- Modelos pequenos cumprem o formato mas raciocinam menos sobre mecanismos de falha (o Gemini 3.8 Flash achou 1 ponto frágil onde o Sonnet 4.6 achou 3).
