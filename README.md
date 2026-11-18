# myworkflowwithai

Meu fluxo de trabalho com IA no [Claude Code](https://claude.com/claude-code): um
conjunto de **agentes** (subagents) e **skills** (slash commands) que levam uma demanda
da ideia até a entrega certa (documento, design, POC ou código) e dela até a Pull Request
revisada. O modelo principal (**Fable**) só **orquestra**: entrevista, planeja, escolhe o
nível de cada tarefa, delega e revisa. Quem desenvolve são os subagentes, com o modelo e o
effort que a dificuldade pede.

## Visão geral do fluxo

```
            ┌──────────────┐        ┌──────────────┐
  ideia ──▶ │ /new-project │        │  /plan-task  │ ◀── task / issue
            └──────┬───────┘        └──────┬───────┘
                   │ entrevista → formato da entrega → plano (t01, t02, ...) → aprovação
                   ▼                       ▼
     ┌─────────────┬──────────────┬────────────────────────────────────────┐
     │ documento   │ design*      │ POC / código                           │
     │ (escrito    │ /figcraft-   │ task-executor no nível de cada tarefa  │
     │  pelo orq.) │ design       │ (sonnet · opus · fable)                │
     │             │      ──handoff──▶ + revisão obrigatória do orquestrador│
     └─────────────┴──────────────┴───────────────────┬────────────────────┘
       * design só se o usuário pedir                 ▼
                                            ┌───────────────────┐
                                            │ commit-pr         │  commits atômicos + push + PR
                                            └─────────┬─────────┘
                                                      ▼
                                            ┌───────────────────┐
                                            │ pr-reviewer       │  review detalhado da PR
                                            └───────────────────┘
```

1. **Planejar** — `/new-project` (projeto do zero) ou `/plan-task` (demanda em projeto
   existente) entrevistam até eliminar ambiguidades, perguntam o **formato da entrega**
   (documento, design/protótipo, POC ou código) e propõem um plano em subtarefas
   numeradas, cada uma com um **nível de dificuldade**. Nada é alterado antes da aprovação.
2. **Executar** — toda subtarefa vai para o `task-executor` com o modelo e effort do seu
   nível (inclusive as complexas, em Fable); design vai para `/figcraft-design`, só se o
   usuário pedir. O orquestrador revisa tudo e sobe o nível quando uma tarefa falha.
3. **Versionar** — `commit-pr` organiza as alterações em commits atômicos
   (Conventional Commits), faz push e abre a PR.
4. **Revisar** — `pr-reviewer` analisa a PR e entrega achados com arquivo:linha,
   severidade e sugestão de correção.

## Estrutura

```
.
├── agents-md/
│   ├── go/AGENTS.md       # padrões para projetos Go
│   └── react-ts/AGENTS.md # padrões para projetos React + TypeScript
├── agents/
│   ├── commit-pr.md         # commits atômicos + push + PR
│   ├── figcraft-designer.md # designer UI/UX no FigCraft: desenvolve (sessão ou subagente)
│   ├── figcraft-reviewer.md # revisor de design no FigCraft (somente leitura)
│   ├── pr-reviewer.md       # revisão criteriosa de PRs (somente leitura)
│   └── task-executor.md     # executa uma subtarefa já planejada
└── skills/
    ├── figcraft-design/     # /figcraft-design (+ references/: build-standard, figcraft-mcp, quality-rubric)
    ├── new-project/SKILL.md # /new-project
    └── plan-task/SKILL.md   # /plan-task
```

## Skills

### `/new-project <ideia>`

Leva uma ideia do zero até a entrega que resolve o problema: documento, design, POC ou MVP.

| Fase | O que acontece |
|------|----------------|
| 1. Interrogatório | Perguntas uma por vez (sempre com uma resposta recomendada): produto, **formato da entrega** (MVP / documento spec-PRD / design e protótipo no FigCraft / POC; oferece design para validação antes do código, mas só gera se o usuário pedir), escopo, tipo de app, stack, qualidade, ambiente/deploy e repositório, pulando o que o formato não precisa. |
| 2. Especificação e plano | Especificação, definição de pronto e plano t00..tNN com nível e modelo·effort de cada tarefa. **Aprovar e construir** / **Ajustar** / **Só salvar a especificação**. |
| 3. Construção | `git init`, grava `docs/SPEC.md`. Documento: o orquestrador escreve. Design: chama `/figcraft-design` (modo enxuto + handoff). MVP/POC: tudo pelo `task-executor` (t01 = scaffold em Média-alta). |
| 4. Revisão | O orquestrador lê o diff e roda a verificação de cada tarefa; reprovada volta ao executor, subindo o nível se faltou capacidade. |
| 5. Entrega | Instala do zero, roda lint/testes, exercita a jornada principal e reporta a definição de pronto com honestidade. Oferece o commit inicial via `commit-pr`. |

### `/plan-task <descrição | issue | arquivo>`

Transforma uma task em entrega dentro de um projeto existente.

| Fase | O que acontece |
|------|----------------|
| 1. Contexto | Lê a issue/arquivo citado e explora o código para só perguntar o que o código não responde. |
| 2. Entrevista | Formato da entrega quando não é óbvio (código / documento técnico / design no FigCraft / POC; design só se o usuário pedir) e rodadas de 1–4 perguntas objetivas até não restar ambiguidade. |
| 3. Resumo e plano | Escopo, premissas, critérios de aceite e plano t01..tNN com nível e modelo·effort. **Aprovar e executar** / **Ajustar o plano** / **Só salvar o plano** (em `.claude/plans/<slug>.md`). |
| 4. Execução | Documento: o orquestrador escreve. Design: `/figcraft-design`. Código/POC: `task-executor` no nível de cada tarefa, em paralelo quando possível. |
| 5. Revisão | Obrigatória para toda tarefa: diff real, verificação, aderência ao pedido; reprovada volta ao executor com escalada de nível. |
| 6. Encerramento | Roda a suíte completa, reporta status por tarefa e oferece `commit-pr` + `pr-reviewer`. |

### `/figcraft-design <produto ou telas>`

Design e protótipo no [FigCraft](https://github.com/storytold/figcraft) via MCP, com
postura de designer de produto sênior (UI/UX).

| Fase | O que acontece |
|------|----------------|
| 0. Contexto | Testa o MCP (ao vivo no app ou headless), lê o projeto (`.figcraft` existentes, `docs/SPEC.md`, stack do front, tokens e marca) e pergunta a profundidade: **Enxuta** (ramos essenciais, resto vira premissa) ou **Completa**. |
| 1. Entrevista | Uma pergunta por vez, com resposta recomendada (e mockups ASCII nas escolhas visuais): produto, usuários, jornadas, conteúdo, direção visual, plataformas e navegação, acessibilidade, handoff, protótipo, arquivo. |
| 1b. Referências | Com o conceito claro: analisa as referências citadas ou, sem nenhuma, busca 6–8 variadas na internet (Dribbble, Behance, Pinterest, Mobbin, galerias). Gera `design/references/moodboard.html`, pergunta quais agradam e o que aproveitar de cada uma (cores, tipografia, layout, componentes), resolve conflitos e fecha o **Mapa de referências** que vira a direção visual: base, não cópia. |
| 2. Brief e plano | Brief de design + inventário de componentes, padrões e telas, com nível de cada tarefa. Na mesma pergunta: **Aprovar e desenhar** / **Ajustar** / **Só salvar o brief** e o **modo de revisão**: em todas as etapas / nos marcos / só no final / sem revisão. |
| 3. Construção | Mobile-first, em tarefas T01..Tnn seguindo `references/build-standard.md`: tokens (Light/Dark), componentes com variantes, página **Design Patterns**, telas mobile → desktop só com instâncias, protótipo. `figcraft-designer` desenvolve (modelo·effort pelo nível) e se autoavalia; `figcraft-reviewer` entra conforme o modo de revisão (reprovado volta com os achados). Pede ok nos marcos. |
| 4. Revisão final | `figcraft-reviewer` no arquivo inteiro (exceto no modo sem revisão, onde é oferecida). |
| 5. Entrega | Salva, completa o `design/BRIEF.md` e, se o design vai virar código, gera o handoff (`design/exports/*.png` + `design/HANDOFF.md`) para o `task-executor`. |

| Modo de revisão | Revisor roda | Indicado para |
|---|---|---|
| Em todas as etapas | após cada tarefa + final | design system/biblioteca, handoff crítico |
| Nos marcos | após o sistema, a 1ª tela e no final | padrão para > 5 telas |
| Só no final | uma vez, no arquivo inteiro | ≤ 5 telas, validação de ideia |
| Sem revisão | nunca (só autochecagem do designer) | rascunho, exploração, pitch |

### Escala de dificuldade (comum às três skills)

O orquestrador não desenvolve: classifica cada subtarefa e passa `model` + `effort` na
chamada do `task-executor` (ou do `figcraft-designer`). Em `/new-project` e `/plan-task` o
orquestrador é o **Fable**; em `/figcraft-design` é o **Opus** (o trabalho de orquestração
do design é bem especificado e o Opus entrega o mesmo resultado por cerca de metade do custo).

| Nível | Modelo · effort | Exemplos |
|-------|-----------------|----------|
| Baixa | sonnet · medium | config, boilerplate, docs, teste seguindo exemplo, CRUD idêntico a um existente |
| Média-baixa | sonnet · high | lógica simples bem especificada, tela a partir de design pronto, testes novos |
| Média | opus · medium | regra de negócio localizada com casos de borda, várias peças num módulo |
| Média-alta | opus · high | scaffold/base do projeto, fatia vertical de feature, refatoração com contrato definido |
| Alta | opus · xhigh | auth/segurança, pagamentos, concorrência, performance, migração de risco, bug de causa desconhecida |
| Alta-complexa | fable · medium | arquitetura delegada, contratos de API públicos, integração externa mal documentada, mudança transversal |
| Complexa | fable · high | problema aberto de ponta a ponta, algoritmo difícil, sistema distribuído, bug sistêmico |

Regras: o nível mais baixo que resolve com folga; falha por especificação → corrige o
cartão e reenvia no mesmo nível; falha por capacidade → sobe um nível; 3ª falha → leva ao
usuário.

## Agentes

| Agente | Modelo | Ferramentas | Função |
|--------|--------|-------------|--------|
| `commit-pr` | sonnet | Bash, Read | Separa o working tree em commits atômicos (Conventional Commits), cria branch se estiver na padrão, faz push e abre a PR via `gh`. |
| `figcraft-designer` | opus·high (o orquestrador troca pelo nível) | todas + MCP `figcraft`/`figcraft-app` | Designer UI/UX no FigCraft. Como sessão (`claude --agent figcraft-designer`) lê a skill e conduz tudo; como subagente lê só o `build-standard.md`, desenvolve a partir de um cartão de tarefa, se autoavalia pela rubrica e corrige os achados do revisor (ou devolve as perguntas pendentes). |
| `figcraft-reviewer` | opus (medium por tarefa, high em marco/final) | Read + MCP `figcraft`/`figcraft-app` só leitura | Revisa uma tarefa, um marco ou o arquivo inteiro contra o pedido, o brief e `quality-rubric.md`. Veredito APROVADO / COM RESSALVAS / REPROVADO e achados por severidade com id do nó e correção sugerida. Nunca edita. |
| `pr-reviewer` | opus | Bash, Read | Revisão somente-leitura de uma PR/branch: correção, casos de borda, segurança, contratos, performance, testes. Saída com veredito e achados por severidade. |
| `task-executor` | sonnet·medium (o orquestrador troca pelo nível: sonnet, opus ou fable) | Bash, Read, Edit, Write, Glob, Grep, WebFetch, WebSearch | Escreve todo o código: executa uma única subtarefa do cartão, toma só as decisões delegadas, roda a verificação e reporta (decisões, pontos de atenção, ajuste de nível sugerido). Para e pergunta se a especificação for ambígua. |

### Garantias importantes

- **`commit-pr`**: nunca adiciona assinatura/`Co-Authored-By` de IA; nunca usa
  `--force`, `--amend`, `--no-verify`, rebase ou reset; nunca commita segredos
  (`.env`, chaves, tokens); não corrige código se um hook falhar — apenas reporta.
- **`figcraft-reviewer`**: não edita, não seleciona, não salva; monta a lista de requisitos
  antes de olhar o design e recebe o cartão da tarefa, não as conclusões do designer.
- **`pr-reviewer`**: estritamente somente-leitura; não publica comentários, não aprova
  e não faz merge sem pedido explícito; todo achado é verificado no código real.
- **`task-executor`**: só altera os arquivos listados na tarefa; não commita, não
  instala dependências não pedidas e só toma as decisões que o cartão delegou.
- **Orquestrador (`/new-project`, `/plan-task`, `/figcraft-design`)**: não escreve código
  nem desenha; não gera design sem o usuário pedir; nada é alterado antes da aprovação.

## AGENTS.md por stack

Modelos de `AGENTS.md` com os padrões que os agentes devem seguir em qualquer projeto
da stack: tecnologias obrigatórias e proibidas, arquitetura, segurança, testes e
checklists de entrega.

| Arquivo | Stack | Destaques |
|---------|-------|-----------|
| `agents-md/go/AGENTS.md` | Go | Chi, PostgreSQL sem ORM, golang-migrate, UUID, swag, slog, OpenTelemetry, Prometheus, Testify/GoMock/testcontainers |
| `agents-md/react-ts/AGENTS.md` | React + TypeScript | Vite, React Router, TanStack Query, Zustand, React Hook Form + Zod, Tailwind + shadcn/ui, Vitest/MSW/Playwright |

Para usar, copie o arquivo da stack para a raiz do projeto:

```bash
cp agents-md/go/AGENTS.md /caminho/do/projeto/AGENTS.md
```

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
/figcraft-design app de agendamento para barbearias
```

```bash
claude --agent figcraft-designer "app de agendamento para barbearias"
```

Os agentes podem ser chamados diretamente pedindo em linguagem natural:

```text
commita essas alterações e abre a PR
revisa a PR #57
```

## Requisitos

- [Claude Code](https://claude.com/claude-code)
- `git` e [`gh`](https://cli.github.com/) autenticado (para `commit-pr` e `pr-reviewer`)
- FigCraft + `figcraft-cli` (para `/figcraft-design` e `figcraft-designer`), veja abaixo

### FigCraft MCP (macOS)

O app (`brew install --cask figcraft`) não traz o servidor MCP: ele vem no
`figcraft-cli-<versão>-macos-universal.zip` da [release](https://github.com/storytold/figcraft/releases)
de mesma versão do app.

```bash
V=0.17.0
mkdir -p ~/.local/opt ~/.local/bin && cd "$(mktemp -d)"
curl -LO https://github.com/storytold/figcraft/releases/download/v$V/figcraft-cli-$V-macos-universal.zip
curl -LO https://github.com/storytold/figcraft/releases/download/v$V/SHA256SUMS.txt
grep "figcraft-cli-$V-macos" SHA256SUMS.txt | shasum -a 256 -c -
unzip -q figcraft-cli-$V-macos-universal.zip
mv figcraft-cli-$V-macos-universal ~/.local/opt/figcraft-cli-$V
ln -sfn ~/.local/opt/figcraft-cli-$V/figcraft-cli ~/.local/bin/figcraft-cli

# headless (padrão), confinado a pastas
claude mcp add figcraft -s user -- ~/.local/bin/figcraft-cli mcp \
  --automation-read-root ~/Downloads --automation-read-root ~/Desktop \
  --automation-read-root ~/Pictures --automation-write-root ~/Documents
# ao vivo: controla o app aberto com `open -a FigCraft --args --control 7979`
claude mcp add figcraft-app -s user -- ~/.local/bin/figcraft-cli mcp --attach 7979 --timeout 300
```

`figcraft-app` aparece como "Failed to connect" enquanto o app não estiver aberto com
`--control 7979`. Depois de abrir o app, reconecte em `/mcp`.
