# AGENTS.md — Frontend React + TypeScript

Regras, padrões e convenções que agentes de código devem seguir ao criar ou modificar frontends em React + TypeScript.

Objetivo: código tipado de ponta a ponta, organizado por feature, acessível, performático, seguro, testável e desacoplado do backend por contratos claros.

> Se o projeto já tiver uma convenção diferente e consolidada, siga a do projeto e não misture estilos. Para trocar uma tecnologia obrigatória, é preciso uma decisão explícita registrada (README ou `docs/`).

---

## 1. Stack obrigatória

| Responsabilidade             | Tecnologia                                                        |
| ---------------------------- | ----------------------------------------------------------------- |
| Linguagem                    | TypeScript em modo `strict`                                       |
| UI                           | React (function components + Hooks)                              |
| Build / dev server           | Vite                                                              |
| Roteamento                   | React Router (framework mode com SSR quando houver sessão/SEO)    |
| Estado do servidor           | TanStack Query                                                    |
| Estado global do cliente     | Zustand                                                           |
| HTTP                         | Axios (instância única com interceptors)                          |
| Formulários                  | React Hook Form + `@hookform/resolvers/zod`                       |
| Validação / schemas          | Zod                                                               |
| Estilo                       | Tailwind CSS (tokens via variáveis CSS)                           |
| Componentes base             | shadcn/ui (sobre Radix UI) + `class-variance-authority`           |
| Ícones                       | lucide-react                                                      |
| Toasts                       | sonner                                                            |
| i18n                         | react-i18next (quando multilíngue)                                |
| Markdown                     | react-markdown + remark-gfm (sem HTML bruto)                      |
| Tipos da API                 | `openapi-typescript` gerado do Swagger/OpenAPI do backend         |
| Testes unitários/componentes | Vitest + React Testing Library + user-event + jest-dom            |
| Mock de API                  | MSW                                                               |
| E2E                          | Playwright                                                        |
| Lint                         | Oxlint                                                            |
| Formatação                   | Prettier + `prettier-plugin-tailwindcss`                          |
| Git hooks                    | Husky + lint-staged                                               |
| Monitoramento                | Sentry (com scrub de dados sensíveis), quando o projeto usar      |
| Gerenciador de pacotes       | **pnpm** (o lockfile existente manda; nunca misturar gerenciadores) |

### Proibido sem decisão explícita

- Redux, MobX, Recoil, Jotai (servidor = TanStack Query; cliente = Zustand).
- Create React App, Webpack, Next.js.
- Material UI, Ant Design, Chakra, Bootstrap.
- styled-components, Emotion ou CSS-in-JS em runtime.
- Formik, Yup (usar React Hook Form + Zod).
- Moment.js (usar `Intl`; date-fns se necessário).
- jQuery ou manipulação direta do DOM.
- `fetch`/`axios` soltos fora da camada de API.

### Versões

- Usar as versões do `package.json`; não subir major sem decisão.
- Lockfile sempre commitado e atualizado junto com `package.json`.

---

## 2. TypeScript

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

- **Proibido `any`.** Tipo desconhecido é `unknown`, estreitado por validação (Zod).
- Proibido `@ts-ignore`; `@ts-expect-error` só com comentário justificando.
- Não usar non-null assertion (`!`) para esconder `undefined`.
- Props de todo componente tipadas; retorno explícito em funções públicas de `api/` e `hooks/`.
- Union de literais em vez de `enum` (`type Priority = 'low' | 'medium' | 'high'`).
- Tudo que vem de fora (API, URL, `localStorage`, cookies, `postMessage`, env) é não confiável e validado na fronteira.

---

## 3. Estrutura (por feature)

```text
src/
├── root.tsx / app/          # composição: providers, layout raiz, error boundary
├── routes.ts                # declaração das rotas
├── routes/                  # módulos de rota (finos, lazy)
├── features/
│   └── <feature>/
│       ├── api/             # funções HTTP, query keys, hooks do TanStack Query
│       ├── components/
│       ├── hooks/
│       ├── schemas/         # schemas Zod
│       ├── locales/         # pt-BR.json, en.json (se i18n)
│       ├── types.ts
│       └── index.ts         # API pública da feature
├── components/
│   ├── ui/                  # shadcn/ui: genéricos, sem regra de negócio
│   ├── common/              # EmptyState, ErrorState, ConfirmDialog...
│   └── layout/
├── hooks/                   # hooks genéricos (useDebounce, useMediaQuery)
├── lib/                     # http, query-client, errors, utils (cn), monitoring
│   └── .server/             # código só de servidor (sessão, proxy, segredos)
├── stores/                  # Zustand (sessão de UI, preferências)
├── config/                  # env validado (env.server.ts / env.ts)
├── i18n/
├── types/                   # tipos globais + api.ts gerado
├── styles/index.css         # Tailwind + tokens de tema
└── test/                    # setup, MSW, renderWithProviders
tests/e2e/                   # Playwright
```

