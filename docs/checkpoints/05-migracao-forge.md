# Checkpoint 05 — Migrando o Forge de lote para tempo real (cadeia de prompts)

## 1. Prompts parametrizáveis da cadeia

| Elo | Prompt | Entradas | Saída |
|---|---|---|---|
| 1 | [`devops/diagnostico-pipeline-batch`](../../devops/diagnostico-pipeline-batch/prompt.md) | `{{estado_atual}}` | diagnóstico com IDs I*, A*, F*, C*, Q* |
| 2 | [`devops/roteiro-migracao-event-driven`](../../devops/roteiro-migracao-event-driven/prompt.md) | `{{diagnostico}}` (saída do elo 1), `{{requisitos}}` | etapas E* com conclusão, contratos em risco e volta |
| 3 | [`devops/plano-executavel-de-etapa`](../../devops/plano-executavel-de-etapa/prompt.md) | `{{roteiro}}` (saída do elo 2), `{{etapa}}` | plano executável e reversível de uma etapa |

Cada prompt diz ao modelo qual é o seu papel na cadeia, o que recebe de quem e quem consome a saída, para que não resolva o problema inteiro. Os IDs são o contrato entre os elos: o roteiro cita F1 e C3 do diagnóstico; o plano cita E4, R1 e Q5 do roteiro.

## 2. Execução

**Modelo:** Claude Sonnet 4.6 (`claude-sonnet-4-6`, Anthropic) nos três elos.

**Por que esse modelo:** numa cadeia, erro no elo 1 contamina os seguintes, então o elo mais barato não pode ser o mais fraco. Comparei o Gemini 3.8 Flash no elo 1 (6 s, US$ 0,006) com o Sonnet (35 s, US$ 0,03): o Flash cumpriu o formato, mas achou um ponto frágil onde o Sonnet achou três com mecanismo explicado, e fez cinco perguntas contra sete. Como o diagnóstico é a única fonte do roteiro, fiquei com o Sonnet e mantive o mesmo modelo nos elos 2 e 3 para consistência de estilo. Custo total da cadeia: US$ 0,11 em três chamadas de 35 a 42 s.

### Elo 1 — diagnóstico (35 s, US$ 0,031)

Entrada: [`forge-estado-atual.txt`](../../devops/diagnostico-pipeline-batch/exemplos/forge-estado-atual.txt).

