# Checkpoint 06 — Endurecendo a NetworkPolicy do Sentinel

## 1. Prompt parametrizável

Está em [`devops/networkpolicy-sentinel/prompt.md`](../../devops/networkpolicy-sentinel/prompt.md). Três parâmetros: `{{manifesto}}` (o YAML barrado), `{{regras}}` (o padrão da Aegis) e `{{mapa_servicos}}` (namespaces, labels e portas). O método é ler regras → mapear fluxos → default-deny → allows → autoverificar, com regras de construção (seletor combinado no mesmo item, portas mínimas, comentário por regra) e uma checklist de sete perguntas que o próprio modelo responde antes de entregar.

## 2. Execução: iterações v1 → verificação → v2 → verificação → v3

**Modelo:** Gemini 3.8 Flash (`gemini-3.8-flash`, Google) com `thinkingLevel: LOW`.

**Por que esse modelo:** o YAML final é determinístico e testável (Checkpoint 08 exige 5 s e US$ 0,01 por chamada), então o candidato precisa ser barato. Comparei com o Claude Haiku 4.5 em todas as rodadas: o Haiku demorou 10 a 15 s, imprimiu a "preparação silenciosa" antes do YAML e envolveu o YAML em cercas de código apesar da instrução contrária. O Gemini obedeceu ao formato nas três versões e ficou entre US$ 0,0046 e 0,0064 por chamada. A revisão de segurança simulada usou o Claude Sonnet 4.6, por ser a tarefa em que raciocínio e exigência importam mais que custo.

### v1 — primeira saída e o que a verificação apontou

Rodei a v1 do prompt nos dois modelos com os asserts que o Checkpoint 08 pede. Os dois falharam. O que a verificação (asserts + leitura minha) apontou:

1. **Os dois outputs continham o literal `- {}`** e falharam o assert `not-contains`. Não era regra allow-all: era a própria checklist de autoverificação, que citava o literal proibido na pergunta "(b) Alguma regra `- {}` ou seletor vazio em allow?". O prompt sabotava o próprio teste.
2. **Perguntas da checklist com polaridade mista.** (b) e (d) esperavam NÃO; as outras esperavam SIM. O Gemini respondeu "SIM (não existe)", que não é SIM nem NÃO.
3. **Comentários duplicados** no Gemini: um acima do item de `ingress`/`egress` e outro repetido dentro de `from`/`to`.
4. **Haiku despejou a análise preliminar** (tabela de fluxos, mapeamento) antes do YAML, apesar de "YAML puro, sem texto antes".
5. **Latência:** Gemini 6,6 s, Haiku 15 s. Haiku também passou de US$ 0,01.

O que a v1 já fazia certo e vale registrar: o Gemini apontou nas lacunas que o mapa não traz a porta de escuta do Sentinel e por isso o ingress ficou sem `ports`. Era a observação mais importante da rodada e veio do próprio modelo.

### v2 — correções no prompt

- Checklist reescrita com todas as perguntas na polaridade positiva ("(b) Nenhuma regra vazia nem seletor vazio em allow?"), uma linha por item, e sem o literal `- {}` em lugar nenhum do prompt (virou "regra vazia, um item de lista cujo único conteúdo são chaves vazias").
- "Os passos 1 e 2 são preparação: nada deles vai para a saída."
- "Um único comentário por item de `ingress`/`egress`; não repita dentro de `from`/`to`."
- "Lacunas e limitações" limitada a 4 linhas, citando explicitamente a porta de escuta do serviço protegido como exemplo de dado ausente.

Resultado: Gemini passou em todos os asserts (4,5 s, US$ 0,0046, 91 linhas). Haiku continuou imprimindo a preparação e ficou em 9,9 s.

### Verificação da v2 por revisora de segurança simulada

Submeti o YAML da v2 a um prompt de revisão (papel de revisora de segurança da plataforma, com as regras e o mapa, pedindo perguntas de revisão, problemas verificáveis com gravidade e veredito). Modelo: Claude Sonnet 4.6. Resultado: **12 perguntas, 6 problemas, veredito REPROVADO.** Os pontos que mudaram o prompt:

- **P2 (alta): ingress sem porta é um allow-all de portas**, violando "nada de allow all". Verdadeiro em substância, mas o mapa não traz a porta do Sentinel e inventar é proibido. A resolução certa não é inventar uma porta: é tornar o bloqueio explícito dentro do YAML.
- **P1 (alta): a combinação namespaceSelector + podSelector no mesmo item precisa ser documentada no comentário** para que uma edição futura não os separe em OU. Barato de atender.
- **Perguntas 1 e 2: o que acontece ao aplicar sobre o que já existe no namespace?** A v2 criou `sentinel-allow` com o mesmo nome do manifesto barrado por coincidência; virou regra.
- P3 (NodeLocal DNS), P4 (egress 443 oculto), P5 (annotations) e P6 (comentário admitindo incompletude) ficaram registrados como perguntas ao time, não como mudança de prompt: dependem de dado que não está nas entradas.