### Regras de importação

- Alias `@/` → `src/`. Nada de `../../../`.
- Uma feature só importa de outra pelo `index.ts` dela.
- `components/ui` e `lib/` nunca importam de `features/` ou `routes/`.
- Código de servidor (`*.server.ts`, pastas `.server/`) nunca é importado por código de cliente.

```text
routes → features → components / hooks / stores → lib / config / types
```

---

## 4. Componentes

- Function components; class component só para Error Boundary se necessário.
- Um componente por arquivo; componente em PascalCase, arquivo em kebab-case (`project-card.tsx` exporta `ProjectCard`).
- **Named exports**; `default` só onde a ferramenta exige (módulos de rota).
- Não definir componentes dentro de outros componentes.
- `key` estável (ID) em listas; nunca índice em listas reordenáveis/filtráveis.
- Componente com mais de ~200 linhas ou misturando dados + formulário + apresentação deve ser dividido.
- Rotas são finas: leem parâmetros, compõem componentes das features, definem layout.

| Tipo     | Convenção               | Exemplo                                       |
| -------- | ----------------------- | --------------------------------------------- |
| Hook     | `use-x.ts` → `useX`     | `use-projects.ts` → `useProjects`             |
| Store    | `x-store.ts`            | `ui-store.ts` → `useUIStore`                  |
| Schema   | `x.schema.ts`           | `project.schema.ts`                           |
| Teste    | ao lado do arquivo      | `project-card.test.tsx`                       |

---

## 5. Estado

| Estado                                             | Onde                                  |
| -------------------------------------------------- | ------------------------------------- |
| Dados da API                                       | TanStack Query                        |
| Formulário                                         | React Hook Form                       |
| UI local (aberto/fechado, aba)                     | `useState` / `useReducer`             |
| Compartilhado entre componentes próximos           | elevar estado / context local         |
| Global de cliente (preferências, UI)               | Zustand                               |
| Filtros, paginação, abas, busca compartilháveis    | URL (search params)                   |

- **Nunca copiar dados da API para Zustand ou `useState`.** O cache do Query é a fonte da verdade.
- Não usar `useEffect` para derivar estado (calcular no render) nem para buscar dados (usar Query).
- Zustand: stores pequenas por responsabilidade, consumidas **com seletor** (`useUIStore((s) => s.sidebarOpen)`).

---

## 6. Comunicação com a API

- **Uma única instância HTTP** em `src/lib/http.ts`: base URL, timeout, credenciais, renovação de sessão **single-flight** (uma renovação concorrente; as demais requisições aguardam) e normalização de erros.
- Componentes nunca chamam `http`/`axios`/`fetch`. Só a camada `features/<f>/api`.
- Todo erro vira um tipo único:

```ts
export type ApiError = {
  status: number;                    // 0 = erro de rede
  code: string;                      // código estável do backend
  message: string;                   // seguro para exibir
  fields?: Record<string, string>;   // erros por campo
};
```

- A UI decide pelo `code`/`status`, nunca pelo texto da mensagem.
- **Tipos da API gerados** do OpenAPI/Swagger do backend (`openapi-typescript`, convertendo Swagger 2.0 → OpenAPI 3 antes) em `src/types/api.ts`, via script (`pnpm gen:api`). Arquivo gerado não é editado à mão.
- Envelope `{ data: ... }` é desembrulhado na camada `api/`.

```ts
// features/projects/api/projects-api.ts
export async function getProject(id: string): Promise<Project> {
  const { data } = await http.get<{ data: Project }>(`/projects/${encodeURIComponent(id)}`);
  return data.data;
}

// features/projects/api/projects-queries.ts
export const projectKeys = {
  all: ['projects'] as const,
  lists: () => [...projectKeys.all, 'list'] as const,
  detail: (id: string) => [...projectKeys.all, 'detail', id] as const,
};

export function useProject(id: string) {
  return useQuery({ queryKey: projectKeys.detail(id), queryFn: () => getProject(id) });
}
```

