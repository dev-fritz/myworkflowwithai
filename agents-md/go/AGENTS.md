# AGENTS.md — Backend Go

Regras, padrões e convenções que agentes de código devem seguir ao criar ou modificar backends em Go.

Objetivo: código idiomático, simples, modular, testável, observável, seguro e pronto para produção, sem complexidade antecipada.

> Se o projeto já tiver uma convenção diferente e consolidada, siga a do projeto e não misture estilos. Para trocar uma tecnologia obrigatória, é preciso uma decisão explícita registrada (README ou `docs/`).

---

## 1. Stack obrigatória

| Responsabilidade      | Tecnologia                                                           |
| --------------------- | -------------------------------------------------------------------- |
| Linguagem             | Go (a versão do `go.mod`)                                            |
| HTTP / roteamento     | `github.com/go-chi/chi/v5` (+ `github.com/go-chi/cors`)              |
| Banco de dados        | PostgreSQL via `database/sql` + `github.com/jackc/pgx/v5/stdlib`     |
| Acesso a dados        | **SQL escrito à mão. Sem ORM** (nada de GORM, ent, bun, sqlboiler)   |
| Migrations            | `github.com/golang-migrate/migrate/v4`                               |
| IDs                   | `github.com/google/uuid` (UUID em toda PK)                           |
| Validação de entrada  | `github.com/go-playground/validator/v10`                             |
| Autenticação          | `github.com/golang-jwt/jwt/v5`                                       |
| Senhas / cripto       | `golang.org/x/crypto` (bcrypt ou argon2id)                           |
| Configuração local    | `github.com/joho/godotenv` (só para carregar `.env` em dev)          |
| Documentação da API   | `github.com/swaggo/swag` + `github.com/swaggo/http-swagger/v2`       |
| Logs                  | `log/slog` (biblioteca padrão), saída JSON em produção               |
| Tracing               | OpenTelemetry (`otelhttp`, `otelsql`, exporter OTLP)                 |
| Métricas              | `github.com/prometheus/client_golang`                                |
| Erros em produção     | Sentry (`github.com/getsentry/sentry-go`), quando o projeto usar     |
| Storage de arquivos   | S3-compatível (`github.com/minio/minio-go/v7`) atrás de interface    |
| Filas                 | RabbitMQ (`github.com/rabbitmq/amqp091-go`), só quando necessário    |
| Testes                | `testing` + Testify; mocks com `go.uber.org/mock` (GoMock)           |
| Testes de integração  | `testcontainers-go` (PostgreSQL real)                                |
| Lint                  | `golangci-lint` (com `gosec`, `errorlint`, `bodyclose`, `noctx`...)  |

### Proibido sem decisão explícita

- Gin, Echo, Fiber ou outro framework HTTP.
- Qualquer ORM ou query builder que esconda o SQL.
- `AutoMigrate` ou migrations geradas por código.
- Loggers de terceiros (zap, zerolog, logrus) quando `slog` resolve.
- Bibliotecas de DI/containers mágicos (wire, fx, dig). DI é manual, por construtor.
- Dependência nova quando a biblioteca padrão ou uma dependência existente já resolve.

---

## 2. Estrutura do projeto

```text
.
├── cmd/
│   ├── api/main.go          # servidor HTTP
│   ├── migrate/main.go      # up | down | version
│   ├── seed/main.go         # dados de desenvolvimento
│   └── worker/main.go       # consumers/jobs (se houver)
├── internal/
│   ├── bootstrap/           # container.go (DI) e routes.go (rotas)
│   ├── config/              # carga e validação das variáveis de ambiente
│   ├── database/            # conexão, transações, migrate
│   ├── apperrors/           # erros semânticos da aplicação
│   ├── dtos/                # contratos de request/response
│   ├── handlers/            # camada HTTP
│   ├── middlewares/
│   ├── models/              # entidades
│   ├── services/            # regras de negócio
│   ├── repositories/        # SQL
│   └── workers/             # consumers e tarefas agendadas
├── pkg/                     # infraestrutura reutilizável sem regra de domínio (storage, queue, telemetry, ai...)
├── migrations/              # 000001_<nome>.up.sql / .down.sql
├── docs/                    # Swagger gerado + documentação
├── tests/                   # integração, fixtures, mocks gerados
├── scripts/
├── .env.example
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── .golangci.yml
├── README.md
└── AGENTS.md
```

