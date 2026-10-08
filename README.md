# myworkflowwithai

Meu fluxo de trabalho com IA no [Claude Code](https://claude.com/claude-code): um
conjunto de **agentes** (subagents) e **skills** (slash commands) que levam uma demanda
da ideia até a Pull Request revisada — com o agente principal orquestrando, modelos
menores executando o trabalho simples e tudo passando por revisão.

## Visão geral do fluxo

```
            ┌──────────────┐        ┌──────────────┐
  ideia ──▶ │ /new-project │        │  /plan-task  │ ◀── task / issue
            └──────┬───────┘        └──────┬───────┘
                   │ entrevista → plano (t01, t02, ...) → aprovação
                   ▼                       ▼
            ┌───────────────────────────────────────┐
            │ execução: task-executor (haiku/sonnet)│
            │ + agente principal nas complexas      │
            │ + revisão obrigatória do orquestrador │
            └──────────────────┬────────────────────┘
                               ▼
                     ┌───────────────────┐
                     │ commit-pr         │  commits atômicos + push + PR
                     └─────────┬─────────┘
                               ▼
                     ┌───────────────────┐
                     │ pr-reviewer       │  review detalhado da PR
                     └───────────────────┘
```

1. **Planejar** — `/new-project` (projeto do zero) ou `/plan-task` (demanda em projeto
   existente) entrevistam até eliminar ambiguidades e propõem um plano em subtarefas
   numeradas. Nada é alterado antes da aprovação.
2. **Executar** — subtarefas simples/médias vão para o `task-executor` com modelos
   menores; as complexas ficam com o agente principal, que revisa todo o trabalho
   delegado.
3. **Versionar** — `commit-pr` organiza as alterações em commits atômicos
   (Conventional Commits), faz push e abre a PR.
4. **Revisar** — `pr-reviewer` analisa a PR e entrega achados com arquivo:linha,
   severidade e sugestão de correção.

## Estrutura

```
.
├── agents/
│   ├── commit-pr.md       # commits atômicos + push + PR
│   ├── pr-reviewer.md     # revisão criteriosa de PRs (somente leitura)
│   └── task-executor.md   # executa uma subtarefa já planejada
└── skills/
    ├── new-project/SKILL.md   # /new-project
    └── plan-task/SKILL.md     # /plan-task
```

## Skills

### `/new-project <ideia>`

Cria um projeto novo do zero até um MVP funcional.

| Fase | O que acontece |
|------|----------------|
| 1. Interrogatório | Perguntas uma por vez (sempre com uma resposta recomendada) cobrindo produto, escopo do MVP, tipo de app, stack, qualidade, ambiente/deploy e repositório. Descobre sozinho o que dá (ferramentas instaladas, versões). |
| 2. Especificação e plano | Apresenta a especificação do MVP, a definição de pronto e o plano t01..tNN. Pede aprovação: **Aprovar e construir** / **Ajustar** / **Só salvar a especificação**. |
| 3. Construção | `git init`, grava `docs/SPEC.md`, consulta versões atuais dos pacotes e executa o plano (t01 = scaffold, sempre pelo agente principal). |
| 4. Revisão | O orquestrador lê o diff e roda a verificação de cada tarefa delegada; re-delega ou corrige quando necessário. |
| 5. Entrega | Instala do zero, roda lint/testes, exercita a jornada principal e reporta a definição de pronto com honestidade. Oferece o commit inicial via `commit-pr`. |

### `/plan-task <descrição | issue | arquivo>`

Transforma uma task em entrega dentro de um projeto existente.

| Fase | O que acontece |
|------|----------------|
| 1. Contexto | Lê a issue/arquivo citado e explora o código para só perguntar o que o código não responde. |
| 2. Entrevista | Rodadas de 1–4 perguntas objetivas até não restar ambiguidade. |
| 3. Resumo e plano | Escopo, premissas, critérios de aceite e plano t01..tNN. Pede aprovação: **Aprovar e executar** / **Ajustar o plano** / **Só salvar o plano** (em `.claude/plans/<slug>.md`). |
| 4. Execução | Segue as dependências, delegando em paralelo quando possível. |
| 5. Revisão | Obrigatória para toda tarefa delegada: diff real, verificação, aderência ao pedido. |
| 6. Encerramento | Roda a suíte completa, reporta status por tarefa e oferece `commit-pr` + `pr-reviewer`. |

**Regra de delegação** (comum às duas skills):

| Complexidade | Exemplos | Executor |
|--------------|----------|----------|
| simples | boilerplate, configs, CRUD seguindo padrão, testes seguindo exemplo, docs | `task-executor` + `haiku` |
| média | lógica localizada e bem especificada | `task-executor` + `sonnet` |
| complexa | arquitetura, design de API, auth/segurança, concorrência, integrações | agente principal |

## Agentes

| Agente | Modelo | Ferramentas | Função |
|--------|--------|-------------|--------|
| `commit-pr` | sonnet | Bash, Read | Separa o working tree em commits atômicos (Conventional Commits), cria branch se estiver na padrão, faz push e abre a PR via `gh`. |
| `pr-reviewer` | opus | Bash, Read | Revisão somente-leitura de uma PR/branch: correção, casos de borda, segurança, contratos, performance, testes. Saída com veredito e achados por severidade. |
| `task-executor` | haiku (o orquestrador pode trocar por chamada) | Bash, Read, Edit, Write | Executa uma única subtarefa especificada, roda a verificação e reporta. Para e pergunta se a especificação for ambígua. |

### Garantias importantes

- **`commit-pr`**: nunca adiciona assinatura/`Co-Authored-By` de IA; nunca usa
  `--force`, `--amend`, `--no-verify`, rebase ou reset; nunca commita segredos
  (`.env`, chaves, tokens); não corrige código se um hook falhar — apenas reporta.
- **`pr-reviewer`**: estritamente somente-leitura; não publica comentários, não aprova
  e não faz merge sem pedido explícito; todo achado é verificado no código real.
- **`task-executor`**: só altera os arquivos listados na tarefa; não commita, não
  instala dependências não pedidas e não toma decisões de design.

## Instalação

Os arquivos ficam em `~/.claude/` (escopo do usuário, valem para todos os projetos) ou
em `<projeto>/.claude/` (escopo do projeto).

```bash
git clone https://github.com/dev-fritz/myworkflowwithai.git
cd myworkflowwithai

mkdir -p ~/.claude/agents ~/.claude/skills
cp agents/*.md ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
```

Para manter sincronizado com o repositório, use links simbólicos em vez de cópia:

```bash
ln -s "$PWD"/agents/*.md ~/.claude/agents/
ln -s "$PWD"/skills/* ~/.claude/skills/
```

Reinicie o Claude Code e confira com `/agents` e digitando `/` para ver as skills.

## Uso

```text
/new-project um app para controlar gastos compartilhados entre amigos
/plan-task https://github.com/org/repo/issues/42
```

Os agentes podem ser chamados diretamente pedindo em linguagem natural:

```text
commita essas alterações e abre a PR
revisa a PR #57
```

## Requisitos

- [Claude Code](https://claude.com/claude-code)
- `git` e [`gh`](https://cli.github.com/) autenticado (para `commit-pr` e `pr-reviewer`)