### TanStack Query

- Leitura com `useQuery`/`useInfiniteQuery`; escrita com `useMutation`.
- Query keys centralizadas numa factory por feature; nunca strings soltas.
- Depois de mutation: invalidar/atualizar as queries afetadas, nunca recarregar a página.
- Otimista só quando a UX exige (drag and drop), sempre com rollback em `onError`.
- Defaults em `query-client.ts`: `staleTime` sensato, sem retry para 4xx.
- Polling de jobs com `refetchInterval` condicional que para em status final.
- No SSR: `QueryClient` **por requisição**, `prefetchQuery` no loader + `dehydrate` / `HydrationBoundary`. Nunca compartilhar cache entre usuários no servidor.
- No logout: `queryClient.clear()`.

---

## 7. Roteamento

- Rotas lazy (code splitting por rota) obrigatório.
- Layout autenticado redireciona para login preservando o destino (validado — ver §9).
- Rota `*` de não encontrado e Error Boundary por rota/layout.
- Params e search params validados (Zod) antes do uso.
- Navegação interna com `<Link>`/`<NavLink>`, nunca `<a href>`.
- Em framework mode, usar os tipos gerados (`./+types/<rota>`); `typecheck` roda o typegen.

---

## 8. Autenticação e autorização

### Sessão (padrão preferido: BFF)

- **Tokens nunca chegam ao JavaScript do navegador.** Ficam em cookie de sessão do servidor do front, criptografado, `HttpOnly`, `SameSite=Lax`, `Secure` em produção.
- Navegador → `/api/*` (mesma origem) → proxy no servidor do front → API com `Authorization: Bearer`.
- Renovação de token só no servidor, single-flight por refresh token, num único módulo.
- **Se for SPA pura** (sem servidor próprio): access token **em memória**, refresh via cookie `HttpOnly` do backend. Nunca token em `localStorage`/`sessionStorage`; nunca `localStorage.getItem('token')` espalhado.
- `401` após tentativa de renovação: encerrar sessão, limpar cache do Query, ir para login.
- Fluxos com token na URL (OAuth, verificação de e-mail, reset de senha, convite) consomem o token e **o removem da URL** (`replace`).

### Autorização

- **O frontend não é barreira de segurança.** Ele só adapta a UI às permissões que o backend informa.
- Checagem centralizada (`useCan('resource:action')`), nunca comparação de papéis espalhada.
- Tratar `403` e `404` com estados próprios, não tela quebrada.

---

## 9. Segurança

- **XSS**: nunca `dangerouslySetInnerHTML` com conteúdo não confiável; se inevitável, DOMPurify. Markdown com `react-markdown` **sem** `rehype-raw`.
- **URLs de usuário**: links/imagens vindos de dados só com protocolos permitidos (`http`, `https`, `mailto`); bloquear `javascript:`/`data:`.
- **Open redirect**: `?redirect=`/`?next=` aceitam só caminhos internos (começam com `/`, não `//`, sem esquema).
- Links externos `target="_blank"` com `rel="noopener noreferrer"`.
- **CSP** com nonce por requisição (scripts inline usam o nonce), `frame-ancestors 'none'`, `connect-src` restrito; headers `X-Content-Type-Options`, `Referrer-Policy`, HSTS em produção.
- **CSRF**: cookies `SameSite=Lax` + mutações só por métodos não-GET + checagem de `Origin` no proxy.
- Servidor do front: `trust proxy` correto, limite de tamanho de corpo, logs **sem query string**.
- Variáveis públicas (`VITE_*` ou entregues ao cliente) são públicas: **nunca segredos nelas**.
- Upload por URL pré-assinada vai direto ao storage **sem** header `Authorization`.
- Não logar tokens, senhas, cookies, dados pessoais ou conteúdo de formulários (console, Sentry, analytics). Sentry com scrub.
- Validação no cliente é UX; o backend sempre valida.
- Dependência nova: avaliar manutenção, tamanho, licença e vulnerabilidades (`pnpm audit`).
- Cookies não-HttpOnly só para preferências (tema, idioma), nunca para identidade.

---

## 10. Variáveis de ambiente

- **Com servidor próprio (SSR/BFF):** lidas em runtime só no servidor, validadas com Zod em `src/config/env.server.ts`; o cliente recebe o que precisa pelo loader da raiz. Produção recusa valores de desenvolvimento.
- **SPA pura:** prefixo `VITE_`, validadas em `src/config/env.ts`, tipadas em `vite-env.d.ts`.
- Nunca acessar `process.env`/`import.meta.env` fora do módulo de config.
- `.env`/`.env.local` fora do git; `.env.example` completo com valores seguros.

