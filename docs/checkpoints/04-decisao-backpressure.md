# Checkpoint 04 — Segurando a sobrecarga do Relay

## 1. Prompt parametrizável

Está em [`devops/decisao-backpressure/prompt.md`](../../devops/decisao-backpressure/prompt.md). Três parâmetros: `{{estado}}` (throughput, pico, retenção, consumidores), `{{restricoes}}` (SLAs, orçamento, regras não negociáveis) e `{{opcoes}}` (os caminhos que o time colocou na mesa). O método força a ordem classificar restrições → calcular → comparar → recomendar → riscos e métricas, e a regra de ouro do prompt é que a recomendação só aparece depois da tabela de comparação.

## 2. Execução

**Modelo:** Claude Sonnet 4.6 (`claude-sonnet-4-6`, Anthropic), sem raciocínio estendido.

**Por que esse modelo:** o enunciado diz que o raciocínio importa tanto quanto a recomendação. Testei quatro configurações (tabela na curadoria). O Sonnet 4.6 foi o único que não inventou nada: não criou ferramenta, não deu prazo em sprints, marcou cada estimativa com premissa e devolveu "indeterminado" onde o estado não permite concluir. O Gemini 3.1 Pro, mais caro, inventou "lambdas", "HPA" e prazos de "1 sprint" e "3 sprints" que não estão no cenário. Custo de US$ 0,05 e 60 s por decisão, aceitável para um prompt que roda uma vez por ADR.

**Resultado real** (promptfoo, `--no-cache`, 2026-10-01, v2 do prompt, 60 s, US$ 0,050):