- `main.go` é pequeno: carrega config, monta o container, inicia o servidor e faz graceful shutdown.
- `pkg/` não é depósito genérico: o que só faz sentido para a aplicação fica em `internal/`.
- Nomes de pacote curtos, minúsculos, sem `_` (`handlers`, `services`, `repositories`).

---

## 3. Camadas e responsabilidades

```text
HTTP → Middlewares → Handler → Service → Repository → PostgreSQL
                                  └──→ interfaces externas (storage, fila, e-mail, IA, APIs)
```

| Camada       | Faz                                                                                   | Não faz                                   |
| ------------ | ------------------------------------------------------------------------------------- | ----------------------------------------- |
| Handler      | decodifica/valida o formato, chama o service, mapeia erro → status, escreve resposta  | regra de negócio, SQL                     |
| Service      | regras de negócio, autorização, transações, orquestra repositories e integrações      | SQL, `http.Request`/`ResponseWriter`      |
| Repository   | SQL, scan, persistência                                                               | regra de negócio, enviar e-mail, gerar JWT |
| Middleware   | auth, CORS, rate limit, logs, métricas, recover, headers de segurança                 | regra de negócio                          |

### Regras

- **Dependency injection explícita por construtor**, montada em `internal/bootstrap/container.go`. Sem variáveis globais (`var DB *sql.DB`) nem singletons.
- **Interfaces pequenas, definidas pelo consumidor**, só quando trazem desacoplamento ou testabilidade (repositories, storage, fila, e-mail, IA, gateways de pagamento, relógio).
- Services dependem de interfaces, nunca de SDKs concretos (`ObjectStorage`, não `minio.Client`; `ChatClient`, não o SDK do provider).
- DTOs separados dos models: o cliente nunca envia `id`, `created_at`, `role`, `is_admin` ou qualquer campo controlado pelo backend (evita mass assignment).
- Não transformar Go em Java: sem factories, hierarquias ou abstrações sem problema concreto.

---

## 4. Banco de dados e SQL

- **Sem ORM.** SQL explícito nos repositories, com `database/sql` + driver `pgx`.
- **Sempre queries parametrizadas** (`$1`, `$2`). Nunca concatenar/`fmt.Sprintf` valores na query.
  - Partes dinâmicas que não podem ser parâmetros (colunas de ordenação, direção) vêm de **allowlist** no código, nunca da entrada do usuário diretamente.
- Nada de `SELECT *`: listar as colunas.
- Sempre `QueryContext`/`ExecContext`/`QueryRowContext` com o `ctx` da requisição.
- Fechar `rows` e checar `rows.Err()`.
- **Evitar N+1**: JOIN, `WHERE id = ANY($1)` ou batch. Operações em massa nunca fazem uma query por item.
- Listagens sempre paginadas (`limit` com teto máximo no servidor + `offset` ou cursor).
- Transações quando várias escritas precisam ser atômicas; fronteira da transação no service (helper do tipo `tx.WithinTx(ctx, func(ctx) error)`), propagada pelo contexto.
- Concorrência em contadores/limites: usar constraints, `SELECT ... FOR UPDATE` ou advisory lock, não "ler, checar e gravar" sem trava.
- Constraints no banco (`NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`) são a última linha de defesa; o service traduz violações (ex.: unique → `409 Conflict`).
- Timestamps em `TIMESTAMPTZ`, armazenados em UTC.

```go
const q = `
    SELECT id, name, email, created_at
    FROM users
    WHERE id = $1`

err := r.db.QueryRowContext(ctx, q, id).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
```

### UUID

- Toda PK é `UUID` no banco e `uuid.UUID` no Go. Nunca `string` para IDs internos.
- Conversão para string só na fronteira (JSON, URL). Parâmetro de URL inválido → `400`.

---

## 5. Migrations (golang-migrate)

- Arquivos em `migrations/` no formato `000001_create_users.up.sql` + `000001_create_users.down.sql`. Toda migration tem `up` **e** `down`.
- Executadas por um entrypoint separado (`go run ./cmd/migrate up|down|version` ou `make migrate`). **A API nunca roda migrations no startup.**
- **Nunca editar uma migration já aplicada** em ambiente compartilhado: criar uma nova.
- Mudanças de schema devem ser compatíveis com a versão anterior da aplicação rodando (expand → migrate → contract): adicionar coluna nullable/com default, fazer backfill, só depois remover o antigo.
- Índices em tabelas grandes: `CREATE INDEX CONCURRENTLY` (em migration própria, sem transação).
- Seeds (`cmd/seed`) são dados de desenvolvimento; nunca misturar com migrations.