### v3 — correções no prompt

- Se o mapa não trouxer a porta de escuta do serviço protegido, a regra de ingress sai sem `ports` e o comentário começa com `# A CONFIRMAR: porta de escuta ausente no mapa; restringir ports antes de aplicar`; isso é bloqueio de aprovação e vai também em "Lacunas e limitações".
- Comentário de fluxo entre namespaces termina com `(mesmo item = E lógico)`.
- A política de allow principal mantém o `metadata.name` e o `namespace` do manifesto original, para que o `kubectl apply` substitua a versão permissiva em vez de coexistir com ela.

**Saída final real** (Gemini 3.8 Flash, promptfoo `--no-cache`, 2026-10-01, 4,9 s, US$ 0,005):

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: sentinel-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # A CONFIRMAR: porta de escuta ausente no mapa; restringir ports antes de aplicar. Libera: relay-prod/app=relay → sentinel-prod/app=sentinel, todas as portas/TCP, consumo de eventos (mesmo item = E lógico)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: relay-prod
          podSelector:
            matchLabels:
              app: relay
    # A CONFIRMAR: porta de escuta ausente no mapa; restringir ports antes de aplicar. Libera: edge/app=api-gateway → sentinel-prod/app=sentinel, todas as portas/TCP, gateway de API da plataforma (mesmo item = E lógico)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: edge
          podSelector:
            matchLabels:
              app: api-gateway
  egress:
    # Libera: sentinel-prod/app=sentinel → forge-prod/app=forge, 5432/TCP, Forge warehouse Postgres (mesmo item = E lógico)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: forge-prod
          podSelector:
            matchLabels:
              app: forge
      ports:
        - protocol: TCP
          port: 5432
    # Libera: sentinel-prod/app=sentinel → cerebro-prod/app=cerebro, 9200/TCP, Cerebro busca Elasticsearch (mesmo item = E lógico)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cerebro-prod
          podSelector:
            matchLabels:
              app: cerebro
      ports:
        - protocol: TCP
          port: 9200
    # Libera: sentinel-prod/app=sentinel → kube-system/k8s-app=kube-dns, 53/UDP, DNS interno (mesmo item = E lógico)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Libera: sentinel-prod/app=sentinel → kube-system/k8s-app=kube-dns, 53/TCP, DNS interno (mesmo item = E lógico)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: TCP
          port: 53

AUTOVERIFICAÇÃO
(a) SIM — default-deny-all define podSelector: {} e policyTypes: [Ingress, Egress] sem regras de allow
(b) SIM — sentinel-allow usa matchLabels: {app: sentinel} e todas as regras contêm seletores/portas específicos
(c) SIM — sentinel-allow cobre entrada de Relay e API Gateway, e saída para Forge, Cerebro e DNS interno
(d) SIM — sentinel-allow contém apenas os fluxos autorizados nas regras do padrão
(e) SIM — sentinel-allow combina namespaceSelector e podSelector no mesmo item em todas as regras de ingress e egress
(f) SIM — sentinel-allow restringe 5432/TCP para Forge, 9200/TCP para Cerebro e 53/UDP+TCP para DNS
(g) SIM — sentinel-allow possui comentário formatado na linha imediatamente anterior a cada item de ingress e egress

Correções feitas: Nenhuma

Lacunas e limitações
- Porta de escuta de Sentinel ausente no mapa; regras de ingress liberam todas as portas e requerem definição antes de aplicar.
- Comunicação cross-namespace depende de NetworkPolicies recíprocas permitirem os fluxos nos namespaces remotos.
- NetworkPolicy nativa não provê inspeção L7/FQDN nem auditoria nativa de logs de conexão.
```

### Verificação da v3 e o falso positivo da revisora

Submeti a v3 à mesma revisora. Veredito: REPROVADO de novo, por dois grupos de problemas:

- **P1: ingress sem porta.** Continua verdadeiro e continua sem solução dentro das entradas. A diferença é que agora o YAML carrega o bloqueio no comentário e nas lacunas; a decisão de aplicar é humana e depende de um dado que o time precisa fornecer. Aceito como estado final do prompt.
- **P2, P3, P4: "namespaceSelector e podSelector como itens separados dentro do mesmo elemento de lista, indentação ambígua, pode virar OU".** Falso. Parseei o YAML com o `js-yaml` que o promptfoo usa; cada regra tem exatamente um peer e esse peer tem as chaves `namespaceSelector` e `podSelector` juntas:

```
sentinel-allow: podSelector={"matchLabels":{"app":"sentinel"}} policyTypes=["Ingress","Egress"]
  ingress[0]: 1 peer(s) | peer0 keys=namespaceSelector+podSelector | ns=relay-prod   pod={"app":"relay"}       | ports=(sem ports)
  ingress[1]: 1 peer(s) | peer0 keys=namespaceSelector+podSelector | ns=edge         pod={"app":"api-gateway"} | ports=(sem ports)
  egress[0]:  1 peer(s) | peer0 keys=namespaceSelector+podSelector | ns=forge-prod   pod={"app":"forge"}       | ports=TCP/5432
  egress[1]:  1 peer(s) | peer0 keys=namespaceSelector+podSelector | ns=cerebro-prod pod={"app":"cerebro"}     | ports=TCP/9200
  egress[2]:  1 peer(s) | peer0 keys=namespaceSelector+podSelector | ns=kube-system  pod={"k8s-app":"kube-dns"}| ports=UDP/53,TCP/53
