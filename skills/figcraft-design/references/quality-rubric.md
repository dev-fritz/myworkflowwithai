# Rubrica de qualidade de design

Fonte única de critérios para os dois agentes:
- **`figcraft-designer`** usa como autochecagem antes de devolver qualquer tarefa.
- **`figcraft-reviewer`** usa para julgar e classificar os achados.

Cada item tem um jeito de verificar no FigCraft. "Olhar" = `screenshot`; "medir" = `get_node`
com `fields`, `inspect.properties` ou `node.find`.

## Severidades

| Severidade | Significa | Bloqueia aprovação? |
|---|---|---|
| **Crítico** | Não é o que foi pedido: tela, estado, fluxo ou conteúdo exigido ausente ou errado; algo quebrado (texto cortado, sobreposição, frame vazio, hotspot morto). | Sim |
| **Alto** | Viola um princípio inegociável: valor solto em vez de token, elemento repetido fora de componente, contraste abaixo de AA, alvo de toque < 44, direção visual do brief ignorada, mobile e desktop inconsistentes. | Sim |
| **Médio** | Problema de craft que um designer sênior não entregaria: hierarquia fraca, ritmo de espaçamento irregular, desalinhamento, tipografia sem escala. | Não, mas deve ser corrigido se houver nova rodada |
| **Baixo** | Polimento: microajustes ópticos, nome de camada, texto que pode melhorar. | Não |

## A. Aderência ao pedido (sempre primeiro)

1. Todas as telas, estados e variantes da tarefa existem, com os nomes combinados (`s03 Lista: vazio`).
2. Jornada coberta de ponta a ponta: cada passo do brief tem a tela correspondente e as ações levam ao próximo.
3. Conteúdo do domínio real, em pt-BR, com volume realista (nomes longos, 0/1/muitos itens, R$, dd/mm/aaaa). Nada de "Lorem ipsum", "Title", "Button".
4. Direção visual do brief respeitada: paleta, tipografia, raio, densidade, tema, personalidade (os 3 adjetivos são perceptíveis?). Com Mapa de referências: cada aspecto vem da referência combinada e o que o usuário pediu para evitar não aparece (desvio sem adaptação registrada = Alto); referência copiada literalmente (tela inteira, logo, ilustração ou imagem de terceiro no arquivo) = Alto.
5. Nada fora do escopo foi inventado (telas, features ou componentes não pedidos).

## B. Hierarquia e composição

1. **Uma ação primária por tela**, visualmente dominante; secundárias claramente subordinadas.
2. Teste do "olhar semicerrado": no screenshot pequeno (`maxSize: 400`) ainda se distingue título → conteúdo principal → ação.
3. Agrupamento por proximidade: espaço dentro do grupo < espaço entre grupos (ex.: 8 dentro, 24 entre).
4. Alinhamento: tudo em poucos eixos verticais; margens laterais iguais (16 no mobile); nada "quase alinhado".
5. Densidade coerente com o brief; respiro suficiente nas bordas; nenhuma área morta sem intenção.

## C. Espaçamento e grid

1. Todos os espaçamentos, paddings e gaps na escala de 4 (`space/*`), ligados a variável.
2. Ritmo vertical consistente entre seções equivalentes em telas diferentes.
3. Conteúdo desktop respeita `Grid/Desktop` (máx. 1200 de largura útil ou margem 80); linhas de texto com 45–75 caracteres.
4. Raios aninhados: raio interno = raio externo − padding (ou visualmente equivalente).

## D. Tipografia

1. Só estilos de texto do sistema; no máximo 2 famílias.
2. Escala com contraste real entre níveis (ex.: 28/20/16/14, não 16/15/14).
3. Peso e cor criam hierarquia junto com o tamanho (`text/secondary` para apoio).
4. Line-height confortável (corpo 140–150%, títulos 120–130%); nada truncado sem intenção (`text.fit` quando necessário).

## E. Cor e contraste

1. Toda cor vem de `Theme/*` (fill e stroke ligados); nenhum hex solto em tela ou componente.
2. Uma cor de destaque usada com parcimônia (≈ 60/30/10 entre neutro de fundo, superfícies e destaque).
3. Contraste AA: 4.5:1 para texto, 3:1 para texto grande e bordas/ícones de UI. Medir os pares críticos (texto secundário sobre surface, texto sobre primary, placeholder, disabled informativo).
4. Estado nunca comunicado só por cor (erro tem ícone ou texto; selecionado tem forma ou check).
5. Dark mode (se houver): trocar o modo e olhar; sem texto sumindo, sem sombra virando borrão.

## F. Sistema (tokens e componentes)

1. Telas compostas 100% de instâncias (exceto frames de layout); `node.find` não acha retângulos/textos soltos que repetem um padrão.
2. Componentes com auto layout, `sizingH` pensado para esticar de 360 a 1440, propriedades consistentes (`Variant`, `Size`, `State`).
3. Estados de componente: default, pressed, focus, disabled (+ hover no desktop, error/loading quando couber).
4. Overrides nas instâncias usam propriedades, não detach.
5. Todo componente novo está na página Design Patterns com rótulo e uma linha de uso.

## G. Interação, acessibilidade e protótipo

1. Alvos de toque ≥ 44×44 (medir a área clicável, não o ícone).
2. Foco visível nos interativos.
3. Estados de tela-chave: vazio, carregando (skeleton), erro, sucesso.
4. Protótipo: cada flow percorre a jornada sem hotspot morto; overlays fecham; `back` funciona.

## H. Polimento

1. Ícones do mesmo conjunto, mesmo peso e tamanho óptico, alinhados ao texto.
2. Nomes de camadas legíveis; peças internas com `.` ou `_`.
3. Mobile ↔ desktop: mesmo conteúdo, mesmos componentes, layout recomposto (não esticado).
4. Microcopy clara, no tom de voz do brief, verbos de ação nos botões ("Salvar alterações", não "OK").