---

## 6. Configuração e variáveis de ambiente

- Toda configuração vem de variáveis de ambiente, carregadas **uma vez** em `internal/config` e injetadas nas dependências.
- `godotenv` só carrega `.env` em desenvolvimento.
- **Não usar `os.Getenv()` fora de `internal/config`.**
- Validar na inicialização: variável obrigatória ausente ou inválida derruba o processo com mensagem clara.
- Em produção, **recusar valores de desenvolvimento** (segredo padrão, `sslmode=disable`, CORS `*`, etc.).
- `.env` nunca é commitado. `.env.example` lista todas as variáveis com valores de exemplo seguros e é atualizado junto com o código.

---

## 7. HTTP com Chi

- Rotas registradas em `internal/bootstrap/routes.go`, versionadas em `/api/v1`. Mudança incompatível dentro de uma versão exige decisão explícita.
- Grupos de rotas com middlewares por grupo (`r.Group`, `r.With(authMiddleware)`).
- Ordem típica de middlewares: request ID → recover → real IP (confiável) → logging/metrics/tracing → security headers → CORS → body limit → rate limit → auth.
- Métricas e spans usam o **template da rota** (`/users/{id}`), nunca a URL crua.
- Expor `/healthz` (liveness) e `/readyz` (readiness, checa dependências críticas).

### Handlers

```go
func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req dtos.CreateUserRequest
    if err := decodeJSON(w, r, &req); err != nil { // MaxBytesReader + DisallowUnknownFields + validator
        respondError(w, r, err)
        return
    }

    user, err := h.service.Create(r.Context(), req)
    if err != nil {
        respondError(w, r, err)
        return
    }

    respondJSON(w, http.StatusCreated, dtos.NewUserResponse(user))
}
```

- Um helper único de decode (limite de corpo, JSON estrito, validação) e um de resposta. Não reinventar por endpoint.

### Formato de resposta

```json
{ "data": { "id": "..." } }
```

```json
{ "error": { "code": "USER_NOT_FOUND", "message": "user not found", "fields": { "email": "invalid" } } }
```

- `code` é estável e em `UPPER_SNAKE_CASE`; o frontend decide pelo código, não pela mensagem.
- Mesmo formato em todos os endpoints.

### Status HTTP

`200` OK · `201` Created · `204` No Content · `400` entrada inválida · `401` não autenticado · `403` sem permissão · `404` não encontrado · `409` conflito · `422` regra de negócio violada · `429` rate limit · `500` interno · `502/503` dependência externa. Nunca `200` com erro no corpo.

---

## 8. Erros

- Tratar todo erro. Nunca `_ = fn()` para erro que importa.
- Envolver com contexto: `fmt.Errorf("create user: %w", err)`; comparar com `errors.Is`/`errors.As`.
- Erros semânticos em `internal/apperrors` (validation, not found, conflict, unauthorized, forbidden, unprocessable, external) com código estável; o handler converte para status.
- **Nunca devolver ao cliente** stack trace, mensagem do Postgres, SQL, caminho de arquivo ou erro de SDK. `500` sai com mensagem genérica; o detalhe vai para log/Sentry.
- `context.Canceled` de cliente que desconectou não é erro de servidor (logar em debug, não reportar ao Sentry).
- `panic` só para bug de programação; um middleware de recover converte em `500` e reporta.

---

## 9. Context, concorrência e ciclo de vida

- Toda função que faz I/O recebe `ctx context.Context` como primeiro parâmetro e o propaga até o banco/rede.
- Não usar `context.Background()` no meio do fluxo para "escapar" do contexto da requisição. Trabalho que precisa sobreviver à requisição vai para fila/worker ou usa `context.WithoutCancel` conscientemente.
- Toda chamada externa tem timeout (`context.WithTimeout`) proporcional à operação.
- `http.Server` sempre com `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout` e `IdleTimeout` (streams/SSE controlam deadline por escrita).
- `http.Client` próprio com `Timeout`; nunca `http.DefaultClient` para chamadas externas.
- Goroutines sempre com dono e forma de encerrar (`ctx`, `errgroup`, `WaitGroup`). Nada de `go func(){ for {...} }()` sem cancelamento.
- Estado compartilhado protegido (`sync.Mutex`, `atomic`, channels). Rodar testes com `-race`.
- **Graceful shutdown** em `SIGINT`/`SIGTERM`: parar de aceitar requisições, drenar as em andamento, encerrar workers/consumers, fazer flush de tracing/Sentry, fechar conexões.