```
INVENTÁRIO
I1 | Ingestão | forge-batch-ingest | job cron, agenda horária (a cada 60 min), formato e volume por ciclo A CONFIRMAR: formato de entrada e volume por ciclo
I2 | Transformação | etapa 1 de 14 | ordem e função específica A CONFIRMAR: identidade de cada etapa
I3 | Transformação | etapas 2–13 de 14 | ordem e função específica A CONFIRMAR: identidade de cada etapa
I4 | Transformação | etapa 14 de 14 | última etapa da cadeia Spark; função específica A CONFIRMAR
I5 | Transformação | pipeline Spark (conjunto das 14 etapas) | duração total declarada ~40 min; executado sequencialmente (inferido, dado "encadeadas")
I6 | Destino | tabelas no data warehouse | particionadas por hora; nome do DW e esquema das tabelas A CONFIRMAR
I7 | Consumidores | Sentinel | lê tabelas agregadas geradas pelo Forge; horário e frequência de leitura A CONFIRMAR
I8 | Consumidores | Cerebro | indexa eventos transformados gerados pelo Forge; horário e frequência A CONFIRMAR
I9 | Consumidores | relatórios de billing da Pepper | rodam de madrugada; dependência das tabelas do Forge; horário exato e janela de tolerância A CONFIRMAR

ACOPLAMENTOS
A1 | Sentinel depende de Forge | contrato: tabelas agregadas, particionadas por hora | declarado; horário-limite e esquema A CONFIRMAR
A2 | Cerebro depende de Forge | contrato: eventos transformados (tabelas ou outro canal A CONFIRMAR), partição por hora (inferido, dado particionamento do destino) | declarado; esquema e horário-limite A CONFIRMAR
A3 | Pepper (billing) depende de Forge | contrato: tabelas do DW disponíveis antes do início dos relatórios noturnos; horário-limite exato A CONFIRMAR | declarado; esquema A CONFIRMAR
A4 | etapas 2–14 dependem da etapa anterior | contrato: saída de cada etapa é entrada da seguinte, dentro do mesmo pipeline Spark | inferido, dado "encadeadas"
A5 | forge-batch-ingest depende do cron | contrato: disparo a cada 60 min; sem mecanismo de disparo alternativo declarado | declarado; comportamento em caso de atraso do cron A CONFIRMAR

PONTOS FRÁGEIS
F1 | pipeline completo (ingestão + 14 etapas) | acúmulo de volume por falha em cascata: quando um ciclo de 60 min falha, os dados desse ciclo não são descartados nem reprocessados imediatamente; no ciclo seguinte o job processa o volume normal mais o volume retido do ciclo anterior, dobrando a carga sobre o Spark e o destino; se o ciclo seguinte também atrasar ou falhar, o acúmulo pode crescer além do dobro (inferido); consumidores que dependem de partições horárias recebem dados atrasados ou ausentes | evidência: "se um lote falha, o próximo acumula o dobro de volume"
F2 | janela de tempo entre fim do pipeline e início dos relatórios da Pepper | mecanismo: o pipeline leva ~40 min de processamento após o disparo do cron; se o cron disparar no início da hora e o pipeline durar 40 min, a partição fica disponível ~40 min depois; qualquer atraso (falha, reprocessamento, acúmulo F1) pode ultrapassar o horário-limite dos relatórios noturnos da Pepper, causando execução sobre dados incompletos ou ausentes | evidência: duração declarada de ~40 min + relatórios de madrugada declarados; margem exata A CONFIRMAR
F3 | cadeia de 14 etapas Spark | mecanismo: falha em qualquer etapa intermediária interrompe toda a cadeia (inferido, dado "encadeadas"); não há declaração de ponto de retomada parcial (checkpoint), logo uma falha na etapa 13 exige reprocessamento desde o início (inferido) | evidência: "14 etapas de processamento encadeadas"; ausência de menção a checkpoints

CONTRATOS A PRESERVAR
C1 | Sentinel | recebe tabelas agregadas particionadas por hora produzidas pelo Forge | A CONFIRMAR: horário-limite de disponibilidade, esquema, granularidade da agregação; quebra se partições atrasarem ou o esquema mudar
C2 | Cerebro | recebe eventos transformados do Forge (canal exato A CONFIRMAR: tabela, fila ou outro) | A CONFIRMAR: contrato de esquema, horário-limite e frequência de indexação; quebra se eventos chegarem fora de ordem, atrasados ou com esquema alterado
C3 | Pepper (billing) | recebe tabelas do DW completas e disponíveis antes do início dos relatórios noturnos | A CONFIRMAR: horário-limite exato, quais tabelas, esquema; quebra se dados estiverem incompletos ou indisponíveis no horário dos relatórios

PERGUNTAS EM ABERTO
Q1 | Existe mecanismo de reprocessamento ou reexecução manual quando um ciclo falha? | define se a migração precisa preservar ou substituir lógica de recuperação de falhas; ausência de reprocessamento agrava F1 | F1, A5
Q2 | Qual o horário-limite (deadline) que Sentinel, Cerebro e Pepper toleram para receber os dados de cada partição horária? | sem esse dado o roteiro não pode dimensionar SLA do novo modelo nem avaliar risco de quebra de contrato | C1, C2, C3, F2
Q3 | As 14 etapas Spark possuem checkpoints ou pontos de retomada intermediários? | determina o custo de reprocessamento em caso de falha parcial e influencia a estratégia de decomposição das etapas na migração | F3, I5
Q4 | Qual o volume típico de dados por ciclo de 60 min (e o volume no cenário de acúmulo)? | necessário para avaliar viabilidade de processamento contínuo e dimensionar o novo modelo | I1, F1
Q5 | O destino (data warehouse) e o esquema das tabelas são compartilhados com outros sistemas além de Sentinel, Cerebro e Pepper? | outros consumidores não declarados podem ter contratos que a migração quebraria sem aviso | I6, C1, C2, C3
Q6 | O forge-batch-ingest lê de qual fonte (banco transacional, fila, lago de dados, API)? | o tipo de fonte determina se o disparo por evento é viável e qual mecanismo de captura de mudança deve ser avaliado | I1, A5
Q7 | Há janela de manutenção ou indisponibilidade programada do DW que o pipeline já respeita? | restrições de janela existentes precisam ser preservadas ou renegociadas no roteiro | I6, C3, F2
```

