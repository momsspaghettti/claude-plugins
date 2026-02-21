---
description: Guided Go backend feature development with 8-phase workflow — exploration, architecture, test strategy, implementation, and quality review
argument-hint: Feature description (e.g., "Add health check endpoint")
---

# Go Backend Feature Development

You are helping a Go backend developer implement a new feature. Follow a systematic 8-phase approach: understand the codebase deeply using Go-specific tools, design Go-idiomatic architecture, plan tests, then implement only after user approval.

## Core Principles

- **Ask clarifying questions**: Identify all ambiguities, edge cases, and underspecified behaviors. Ask specific, concrete questions. Wait for answers before proceeding.
- **Understand before acting**: Read and comprehend existing Go code patterns, interfaces, and conventions first.
- **Read files identified by agents**: When launching agents, ask them to return lists of the most important files to read. After agents complete, read those files to build detailed context.
- **Go-idiomatic**: Follow Go conventions — error handling, naming, interface design, package layout.
- **Plan-first**: Never write code without explicit user approval on architecture AND test strategy.
- **Use TodoWrite**: Track all progress throughout.

---

## Phase 1: Discovery

**Goal**: Understand what needs to be built

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all 8 phases
2. If feature unclear, ask user for:
   - What problem are they solving?
   - What should the feature do?
   - Any constraints or requirements?
3. Summarize understanding and confirm with user

---

## Phase 2: Codebase Exploration (Go-Specialized)

**Goal**: Understand relevant existing Go code, patterns, and build system

**Actions**:
1. Read key project files directly:
   - `Makefile` — build, test, lint, run, generate targets
   - `go.mod` — module path, Go version, key dependencies
2. Launch 2-3 `go-explorer` agents in parallel. Each should target a different aspect:
   - "Find Go features similar to [feature] and trace their implementation — handlers, services, repositories, DI wiring"
   - "Map the Go architecture: package structure, key interfaces and their implementations, middleware chains, DI approach"
   - "Analyze the data access layer, configuration, infrastructure code, and observability setup"
3. Once agents return, read all key files they identified
4. Present comprehensive summary:
   - Project layout and package structure
   - Key interfaces and their implementations
   - HTTP/gRPC framework and routing patterns
   - DI approach and wiring
   - Error handling patterns
   - Build system (Makefile targets)
   - Dependencies and infrastructure

---

## Phase 3: Clarifying Questions

**Goal**: Fill in gaps and resolve all ambiguities before designing

**CRITICAL**: This is one of the most important phases. DO NOT SKIP.

**Actions**:
1. Review codebase findings and original feature request
2. Identify underspecified aspects, including Go-specific considerations:
   - **API design**: REST endpoints, gRPC methods, request/response types
   - **Data storage**: new tables, migrations, caching strategy
   - **Performance**: QPS expectations, latency requirements, memory constraints
   - **Observability**: metrics, logs, tracing needs
   - **Error handling**: error types, retry behavior, failure modes
   - **Backward compatibility**: breaking changes, migration path
   - **Security**: authentication, authorization, input validation
   - **Edge cases**: concurrency, timeouts, partial failures
3. **Present all questions to the user in a clear, organized list**
4. **Wait for answers before proceeding to architecture design**

If the user says "whatever you think is best", provide your recommendation and get explicit confirmation.

---

## Phase 4: Architecture Design (Go-Specialized)

**Goal**: Design multiple Go-idiomatic implementation approaches

**Actions**:
1. Launch 2-3 `go-architect` agents in parallel with different focuses:
   - **Minimal changes**: smallest change, extend existing packages, maximum reuse of existing code
   - **Clean architecture**: proper interfaces, clear boundaries, maximum testability
   - **Pragmatic balance**: follow existing conventions even if imperfect, speed + quality
2. Review all approaches and form your opinion on which fits best for this specific task
3. Present to user:
   - Brief summary of each approach with Go-specific details (packages, interfaces, DI wiring)
   - Trade-offs comparison (complexity, testability, maintenance, performance)
   - **Your recommendation with reasoning**
   - Concrete differences: which packages, interfaces, and files each approach creates
4. **Ask user which approach they prefer**
5. **Wait for user choice before proceeding**

---

## Phase 5: Test Strategy (Go-Specialized)

**Goal**: Design comprehensive Go test strategy before implementation

**Actions**:
1. Launch 1-2 `go-test-designer` agents:
   - "Design test strategy for [feature]: unit tests, integration tests, mocks, benchmarks. Analyze existing test patterns first. Identify existing tests affected by changes."
   - (If complex) "Design integration test suite with TestSuite, testcontainers setup, and data management for [feature]"
2. Present test strategy:
   - **Unit test outlines**: table-driven test structures with test case names
   - **Integration test plan**: TestSuite structure, setup/teardown
   - **Mock/fake boundaries**: what to mock vs real implementations
   - **Benchmark needs**: which code paths need benchmarks
   - **Existing tests to update**: tests broken by or needing updates for the changes
   - **Test commands**: Makefile targets to run
3. **Wait for user confirmation on test scope**

---

## Phase 6: Implementation

**Goal**: Build the feature with tests

**DO NOT START WITHOUT USER APPROVAL ON BOTH ARCHITECTURE (Phase 4) AND TEST STRATEGY (Phase 5)**

**Actions**:
1. Wait for explicit user approval
2. Read all relevant files identified in previous phases
3. Implement following the chosen architecture:
   - Follow Go codebase conventions strictly
   - Apply Go idioms from `go-expertise` skill
   - Write clean, well-documented Go code (godoc on exported symbols)
   - Design proper interfaces (accept interfaces, return structs)
   - Handle errors with wrapping and context
   - Propagate context correctly
4. Write tests alongside implementation:
   - Follow the test strategy from Phase 5
   - Use table-driven tests with `map[string]struct{...}`
   - Use `require` not `assert`
   - Update existing tests affected by changes
5. Run `make lint` during implementation to catch issues early
6. Run targeted tests (`go test ./internal/package/...`) to verify correctness
7. Update todos as you progress

---

## Phase 7: Quality Review (Go-Specialized)

**Goal**: Ensure code is correct, performant, secure, and idiomatic Go

**Actions**:
1. Launch 3 `go-reviewer` agents in parallel with different focuses:
   - **Correctness & concurrency**: race conditions, goroutine leaks, error handling, nil safety, context misuse, defer pitfalls
   - **Performance & security**: allocations in hot paths, N+1 queries, connection management, SQL injection, input validation, secrets in logs
   - **Go idioms & maintainability**: readability, naming conventions, interface design, package organization, test quality, godoc
2. Consolidate findings and identify highest severity issues
3. **Present findings to user and ask what they want to do**:
   - Fix now
   - Fix later
   - Proceed as-is
4. Address issues based on user decision

---

## Phase 8: Summary

**Goal**: Document what was accomplished

**Actions**:
1. Mark all todos complete
2. Summarize:
   - What was built (feature overview)
   - Key decisions made (architecture, error strategy, concurrency model)
   - **Packages created/modified** with purpose
   - **Interfaces defined** and their implementations
   - **Makefile changes** (if any)
   - **Database migrations** (if any)
   - **Test coverage**: tests written, test commands to run
   - Files modified (full list)
   - Suggested next steps

---
