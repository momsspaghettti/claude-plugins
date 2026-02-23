---
name: go-expertise
description: This skill should be used when designing, implementing, or reasoning about Go backend code, "Go project layout", "Go error handling patterns", "Go concurrency patterns", "dependency injection in Go", "Go testing conventions", "table-driven tests", "Go Makefile targets", "Go observability", "structured logging in Go", or when senior Go backend development knowledge is needed. Preloaded by go-explorer, go-architect, and go-test-designer agents.
---

# Senior Go Backend Developer Knowledge

Comprehensive patterns and conventions for building production Go backend services.

## Project Layout

Standard Go project structure:

- `cmd/` — Application entry points. Each subdirectory is a binary (`cmd/server/`, `cmd/worker/`, `cmd/migrate/`).
- `internal/` — Private packages. Cannot be imported by external modules. Use for business logic, handlers, repositories.
- `pkg/` — Public packages meant for external consumption. Use sparingly — most code belongs in `internal/`.
- `api/` — API definitions: OpenAPI specs, protobuf files, GraphQL schemas.
- `migrations/` — Database migration files (SQL or Go-based).
- `config/` or `configs/` — Configuration files and templates.
- `scripts/` — Build, CI, and utility scripts.

Explore each project's actual layout — not all projects follow this exactly.

## Error Handling

### Wrapping Errors

Always wrap errors with context using `fmt.Errorf` and `%w`:

```go
if err != nil {
    return fmt.Errorf("failed to fetch user %d: %w", userID, err)
}
```

### Sentinel vs Typed Errors

- **Sentinel errors** (`var ErrNotFound = errors.New("not found")`) — for simple, well-known error conditions. Check with `errors.Is()`.
- **Typed errors** (`type ValidationError struct{...}`) — when errors carry additional data. Check with `errors.As()`.
- Define domain errors in the package that owns the concept.

### Error Propagation

- Wrap at each layer with additional context
- Don't log and return — do one or the other
- Handle errors at the boundary (HTTP handler, gRPC interceptor)
- Use `errors.Is()` and `errors.As()` for checking, never string comparison

## Concurrency Patterns

### Worker Pools

```go
func processItems(ctx context.Context, items []Item, workers int) error {
    g, ctx := errgroup.WithContext(ctx)
    ch := make(chan Item)
    g.Go(func() error { defer close(ch); /* send items */ })
    for i := 0; i < workers; i++ {
        g.Go(func() error { /* process from ch */ })
    }
    return g.Wait()
}
```

### Key Concurrency Rules

- Always pass `context.Context` as first parameter
- Use `errgroup` for managing goroutine lifecycles
- Prefer `sync.Mutex` for simple shared state, channels for communication
- Always handle goroutine cleanup — no goroutine leaks
- Use `context.WithCancel` or `context.WithTimeout` for cancellation
- Run tests with `-race` flag

## Dependency Injection

### Constructor Injection (Preferred)

```go
type UserService struct {
    repo   UserRepository
    cache  Cache
    logger *slog.Logger
}

func NewUserService(repo UserRepository, cache Cache, logger *slog.Logger) *UserService {
    return &UserService{repo: repo, cache: cache, logger: logger}
}
```

### Functional Options

For constructors with many optional parameters:

```go
type Option func(*Server)
func WithPort(port int) Option { return func(s *Server) { s.port = port } }
func NewServer(opts ...Option) *Server { /* apply opts */ }
```

### DI Frameworks

Some projects use `wire` (compile-time) or `fx` (runtime). Check `go.mod` and look for `wire_gen.go` files or `fx.New()` calls.

## High-Load Patterns

- **Connection pooling** — `sql.DB` already pools; configure `MaxOpenConns`, `MaxIdleConns`, `ConnMaxLifetime`
- **Caching** — in-memory (`sync.Map`, `groupcache`) or external (Redis). Always set TTLs.
- **Circuit breaker** — use `sony/gobreaker` or similar for external service calls
- **Rate limiting** — `golang.org/x/time/rate` for token bucket
- **Graceful degradation** — fallback responses when dependencies are down
- **Batch processing** — aggregate operations to reduce I/O

## Observability