```
RESTRIÇÕES

ELIMINATÓRIAS:
- Perda de mensagem inaceitável: qualquer opção que descarte mensagens sob pico é eliminada sem análise adicional.
- SLA Sentinel ≤ 60s: qualquer opção que, isolada, não garanta esse teto para alerting é eliminada (ou rebaixada a componente secundário).

DE PESO:
- Orçamento 8% acima do previsto: preferir caminhos de menor custo incremental; opções com escalonamento automático ou particionamento por cliente pesam mais no critério custo.
- SLA Forge ≤ 15min de atraso: critério de comparação de tempo de drenagem; não é eliminatório, mas uma opção que drene em mais de 15min para o Forge perde na comparação.

---

CONTAS

Acúmulo no pico:
acúmulo = (taxa_pico − taxa_sustentada) × duração
acúmulo = (320.000 − 180.000) msg/s × (25 × 60) s
acúmulo = 140.000 × 1.500 = 210.000.000 mensagens acumuladas no evento

Capacidade da retenção atual:
DADO FALTANTE: volume médio por mensagem e capacidade de armazenamento em bytes do Relay. Não é possível converter 4h de retenção em número máximo de mensagens sem esses dados. A comparação abaixo usa tempo, não volume.

Tempo de drenagem após o pico (retorno à taxa sustentada):
ESTIMATIVA (premissa: consumo volta à taxa sustentada de 180.000 msg/s imediatamente após o pico; sem dado de taxa por consumidor, usa-se a taxa total como teto de consumo agregado)
drenagem = acúmulo ÷ (taxa_sustentada − taxa_chegada_pós_pico)
DADO FALTANTE: taxa de chegada pós-pico. Se a chegada voltar à taxa sustentada (180.000 msg/s), o denominador é zero — o sistema não drena, apenas para de acumular.
ESTIMATIVA (premissa alternativa: chegada pós-pico cai abaixo de 180.000 msg/s, liberando capacidade de consumo; sem o valor exato, o tempo de drenagem é indeterminado).

Cobertura da retenção:
A retenção de 4h (14.400s) cobre o pico de 25min (1.500s) em tempo — o pico cabe dentro da janela de retenção. Se o volume acumulado (210M msgs) não exceder a capacidade de armazenamento (indeterminado por falta de dado de volume/msg), a retenção cobre o pico.
A retenção cobre o pico? INDETERMINADO — o tempo é suficiente (25min < 4h), mas a capacidade volumétrica não pode ser confirmada sem o tamanho médio de mensagem e o limite de armazenamento do Relay.

---

COMPARAÇÃO

| Critério | Prioridade ao Sentinel | Dead-letter queue (DLQ) | Prioridade ao Sentinel + DLQ para Forge (ADIÇÃO) | Escalonamento automático de consumidores |
|---|---|---|---|---|
| SLA Sentinel (≤60s) | Atende: Sentinel recebe prioridade de consumo, atraso minimizado | Indeterminado: DLQ não diferencia consumidores; Sentinel pode atrasar junto com Forge | Atende: Sentinel priorizado; Forge vai para DLQ sem competir | Indeterminado: depende de velocidade de escalonamento vs. rampa do pico; sem dado de tempo de boot de consumidor |
| SLA Forge (≤15min) | Indeterminado: depende do tempo de drenagem pós-pico (ver CONTAS; indeterminado) | Indeterminado: reprocessamento da DLQ pode exceder 15min sem dado de throughput de reprocessamento | Atende estruturalmente: Forge drena da DLQ quando Sentinel está em dia; tempo real indeterminado sem taxa de drenagem | Indeterminado: se escalonamento for rápido o suficiente, ambos atendem; sem dado de tempo de escalonamento |
| Risco de perda de dados | Baixo: mensagens do Forge ficam no Relay enquanto há retenção; risco se retenção volumétrica for excedida (indeterminado) | Baixo: DLQ persiste mensagens não processadas; risco se DLQ não tiver retenção configurada | Baixo: DLQ persiste Forge; Sentinel consome direto; risco residual igual ao da DLQ isolada | Médio: se escalonamento falhar ou for lento, mensagens acumulam no Relay e podem exceder retenção |
| Custo de infra | Menor: sem novo componente; requer apenas configuração de prioridade no Relay | Menor-médio: DLQ é componente adicional, mas simples; custo de armazenamento da fila depende de volume (indeterminado) | Menor-médio: combina configuração de prioridade + DLQ; custo incremental relativo à DLQ isolada | Maior: escalonamento automático implica instâncias adicionais sob pico; custo proporcional à frequência e duração dos picos |
| Complexidade de implementação | Baixa: configuração de consumer groups com prioridade | Baixa-média: requer lógica de reprocessamento e monitoramento da DLQ | Média: combina duas configurações; requer orquestração de roteamento por consumidor | Alta: requer lógica de autoscaling, métricas de trigger, testes de rampa |
| Tempo até produção | Menor: mudança de configuração | Menor-médio: desenvolvimento da lógica de reprocessamento | Médio: soma dos dois anteriores | Maior: desenvolvimento, testes de carga e validação de rampa |
| O que quebra se a premissa falhar | Se retenção volumétrica for excedida, Forge perde mensagens (premissa: 4h cobre o volume) | Se throughput de reprocessamento for menor que a taxa de chegada, DLQ cresce indefinidamente | Se DLQ não drenar em 15min, Forge viola SLA; Sentinel continua protegido | Se tempo de escalonamento for maior que a rampa do pico, acúmulo ocorre antes de novos consumidores estarem prontos |

---

RECOMENDAÇÃO

CAMINHO: Prioridade ao Sentinel + DLQ para Forge (ADIÇÃO — combinação das opções "dar prioridade ao Sentinel" e "guardar o que não foi processado numa fila separada").

JUSTIFICATIVA: É o único caminho que atende estruturalmente as duas restrições eliminatórias sem adicionar custo variável proporcional ao pico. O Sentinel recebe prioridade de consumo no Relay, protegendo o SLA de 60s sem novo componente de escalonamento. O Forge é roteado para DLQ durante o pico, eliminando competição por capacidade de consumo e preservando todas as mensagens (restrição de não-perda). O custo incremental é o armazenamento da DLQ, menor do que instâncias adicionais de consumidores sob pico, relevante dado o orçamento 8% acima do previsto. O acúmulo calculado de 210M mensagens em 25min precisa caber na DLQ; isso deve ser validado com o tamanho médio de mensagem antes do deploy.

VALE ENQUANTO: a taxa de drenagem da DLQ pelo Forge, após o pico, for suficiente para consumir o acumulado em menos de 15min. Se o tempo de drenagem medido em produção exceder 15min, reavaliar o throughput do consumidor Forge e considerar escalonamento automático exclusivo para ele.

---

DESCARTADOS

- Dead-letter queue (DLQ) isolada: não diferencia Sentinel de Forge; Sentinel pode atrasar junto com Forge, violando SLA de 60s (restrição eliminatória).
- Escalonamento automático de consumidores: maior custo incremental (relevante com orçamento estourado), maior complexidade e tempo até produção; SLA do Sentinel é indeterminado sem dado de tempo de escalonamento vs. rampa do pico; perde na comparação de custo e risco para a combinação recomendada.
- Dividir o Relay por cliente: não endereça o problema de prioridade entre Sentinel e Forge dentro do mesmo cliente; resolve isolamento entre clientes distintos, que não é o problema descrito no estado atual; pode ser reavaliado se o problema de "cliente barulhento" for confirmado com dados.

---

RISCOS E MITIGAÇÕES

RISCO: Volume acumulado de 210M msgs no pico excede capacidade de armazenamento da DLQ ou do Relay, causando descarte | MITIGAÇÃO: Medir tamanho médio de mensagem e provisionar DLQ com capacidade mínima de 210M msgs × tamanho médio antes do go-live; configurar alerta de ocupação da DLQ em 70% da capacidade.

RISCO: Taxa de drenagem da DLQ pelo Forge insuficiente para respeitar SLA de 15min após pico | MITIGAÇÃO: Medir throughput real do Forge em ambiente de staging com carga de reprocessamento; se insuficiente, adicionar escalonamento automático exclusivo para o consumidor Forge (não para o Sentinel).

RISCO: Configuração de prioridade no Relay não isola completamente o Sentinel sob pico extremo (ex.: pico acima de 320k msg/s) | MITIGAÇÃO: Definir limite de consumo mínimo garantido para o Sentinel no Relay (reserva de capacidade); testar com carga sintética acima de 320k msg/s antes do deploy.

RISCO: DLQ sem política de retenção configurada pode perder mensagens do Forge se o reprocessamento atrasar além da janela | MITIGAÇÃO: Configurar retenção da DLQ com valor igual ou superior à retenção do Relay (mínimo 4h); alinhar com a restrição de não-perda.

---

MÉTRICAS DE SUCESSO

- Latência de consumo do Sentinel: mede o atraso entre publicação e consumo pelo Sentinel; limiar de alerta: 45s (margem de 15s antes do SLA de 60s); se violado, indica que a prioridade não está sendo respeitada ou que o Relay está saturado.
- Profundidade da DLQ do Forge: mede o volume de mensagens pendentes de reprocessamento; limiar a definir com baseline (depende do tamanho médio de mensagem, dado faltante); se crescer monotonicamente após o pico, indica que a taxa de drenagem é insuficiente.
- Tempo de drenagem da DLQ após pico: mede o intervalo entre fim do pico e DLQ zerada; limiar de alerta: 12min (margem de 3min antes do SLA de 15min do Forge); se violado, acionar avaliação de escalonamento do Forge.
- Taxa de descarte de mensagens no Relay e na DLQ: mede qualquer evento de drop; limiar: zero (restrição eliminatória de não-perda); qualquer valor acima de zero exige investigação imediata e rollback do roteamento.
- Ocupação de armazenamento da DLQ: mede uso percentual da capacidade provisionada; limiar de alerta: 70% da capacidade; se violado, indica necessidade de expansão antes do próximo pico.
```