```

Um peer por regra, com as duas chaves no mesmo objeto, é a semântica E. A revisora de IA errou com confiança, em três itens de gravidade alta. É o argumento mais forte deste checkpoint a favor de verificação mecânica (parser, `kubectl apply --dry-run`, testes do Checkpoint 08) por cima da verificação por modelo.

## 3. Curadoria

**Técnica de criação:** meta-prompting. O meta-prompt (no final) descreveu o problema, as sete garantias (default-deny, nada de regra vazia, seletores exatos com namespace e pod no mesmo item, portas mínimas, comentário por regra, autoverificação com checklist, não presumir CNI) e os três placeholders. Sonnet 5.5 gerou a v1 em uma rodada.

**Framework:** papel + método em passos + regras de construção + regras de "não invente" + formato com autoverificação embutida. A autoverificação é a parte que faz este prompt ser diferente dos anteriores: a primeira rodada de revisão acontece dentro da própria chamada.

**O que o processo de verificação ensinou:**

1. Teste determinístico pega erro de prompt antes de pegar erro de YAML: o `- {}` na checklist era um bug do prompt que nenhuma leitura humana do YAML pegaria.
2. Revisão por IA achou o problema de substância (ingress sem porta) na v2 e um falso positivo grave na v3. Vale como gerador de perguntas; não vale como veredito.
3. A resposta certa para dado ausente não é inventar nem calar: é um bloqueio visível no artefato (`A CONFIRMAR` no comentário, lacuna listada), que a revisão humana resolve com o time dono do serviço.

**Comparação de modelos** (mesmas entradas, asserts do Checkpoint 08):

| Modelo | Versão | Asserts | Latência | Custo | Observação |
|---|---|---|---|---|---|
| Gemini 3.8 Flash (thinking LOW) | v1 | falha (`- {}` ecoado, latência) | 6,6 s | US$ 0,0064 | Flagrou a porta de ingress ausente |
| Claude Haiku 4.5 | v1 | falha (`- {}`, latência, custo) | 15,0 s | US$ 0,0131 | Preamble antes do YAML |
| **Gemini 3.8 Flash (thinking LOW)** | **v2** | **passa** | **4,5 s** | **US$ 0,0046** | |
| Claude Haiku 4.5 | v2 | falha (latência) | 9,9 s | US$ 0,0097 | Preamble e cercas de código |
| Gemini 3.8 Flash (thinking LOW) | v3 | falha só latência | 5,2 s | US$ 0,0052 | Comentários mais longos |
| Gemini 3.8 Flash (thinking LOW) | v3.1 (comentário curto) | passa / falha latência (4,9 s / 7,2 s em duas rodadas) | 4,9 a 7,2 s | US$ 0,0050 | Na borda do limite; tratado no Checkpoint 08 |

**Sanitização:** manifesto, regras e mapa trazem só nomes de namespaces, labels e portas internas. Nada que identifique cliente. Num ambiente real, removeria IPs de `ipBlock` e nomes de namespaces de clientes antes de enviar ao provedor.

---

### Meta-prompt usado (registro do caminho, não entra na biblioteca)

```
Você é um engenheiro de prompts sênior que escreve prompts para um playbook de operações de SRE e segurança. Os prompts são parametrizáveis, autocontidos e previsíveis.

Escreva o PROMPT FINAL (não execute a tarefa) para o caso abaixo.

## O problema que o prompt resolve
A revisão de segurança barrou um manifesto de NetworkPolicy do Kubernetes por ser permissivo demais (podSelector vazio e regras `- {}` que liberam qualquer origem e qualquer destino). O prompt recebe três blocos como parâmetro: o MANIFESTO permissivo, as REGRAS do padrão interno da empresa (quem pode entrar, para onde pode sair, default-deny obrigatório, comentário obrigatório por regra) e o MAPA DE SERVIÇOS (namespace, labels e portas de cada serviço do cluster). Devolve a versão corrigida e endurecida, pronta para `kubectl apply`.