### Structured Logging

Use `slog` (stdlib) or `zerolog`/`zap`. Always structured, never `fmt.Printf`:

```go
slog.Info("user created", "user_id", user.ID, "email", user.Email)
```

### Metrics, Tracing, Profiling

- **Metrics** — Prometheus client (`prometheus/client_golang`), expose `/metrics`
- **Tracing** — OpenTelemetry (`go.opentelemetry.io/otel`), propagate via context
- **Profiling** — `net/http/pprof` for CPU/memory profiling

## Testing Conventions

### Table-Driven Tests

Prefer `map[string]struct{...}` approach:

```go
func TestParseAmount(t *testing.T) {
    tests := map[string]struct {
        input    string
        expected int
        wantErr  bool
    }{
        "valid amount":   {input: "100", expected: 100},
        "negative":       {input: "-5", expected: -5},
        "invalid string": {input: "abc", wantErr: true},
    }
    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got, err := ParseAmount(tc.input)
            if tc.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            require.Equal(t, tc.expected, got)
        })
    }
}
```

### TestSuite for Integration Tests

Use `testify` suite for tests with shared setup/teardown:

```go
type UserServiceSuite struct {
    suite.Suite
    db      *sql.DB
    service *UserService
}

func (s *UserServiceSuite) SetupSuite()    { /* DB setup */ }
func (s *UserServiceSuite) TearDownSuite() { /* DB cleanup */ }
func (s *UserServiceSuite) TestCreateUser() {
    user, err := s.service.Create(ctx, input)
    s.Require().NoError(err)
    expected := &User{ID: 1, Email: "test@example.com", Name: "Test User"}
    s.Require().Equal(expected, user)
}
func TestUserServiceSuite(t *testing.T) { suite.Run(t, new(UserServiceSuite)) }
```

### Testing Rules

- **Use `require`, not `assert`** — fail fast, don't continue on broken state
- **Use `s.Require()` in suites** — same principle
- **httptest** for HTTP handler testing
- **bufconn** for gRPC testing without network
- **testcontainers** for integration tests with real databases
- **Benchmarks** with `testing.B` for performance-critical code
- **Fuzzing** with `testing.F` for input validation
- **Compare whole objects** — pre-build the expected struct and assert with `require.Equal(t, expected, actual)` instead of comparing field by field
- **Always run with `-race`** in CI

## Makefile Conventions

Check the project's Makefile for standard targets:

- `make build` — compile the application
- `make test` — run unit tests
- `make test-race` — run tests with race detector
- `make lint` — run linter (usually `golangci-lint`)
- `make run` — start the application locally
- `make migrate` — run database migrations
- `make generate` — run `go generate` (protobuf, mocks, wire)
- `make coverage` — generate test coverage report

Always discover and use existing Makefile targets rather than running raw commands.

## Code Style

### Naming

- Use `MixedCaps` / `mixedCaps`, never underscores (except in test function names)
- Short receiver names: `s` for `*Server`, `u` for `*UserService`
- Interfaces: single-method interfaces use `-er` suffix (`Reader`, `Handler`)
- Constructors: `NewXxx` pattern
- Getters: just `Name()`, not `GetName()`

### Interface Design

- **Accept interfaces, return structs** — define interfaces where they're consumed, not where they're implemented
- Keep interfaces small — prefer single-method interfaces
- Don't create interfaces preemptively — wait until there are multiple implementations or a testing need

### Zero Values

- Design structs so zero values are useful
- `sync.Mutex{}` is ready to use (unlocked)
- `bytes.Buffer{}` is ready to use (empty)
- Document when zero value is not safe

## HTTP + gRPC Patterns

Explore each project's specific stack from code:

- **HTTP routers** — `chi`, `gin`, `echo`, `gorilla/mux`, or stdlib `net/http`
- **gRPC** — look for `.proto` files, generated `_grpc.pb.go`, service registrations
- **Middleware** — authentication, logging, tracing, recovery, rate limiting
- **Handler registration** — how routes are wired (central router file vs per-package)
- **Request/response patterns** — JSON marshaling, protobuf, validation

Don't assume the stack — always read the code to discover which frameworks and patterns are used.