Checagem humana: a conta de acúmulo está certa (140 mil msgs/s × 1.500 s = 210 milhões). A observação sobre o denominador zero é correta e importante: "throughput sustentado de 180k" é a taxa de consumo, então se a chegada volta a 180k depois do pico, a fila não drena, só para de crescer; drenar exige folga de consumo ou chegada abaixo do sustentado. A recomendação (prioridade ao Sentinel + fila separada para o Forge) respeita as duas eliminatórias, e o "vale enquanto" aponta exatamente o dado que falta medir.

## 3. Curadoria

**Técnica de criação:** meta-prompting. O meta-prompt (no final) descreveu o problema, as sete garantias (classificar restrições, mostrar contas, comparar ≥ 3 caminhos com critérios fixos, recomendar só depois, descartados, riscos, não inventar) e o formato de sete seções. Sonnet 5.5 gerou a v1 em uma rodada.

**Framework:** é um prompt de decisão, então a estrutura central é a tabela de comparação com linhas fixas (SLA por consumidor, risco de perda, custo, complexidade, tempo, o que quebra se a premissa falhar). Isso é o que impede a resposta de "cuspir" uma recomendação: o modelo precisa preencher a tabela antes.

**O que refinei (v1 → v2):**

1. Placeholders `{ESTADO}`, `{RESTRICOES}`, `{OPCOES}` (chave simples e maiúsculas) viraram `{{estado}}`, `{{restricoes}}`, `{{opcoes}}`.
2. Na v1, a regra "se faltar dado, escreva DADO FALTANTE e não calcule" fez o Sonnet recusar a conta de tempo de drenagem por não ter a taxa de consumo por consumidor. O Gemini 3.8 Flash, no mesmo prompt, calculou ~19 min assumindo chegada zero, sem dizer a premissa. A v2 ganhou uma regra: se faltar a taxa por consumidor, calcule com a taxa sustentada como premissa explícita e compare com o SLA. Na v2 o Sonnet fez a conta, declarou a premissa e descobriu o problema do denominador zero, que é a conclusão mais útil da análise.
3. A v2 ficou em 85 linhas (limite pedido: 40 a 60). Não apertei mais porque o excesso está na tabela e nas contas, que é onde o valor está. Registrei no README.

