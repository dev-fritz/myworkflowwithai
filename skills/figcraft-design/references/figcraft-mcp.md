# FigCraft via MCP — guia de campo

Referência curta para desenhar no FigCraft (app open source em Rust, arquivos `.figcraft`) pelo
servidor MCP do `figcraft-cli`. Fonte: `docs/mcp.md` e `docs/cli.md` do repositório
`storytold/figcraft` (versão 0.17). Em caso de dúvida sobre parâmetros, **pergunte ao próprio
servidor**: `list_commands {filter: "variable."}` lista ids, rótulos e a documentação dos
parâmetros; o recurso `figcraft://command/<id>` mostra um comando.

## Servidores (escopo de usuário, valem em qualquer projeto)

| Servidor | Modo | Quando usar |
|---|---|---|
| `figcraft` | **Headless**: motor em memória, sem janela. O documento só existe no disco depois de `save_file`. | Padrão. Funciona sem o app aberto. |
| `figcraft-app` | **Ao vivo**: controla o app aberto pela porta 7979; o usuário vê cada mudança. | Quando o usuário quer acompanhar. O app precisa ter sido aberto com `--control 7979`. |

- Confinamento do headless: lê em `~/Downloads`, `~/Desktop`, `~/Pictures` e lê/escreve em
  `~/Documents`. Caminho fora disso → erro `automation path rejected` (salve dentro do projeto,
  em `~/Documents/...`). Com roots ativas, nada vai para a rede (sem publicar/compartilhar).
- Abrir o app em modo ao vivo (feche o FigCraft antes, `--args` só vale ao iniciar):
  `open -a FigCraft --args --control 7979` (opcional: `--automation-write-root ~/Documents`).
  O `figcraft-app` falha na abertura da sessão se o app não estiver escutando na 7979
  (`claude mcp list` mostra "Failed to connect"). Depois de abrir o app, reconecte em `/mcp`.
- Teste de conexão: `figcraft-app` → `inspect_ui {}`; headless → `inspect_document {depth: 1, outline: true}`.
- Não edite o mesmo arquivo nos dois modos ao mesmo tempo: se o app tem o arquivo aberto, use
  `figcraft-app`; o headless salvando por cima gera conflito ("the file changed on disk").
- Instalação / atualização do CLI no macOS: o app `.dmg` **não** traz o servidor MCP. Baixe
  `figcraft-cli-<versão>-macos-universal.zip` da release do GitHub (mesma versão do app), confira o
  SHA256 em `SHA256SUMS.txt`, extraia em `~/.local/opt/figcraft-cli-<versão>/` e aponte o link
  `~/.local/bin/figcraft-cli`. Registro: `claude mcp add figcraft -s user -- ~/.local/bin/figcraft-cli mcp ...`.

## Ferramentas

| Ferramenta | Uso |
|---|---|
| `create_node {type, x, y, width, height, parent, index, name, text, props}` | frame, rectangle, ellipse, polygon, star, line, arrow, text, vector, section, component, slice → `{id, node}` |
| `set_properties {ids, props}` | geometria, fills, strokes, effects, raio, auto layout, sizing, texto, `devStatus` |
| `run_command {command, params}` | qualquer um dos 400+ comandos (menus inteiros) |
| `batch {commands, label?, onError?}` | vários comandos = 1 undo step; `"as": "card"` + `{"$ref": "card"}` para usar ids de passos anteriores |
| `inspect_document {depth, fields?, outline?}` / `get_node {id, depth, fields?, outline?}` | ler a árvore — **sempre** com `depth` baixo e `outline: true` ou `fields` |
| `screenshot {ids?, scale?, maxSize?}` | PNG para conferir visualmente (use `maxSize` ~1200) |
| `export {ids, format, scale, path?}` | png/jpg/svg/pdf; com `path` grava em disco |
| `open_file {path}` / `save_file {path?}` | abrir `.figcraft`, `.fig`, `.sketch`, `.svg`, imagem; salvar `.figcraft` |
| `select`, `undo`, `redo`, `pointer`, `key`, `type_text`, `inspect_ui` | seleção, histórico e gestos |

Todas as ferramentas que agem num documento aceitam `document` (id de `doc.list` ou caminho de
arquivo aberto). No modo ao vivo, **passe sempre `document`**: o usuário pode trocar de aba no meio.

## Convenções que evitam retrabalho

- Coordenadas em px, y para baixo; `x`/`y` são **relativos ao frame pai** (à página sem pai).
- Filhos de auto layout não usam x/y: configure `layout` (`horizontal`, `vertical`, `grid`),
  `gap`, `padding`, `mainAlign` (`min|center|max|spaceBetween|spaceEvenly|spaceAround`),
  `crossAlign` (`min|center|max|baseline`), `sizingH`/`sizingV` (`fixed|hug|fill`),
  `minWidth`/`maxWidth`, `wrap` + `crossGap`. Na mesma chamada, `layout` é aplicado antes das
  chaves que o ajustam.
- Um filho não pode dar `fill` num eixo em que o pai faz `hug`: o pai vira `fixed` nesse eixo.
  Em linha `hug`, um filho `sizingV: "fill"` pode colapsar para altura 0: use `hug` ou altura fixa.
- Cores `"#RRGGBB"` / `"#RRGGBBAA"`; gradiente `{"type": "linearGradient", "stops": ["#7048E8", "#1098AD"]}`;
  `fill: "none"` remove a pintura.
