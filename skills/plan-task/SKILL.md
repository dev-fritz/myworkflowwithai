---
name: plan-task
description: >-
  Fluxo completo para transformar uma task/demanda em entrega: entrevista o usuário com
  perguntas até eliminar ambiguidades, apresenta um resumo com o plano quebrado em
  subtarefas pequenas numeradas (t01, t02, ...) e pede aprovação; após aprovado, executa
  as subtarefas delegando as simples a subagents com modelos menores/rápidos (haiku,
  sonnet) enquanto o agente principal faz as complexas e revisa tudo para garantir
  qualidade e aderência ao que foi pedido. Use quando o usuário invocar /plan-task,
  colar uma task/issue/ticket, ou pedir para "planejar", "quebrar em tarefas" ou
  "me entrevistar sobre" uma demanda. NÃO use para perguntas rápidas ou mudanças triviais
  de uma linha.
argument-hint: "<descrição da task, link da issue ou caminho de arquivo>"
model: claude-opus-5-5
---

# plan-task

Você é o **orquestrador**. Sua responsabilidade é a qualidade final: entender
exatamente o que foi pedido, planejar, delegar o que é simples e verificar tudo.

Task recebida: $ARGUMENTS
(Se vazio, peça ao usuário para descrever a task antes de continuar.)

## Fase 1 — Contexto (antes de perguntar)

- Se a task referenciar issue/ticket/arquivo, leia-o (`gh issue view`, `Read`).
- Explore o código relevante (somente leitura; use o subagent `Explore` para varreduras
  amplas) para entender arquitetura, convenções e onde a mudança vai acontecer.
- Objetivo: só perguntar ao usuário o que o código **não** responde.

## Fase 2 — Entrevista

- Use `AskUserQuestion` em rodadas de 1 a 4 perguntas objetivas, com opções concretas
  e a recomendada primeiro, marcada "(Recomendado)".
- Cubra, conforme relevante: objetivo/problema real, escopo (dentro e fora),
  critérios de aceite, comportamento em casos de borda e erros, restrições técnicas
  (libs, compatibilidade, performance, segurança), UX/API esperada, testes esperados,
  migração/rollout.
- Cada rodada deve se basear nas respostas anteriores. Não repita perguntas já
  respondidas nem pergunte o que tem padrão óbvio — assuma o padrão e registre como
  premissa.
- Pare quando não houver mais ambiguidade que mude o que será implementado.

## Fase 3 — Resumo e plano (aguarde aprovação)

Apresente no chat:

```
## Resumo do entendimento
<objetivo em 2–4 frases>

**Escopo:** ... | **Fora do escopo:** ...
**Premissas:** ...
**Critérios de aceite (globais):**
- [ ] ...

## Plano
| ID  | Tarefa | Depende de | Complexidade | Executor |
|-----|--------|------------|--------------|----------|
| t01 | ...    | —          | simples      | haiku    |
| t02 | ...    | t01        | média        | sonnet   |
| t03 | ...    | t01, t02   | complexa     | main     |

### t01 — <título>
- **Objetivo:** ...
- **Arquivos:** ...
- **Critérios de aceite:** ...
- **Verificação:** <comando/teste que prova que está pronto>
```

Regras do plano:
- Tarefas pequenas (idealmente ≤ ~150 linhas de diff), com entrega verificável e
  ordenadas por dependência. Numere t01, t02, ... com dois dígitos.
- Complexidade → executor:
  - **simples** (mecânico, padrão claro, poucos arquivos: renomear, boilerplate,
    ajustar config, testes seguindo exemplo existente, docs) → `task-executor` com
    `model: "haiku"`.
  - **média** (lógica localizada, bem especificada) → `task-executor` com
    `model: "sonnet"`.
  - **complexa** (arquitetura, design de API, segurança, concorrência, mudança
    transversal, algo ambíguo) → o agente principal executa.

Depois pergunte com `AskUserQuestion`: **Aprovar e executar** / **Ajustar o plano** /
**Só salvar o plano**. Não altere nenhum arquivo antes da aprovação. Em "Ajustar",
volte à fase 2/3. Em "Só salvar", grave o plano em `.claude/plans/<slug>.md` no projeto
e encerre.

## Fase 4 — Execução

- Siga a ordem de dependências. Tarefas independentes entre si podem ser delegadas em
  paralelo (várias chamadas `Agent` na mesma mensagem), desde que não mexam nos mesmos
  arquivos.
- Ao delegar, use `subagent_type: "task-executor"` e o `model` definido no plano. O
  prompt deve ser autocontido — o subagent NÃO vê esta conversa:
  - ID e objetivo da tarefa; contexto necessário (decisões da entrevista que afetam
    esta tarefa); arquivos a criar/alterar; padrões a seguir (aponte um arquivo de
    exemplo); critérios de aceite; comando de verificação; o que NÃO fazer
    (não commitar, não mexer fora dos arquivos listados).
- Informe o progresso ao usuário em uma linha por tarefa (ex.: "t02 ✅ (sonnet)").

## Fase 5 — Revisão pelo orquestrador (obrigatória para toda tarefa delegada)

Não confie no relatório do subagent. Para cada tarefa:
1. Leia o diff real (`git diff`) dos arquivos tocados.
2. Rode o comando de verificação e confira os critérios de aceite.
3. Confira aderência ao que o usuário pediu na entrevista e às convenções do repo.
4. Se falhar: re-delegue uma vez com feedback específico (subindo haiku → sonnet se o
   problema foi de capacidade); se falhar de novo, corrija você mesmo.

## Fase 6 — Encerramento

- Rode a suíte de testes/lint relevante do projeto uma vez com tudo integrado.
- Relatório final: tabela t01..tNN com status, executor e observações; critérios de
  aceite globais marcados; desvios do plano e o motivo; pendências.
- Pergunte se deseja commitar e abrir PR com o agente `commit-pr` e, depois, revisar
  com `pr-reviewer`. Não faça commit sem o usuário pedir.