**Comparação de modelos** (mesmo cenário):

| Modelo | Versão | Saída | Latência | Custo | Veredito |
|---|---|---|---|---|---|
| Claude Sonnet 4.6 | v1 | 66 linhas, completa | 58 s | US$ 0,050 | Correta, recusou a conta de drenagem |
| Gemini 3.1 Pro (8k tokens) | v1 | 19 linhas, truncou na tabela | 66 s | US$ 0,099 | Inutilizável |
| Gemini 3.1 Pro (20k tokens) | v1 | 39 linhas, completa | 74 s | US$ 0,108 | Inventou "lambdas", "HPA", prazos em sprints |
| Gemini 3.8 Flash (thinking LOW) | v1 | 42 linhas | 8 s | US$ 0,007 | Calculou drenagem sem declarar premissa; eliminou DLQ por interpretá-la como descarte |
| **Claude Sonnet 4.6** | **v2** | **85 linhas** | **60 s** | **US$ 0,050** | **Escolhido: conta com premissa, sem invenção** |

O Flash a US$ 0,007 é tentador, e para uma primeira leitura rápida serve. Para a decisão que vai para o ADR, a diferença entre "19 min de drenagem" (Flash, premissa oculta de chegada zero) e "não drena se a chegada voltar a 180k" (Sonnet) é a diferença entre uma decisão certa e uma errada.

**Sanitização:** o cenário tem só números de capacidade e nomes de sistemas internos; nada a remover. Num caso real, o que eu tiraria seriam valores de contrato (SLA com cliente nomeado) e orçamento em moeda, trocando por percentuais, como o enunciado já faz ("8% acima").

---

### Meta-prompt usado (registro do caminho, não entra na biblioteca)