---

## 11. Formulários

- React Hook Form + Zod; o schema é a fonte da verdade e o tipo vem de `z.infer`.
- Erros de validação do backend (`fields`) aplicados com `setError`.
- Botão de envio desabilitado enquanto a mutation está pendente (sem envio duplicado).
- Todo campo com `<label>` e erro acessível (`aria-invalid`, `aria-describedby`).

---

## 12. Estilo (Tailwind + shadcn/ui)

- Classes utilitárias + `cn()` (`clsx` + `tailwind-merge`).
- Cores, raios, espaçamentos e fontes vêm dos **tokens do tema** (variáveis CSS); nada de `bg-[#3a7bd5]` espalhado.
- Tema claro e escuro via tokens; preferência aplicada no SSR sem flash.
- Componentes shadcn adicionados pela CLI em `components/ui`, ajustáveis mas genéricos; variações com `cva`.
- Sem `style={{}}` exceto valores realmente dinâmicos; sem CSS por componente sem justificativa.
- **Mobile-first e responsivo.**

### Mobile

- Área de toque mínima de 44×44px em `pointer-coarse` (pode ser por pseudo-elemento `::after`, sem inflar o visual).
- Ações que aparecem no hover ficam sempre visíveis no toque (`pointer-fine:opacity-0 pointer-fine:group-hover:opacity-100 pointer-fine:focus-visible:opacity-100`).
- **Sem overflow horizontal**: grids com coluna base (`grid-cols-1 sm:grid-cols-2`), `minmax(0,1fr)` em templates; barras com muitas ações viram ícones/menu.
- Inputs com 16px no toque (evita zoom do iOS); respeitar `env(safe-area-inset-*)` em barras fixas.
- Popover no desktop, bottom sheet no mobile.

---

## 13. Acessibilidade

- HTML semântico; nunca `div` com `onClick` no lugar de `button`/`a`.
- Tudo operável por teclado, com foco visível.
- `alt` em imagens informativas, `alt=""` nas decorativas; `aria-label` em botões só-ícone.
- Diálogos, menus e popovers com primitives Radix (via shadcn).
- Contraste WCAG AA; mensagens assíncronas anunciadas (`aria-live`/toasts acessíveis).
- Operações destrutivas pedem confirmação.

---

## 14. Estados de interface

Toda tela/componente com dados trata: **carregando** (skeleton), **vazio** (explicação + ação), **erro** (mensagem + tentar novamente), **sem permissão** e **sucesso**. Mutações dão feedback (toast) e tratam falha.

---

## 15. Performance

- Code splitting por rota; componentes pesados raros (editores, gráficos) com `lazy`.
- `memo`/`useMemo`/`useCallback` só com problema medido ou quando a referência estável é necessária.
- Listas grandes virtualizadas (TanStack Virtual) ou paginadas **no backend**.
- Debounce em buscas; imagens com dimensões e `loading="lazy"`.
- Checar impacto no bundle antes de adicionar dependência.

---

## 16. i18n, datas e números

- Se multilíngue: react-i18next, **nenhum texto de interface fixo** em componentes; os idiomas têm as mesmas chaves (teste garante).
- Formatação com `Intl` (`DateTimeFormat`, `NumberFormat`, `RelativeTimeFormat`).
- Datas só-dia (`YYYY-MM-DD`) não passam por conversão de fuso.
- Dinheiro chega em unidades inteiras (centavos) e é formatado no cliente.

---

## 17. Testes

| Tipo                    | Ferramenta                                  |
| ----------------------- | ------------------------------------------- |
| Funções, hooks, stores  | Vitest (`renderHook`)                       |
| Componentes             | Vitest + Testing Library + user-event       |
| API                     | MSW (mock na fronteira de rede)             |
| Fluxos críticos         | Playwright (desktop + viewport mobile)      |