---

## 10. Segurança

### Segredos

- Nenhum segredo no código, em testes commitados, em imagens Docker ou em logs.
- Segredos só por variável de ambiente (ou secret manager). `.env` fora do git.
- Comparação de segredos/tokens com `subtle.ConstantTimeCompare` (ou `hmac.Equal`).
- Dado sensível de terceiros armazenado no banco (tokens OAuth, chaves de API, secrets de webhook) é **criptografado em repouso** (AES-256-GCM com chave vinda do ambiente).
- Tokens opacos (reset de senha, verificação de e-mail, convites, API tokens/PATs, refresh tokens) são gerados com `crypto/rand`, **armazenados só como hash** (SHA-256), têm expiração e são de uso único quando aplicável.

### Senhas

- Hash com **bcrypt (cost ≥ 12)** ou **argon2id**. Nunca MD5/SHA-1/SHA-256 puro.
- Validar tamanho máximo (bcrypt aceita até 72 bytes) e mínimo.
- Respostas de login/recuperação não revelam se o e-mail existe (evitar enumeração).

### Autenticação (JWT)

- Validar assinatura, **algoritmo esperado fixo** (rejeitar `none` e algoritmos inesperados), `exp`, `iat`/`nbf`, `iss`/`aud` quando usados.
- Access token curto; refresh token longo, rotacionado a cada uso, revogável e armazenado como hash.
- Nada sensível nas claims (são só base64).
- O middleware coloca a identidade no `context.Context`; handlers/services leem dali, nunca do corpo da requisição.
- Trocar senha/sair de todos os dispositivos revoga as sessões.

### Autorização

- Autenticação ≠ autorização. **Toda operação protegida é checada no backend**, no service (ou policy), recurso por recurso — inclusive em listas, buscas, exports e endpoints auxiliares.
- Verificar posse/escopo do recurso (evita IDOR): o ID na URL não basta.
- Recurso que o usuário não pode ver responde `404` (não confirma existência) quando apropriado.
- Permissões centralizadas (ex.: `can(user, "resource:action")`), não comparação de strings de papel espalhada.

### Entrada e saída

- Validar toda entrada externa (body, query, path, headers, mensagens de fila, webhooks recebidos).
- `http.MaxBytesReader` em todo corpo; limites também em multipart e uploads.
- `json.Decoder.DisallowUnknownFields()` nos DTOs de entrada.
- Webhooks recebidos: verificar assinatura HMAC/token com comparação em tempo constante e tolerância de timestamp (anti-replay).
- HTML gerado a partir de dados do usuário (e-mails, Markdown) via `html/template` ou sanitizador; nunca concatenar HTML.

### Rede e HTTP

