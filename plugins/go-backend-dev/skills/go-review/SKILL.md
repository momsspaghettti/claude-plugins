---
name: go-review
description: This skill should be used when reviewing Go backend code changes, "review Go code", "check for race conditions", "Go code review checklist", "check Go error handling", "review concurrency safety", "Go security review", or when performing quality review of Go backend implementations. Preloaded by the go-reviewer agent.
---

# Go Backend Code Review Checklist

Systematic checklist for reviewing Go backend code. Evaluate each category and report only issues with confidence >= 80.

## 1. Concurrency Safety

### Race Conditions
- Shared mutable state accessed without synchronization (`sync.Mutex`, `sync.RWMutex`, atomic operations)
- Map access from multiple goroutines without protection (maps are not goroutine-safe)
- Slice append from multiple goroutines
- Check-then-act patterns without locking

### Goroutine Leaks
- Goroutines blocked on channels that are never closed or written to
- Missing `context.Context` cancellation propagation
- `select` without `ctx.Done()` case
- Goroutines spawned without lifecycle management

### Channel Usage
- Unbuffered channels causing unexpected blocking
- Sending on closed channels (panic)
- Missing `close()` on channels when producer is done
- Channel direction not specified in function signatures

### Mutex Patterns
- `defer mu.Unlock()` not used (risk of missing unlock on error paths)
- Nested locks without consistent ordering (deadlock risk)
- Holding locks across I/O or network calls (performance, deadlock)
- Using `sync.Mutex` where `sync.RWMutex` would improve read-heavy workloads

## 2. Error Handling

### Error Wrapping
- Errors returned without context — must wrap with `fmt.Errorf("context: %w", err)`
- Using `%v` instead of `%w` (breaks `errors.Is()` / `errors.As()` chain)
- Over-wrapping (adding noise without useful context)

### Sentinel Errors
- String comparison on error messages instead of `errors.Is()`
- Missing sentinel error definitions for domain errors
- Sentinel errors not using `errors.New()` (using `fmt.Errorf` for sentinels)

### Unchecked Errors
- Ignoring returned errors (assign to `_` without justification)
- Missing error checks on `Close()`, `Flush()`, `Write()` calls
- `defer file.Close()` without error handling on writes (data may not be flushed)

### Defer Pitfalls
- `defer` in loops (resource accumulation until function returns)
- Deferred function capturing loop variable by reference
- Relying on `defer` order for correctness without understanding LIFO

## 3. Performance

### Allocations in Hot Paths
- String concatenation in loops (use `strings.Builder`)
- Creating slices without pre-allocated capacity when size is known
- Unnecessary pointer indirection causing heap allocation
- Repeated `[]byte` to `string` conversions

### Connection & Resource Management
- Missing connection pool configuration (`MaxOpenConns`, `MaxIdleConns`)
- HTTP clients without timeouts (`http.DefaultClient` has no timeout)
- Missing `defer resp.Body.Close()` after HTTP requests
- Connection leaks on error paths

### Sync.Pool
- Not using `sync.Pool` for frequently allocated temporary objects in hot paths
- Incorrect `sync.Pool` usage (storing pointers to stack objects)

### N+1 Queries
- Database queries inside loops — batch or join instead
- Missing database indexes for query patterns
- Large result sets loaded entirely into memory

## 4. Security

### SQL Injection
- String interpolation in SQL queries — use parameterized queries
- `fmt.Sprintf` for building SQL — use query builder or `$1` placeholders
- Dynamic table/column names without allowlist validation

### Input Validation
- Missing validation on user input (request bodies, query params, path params)
- Integer overflow on user-provided numbers
- Path traversal via user-provided file paths
- Missing rate limiting on authentication endpoints

### Sensitive Data
- Passwords, tokens, or API keys in log output
- Sensitive fields not redacted in error messages
- Secrets in environment variables logged at startup
- Missing `json:"-"` on sensitive struct fields

### Rate Limiting
- Missing rate limiting on public endpoints
- Rate limiter not keyed by appropriate identifier (user, IP, API key)

## 5. Service Configuration

### Environment Variables
- Missing validation of required environment variables at startup
- No defaults for optional configuration
- Parsing errors not caught early (deferred to first use)

### Graceful Shutdown
- Missing `signal.NotifyContext` or `signal.Notify` for shutdown signals
- Missing `server.Shutdown(ctx)` with timeout
- Background workers not stopped on shutdown
- Database connections not closed on shutdown

### Secrets Management
- Hardcoded secrets in source code
- Secrets in configuration files committed to version control
- Missing rotation support for credentials

## 6. Go Idioms

### Naming Conventions
- Non-idiomatic names (underscores, `Get` prefix on getters, verbose names)
- Inconsistent receiver names across methods of same type
- Package names that stutter with exported names (`user.UserService` → `user.Service`)

### Interface Design
- Interfaces defined in implementor package (should be in consumer package)
- Large interfaces when small ones would suffice
- Preemptive interfaces without multiple implementations or test need

### Zero Values
- Structs requiring initialization but not using constructor (`New` function)
- Nil pointer dereference on zero-value structs with pointer fields

### Godoc
- Exported types/functions missing doc comments
- Doc comments not starting with the name of the element
- Package-level doc missing

## 7. Code Readability

### Function Length
- Functions exceeding ~50 lines (consider extracting)
- Functions doing more than one thing (violating SRP)
- Deep nesting (>3 levels) — prefer early returns

### Package Organization
- Package too large (>20 files) — consider splitting
- Circular dependencies between packages
- `utils` or `helpers` packages (anti-pattern — move functions to domain packages)

### Comment Quality
- Comments explaining "what" instead of "why"
- Outdated comments that don't match the code
- Commented-out code that should be deleted

## Review Process

1. **Read the diff** — understand what changed and why
2. **Check each category** systematically
3. **Use gopls** for semantic analysis (find references, implementations)
4. **Run `make lint`** and `go vet` to catch static analysis issues
5. **Rate confidence** for each finding (0-100 scale)
6. **Report only issues >= 80 confidence**
7. **Group by severity** — Critical (will cause bugs/outages) vs Important (should fix)
8. **Provide concrete fixes** — don't just point out problems, suggest solutions