- Testar comportamento, não implementação.
- Consultas por papel/label (`getByRole`, `getByLabelText`); `data-testid` só sem alternativa acessível.
- `userEvent`, não `fireEvent`.
- **Mockar a rede com MSW**, nunca módulos internos (`http`, hooks de query).
- Isolamento: `QueryClient` novo por teste (retry desligado), stores e handlers MSW resetados; helper `renderWithProviders`.
- Determinístico: sem relógio real, rede real ou dependência de ordem.
- Componentes de feature cobrem: sucesso, carregando, vazio, erro, validação do backend e permissões.
- Código de segurança (redirect seguro, sanitização, sessão, proxy, CSP) tem testes unitários próprios.
- E2E mobile falha se houver overflow horizontal (`scrollWidth > clientWidth`).

---

## 18. Lint, formatação e scripts

- Oxlint (com regras de hooks e jsx-a11y), Prettier com plugin do Tailwind, `tsc` no typecheck.
- Husky + lint-staged rodam lint e format nos arquivos alterados.
- Não desabilitar regra para contornar problema; se inevitável, na linha e com justificativa.
- Sem `console.log` no código commitado.

```json
{
  "scripts": {
    "dev": "...",
    "build": "...",
    "typecheck": "tsc (+ typegen do router, se houver)",
    "lint": "oxlint",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "test": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "gen:api": "gera src/types/api.ts a partir do OpenAPI"
  }
}
```

---

## 19. Docker e deploy

- `Dockerfile` multi-stage: instala + builda; imagem final só com o necessário (dependências de produção do servidor Node, ou estáticos num servidor web), **usuário não-root**, sem código-fonte nem ferramentas de dev.
- SPA estática: fallback para `index.html`, assets com hash com cache longo, `index.html` sem cache.
- Com SSR/BFF: estado em memória (ex.: single-flight de refresh) implica uma instância, ou lock distribuído para escalar.
- README documenta: versão do Node/pnpm, como rodar, variáveis, scripts, geração de tipos, testes, build e como subir o backend local.

---

## 20. Fluxo para uma nova feature

1. Tipos (gerados da API) → 2. funções de API → 3. query keys + hooks → 4. schemas Zod → 5. componentes → 6. rota → 7. estados (loading/vazio/erro/permissão) → 8. textos i18n → 9. testes → 10. docs.

### Checklist antes de concluir

```text
[ ] Tipos alinhados ao contrato (gen:api rodado se a API mudou)
[ ] HTTP só na camada api/; query keys centralizadas; invalidação após mutations
[ ] React Hook Form + Zod; erros do backend nos campos
[ ] Loading, vazio, erro e sem permissão tratados
[ ] UI respeita permissões vindas do backend
[ ] Sem XSS (sem HTML bruto), sem open redirect, sem segredos no cliente, sem token no storage
[ ] Acessível (teclado, labels, foco, contraste) e responsivo (sem overflow no mobile)
[ ] Textos i18n nos idiomas suportados
[ ] Rota lazy; sem any, sem console.log
[ ] lint, typecheck, format:check, test, build (e e2e se fluxo crítico) passando
```

```bash
pnpm lint && pnpm typecheck && pnpm format:check && pnpm test && pnpm build
pnpm test:e2e   # fluxos críticos
```

---

## 21. Regras para agentes

- Antes de escrever código: ler a estrutura, procurar componentes/hooks semelhantes e **seguir o padrão existente**.
- Reutilizar `components/ui`, `components/common` e helpers de `lib/` antes de criar novos.
- Consultar o contrato da API (OpenAPI/`docs/`) em vez de supor formatos.
- Mudança mínima para o pedido; sem refatoração grande não solicitada.
- Não adicionar dependência sem necessidade clara.
- Nunca desativar teste/lint para "passar", nem commitar `.env`.
- Com várias soluções, escolher a mais simples, idiomática, fácil de testar e de menor impacto.

### O que NÃO fazer

- `any`, `@ts-ignore`, `!` para esconder `undefined`.
- `axios`/`fetch` em componente; buscar dados com `useEffect`.
- Copiar dados da API para store/estado local; query keys soltas.
- Token em `localStorage`; segredo em variável pública; confiar no front para autorização.
- `dangerouslySetInnerHTML` sem sanitizar; `div` clicável; índice como `key`.
- Componente dentro de componente; páginas gigantes; importar internos de outra feature.
- Bibliotecas alternativas de UI/estado/estilo/formulário sem decisão.
- Cores e espaçamentos fora dos tokens; texto de interface fixo em app multilíngue.
- Ignorar estados de carregamento e erro.

---

**Princípio geral:** simplicidade → clareza → tipagem → segurança → testabilidade → acessibilidade → performance. A solução mais simples que atende corretamente vence; toda abstração precisa resolver um problema concreto.