### Elo 2 — roteiro (42 s, US$ 0,041)

Entradas: a saída do elo 1 acima e [`forge-requisitos.txt`](../../devops/roteiro-migracao-event-driven/exemplos/forge-requisitos.txt).

```
PREMISSAS
P1 | O cron horário é o único mecanismo de disparo do Forge; o Relay será a nova fonte de eventos contínuos — sustentado por I1, A5, Q6 e pelo requisito de consumo contínuo.
P2 | Os contratos C1, C2 e C3 devem permanecer íntegros durante toda a migração; nenhuma etapa pode quebrar partições horárias, esquema ou disponibilidade para Sentinel, Cerebro e Pepper — sustentado por C1, C2, C3 e pelo requisito de manter dependentes funcionando.
P3 | A migração ocorre em passos incrementais com reversibilidade explícita; o cron só é desligado após todos os consumidores migrados e período de observação — sustentado por F1, F3 e pelo requisito de ausência de big-bang.
P4 | As perguntas Q2, Q4 e Q6 bloqueiam decisões de dimensionamento e viabilidade; as demais (Q1, Q3, Q5, Q7) informam etapas específicas mas não bloqueiam o início da sequência.
P5 | O volume por ciclo e os horários-limite dos consumidores são desconhecidos; limiares dos critérios de conclusão marcados como A CONFIRMAR onde necessário — sustentado por Q2, Q4.

ETAPAS
E1 | Resolução dos bloqueios críticos | nenhum | depende de: nenhuma | C1, C2, C3
   conclusão: Q2, Q4 e Q6 respondidos e registrados formalmente pelo time | volta: não se aplica (etapa de levantamento); pode ser refeita a qualquer momento antes de E2

E2 | Instrumentação de observabilidade do pipeline atual | nenhum | depende de: E1 | nenhum
   conclusão: métricas de duração por ciclo, volume ingerido, horário de disponibilidade das partições e taxa de falha do cron coletadas por A CONFIRMAR: número mínimo de ciclos consecutivos sem anomalia | volta: remoção da instrumentação sem efeito sobre o pipeline; possível até E4

E3 | Publicação paralela dos eventos do Forge no Relay sem alterar o fluxo batch | Strangler Fig — introdução de canal paralelo sem corte | depende de: E1, E2 | C1, C2, C3
   conclusão: 100 % dos eventos processados pelo batch também publicados no Relay, confirmado por comparação de contagem entre saída do DW e fila do Relay por A CONFIRMAR: número de ciclos | volta: desativação da publicação no Relay; o batch continua inalterado; possível até E5

E4 | Substituição do pipeline Spark monolítico por processamento em micro-lotes contínuos sobre o Relay | Strangler Fig — substituição incremental do núcleo | depende de: E3 | C1, C2, C3
   conclusão: micro-lotes processam o mesmo volume que o batch em janelas de A CONFIRMAR: duração-alvo por bloco, sem atraso acumulado e sem divergência de esquema nas tabelas de destino | volta: reativação do pipeline Spark original; o cron permanece ativo durante toda esta etapa; possível até E5

E5 | Execução paralela de batch e evento-driven com comparação de saídas | Parallel Run — execução dupla com comparação antes do corte | depende de: E4 | C1, C2, C3
   conclusão: saídas do batch e do novo pipeline divergem em menos de A CONFIRMAR: limiar de tolerância (contagem de registros e valores agregados) por A CONFIRMAR: número de ciclos consecutivos | volta: desligamento do novo pipeline; batch assume sozinho; possível até E6

E6 | Migração de Cerebro para consumir do novo pipeline | migração de consumidor por vez | depende de: E5 | C2
   conclusão: Cerebro indexa eventos do novo pipeline sem erros de esquema e dentro do horário-limite A CONFIRMAR por A CONFIRMAR: número de ciclos | volta: Cerebro volta a ler da fonte anterior; possível até E7

E7 | Migração de Sentinel para consumir do novo pipeline | migração de consumidor por vez | depende de: E6 | C1
   conclusão: Sentinel lê tabelas agregadas do novo pipeline sem atraso além de A CONFIRMAR: horário-limite e sem divergência de esquema por A CONFIRMAR: número de ciclos | volta: Sentinel volta à fonte anterior; possível até E8

E8 | Migração dos relatórios de billing da Pepper para o novo pipeline | migração de consumidor por vez | depende de: E7 | C3
   conclusão: relatórios noturnos da Pepper executam sobre dados completos do novo pipeline dentro da janela de madrugada por A CONFIRMAR: número de noites consecutivas sem falha | volta: Pepper volta a depender das tabelas geradas pelo batch; possível até E9

E9 | Período de observação com cron ainda ativo como fallback | corte gradual com chave de liga/desliga | depende de: E8 | C1, C2, C3
   conclusão: nenhum consumidor aciona o fallback batch e nenhum contrato C* é violado por A CONFIRMAR: duração mínima do período de observação | volta: reativação do batch como fonte primária via chave de liga/desliga; possível até E10

E10 | Desligamento do cron e encerramento do pipeline batch | nenhum | depende de: E9 | C1, C2, C3
   conclusão: cron removido, pipeline Spark descomissionado e ausência de alertas de contrato C* por A CONFIRMAR: período pós-desligamento | volta: não há reversão automatizada após este ponto; reativação exige redeployment manual

DEPENDÊNCIAS E ORDEM
Ordem linear obrigatória: E1 → E2 → E3 → E4 → E5 → E6 → E7 → E8 → E9 → E10; E6, E7 e E8 são sequenciais entre si (um consumidor por vez) mas cada uma pode ser antecipada ou postergada independentemente desde que E5 esteja concluída; nenhuma etapa admite paralelismo com a seguinte dado o requisito de reversibilidade.

RISCOS RESIDUAIS
R1 | acúmulo de volume no Relay se o novo pipeline atrasar equivale ao F1 do batch | E4, E5, E9 | F1
R2 | janela entre fim do processamento contínuo e início dos relatórios da Pepper pode ser insuficiente se o volume real (Q4) for maior que o estimado | E8, E9 | F2, C3
R3 | falha em etapa intermediária do novo pipeline sem checkpoints replica o risco F3 no modelo evento-driven | E4, E5 | F3
R4 | consumidores não declarados do DW (Q5) podem ser afetados por mudanças de esquema ou particionamento sem que a migração os identifique | E4, E5, E10 | C1, C2, C3

PENDÊNCIAS A CONFIRMAR
A CONFIRMAR: horário-limite de disponibilidade de dados para Sentinel, Cerebro e Pepper (Q2) — bloqueia definição dos limiares de conclusão de E6, E7 e E8
A CONFIRMAR: volume típico por ciclo de 60 min e volume no cenário de acúmulo (Q4) — bloqueia dimensionamento dos micro-lotes em E4 e avaliação de R2
A CONFIRMAR: fonte de leitura do forge-batch-ingest (Q6) — bloqueia viabilidade técnica de E3 (publicação no Relay)
A CONFIRMAR: existência de mecanismo de reprocessamento manual (Q1) — informa se E4 precisa incluir lógica de recuperação equivalente
A CONFIRMAR: existência de checkpoints nas 14 etapas Spark (Q3) — informa custo de fallback em E4 e E5 e mitiga R3
A CONFIRMAR: outros consumidores do DW além de Sentinel, Cerebro e Pepper (Q5) — bloqueia garantia de ausência de quebra de contrato em E4 e E10
A CONFIRMAR: janelas de manutenção do DW (Q7) — informa restrições de horário para E4 e E8
```