```
Você é um engenheiro de prompts sênior que escreve prompts para um playbook de operações de SRE. Os prompts são reutilizados pelo time, então precisam ser parametrizáveis, autocontidos e previsíveis.

Escreva o PROMPT FINAL (não execute a tarefa) para o caso abaixo.

## O problema que o prompt resolve
O time precisa decidir uma estratégia de backpressure para um barramento de eventos que recebe toda a telemetria dos clientes e alimenta consumidores com SLAs diferentes. Quando um cliente dispara muito mais dado que o normal, o barramento recebe mais do que entrega, a fila acumula e o atraso empurra os consumidores. Não existe resposta única: há vários caminhos conhecidos (priorizar um consumidor, dead-letter queue, particionar por cliente, autoscaling de consumidores, combinações), e cada um tem um preço em custo, complexidade, risco de perda e latência.

O prompt recebe três blocos como parâmetro: o ESTADO atual do barramento (throughput, pico, retenção, consumidores), as RESTRIÇÕES que a solução precisa respeitar (SLAs por consumidor, orçamento, regras não negociáveis como "perda de mensagem é inaceitável") e as OPÇÕES que o time já colocou na mesa. O modelo pode propor opções além das listadas, desde que diga que são adições.

A decisão é cara, então o raciocínio importa tanto quanto a recomendação. O prompt precisa fazer o modelo COMPARAR mais de um caminho, pesando prós e contras contra as restrições, ANTES de recomendar. Uma resposta que já abre com a recomendação está errada.

## O que o prompt precisa garantir na resposta
1. Começar extraindo das restrições quais são ELIMINATÓRIAS (uma opção que viola é descartada) e quais são de PESO (entram na comparação). Ex.: "perda de mensagem inaceitável" é eliminatória; "orçamento 8% acima" é de peso.
2. Fazer as contas que o estado permite: por exemplo, quanto a fila acumula num pico de X msgs/s acima do sustentado durante Y minutos, e se a retenção cobre isso. Mostrar a conta, não só o resultado. Se faltar dado para a conta, dizer qual.
3. Comparar pelo menos três caminhos (das opções listadas e/ou combinações) numa tabela com critérios fixos: atende SLA de cada consumidor?, risco de perda de dados, custo de infra, complexidade de implementação, tempo até estar em produção, e o que quebra quando a premissa falha.
4. Só depois recomendar: um caminho (pode ser combinação), com a justificativa amarrada às restrições, o que foi descartado e por quê, e a condição sob a qual a recomendação deixa de valer.
5. Listar os riscos da recomendação e como mitigá-los, e as métricas que o time deve observar para saber se funcionou.
6. Nunca inventar números que não estejam no estado ou nas restrições; estimativas de custo ou capacidade devem vir marcadas como estimativa e com a premissa explícita.
7. Tom de documento de decisão de arquitetura: objetivo, sem floreio, bom para colar num ADR.

## Requisitos de forma do prompt
- Em português do Brasil.
- Exatamente três parâmetros: `{{estado}}`, `{{restricoes}}` e `{{opcoes}}`, cada um dentro de uma tag XML própria. Nenhum outro placeholder.
- Deve definir papel, método em passos (classificar restrições → calcular → comparar → recomendar → riscos e métricas), regras de "não invente" e um formato de saída fixo com seções nomeadas nesta ordem: RESTRIÇÕES (eliminatórias / de peso), CONTAS, COMPARAÇÃO (tabela), RECOMENDAÇÃO, DESCARTADOS, RISCOS E MITIGAÇÕES, MÉTRICAS DE SUCESSO.
- A tabela de comparação é a única parte em markdown; o resto é texto com rótulos em maiúsculas.
- Saída objetiva: cerca de 40 a 60 linhas.
- Conciso: o prompt inteiro em cerca de 45 linhas.

Responda apenas com o texto do prompt, sem comentários antes ou depois, sem cercas de código ao redor do prompt inteiro.
```
