# DevOps

Prompts voltados a **infraestrutura, automação e operação** de sistemas: pipelines de CI/CD, containers, orquestração, provisionamento, observabilidade, confiabilidade e segurança operacional.

## Escopo

Entram aqui prompts relacionados a:

- Pipelines de CI/CD (GitHub Actions, GitLab CI, Jenkins etc.).
- Containers e orquestração (Docker, Kubernetes, Helm).
- Infraestrutura como código (Terraform, Pulumi, Ansible).
- Provedores de nuvem (AWS, GCP, Azure) e seus recursos.
- Observabilidade (logs, métricas, tracing, alertas, dashboards).
- Confiabilidade, SRE, postmortems e análise de incidentes.
- Segurança operacional (hardening, secrets, políticas de acesso).

## Fora de escopo

- Escrita de código de aplicação → usar `desenvolvimento/`.
- Conteúdo educacional sobre DevOps (aulas, artigos, vídeos) → usar `criacao-conteudo/`.

## Prompts

- [triagem-de-pods](./triagem-de-pods/) — Triagem da saúde dos pods de um namespace Kubernetes a partir de um snapshot, com causa provável e próxima ação por pod.
- [nota-de-triagem](./nota-de-triagem/) — Converte um alerta cru do Sentinel na nota de triagem padronizada de cinco linhas (alerta, impacto, hipótese, ação, escalonamento).
- [causa-raiz](./causa-raiz/) — Análise de causa-raiz de degradação num cluster Elasticsearch cruzando configuração, métricas e logs, separando causa de consequência.
- [decisao-backpressure](./decisao-backpressure/) — Apoia a decisão de estratégia de backpressure num barramento de eventos comparando caminhos contra as restrições antes de recomendar, em formato de ADR.
- [diagnostico-pipeline-batch](./diagnostico-pipeline-batch/) — Elo 1 da cadeia de migração batch → event-driven: transforma a descrição do pipeline atual num diagnóstico estruturado com IDs.
- [roteiro-migracao-event-driven](./roteiro-migracao-event-driven/) — Elo 2 da cadeia: recebe o diagnóstico e os requisitos e devolve a sequência de etapas verificáveis e reversíveis da migração.
- [plano-executavel-de-etapa](./plano-executavel-de-etapa/) — Elo 3 da cadeia: recebe o roteiro e o ID de uma etapa e devolve o plano executável com rollback.