### Elo 3 — plano executável da etapa E5 (37 s, US$ 0,038)

Entradas: a saída do elo 2 acima e `etapa = E5` (a execução paralela com comparação, que é a etapa de maior risco).

```
ETAPA
E5 | Execução paralela de batch e evento-driven com comparação de saídas | Parallel Run — execução dupla com comparação antes do corte | contratos em risco: C1, C2, C3 | conclusão: saídas do batch e do novo pipeline divergem em menos de A CONFIRMAR: limiar de tolerância (contagem de registros e valores agregados) por A CONFIRMAR: número de ciclos consecutivos | volta: desligamento do novo pipeline; batch assume sozinho; possível até E6

PRÉ-CONDIÇÕES
P1 | E4 concluída: micro-lotes processam o mesmo volume que o batch sem atraso acumulado e sem divergência de esquema | verificar o critério de conclusão de E4 registrado formalmente pelo time
P2 | E3 concluída: 100 % dos eventos do batch também publicados no Relay | verificar comparação de contagem entre saída do DW e fila do Relay para os ciclos aceitos em E3
P3 | Pipeline batch (cron horário) ainda ativo e gerando partições horárias normalmente | consultar o histórico de execuções do cron e confirmar ausência de falhas recentes
P4 | Tabelas de destino do novo pipeline e do batch identificadas e acessíveis para consulta de contagem e agregados | verificar que ambas as tabelas respondem a consultas de contagem sem erro
P5 | Mecanismo de comparação de saídas definido e testado em ambiente não-produtivo | confirmar que a rotina de comparação retorna diferença zero em dados idênticos

PASSOS
1. Confirmar que o novo pipeline está processando em paralelo ao batch sem gravar nas mesmas tabelas de destino dos consumidores | verificação: tabelas de destino dos consumidores (C1, C2, C3) continuam sendo alimentadas exclusivamente pelo batch
2. Para cada ciclo concluído pelo batch, consultar a contagem de linhas da partição gerada pelo batch e a contagem equivalente gerada pelo novo pipeline | verificação: ambas as contagens retornam valores para o mesmo intervalo temporal
3. Calcular a diferença absoluta de contagem de registros entre as duas saídas por ciclo | verificação: diferença registrada e armazenada em log de comparação com carimbo de tempo
4. Calcular a diferença de valores agregados definidos pelo time entre as duas saídas por ciclo | verificação: diferença registrada no mesmo log de comparação
5. Repetir os passos 2 a 4 por A CONFIRMAR: número de ciclos consecutivos | verificação: log de comparação contém entradas para todos os ciclos do período sem lacunas
6. Ao final do período, verificar se todas as diferenças ficaram abaixo de A CONFIRMAR: limiar de tolerância | verificação: nenhuma entrada do log supera o limiar em contagem nem em agregados

OBSERVAR DURANTE A EXECUÇÃO
M1 | diferença de contagem de registros por ciclo | zero ou abaixo de A CONFIRMAR: limiar | qualquer ciclo com diferença acima do limiar
M2 | diferença de valores agregados por ciclo | zero ou abaixo de A CONFIRMAR: limiar | qualquer ciclo com divergência acima do limiar
M3 | atraso acumulado do novo pipeline em relação ao fechamento da partição batch | A CONFIRMAR: tolerância de atraso | atraso crescente entre ciclos consecutivos
M4 | taxa de falha do novo pipeline por ciclo | zero falhas | qualquer falha não recuperada automaticamente
M5 | disponibilidade das tabelas de destino para Sentinel, Cerebro e Pepper | sem interrupção | consulta sem resposta ou erro de esquema

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

DEFINIÇÃO DE PRONTO
- Log de comparação contém entradas para A CONFIRMAR: número de ciclos consecutivos sem lacunas
- Nenhuma entrada do log supera A CONFIRMAR: limiar de tolerância em contagem de registros
- Nenhuma entrada do log supera A CONFIRMAR: limiar de tolerância em valores agregados
- Nenhum gatilho de rollback foi acionado durante o período de observação
- Contrato C1 atendido: Sentinel leu tabelas agregadas sem erro e sem atraso inaceitável em todos os ciclos do período
- Contrato C2 atendido: Cerebro indexou eventos sem erro de esquema em todos os ciclos do período
- Contrato C3 atendido: relatórios da Pepper executaram sobre dados completos em todos os ciclos do período
- Resultado formal registrado e aprovado pelo time antes de iniciar E6

PENDÊNCIAS A CONFIRMAR
A CONFIRMAR: limiar de tolerância de diferença de contagem de registros — bloqueia passos 3, 6 e gatilhos G1
A CONFIRMAR: limiar de tolerância de diferença de valores agregados — bloqueia passos 4, 6 e gatilhos G2
A CONFIRMAR: número de ciclos consecutivos exigidos para conclusão — bloqueia passo 5 e a definição de pronto
A CONFIRMAR: tolerância de atraso acumulado do novo pipeline — bloqueia gatilho G3 e monitoramento M3
A CONFIRMAR: existência de outros consumidores do DW além de Sentinel, Cerebro e Pepper (Q5) — bloqueia garantia de ausência de quebra de contrato durante E5 (risco R4)
A CONFIRMAR: existência de tabelas intermediárias com escrita ativa pelo novo pipeline durante E5 — bloqueia o item de efeito colateral do procedimento de rollback
A CONFIRMAR: agregados específicos a comparar entre as duas saídas — bloqueia passo 4 e monitoramento M2
```

