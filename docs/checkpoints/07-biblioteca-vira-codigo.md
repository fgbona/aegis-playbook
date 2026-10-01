# Checkpoint 07 — A biblioteca vira código

## Entrega

- **Repositório:** https://github.com/fgbona/aegis-playbook — criado a partir do template [`fabricioveronez/prompt-registry`](https://github.com/fabricioveronez/prompt-registry) (clone como ponto de partida, commit inicial `chore: inicia playbook a partir do template prompt-registry`).
- **Prompt de exemplo no formato completo:** [`devops/triagem-de-pods/`](../../devops/triagem-de-pods/), com [`prompt.md`](../../devops/triagem-de-pods/prompt.md) (frontmatter + texto com `{{snapshot}}`) e [`README.md`](../../devops/triagem-de-pods/README.md) (mesmo frontmatter + objetivo, quando usar, exemplo real, limitações). Os outros oito prompts seguem o mesmo formato.

## Como o playbook foi mapeado nas convenções do template

| Convenção do template | Como ficou no playbook |
|---|---|
| Categorias como pastas na raiz, kebab-case, sem aninhar | Todos os nove prompts caem em `devops/`. As outras quatro categorias do template (desenvolvimento, produtividade, finanças, criação de conteúdo) foram removidas: só tinham um README com "nenhum prompt cadastrado", e pasta vazia num playbook de operações é ruído, não estrutura. A convenção que fica é a regra, não a lista de exemplo: categoria nova nasce com o primeiro prompt dela. |
| Um prompt por pasta, nomeada pelo resultado, não pela técnica | `triagem-de-pods`, `nota-de-triagem`, `causa-raiz`, `decisao-backpressure`, `networkpolicy-sentinel`. A cadeia do Checkpoint 05 virou três pastas (`diagnostico-pipeline-batch`, `roteiro-migracao-event-driven`, `plano-executavel-de-etapa`), porque cada elo é um prompt com seus próprios inputs; o README de cada um aponta os outros dois. |
| `prompt.md` = frontmatter + texto puro com `{{variavel}}` | Os placeholders gerados pelo meta-prompting vieram em maiúsculas ou com chave simples (`{{SNAPSHOT}}`, `{ESTADO}`) e foram normalizados para minúsculas com duas chaves. Nenhum texto explicativo dentro do `prompt.md`. |
| `README.md` = mesmo frontmatter + documentação humana | Frontmatter idêntico byte a byte (verificado com `diff` em cada commit). Seções: Objetivo, Quando usar, Exemplo de uso (com saída real do modelo), Limitações conhecidas. |
| Frontmatter com `nome`, `descricao`, `versao`, `tags`, `inputs` | `inputs` lista exatamente os placeholders do prompt, com descrição. É aqui que "todo prompt é parametrizável" vira estrutura: o placeholder só existe se estiver documentado no `inputs`, e vice-versa. |
| `versao` semver, nascendo em `1.0.0` | Todos em `1.0.0`. As iterações v1 → v2 → v3 dos checkpoints aconteceram antes do primeiro commit do prompt; a partir daqui, mudança de texto sobe a versão por commit semântico. |
| Commits semânticos com escopo na categoria | `feat(devops): adiciona prompt de triagem de pods`, `feat(devops): adiciona cadeia de prompts da migração do Forge para event-driven`, e assim por diante, um commit por prompt (ou por cadeia). |
| Índices nos READMEs (raiz e categoria) atualizados a cada prompt | Atualizados no mesmo commit do prompt; o README raiz ganhou uma seção do playbook com a tabela de checkpoints. |

## O que foi acrescentado ao template (e por quê)

- **`exemplos/` dentro de cada prompt.** As entradas de exemplo (snapshots, alertas, artefatos, manifestos) são dados, não prompt; viver ao lado do `prompt.md` permite que o README e o `promptfooconfig.yaml` as referenciem com `file://` sem duplicar texto.
- **`promptfooconfig.yaml` ao lado do `prompt.md`** (Checkpoint 08 em diante). O teste viaja com o prompt.
- **`docs/checkpoints/`.** Registro de decisão por checkpoint: prompt, execução real (modelo, latência, custo, output), curadoria, meta-prompt usado e comparações de modelos. O template não tem esse espaço, e o `README.md` do prompt não é o lugar certo: ele documenta o prompt, não a história de como ele nasceu.
- **`CLAUDE.md` revisado.** A frase "não há código executável, build, testes ou pipeline" deixou de ser verdade; a seção "Testes e pipeline" descreve como rodar e o que o CI faz.

## O que ficou de fora de propósito

- O slash command `/catalogar` do template foi mantido, mas os prompts foram catalogados à mão: o comando grava o texto do prompt byte a byte, e aqui cada prompt passou por refino antes de entrar.
- Meta-prompts não entraram na biblioteca. Estão no final de cada doc de checkpoint, como registro do caminho.