- **CORS com allowlist explícita** de origens; nunca `*` com credenciais.
- Headers de segurança: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Cross-Origin-Resource-Policy` e `Strict-Transport-Security` em produção.
- **Rate limiting** em login, cadastro, recuperação de senha, envio de e-mail e endpoints caros (por IP e/ou por usuário). Responder `429` com `Retry-After`.
- IP do cliente só de `X-Forwarded-For` quando vier de proxy confiável configurado.
- **SSRF**: requisição de saída para URL informada pelo usuário (webhooks, importação, preview) passa por um cliente dedicado que valida o destino, bloqueia IPs privados/loopback/link-local/metadata (inclusive após resolver DNS, a cada conexão), sem seguir redirects e com timeout.

### Arquivos

- Upload direto para o storage por **URL pré-assinada** curta, assinando `Content-Type` e `Content-Length`; confirmar o objeto (`Stat`) antes de marcar como pronto.
- Nunca confiar no `Content-Type`/extensão do cliente: allowlist de tipos servidos inline; o resto sai como `application/octet-stream` + `Content-Disposition: attachment`. Bloquear executáveis/scripts.
- Download via URL pré-assinada de curta duração, depois de checar permissão.
- Nomes de arquivo do usuário nunca viram caminho no disco/chave do storage (usar UUID).

### Logs e privacidade

- Nunca logar: senhas, tokens, `Authorization`, cookies, chaves de API, URLs com credenciais (ex.: webhooks do Slack/Discord), corpos de requisição com dados pessoais.
- Logar URL sem query string quando ela puder conter tokens.
- Dados pessoais: mínimo necessário; suportar exportação e exclusão de conta quando o produto exigir (LGPD).

### Supply chain e verificação

- `golangci-lint` com `gosec` passando; `govulncheck ./...` antes de releases.
- Dependências com versão fixa no `go.mod`; avaliar manutenção e licença antes de adicionar.
- Imagens Docker com tag fixa (nunca `latest`).

---

## 11. Documentação da API (swag)

- Todo endpoint tem anotações swag **acima do handler**: `@Summary`, `@Tags`, `@Accept`, `@Produce`, `@Param`, `@Success`, `@Failure` (todos os erros possíveis), `@Security` quando autenticado, `@Router`.
- Gerar com `swag init -g cmd/api/main.go -o docs` (ou `make swag`) e **commitar** o resultado em `docs/`.
- Swagger desatualizado é bug: alterar endpoint ou DTO sem regenerar não está pronto.
- Swagger UI desabilitado ou protegido em produção, conforme o projeto.

```go
// Create godoc
// @Summary      Create user
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        request body dtos.CreateUserRequest true "User"
// @Success      201 {object} dtos.UserEnvelope
// @Failure      400 {object} dtos.ErrorResponse
// @Failure      409 {object} dtos.ErrorResponse
// @Security     BearerAuth
// @Router       /api/v1/users [post]
func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
```

---

## 12. Infraestrutura externa

### Storage

Interface (`Put`, `Get`, `Delete`, `PresignPut`, `PresignGet`...) com implementação S3-compatível. MinIO em dev; qualquer S3 em produção, sem mudar services.

### Filas (RabbitMQ)

- Só para trabalho realmente assíncrono (e-mail, processamento pesado, integrações lentas). Operação simples e síncrona é chamada direta.
- Mensagens com contrato próprio e enxuto (IDs, não entidades inteiras).
- **Consumers idempotentes**: a mesma mensagem pode chegar mais de uma vez.
- Retry com backoff e dead-letter queue; ack só depois do processamento.
- Efeitos colaterais que dependem de um commit (e-mail, notificação, evento) são publicados **após o commit** (outbox ou hook pós-commit), nunca antes.

### IA e APIs de terceiros

Sempre atrás de interface em `pkg/` (`ChatClient`, `PaymentGateway`, `Mailer`), com timeout, retry quando seguro e mock nos testes.

---

## 13. Observabilidade

- **Logs**: `slog` estruturado (JSON em produção) com `request_id`, `trace_id`, `user_id` quando houver, operação e erro. Nada de `fmt.Println`/`log.Printf`.
- **Tracing**: OpenTelemetry propagado por HTTP, banco (`otelsql`), fila e chamadas externas. Spans para operações significativas (`UserService.Create`), não por linha. `span.RecordError(err)` em falhas relevantes. Sem dados sensíveis em atributos.
- **Métricas**: Prometheus com contagem, latência e status por método + rota (template), filas e métricas de domínio. **Sem labels de alta cardinalidade** (`user_id`, `email`, `request_id`, URL crua).
- **Erros**: Sentry com dados sensíveis removidos.

---

## 14. Testes

| Tipo         | Ferramenta                                      | Onde                             |
| ------------ | ----------------------------------------------- | -------------------------------- |
| Unitário     | `testing` + Testify, table-driven               | ao lado do código (`_test.go`)   |
| Mocks        | GoMock (`//go:generate mockgen ...`)             | `tests/mocks/` (gerados)         |
| Repository   | PostgreSQL real via testcontainers + migrations | `tests/integration/` ou `_test`  |
| API          | `httptest` contra o router real                 | `tests/integration/`             |

- Services: sucesso, entrada inválida, não encontrado, conflito, **sem permissão**, falha de dependência.
- Endpoints: status, corpo, autenticação, **autorização (outro usuário não acessa)**, validação, erros.
- Repositories testados contra Postgres real, não mock de `database/sql`.
- Testes de integração rápidos: um container compartilhado por pacote e isolamento por schema/transação, rodando em paralelo quando possível. `-short` pula integração.
- Testes unitários determinísticos: sem rede real, sem `time.Now()` direto (injetar relógio), sem dependência de ordem.
- Mocks são gerados, não escritos à mão. Toda correção de bug vem com teste que falharia antes.
- `go test -race ./...` no CI.