Checagem humana da cadeia: o diagnóstico identificou o mecanismo do ponto frágil declarado (lote falha → próximo dobra) e inferiu dois que a descrição não cita (janela até o billing da Pepper; cadeia Spark sem checkpoint). O roteiro respeitou os três requisitos: consumo contínuo (E3, E4), continuidade dos consumidores (contratos C1 a C3 em toda etapa) e ausência de big-bang (dez etapas, cron desligado só em E10 após observação em E9). O plano de E5 só detalha E5, cita E4 como pré-condição, e o rollback preserva os dados paralelos para análise.

## 3. Curadoria

**Técnica de criação:** meta-prompting, com um único meta-prompt que pediu os três prompts de uma vez, delimitados. O meta-prompt (no final) descreveu o problema, o papel de cada elo, as garantias comuns (não inventar, seções fixas, saber o próprio lugar na cadeia) e os placeholders de cada um. Sonnet 5.5 gerou os três em uma rodada, já com o esquema de IDs que vira o contrato entre elos. Esse esquema não estava no meu meta-prompt; foi a melhor contribuição do modelo.

**Por que cadeia e não um prompt só:** o enunciado já diz, e a execução confirmou: o elo 2 só conseguiu propor "Strangler Fig" e "Parallel Run" com critério de conclusão por etapa porque recebeu contratos C* e perguntas Q* prontos. Num prompt único o modelo teria de inventar o inventário e planejar ao mesmo tempo, e o resultado sai genérico.

