# Padrão técnico de construção (FigCraft)

Fonte única de **como** construir o arquivo. O `figcraft-designer` segue, o orquestrador usa
para escrever os critérios de aceite dos cartões e o `figcraft-reviewer` cobra (junto com a
rubrica `quality-rubric.md`).

Regras de execução: salvar (`save_file`) ao fim de cada tarefa; montar cada componente ou
tela com `batch` (`as` / `$ref`) para virar um único undo step; ler a árvore com
`outline: true` ou `fields` e `depth` baixo; passar `document` em toda chamada.

## 1. Estrutura

Páginas `00 Capa` (nome, versão, data, status), `01 Design Patterns` (seções `Fundamentos`,
`Componentes`, `Padrões`), `02 Mobile`, `03 Desktop`.

## 2. Fundamentos (tokens)

Documentados visualmente na seção `Fundamentos` (swatches com nome e hex, amostras de
tipografia, régua de espaçamentos):

- Coleção `Primitives`: rampas de cor 50–950 da primária, dos neutros e das semânticas
  (success, warning, danger, info).
- Coleção `Theme`, modos `Light` e (se pedido) `Dark`, com alias para os primitives:
  `bg/canvas`, `bg/surface`, `bg/elevated`, `text/primary`, `text/secondary`,
  `text/disabled`, `text/on-primary`, `border/default`, `border/strong`,
  `primary/default|hover|pressed`, `focus/ring`, `feedback/success|warning|danger|info`.
  Confira o contraste de cada par texto/fundo nos dois modos.
- Coleção `Dimensions` (números): `space/*` na escala de 4 (0, 4, 8, 12, 16, 20, 24, 32,
  40, 48, 64), `radius/sm|md|lg|full`, `size/touch` = 44.
- Estilos de texto: `Display`, `Heading/H1–H3`, `Body/L|M|S`, `Label/M|S`, `Caption`
  (escala mobile como base; `Desktop/…` só para títulos que crescem). Confira os pesos
  disponíveis com `text.fonts` antes.
- Estilos de efeito `Elevation/1–3` e de grid: `Grid/Mobile` (4 colunas, margem 16,
  gutter 16) e `Grid/Desktop` (12 colunas, gutter 24, conteúdo com no máximo 1200
  centralizado ou margem 80).
- Se houver handoff para web, preencha `codeSyntax.WEB` dos tokens com os nomes do código.

## 3. Componentes

Átomos → moléculas → organismos, na seção `Componentes`, cada um com rótulo e uma linha de uso:

- Auto layout em tudo; fills e strokes ligados a `Theme`; textos com estilo; `sizingH`
  Fill/Hug pensado para o componente esticar de 360 a 1440.
- Component sets com propriedades consistentes: `Variant` (primary, secondary, ghost,
  destructive), `Size` (sm, md, lg), `State` (default, hover, pressed, focus, disabled);
  booleanas para ícone opcional, texto para rótulos, instance swap para ícones, slots para
  conteúdo livre (Card, Dialog, BottomSheet).
- Conjunto típico, ajustado ao inventário aprovado: `Icon/*`, Button, IconButton, Input
  (text, password, search, textarea), Select, Checkbox, Radio, Switch, Chip, Badge, Avatar,
  Card, ListItem, Tabs, TopBar, TabBar (mobile), Sidebar (desktop), Dialog, BottomSheet,
  Toast, Banner, EmptyState, Skeleton, Spinner/ProgressBar.
- Nomes iguais aos do código (stack descoberta). Peças internas começam com `.` ou `_`
  (ficam fora da publicação como biblioteca).

## 4. Padrões

Na seção `Padrões`, montados só com instâncias e com nota "quando usar / faça / não faça":
formulário (label, ajuda, erro inline, ação primária), lista (cheia, vazia, carregando,
erro), navegação (mobile ↔ desktop lado a lado), feedback (toast, banner, diálogo de
confirmação), busca e filtros, cabeçalho de página.

## 5. Telas mobile (`02 Mobile`)

Uma seção por jornada; frames 393×852 (a altura cresce se a tela rolar), auto layout
vertical, `Grid/Mobile`, só instâncias e textos com estilo, conteúdo realista em pt-BR.
Estados das telas-chave como frames irmãos (`s03 Lista: vazio`).

Antes de montar cada tela, escreva para si em 3 linhas: ação primária, ordem de leitura (o
que o olho vê 1º, 2º, 3º) e como os grupos se separam (espaço interno vs. entre grupos).

## 6. Telas desktop (`03 Desktop`)

As mesmas telas em 1440×1024 com `Grid/Desktop`, recompostas para o espaço (sidebar,
colunas, master-detail, tabela), com os mesmos componentes (variantes de tamanho e de
navegação desktop). Conteúdo idêntico ao mobile.

## 7. Protótipo

Interações da jornada principal e das secundárias pedidas, um flow por jornada
(`proto.flow.add`), dispositivo `iPhone 16` no mobile e `Desktop` no desktop, overlays para
Dialog e BottomSheet, `back` para voltar.

## 8. Handoff

- `devStatus: readyForDev` nas telas e anotações nos comportamentos que a imagem não mostra
  (validações, regras, animações).
- **Handoff para código** (quando o design vai ser implementado por `/new-project` ou
  `/plan-task`): exporte PNG 2x de cada tela e estado para `design/exports/<id>-<tela>-<mobile|desktop>.png`
  e gere `design/HANDOFF.md` com:
  - tokens: `nome → valor Light / Dark → codeSyntax` (cor, espaçamento, raio, tipografia, sombra);
  - componentes: `nome → variantes e propriedades → onde é usado`;
  - telas: `id · nome → arquivos PNG → componentes usados → estados → comportamentos`;
  - flows do protótipo (de → para, gatilho).

  É o que o `task-executor` (sem acesso ao FigCraft) vai ler para implementar o front.