Esse artefato é crítico: um erro expõe o produto core. Por isso a saída do prompt passa por rodadas de verificação e refino conduzidas por um revisor humano, que vai usar a seção de autoverificação do próprio prompt como ponto de partida.

## O que o prompt precisa garantir na resposta
1. Saída em YAML válido, com um ou mais documentos separados por `---`, contendo: uma NetworkPolicy de default-deny explícita para o namespace (ingress e egress) e uma ou mais NetworkPolicies de allow com os fluxos legítimos. Nada fora do YAML além do bloco de autoverificação ao final.
2. Nenhuma regra `- {}` e nenhum `podSelector: {}` em política de allow. O `podSelector: {}` só é aceito na política de default-deny, porque ali o objetivo é pegar todos os pods.
3. Seletores escritos exatamente com os namespaces e labels do mapa de serviços. Nunca inventar label, namespace ou porta. Tráfego entre namespaces exige `namespaceSelector` combinado com `podSelector` no mesmo item da lista (`from`/`to`), porque itens separados viram OU lógico e abrem mais do que o pretendido. Para selecionar namespace por nome, usar o label `kubernetes.io/metadata.name`.
4. Portas restritas ao necessário: egress para o warehouse só na porta declarada, para a busca só na porta declarada, DNS em UDP e TCP na porta declarada.
5. Toda regra de allow com um comentário `#` na linha acima dizendo qual fluxo legítimo ela libera (origem → destino, finalidade).
6. Ao final, fora do YAML, uma seção AUTOVERIFICAÇÃO em que o modelo confere a própria saída contra uma checklist fixa: (a) existe default-deny para Ingress e Egress? (b) alguma regra `- {}` ou seletor vazio em allow? (c) cada fluxo das regras do padrão tem uma regra correspondente? (d) existe regra que não corresponde a nenhum fluxo do padrão? (e) todo `from`/`to` entre namespaces combina namespaceSelector e podSelector no mesmo item? (f) portas e protocolos batem com o mapa? (g) todo item de allow tem comentário? Cada item respondido com SIM/NÃO e a evidência (nome da política e regra). Se algum item for NÃO, o modelo corrige o YAML antes de entregar e diz o que corrigiu.
7. Não presumir CNI, nem recursos além de NetworkPolicy padrão (`networking.k8s.io/v1`). Se o padrão exigir algo que NetworkPolicy não expressa, dizer isso na autoverificação em vez de inventar.

## Requisitos de forma do prompt
- Em português do Brasil. Comentários do YAML também em português.
- Exatamente três parâmetros: `{{manifesto}}`, `{{regras}}` e `{{mapa_servicos}}`, cada um dentro de uma tag XML própria. Nenhum outro placeholder.
- Deve definir papel, método (ler regras → mapear fluxos no mapa de serviços → escrever default-deny → escrever allows → autoverificar), as regras de "não invente" e o formato de saída (YAML e depois AUTOVERIFICAÇÃO).
- Conciso: o prompt inteiro em cerca de 40 linhas.

Responda apenas com o texto do prompt, sem comentários antes ou depois, sem cercas de código ao redor do prompt inteiro.
```

### Prompt de revisão usado nas rodadas de verificação (ferramenta do processo, não entra na biblioteca)

```
Você é a revisora de segurança da plataforma e vai revisar uma NetworkPolicy antes de ela virar item do playbook. Seja exigente: um erro aqui expõe o produto core. Não reescreva o YAML; só revise.

<regras_do_padrao>
{{regras}}
</regras_do_padrao>

<mapa_de_servicos>
{{mapa_servicos}}
</mapa_de_servicos>

<manifesto_proposto>
{{yaml}}
</manifesto_proposto>

Responda em português, sem introdução, com exatamente estas seções:

PERGUNTAS DE REVISÃO
As perguntas que você faria ao autor antes de aprovar, uma por linha, numeradas, cada uma com o motivo em meia linha. Pense em: o que acontece ao aplicar este manifesto sobre o que já existe no namespace; o que fica aberto que o padrão não cobre; o que a NetworkPolicy padrão não consegue expressar; semântica de seletores (E/OU, namespaceSelector × podSelector, labels de namespace); portas e protocolos; DNS; comportamento com pods que não têm o label protegido; nomes e operação (apply, rollback, auditoria).

PROBLEMAS ENCONTRADOS
Um por linha: "P<n> | gravidade (alta|média|baixa) | onde (política e regra) | o que está errado ou faltando | correção sugerida". Só o que é verificável no YAML, nas regras ou no mapa. Se não houver, escreva "Nenhum".

VEREDITO
APROVADO, APROVADO COM AJUSTES ou REPROVADO, com uma frase de justificativa.
```