**O que refinei:**

1. Placeholders: `{{DESCRICAO_PIPELINE}}` → `{{estado_atual}}`, `{{DIAGNOSTICO}}`/`{{REQUISITOS}}` → minúsculas, `{{ROTEIRO}}`/`{{ETAPA_ID}}` → `{{roteiro}}`/`{{etapa}}`.
2. Elo 1: a regra "uma linha por etapa de transformação" fez o Gemini Flash emitir 14 linhas idênticas ("Etapa Spark 1 … 14") a partir de uma única frase da descrição. Troquei por "uma linha por etapa quando a descrição nomeia as etapas; se só dá a contagem, uma linha para o conjunto". Rerodei o Flash com a regra nova e o inventário caiu para uma linha de transformação. O Sonnet já agrupava antes da mudança, por isso a saída dele foi mantida como entrada do elo 2.
3. Elo 3: na v1 o Sonnet respondeu com cabeçalhos markdown, negrito, linhas `---` e checkboxes, quebrando a consistência visual com os elos 1 e 2 (85 linhas). Adicionei "sem markdown, títulos em maiúsculas como nos elos anteriores". Na v2, 56 linhas, zero markdown, mesmo conteúdo.

**Comparação de modelos:**

| Elo | Modelo | Saída | Latência | Custo | Observação |
|---|---|---|---|---|---|
| 1 | Gemini 3.8 Flash (thinking LOW), prompt v1 | 43 linhas | 6 s | US$ 0,006 | 14 linhas repetidas de inventário; 1 ponto frágil |
| 1 | Gemini 3.8 Flash (thinking LOW), prompt v2 | inventário agrupado | 6 s | US$ 0,006 | Correção confirmada |
| 1 | **Claude Sonnet 4.6** | 36 linhas | 35 s | US$ 0,031 | 3 pontos frágeis com mecanismo, 7 perguntas |
| 2 | **Claude Sonnet 4.6** | 55 linhas | 42 s | US$ 0,041 | 10 etapas, padrões nomeados |
| 3 | Gemini 3.1 Pro (20k tokens) | 40 linhas | 24 s | US$ 0,039 | Rollback manda apagar dados paralelos |
| 3 | Claude Sonnet 4.6, prompt v1 | 85 linhas | 40 s | US$ 0,041 | Correto, mas em markdown |
| 3 | **Claude Sonnet 4.6, prompt v2** | 56 linhas | 37 s | US$ 0,038 | Escolhido |

