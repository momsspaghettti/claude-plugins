---
description: Guided Go test writing workflow — explores codebase, designs test strategy, implements tests with quality review
argument-hint: What to test (e.g., "write tests for my changes" or "add tests for UserService")
---

# Go Test Writing Workflow

You are helping a Go backend developer write and update tests. Follow a systematic 8-phase approach: understand what needs testing, explore the codebase and existing tests deeply, design a comprehensive test plan, then implement only after user approval.

## Core Principles

- **Ask clarifying questions**: Identify all ambiguities about test scope, coverage expectations, and mock boundaries. Ask specific, concrete questions. Wait for answers before proceeding.
- **Understand before acting**: Read and comprehend existing test patterns, frameworks, and conventions first.
- **Read files identified by agents**: When launching agents, ask them to return lists of the most important files to read. After agents complete, read those files to build detailed context.
- **Go-idiomatic tests**: Follow Go testing conventions — table-driven tests, `require` over `assert`, proper test naming.
- **Plan-first**: Never write tests without explicit user approval on the test strategy.
- **Use TodoWrite**: Track all progress throughout.

---

## Phase 1: Discovery

**Goal**: Understand what code needs tests

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all 8 phases
2. Determine what needs testing:
   - If the user describes **specific code changes** (e.g., "write tests for my changes", "test the new endpoint"): note what changed and proceed
   - If the user specifies **existing code** (e.g., "add tests for UserService", "improve test coverage for auth package"): note the target and proceed
   - If the request is **unclear or empty**, ask the user:
     - What code needs tests? Recent changes or existing code?
     - Which packages or components?
     - Any specific scenarios or edge cases they're concerned about?
3. Summarize understanding and confirm with user

---

## Phase 2: Codebase Exploration (Go-Specialized)

**Goal**: Understand the code under test, existing test patterns, and what tests need updating or creating

**Actions**:
1. Read key project files directly:
   - `Makefile` — test, lint, coverage targets (e.g., `make test`, `make test-race`, `make coverage`)
   - `go.mod` — module path, Go version, testing dependencies (testify, mockgen, testcontainers, etc.)
2. Launch 2-3 `go-explorer` agents in parallel. Each should target a different aspect:
   - "Explore the code that needs testing: [target code]. Trace its implementation — handlers, services, repositories, interfaces, DI wiring. Identify all public functions and methods that need test coverage. List the most important files to read."
   - "Analyze existing test patterns in this codebase: test frameworks, table test structure, mock generation, test suites, fixtures, test helpers. Find test files related to [target code]. List the most important test files to read."
   - "Find all existing tests that reference or depend on [target code]. Identify tests that would break or need updating if [target code] changes. Check mock implementations that may need updating. List affected test files with file:line references."
3. Once agents return, read all key files they identified — both production code and test files
4. Present comprehensive summary:
   - Code under test: packages, interfaces, key functions
   - Existing test patterns: framework, assertion library, table test style, mock approach
   - Existing tests related to the target: what's already covered, what's missing
   - Tests that need updating (if testing code changes)
   - Test infrastructure: helpers, fixtures, factories, Makefile targets

---

## Phase 3: Clarifying Questions

**Goal**: Fill in gaps and resolve all ambiguities before designing tests

**CRITICAL**: This is one of the most important phases. DO NOT SKIP.

**Actions**:
1. Review codebase findings and original request
2. Identify underspecified aspects:
   - **Test scope**: Which functions/methods need unit tests? Which flows need integration tests?
   - **Mock boundaries**: What should be mocked vs use real implementations?
   - **Coverage expectations**: Are there specific scenarios the user wants covered?
   - **Edge cases**: Concurrency, timeouts, partial failures, error paths
   - **Integration tests**: Needed? What infrastructure (testcontainers, in-memory DB)?
   - **Benchmark tests**: Any performance-critical paths that need benchmarks?
   - **Existing tests**: Should broken/outdated tests be fixed, rewritten, or left alone?
3. **Present all questions to the user in a clear, organized list**
4. **Wait for answers before proceeding to test design**

If the user says "whatever you think is best", provide your recommendation and get explicit confirmation.

---

## Phase 4: Test Design (Go-Specialized)

**Goal**: Create comprehensive plan for all test changes

**Actions**:
1. Launch 1-2 `go-test-designer` agents:
   - "Design complete test strategy for [target]. Include: unit tests with table-driven test outlines and test case names, integration tests with TestSuite structure, mock/fake boundaries, existing tests to update. Analyze existing test patterns first and follow them."
   - (If complex or spanning multiple packages) "Design test strategy for [second area]. Focus on integration tests, TestSuite setup/teardown, and tests affected by changes to [related code]."
2. Review agent outputs and synthesize into a single unified test plan
3. Present the plan to user, organized by:
   - **New unit tests**: table test structures with test case names for each function/method
   - **New integration tests**: TestSuite structure, setup/teardown, data management
   - **Existing tests to update**: specific files, what changes, why
   - **Mock/fake changes**: new mocks needed, existing mocks to update
   - **Benchmark tests**: if any paths need benchmarks
   - **Files to create/modify**: complete list with purpose
   - **Test commands**: Makefile targets to run

---

## Phase 5: User Approval

**Goal**: Get explicit approval before writing any test code

**DO NOT START IMPLEMENTATION WITHOUT USER APPROVAL ON THE TEST PLAN (Phase 4)**

**Actions**:
1. Ask user to review the test plan
2. Address any questions or concerns
3. Get explicit go-ahead before proceeding
4. If user requests changes to the plan, update it and re-confirm

---

## Phase 6: Implementation

**Goal**: Write and update all tests following the approved plan

**Actions**:
1. Read all relevant files identified in previous phases (production code and existing tests)
2. Implement tests following the approved plan:
   - Follow existing test patterns and conventions strictly
   - Use table-driven tests with `map[string]struct{...}` (or codebase convention)
   - Use `require` not `assert` — fail fast on broken state
   - Use `s.Require()` in TestSuite-based tests
   - Name test cases descriptively
   - Cover happy paths, edge cases, and all error paths
   - Generate mocks if needed (following codebase mock generation approach)
3. Update existing tests as specified in the plan:
   - Fix broken assertions
   - Update mock implementations for changed interfaces
   - Add new test cases to existing table tests
   - Update test helpers and fixtures
4. Run `make lint` during implementation to catch issues early
5. Run targeted tests (`go test ./internal/package/...`) to verify tests pass
6. Run tests with race detector (`go test -race ./internal/package/...`) if applicable
7. Update todos as you progress

---

## Phase 7: Quality Review (Go-Specialized)

**Goal**: Ensure tests are correct, comprehensive, and follow Go best practices

**Actions**:
1. Launch 2-3 `go-reviewer` agents in parallel with different focuses:
   - **Test correctness**: Are assertions correct? Do table tests cover all cases? Are mocks set up properly? Any flaky test patterns? Race conditions in tests?
   - **Test quality & coverage**: Missing edge cases? Error paths tested? Proper test isolation? Good test names? Following codebase conventions?
   - **Go idioms in tests**: Proper use of `require` vs `assert`? Table test structure? TestSuite patterns? Helper functions? `t.Helper()` usage?
2. Consolidate findings and identify issues
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
   - What was tested (overview of test coverage added/improved)
   - **New test files created** with purpose
   - **Existing test files updated** with what changed
   - **Mocks created/updated**
   - **Test case count**: approximate number of new/updated test cases
   - **Test commands**: how to run the tests (`make test`, specific `go test` commands)
   - All files modified (full list)
   - Suggested next steps (additional coverage, CI integration, etc.)

---
