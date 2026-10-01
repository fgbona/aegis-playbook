---
nome: Roteiro de migração para event-driven
descricao: Elo 2 da cadeia de migração batch → event-driven. Recebe o diagnóstico do elo 1 e os requisitos e devolve a sequência de etapas pequenas, verificáveis e reversíveis, nomeando os padrões usados
versao: 1.0.0
tags: [migracao, pipeline-de-dados, roteiro, cadeia-de-prompts, event-driven]
inputs:
  - nome: diagnostico
    descricao: Saída integral do prompt `diagnostico-pipeline-batch` (elo 1), com os IDs I, A, F, C e Q
  - nome: requisitos
    descricao: Requisitos da migração definidos pelo time, um por linha (continuidade, ausência de big-bang, reversibilidade)
---

# Roteiro de migração para event-driven

## Objetivo

Segundo elo da cadeia. Recebe o diagnóstico do elo 1 e os requisitos da migração e devolve a sequência de 5 a 10 etapas pequenas, cada uma com critério de conclusão verificável, contratos em risco e caminho de volta, nomeando padrões conhecidos (Strangler Fig, Parallel Run, migração de consumidor por vez, chave de liga/desliga). Não detalha execução.

## Cadeia

1. [`diagnostico-pipeline-batch`](../diagnostico-pipeline-batch/) — descrição → diagnóstico.
2. **`roteiro-migracao-event-driven`** (este) — diagnóstico + requisitos → etapas.
3. [`plano-executavel-de-etapa`](../plano-executavel-de-etapa/) — roteiro + etapa → plano executável e reversível.

## Quando usar

- Depois do diagnóstico, quando o time precisa decidir a ordem da migração sem big-bang.
- Para gerar o esqueleto de um plano de projeto em que cada etapa já nasce com rollback.

## Exemplo de uso

Entradas: [`exemplos/forge-diagnostico.txt`](./exemplos/forge-diagnostico.txt) em `{{diagnostico}}` (saída real do elo 1) e [`exemplos/forge-requisitos.txt`](./exemplos/forge-requisitos.txt) em `{{requisitos}}`.

Saída real (Claude Sonnet 4.6), três das dez etapas:

```
E3 | Publicação paralela dos eventos do Forge no Relay sem alterar o fluxo batch | Strangler Fig — introdução de canal paralelo sem corte | depende de: E1, E2 | C1, C2, C3
   conclusão: 100 % dos eventos processados pelo batch também publicados no Relay, confirmado por comparação de contagem entre saída do DW e fila do Relay por A CONFIRMAR: número de ciclos | volta: desativação da publicação no Relay; o batch continua inalterado; possível até E5

E4 | Substituição do pipeline Spark monolítico por processamento em micro-lotes contínuos sobre o Relay | Strangler Fig — substituição incremental do núcleo | depende de: E3 | C1, C2, C3
   conclusão: micro-lotes processam o mesmo volume que o batch em janelas de A CONFIRMAR: duração-alvo por bloco, sem atraso acumulado e sem divergência de esquema nas tabelas de destino | volta: reativação do pipeline Spark original; o cron permanece ativo durante toda esta etapa; possível até E5

E5 | Execução paralela de batch e evento-driven com comparação de saídas | Parallel Run — execução dupla com comparação antes do corte | depende de: E4 | C1, C2, C3
   conclusão: saídas do batch e do novo pipeline divergem em menos de A CONFIRMAR: limiar de tolerância (contagem de registros e valores agregados) por A CONFIRMAR: número de ciclos consecutivos | volta: desligamento do novo pipeline; batch assume sozinho; possível até E6
```

Saída completa em [`docs/checkpoints/05-migracao-forge.md`](../../docs/checkpoints/05-migracao-forge.md).

## Limitações conhecidas

- Os limiares dos critérios de conclusão saem como "A CONFIRMAR" quando o diagnóstico não traz número; o roteiro não inventa tolerância.
- A ordem é linear por desenho (requisito de reversibilidade); o prompt não otimiza paralelismo.
