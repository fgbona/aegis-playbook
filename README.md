# Catálogo de prompts

Coleção de prompts em Markdown organizados por categoria/área de domínio. Cada prompt vive em sua própria pasta, contendo o arquivo `prompt.md` (texto puro, pronto para copiar e colar) e um `README.md` com metadados, variáveis e exemplos de uso.

Este repositório faz parte do material dos projetos da pós-graduação em AIOps e Inteligência Artificial com Engenharia Cloud: [pos.veronez.io/pos-aiops](https://pos.veronez.io/pos-aiops/).

Convenções de estrutura, nomenclatura e manutenção estão em [`CLAUDE.md`](./CLAUDE.md).

## Playbook de IA operacional da Aegis

Este repositório é a entrega do desafio IAOps (Módulo 02 · Cap 05 - Avaliação de Prompts) da pós-graduação em AIOps e IA na Engenharia de Cloud. Ele é o playbook da Aegis: prompts parametrizáveis criados por meta-prompting, executados de verdade em dois provedores (Google e Anthropic), testados com promptfoo e gateados em CI.

| Checkpoint | Entrega | Onde |
|---|---|---|
| 01 | Triagem de pods | [`devops/triagem-de-pods`](./devops/triagem-de-pods/) · [doc](./docs/checkpoints/01-triagem-de-pods.md) |
| 02 | Nota de triagem | [`devops/nota-de-triagem`](./devops/nota-de-triagem/) · [doc](./docs/checkpoints/02-nota-de-triagem.md) |
| 03 | Causa-raiz no Cerebro | [`devops/causa-raiz`](./devops/causa-raiz/) · [doc](./docs/checkpoints/03-causa-raiz.md) |
| 04 | Decisão de backpressure | [`devops/decisao-backpressure`](./devops/decisao-backpressure/) · [doc](./docs/checkpoints/04-decisao-backpressure.md) |
| 05 | Cadeia de migração do Forge | [`diagnostico-pipeline-batch`](./devops/diagnostico-pipeline-batch/) → [`roteiro-migracao-event-driven`](./devops/roteiro-migracao-event-driven/) → [`plano-executavel-de-etapa`](./devops/plano-executavel-de-etapa/) · [doc](./docs/checkpoints/05-migracao-forge.md) |
| 06 | NetworkPolicy do Sentinel | [`devops/networkpolicy-sentinel`](./devops/networkpolicy-sentinel/) · [doc](./docs/checkpoints/06-networkpolicy-sentinel.md) |
| 07 | A biblioteca vira código | [doc](./docs/checkpoints/07-biblioteca-vira-codigo.md) |
| 08 | Testes determinísticos (promptfoo) | `promptfooconfig.yaml` em cada prompt estruturado · [doc](./docs/checkpoints/08-testes-deterministicos.md) |
| 09 | Gate de qualidade com LLM-as-judge | [`devops/causa-raiz/promptfooconfig.yaml`](./devops/causa-raiz/promptfooconfig.yaml) · [doc](./docs/checkpoints/09-llm-as-judge.md) |
| 10 | Pipeline em GitHub Actions | [`.github/workflows/`](./.github/workflows/) · [doc](./docs/checkpoints/10-pipeline.md) |

Rodar os testes de um prompt: `cd devops/<prompt> && promptfoo eval -c promptfooconfig.yaml` (precisa de `GOOGLE_API_KEY` e `ANTHROPIC_API_KEY`).

## Como usar

1. Navegar até a categoria de interesse.
2. Abrir o `README.md` do prompt para entender objetivo, variáveis esperadas e limitações.
3. Copiar o conteúdo do `prompt.md` e substituir os placeholders `{{nome_variavel}}` pelos valores desejados.

## Adicionando um prompt

Use o slash command [`/catalogar`](./.claude/commands/catalogar.md) passando o texto do prompt como argumento. Ele analisa, propõe organização (categoria, slug, frontmatter) e, após sua aprovação, escreve os arquivos e atualiza os índices — sem commitar. Convenções completas em [`CLAUDE.md`](./CLAUDE.md).

## Categorias

### [Desenvolvimento](./desenvolvimento/)

Escrita, revisão e refatoração de código, design de APIs e arquitetura, debugging, testes e documentação técnica.

_Nenhum prompt cadastrado ainda._

### [DevOps](./devops/)

Pipelines de CI/CD, containers, orquestração, infraestrutura como código, observabilidade, SRE e segurança operacional.

- [triagem-de-pods](./devops/triagem-de-pods/) — Triagem da saúde dos pods de um namespace Kubernetes a partir de um snapshot, com causa provável e próxima ação por pod.
- [nota-de-triagem](./devops/nota-de-triagem/) — Converte um alerta cru do Sentinel na nota de triagem padronizada de cinco linhas (alerta, impacto, hipótese, ação, escalonamento).
- [causa-raiz](./devops/causa-raiz/) — Análise de causa-raiz de degradação num cluster Elasticsearch cruzando configuração, métricas e logs, separando causa de consequência.
- [decisao-backpressure](./devops/decisao-backpressure/) — Apoia a decisão de estratégia de backpressure num barramento de eventos comparando caminhos contra as restrições antes de recomendar, em formato de ADR.
- [diagnostico-pipeline-batch](./devops/diagnostico-pipeline-batch/) — Elo 1 da cadeia de migração batch → event-driven: transforma a descrição do pipeline atual num diagnóstico estruturado com IDs.
- [roteiro-migracao-event-driven](./devops/roteiro-migracao-event-driven/) — Elo 2 da cadeia: recebe o diagnóstico e os requisitos e devolve a sequência de etapas verificáveis e reversíveis da migração.
- [plano-executavel-de-etapa](./devops/plano-executavel-de-etapa/) — Elo 3 da cadeia: recebe o roteiro e o ID de uma etapa e devolve o plano executável com rollback.
- [networkpolicy-sentinel](./devops/networkpolicy-sentinel/) — Reescreve uma NetworkPolicy permissiva na versão endurecida (default-deny + allows mínimos comentados) com autoverificação, a partir do padrão interno e do mapa de serviços.

### [Produtividade](./produtividade/)

Organização pessoal, gestão de tempo e tarefas, rotina, hábitos, foco e decisões sobre fluxo de trabalho individual.

_Nenhum prompt cadastrado ainda._

### [Finanças](./financas/)

Orçamento, investimentos, planejamento financeiro, impostos e apoio a decisões financeiras.

_Nenhum prompt cadastrado ainda._

### [Criação de Conteúdo](./criacao-conteudo/)

Roteiros, artigos, posts para redes sociais, material didático e copy de divulgação.

_Nenhum prompt cadastrado ainda._

<!--
Ao adicionar um prompt, substituir "Nenhum prompt cadastrado ainda" pela lista:

- [nome-do-prompt](./<slug-da-categoria>/<slug-do-prompt>/) — o que o prompt faz, em uma linha.
-->

## Contribuindo

Antes de adicionar ou alterar um prompt, revisar [`CLAUDE.md`](./CLAUDE.md) — a seção **Manutenção da documentação** lista todos os arquivos que precisam ser atualizados junto com a mudança (este índice incluso).