- **Ligar a variável na criação**: `props: {fill: {"$var": "Theme/bg/surface"}}` (também em
  `stroke`). Outros campos (raio, gap, padding): `variable.bind`. Editar um valor ligado à mão
  desliga a variável; editar um fill com estilo desliga o estilo.
- Estilos: `style.create {type, name, value?}`, `style.apply {style, ids}`; nomes com `/` viram
  pastas (`Heading/H1`). Estilo de texto também carrega espaçamento de parágrafo e features.
- Variáveis: `variable.createCollection`, `variable.create`, `variable.addMode`,
  `variable.renameMode`, `variable.set` (valor por modo, ou `{alias}` para outra variável),
  `variable.bind`, `variable.update` (escopos, `codeSyntax: {WEB, iOS, ANDROID}`),
  `variable.setMode` (modo de um frame). Confira os parâmetros com `list_commands {filter: "variable."}`.
- Componentes: `component.create {ids}` (frame → componente), `component.combineVariants`
  (componentes → component set; nomes `Propriedade=Valor, Outra=Valor`), `component.addVariant`,
  `component.addProperty {type: boolean|text|instanceSwap|slot, ...}`, `component.bindProperty`,
  `component.makeSlot {id}`, `instance.create {component, x, y}`, `instance.place` (solta numa
  posição/slot de auto layout), `instance.setProperties {ids, properties}`, `instance.swap`,
  `instance.slotInsert`. Instância **não** aceita mover/redimensionar/adicionar filhos (fora de
  slots): edite o main ou use propriedades.
- Páginas e seções: `page.add`, `page.rename`, `page.select`, `page.list`;
  `create_node {type: "section"}` organiza blocos (seções só na página ou dentro de seções).
- Achar camadas sem mexer na seleção: `run_command node.find {root?, type?, name?, text?, component?, limit?}`.
- Duplicar com mapa de ids: `edit.duplicate {ids, map: true}` → `map/<idOriginal>` aponta a cópia.
- Texto: `fontFamily`, `fontStyle` (nome do estilo, ex. `"Semi Bold"`), `fontSize`, `lineHeight`
  (`"140%"` ou px), `letterSpacing`, `textCase`. **Antes de escolher pesos**, confira
  `text.fonts {family}`: o Inter embutido não tem Medium (use Semi Bold). `text.missingFonts`
  lista fontes ausentes. `text.fit {ids, maxLines}` ajusta título longo.
- Protótipo: `proto.interaction.add` (trigger + ações `navigate`, `back`, `openOverlay`,
  `scrollTo`, `setVariable`, `conditional`), `proto.flow.add`, `proto.settings {device: "iPhone 16" | "Desktop" | ...}`,
  `proto.preview` (painel ao vivo), `proto.present`.
- Handoff: `set_properties {props: {devStatus: "readyForDev"}}`, `annotation.add`,
  `run_command inspect.code` (CSS/HTML/React/Vue/SwiftUI/Compose/Flutter), `inspect.properties`.
- Modelos prontos para estudar a estrutura: `run_command template.list` →
  `file.newFromTemplate {template: "design-system" | "mobile-app" | "dashboard" | ...}` abre um
  documento novo feito com os mesmos comandos (variáveis Light/Dark, componentes, protótipo).
  Inspecionar um template é o jeito mais rápido de ver o formato exato dos parâmetros.
- Preencher cópias com dados: `data.fill {id, rows}` repete um frame por linha (cards, listas).

## Receitas

Card com auto layout, ligado ao tema, em uma chamada:

```json
{"label": "Card", "commands": [
  {"command": "node.create", "as": "card", "params": {"type": "frame", "name": "Card", "parent": 12,
    "props": {"layout": "vertical", "padding": 16, "gap": 8, "sizingH": "fill", "sizingV": "hug",
              "cornerRadius": 12, "fill": {"$var": "Theme/bg/surface"}}}},
  {"command": "node.create", "params": {"type": "text", "parent": {"$ref": "card"}, "text": "Título",
    "props": {"textStyle": "Heading/H3", "sizingH": "fill"}}},
  {"command": "component.create", "params": {"ids": [{"$ref": "card"}]}}
]}
```

Tela mobile: frame `393×852`, `layout: vertical`, `sizingV: hug` (cresce se rolar) ou fixo com
`overflow` de rolagem, `fill` = `Theme/bg/canvas`, grid `Grid/Mobile`; dentro, só instâncias
(TopBar, conteúdo, TabBar com `absolute: true` + constraints no rodapé, ou como último filho).
Tela desktop: frame `1440×1024`, grid `Grid/Desktop`, normalmente `layout: horizontal`
(Sidebar fixa + conteúdo `fill` com `maxWidth`).

## Armadilhas conhecidas

- `get_node`/`inspect_document` sem `depth`/`fields` devolvem respostas enormes.
- Bordas por lado: um `strokeTopWeight` sozinho desenha a borda inteira. Defina todos os lados
  (ou use `strokeSides`).
- Imagens: prefira JPEG. PNGs incham o `.figcraft`, e imagens substituídas não são removidas do arquivo.
- Headless: nada persiste sem `save_file`. Salve logo no início (com caminho) e ao fim de cada etapa.
- Comando que falha não altera o documento; `batch` padrão é tudo ou nada (`onError: "continue"`
  para seguir passo a passo).
- No modo ao vivo, uma resposta perdida pode ter sido aplicada: inspecione antes de repetir uma edição.
- Verifique sempre: números com `get_node`, pixels com `screenshot`.