**Sanitização:** a descrição cita nomes de sistemas internos e o job `forge-batch-ingest`; nada sensível. O que eu removeria num caso real: nomes de tabelas do warehouse com dados de cliente e o horário exato do billing, que revela janela operacional.

---

### Meta-prompt usado (registro do caminho, não entra na biblioteca)

```
Você é um engenheiro de prompts sênior que escreve prompts para um playbook de operações de SRE e engenharia de dados. Os prompts são parametrizáveis, autocontidos e previsíveis, e aqui formam uma CADEIA: a saída de um prompt é a entrada do seguinte.

Escreva os TRÊS PROMPTS FINAIS da cadeia (não execute a tarefa) para o caso abaixo.

## O problema que a cadeia resolve
Um pipeline de dados roda em lote: um job de cron acorda de hora em hora, lê tudo que acumulou e processa de uma vez numa sequência de etapas de transformação, gravando em tabelas particionadas por hora num data warehouse. Vários sistemas dependem dessas tabelas. O time quer migrar para um modelo orientado a eventos: consumir cada evento de um barramento assim que chega e processar em pequenos blocos, quase em tempo real. A migração é grande demais para um prompt só; jogada de uma vez, a resposta sai rasa e genérica. Por isso ela é quebrada em três elos, cada um resolvendo uma parte e passando o resultado adiante.

## Os três elos
ELO 1 — DIAGNÓSTICO. Entrada: a descrição do estado atual do pipeline (parâmetro `{{estado_atual}}`). Saída: um diagnóstico estruturado que o próximo elo consome: inventário do que existe (ingestão, transformação, destino, consumidores), acoplamentos (quem depende de quê e com que contrato: tabela, partição, horário), pontos frágeis com o mecanismo de falha explicado, o que precisa ser preservado durante a transição (contratos com consumidores), e perguntas em aberto que a descrição não responde. Não propõe solução.

ELO 2 — ROTEIRO. Entradas: o diagnóstico produzido pelo elo 1 (parâmetro `{{diagnostico}}`) e os requisitos da migração (parâmetro `{{requisitos}}`). Saída: a sequência de etapas da migração, do estado atual ao estado alvo, em que cada etapa é pequena, tem critério de conclusão verificável, mantém os consumidores funcionando e tem um caminho de volta. Deve usar padrões conhecidos quando couber (ex.: rodar o novo caminho em paralelo ao antigo e comparar saídas antes de cortar; migrar consumidores um por vez), nomeando o padrão. Não detalha a execução de cada etapa.

ELO 3 — PLANO EXECUTÁVEL DE UMA ETAPA. Entradas: o roteiro produzido pelo elo 2 (parâmetro `{{roteiro}}`) e o identificador da etapa a detalhar (parâmetro `{{etapa}}`). Saída: o plano executável e reversível daquela etapa: pré-condições verificáveis, passos numerados com o que fazer e como verificar que deu certo, o que observar durante (métricas e sinais de problema), o gatilho objetivo de rollback, o procedimento de rollback passo a passo, e a definição de pronto. Deve se restringir à etapa pedida e respeitar o que o roteiro já decidiu.

## Garantias comuns aos três prompts
1. Nunca inventar nomes de sistemas, números, ferramentas ou horários que não estejam nas entradas. Se um dado faltar, escrever "A CONFIRMAR: <o quê>" em vez de supor.
2. Saída estruturada com seções nomeadas em maiúsculas, em ordem fixa, para que o próximo elo encontre o que precisa. Sem introdução nem conclusão.
3. Cada prompt deve dizer ao modelo qual é o seu papel na cadeia (o que recebe de quem e quem vai consumir sua saída) para que não resolva o problema inteiro.
4. Tom técnico e objetivo, em português do Brasil.
5. Cada saída em cerca de 30 a 50 linhas.

## Requisitos de forma
- Cada prompt usa exatamente os placeholders indicados para o seu elo, dentro de tags XML próprias, e nenhum outro.
- Cada prompt em cerca de 30 a 40 linhas.
- Entregue os três prompts separados exatamente pelas linhas delimitadoras `===== ELO 1 =====`, `===== ELO 2 =====` e `===== ELO 3 =====`, sem nenhum outro texto antes, entre ou depois, e sem cercas de código.
```