---

## 15. Docker e desenvolvimento local

- `docker-compose.yml` sobe toda a infraestrutura (PostgreSQL, MinIO, RabbitMQ, Mailpit, Jaeger...) com **versões fixadas**. Nunca exigir instalação local desses serviços.
- `Dockerfile` multi-stage: build com `CGO_ENABLED=0`, imagem final mínima (distroless/alpine), **usuário não-root**, sem ferramentas de dev.
- `Makefile` com alvos padrão: `infra-up`, `infra-down`, `migrate`, `migrate-down`, `seed`, `run`, `worker`, `build`, `test`, `test-short`, `lint`, `fmt`, `vet`, `swag`, `mocks`, `check`.

Fluxo local:

```bash
make infra-up     # docker compose up -d
make migrate      # go run ./cmd/migrate up
make seed         # opcional
make run          # go run ./cmd/api
```

O README documenta esse fluxo, portas, credenciais de desenvolvimento e todas as variáveis.

---

## 16. Estilo e convenções

- `gofmt`/`goimports` sempre. Código idiomático: `if err != nil { return ... }` cedo, sem `else` desnecessário.
- Nomes curtos e claros: `UserService`, `GetByID`, não `UserManagementServiceImpl`. Siglas em maiúsculas (`ID`, `URL`, `HTTP`).
- Arquivos por recurso: `user_handler.go`, `user_service.go`, `user_repository.go`, `user_dto.go`.
- Funções pequenas com uma responsabilidade. Se uma função valida, faz SQL, chama HTTP e publica em fila, divida.
- Comentários explicam o **porquê**, não o quê. Exportados têm doc comment.

---

## 17. Fluxo para uma nova feature

1. Migration (`up` + `down`)
2. Model
3. Repository (SQL) + teste de integração
4. Service (regras + autorização + transação) + testes unitários
5. DTOs (request com `validate`, response)
6. Handler + anotações swag
7. Rota + middlewares
8. `swag init`, testes de API
9. README / `.env.example` / docs atualizados

### Checklist antes de concluir

```text
[ ] UUID como PK; migration nova (nenhuma antiga editada) com up e down
[ ] SQL parametrizado, sem SELECT *, sem N+1, listagem paginada
[ ] Autorização checada no backend (incluindo acesso a recurso de outro usuário)
[ ] Entrada validada, corpo limitado, erros sem detalhes internos
[ ] Nada sensível em logs, spans, métricas ou respostas
[ ] Context propagado; timeouts em chamadas externas
[ ] Swagger regenerado
[ ] Testes unitários + integração cobrindo sucesso, erros e permissão
[ ] gofmt, go vet, golangci-lint, go test -race ./..., go build ./... passando
[ ] .env.example e README atualizados quando necessário
```

```bash
make check        # ou: gofmt -l . && go vet ./... && golangci-lint run && go test -race ./... && go build ./...
make swag
govulncheck ./...
```

---

## 18. Regras para agentes

- Antes de escrever código: ler a estrutura, procurar implementação semelhante e **seguir o padrão existente**.
- Reutilizar helpers existentes (decode, respond, transação, erros, paginação) em vez de criar paralelos.
- Mudança mínima para resolver o pedido. Sem refatorações grandes não solicitadas.
- Não adicionar dependência sem necessidade clara; preferir a biblioteca padrão.
- Não alterar contrato público (rotas, payloads, códigos de erro) de forma incompatível sem pedido explícito.
- Nunca editar migration aplicada, desativar teste/lint para "passar" ou commitar segredos.
- Com várias soluções possíveis, escolher a de menor complexidade, mais idiomática, mais fácil de testar e com menor impacto.

### O que NÃO fazer

- Gin/Echo/Fiber, ORM, AutoMigrate.
- SQL no handler/service; regra de negócio no repository; HTTP no service.
- `os.Getenv` espalhado; segredo no código; `.env` commitado.
- `string` para UUID interno.
- Migration no startup da API.
- Goroutine sem ciclo de vida; erro ignorado; `context.Background()` no meio do fluxo.
- Expor erro do banco ao cliente; logar token/senha.
- Interface/abstração "por hábito"; arquivos gigantes com várias responsabilidades.

---

**Princípio geral:** simplicidade → clareza → segurança → testabilidade → desacoplamento → escala. A solução mais simples que atende corretamente ao requisito vence; toda abstração precisa resolver um problema concreto.
