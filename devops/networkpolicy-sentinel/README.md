---
nome: Endurecimento de NetworkPolicy
descricao: Reescreve um manifesto de NetworkPolicy permissivo na versão endurecida (default-deny + allows mínimos com comentário por regra) a partir das regras do padrão interno e do mapa de serviços, com autoverificação ao final
versao: 1.0.0
tags: [kubernetes, networkpolicy, seguranca, hardening, default-deny]
inputs:
  - nome: manifesto
    descricao: YAML da NetworkPolicy barrada na revisão, como está hoje
  - nome: regras
    descricao: Regras do padrão interno para o namespace (quem pode entrar, para onde pode sair, default-deny, comentário obrigatório)
  - nome: mapa_servicos
    descricao: Mapa dos serviços do cluster com namespace, labels e portas, uma linha por serviço
---

# Endurecimento de NetworkPolicy

## Objetivo

Reescrever um manifesto de NetworkPolicy barrado por ser permissivo demais na versão que o padrão da Aegis exige: default-deny explícito no namespace, políticas de allow com seletores exatos do mapa de serviços, portas mínimas, um comentário por regra dizendo qual fluxo legítimo ela libera e uma autoverificação ao final contra sete perguntas de revisor. Quando o mapa não traz um dado necessário (por exemplo, a porta de escuta do serviço protegido), o prompt marca a regra como bloqueio de aprovação em vez de inventar.

## Quando usar

- Toda vez que uma NetworkPolicy for barrada na revisão de segurança ou precisar nascer já endurecida.
- Como primeira rodada de um processo de verificação e refino: a autoverificação do prompt é o ponto de partida, não o fim. Um revisor humano (ou um prompt de revisão) faz a segunda rodada.

## Exemplo de uso

Entradas: [`exemplos/manifesto-permissivo.yaml`](./exemplos/manifesto-permissivo.yaml) em `{{manifesto}}`, [`exemplos/regras-padrao.txt`](./exemplos/regras-padrao.txt) em `{{regras}}` e [`exemplos/mapa-servicos.txt`](./exemplos/mapa-servicos.txt) em `{{mapa_servicos}}`.

Saída real (Gemini 3.8 Flash, `thinkingLevel: LOW`), política de allow com uma regra de ingress e uma de egress:

```yaml
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
```

Saída completa, as três iterações (v1 → verificação → v2 → verificação → v3) e a rodada com revisora de segurança simulada em [`docs/checkpoints/06-networkpolicy-sentinel.md`](../../docs/checkpoints/06-networkpolicy-sentinel.md).

## Limitações conhecidas

- A política só expressa o lado do namespace protegido. Fluxos entre namespaces também dependem de a política do namespace vizinho permitir o lado oposto; o prompt avisa isso nas lacunas, mas não gera as políticas vizinhas.
- Sem a porta de escuta do serviço protegido no mapa, o ingress sai sem `ports` e marcado `A CONFIRMAR`. O manifesto não deve ser aplicado antes de resolver isso.
- A saída tem cerca de 90 linhas e fica na borda do limite de 5 s do playbook: o Gemini 3.8 Flash oscila entre 4,5 e 7 s; o Haiku 4.5 leva mais de 9 s e ainda imprime a preparação antes do YAML. Ver Checkpoint 08.
- A revisora simulada (Sonnet 4.6) levantou um falso positivo sobre semântica E/OU dos seletores; a estrutura foi verificada com parser YAML. Revisões de IA precisam de checagem humana ou mecânica, não de obediência.
