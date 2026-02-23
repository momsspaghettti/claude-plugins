# Go Backend Development Plugin

A structured workflow for developing features and writing tests in Go backend services. Optimized for Go idioms, gopls semantic navigation, and Makefile-driven workflows.

## Overview

This plugin provides structured workflows for Go backend development. The **feature** workflow guides you through an 8-phase approach to building features: understanding the codebase (with gopls), asking clarifying questions, designing Go-idiomatic architecture, planning tests, and ensuring quality. The **tests** workflow focuses specifically on writing and updating tests for existing or changed code with the same rigor — codebase exploration, test strategy design, implementation, and quality review.

## Philosophy

Go backend development requires specific expertise:
- **Understand the codebase semantically** using gopls for interface discovery, not just text search
- **Design Go-idiomatic architecture** with proper interfaces, error handling, and package layout
- **Plan tests first** — table-driven tests, TestSuite for integration, proper mock boundaries
- **Check the build system** — use existing Makefile targets for lint, test, build
- **Review for Go-specific issues** — race conditions, goroutine leaks, context misuse

This plugin embeds these practices into a structured workflow.

## Commands

### `/go-backend-dev:feature`

Launches the guided 8-phase feature development workflow.

**Usage:**
```bash
/go-backend-dev:feature Add rate limiting middleware
```

### `/go-backend-dev:tests`

Launches the guided test writing workflow — explores codebase, designs test strategy, implements tests with quality review.

**Usage:**
```bash
/go-backend-dev:tests Write tests for my changes
/go-backend-dev:tests Add tests for UserService
/go-backend-dev:tests Improve test coverage for auth package
```

## The Feature Workflow (8 Phases)

### Phase 1: Discovery
Understand what needs to be built. Clarify the feature request, identify constraints.

### Phase 2: Codebase Exploration (Go-Specialized)
- Reads `go.mod` and `Makefile` for project context
- Launches `go-explorer` agents that use gopls for semantic code navigation
- Maps Go-specific architecture: handlers, services, repositories, middleware, DI wiring
- Reports key interfaces and their implementations

### Phase 3: Clarifying Questions
Resolves ambiguities with Go-specific considerations: API design, data storage, performance expectations, observability needs, error handling strategy.

### Phase 4: Architecture Design (Go-Specialized)
- Launches `go-architect` agents with different trade-off profiles
- Designs Go-idiomatic package layout, interfaces, error types, DI wiring
- Presents options with recommendation
- **Waits for user choice**

### Phase 5: Test Strategy (NEW)
- Launches `go-test-designer` agents
- Plans table-driven unit tests, TestSuite integration tests, mock boundaries
- Identifies existing tests that need updating
- **Waits for user confirmation**

### Phase 6: Implementation
- **Requires approval on architecture AND test strategy**
- Writes code following chosen architecture and test plan
- Runs `make lint` and targeted tests during implementation
- Applies Go idioms throughout

### Phase 7: Quality Review (Go-Specialized)
Launches 3 `go-reviewer` agents checking:
1. Correctness & concurrency (races, goroutine leaks, error handling)
2. Performance & security (allocations, queries, SQL injection)
3. Go idioms & maintainability (naming, interfaces, test quality)

Presents findings, **waits for user decision** (fix now / fix later / proceed).

### Phase 8: Summary
Documents what was built: packages, interfaces, Makefile changes, test coverage, next steps.

## The Tests Workflow (8 Phases)

### Phase 1: Discovery
Understand what code needs tests — recent changes, specific packages, or existing code lacking coverage.

### Phase 2: Codebase Exploration (Go-Specialized)
- Reads `go.mod` and `Makefile` for project context
- Launches `go-explorer` agents to understand the code under test, existing test patterns, and affected tests
- Maps what's already covered and what's missing

### Phase 3: Clarifying Questions
Resolves test scope ambiguities: mock boundaries, coverage expectations, edge cases, integration test needs.

### Phase 4: Test Design (Go-Specialized)
- Launches `go-test-designer` agents
- Plans new unit tests, integration tests, mock changes, and existing test updates
- **Waits for user approval**

### Phase 5: User Approval
Presents the complete test plan. **Requires explicit approval before implementation.**

### Phase 6: Implementation
- Writes new tests and updates existing ones following the approved plan
- Runs `make lint` and targeted tests during implementation
- Applies Go test conventions throughout

### Phase 7: Quality Review (Go-Specialized)
Launches `go-reviewer` agents checking test correctness, coverage completeness, and Go test idioms.

### Phase 8: Summary
Documents tests written, files created/modified, test commands to run.

## Agents

### `go-explorer` (yellow)
Explores Go codebases using gopls semantic navigation. Maps architecture layers, traces handler flows, identifies key interfaces and their implementations, discovers DI wiring patterns.

### `go-architect` (green)
Designs Go-idiomatic feature architectures. Plans package layout, interface design, error strategy, middleware integration, DI wiring, concurrency model, and observability.

### `go-reviewer` (red)
Reviews Go code for concurrency bugs, performance issues, security vulnerabilities, and Go idiom violations. Uses confidence scoring (>= 80 threshold) to report only high-priority issues. Runs `make lint` and `go vet`.

### `go-test-designer` (cyan)
Designs Go test strategies: table-driven unit tests, TestSuite-based integration tests, mock/fake boundaries, benchmarks. Identifies existing tests that need updating due to code changes.

## Skills

### `gopls-usage`
How to use gopls CLI for semantic code navigation: definitions, references, implementations, symbols, rename. When to prefer gopls over grep.

### `go-expertise`
Senior Go backend developer knowledge: project layout, error handling, concurrency, DI, high-load patterns, observability, testing conventions, Makefile conventions, code style.

### `go-review`
Go backend code review checklist: concurrency safety, error handling, performance, security, service config, Go idioms, readability. Organized for systematic review.

## LSP Integration

The plugin bundles gopls configuration (`.lsp.json`) that activates automatically, providing semantic code intelligence for all Go files.

## Requirements

- Claude Code installed
- Go project with `go.mod`
- `gopls` installed (`go install golang.org/x/tools/gopls@latest`)
- Makefile (recommended but not required)

## Context7 Integration

All agents use Context7 MCP tools to look up latest documentation for external libraries (testify, chi, gin, gRPC, wire, fx, etc.) when needed. Requires the Context7 plugin to be installed separately.

## When to Use

**Use `/go-backend-dev:feature` for:**
- New Go backend features (endpoints, services, middleware)
- Complex bug fixes requiring architectural understanding
- Features touching multiple packages
- Adding new integrations or data access layers

**Use `/go-backend-dev:tests` for:**
- Writing tests for recent code changes
- Adding test coverage to existing packages
- Updating tests after refactoring
- Improving test quality and coverage

**Don't use for:**
- Single-line bug fixes
- Simple configuration changes
- Non-Go code changes
- CLI tool development (this plugin is optimized for backend services)

## Tips

- **Be specific** in your feature description — fewer clarifying questions needed
- **Trust the exploration phase** — it discovers patterns you should follow
- **Review agent outputs carefully** — they provide valuable codebase insights
- **Approve architecture deliberately** — Phase 4 gives you options for a reason
- **Don't skip test strategy** — Phase 5 ensures comprehensive test coverage
