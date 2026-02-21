---
name: gopls-usage
description: This skill should be used when exploring Go code semantically, "finding interface implementations", "finding references to a type or function", "navigating Go code with gopls", "understanding Go interfaces", "renaming Go symbols", or when semantic code navigation is needed instead of text-based grep search. Preloaded by all go-backend-dev agents.
---

# Using gopls for Go Code Navigation

gopls (Go Please) is the official Go language server. It provides semantic code intelligence — understanding types, interfaces, and references at a level that text-based search cannot match.

## When to Use gopls vs Grep

**Use gopls when:**
- Finding all implementations of an interface
- Finding all callers of a function (semantic references, not string matches)
- Navigating to a symbol's definition
- Understanding interface satisfaction
- Renaming symbols safely across the codebase
- Finding all types that embed a struct

**Use Grep when:**
- Searching for string literals, comments, or configuration values
- Finding TODO/FIXME markers
- Searching across non-Go files (YAML, JSON, Makefile, etc.)
- Pattern matching with regex (error messages, log strings)

## gopls CLI Commands

### Find Definition

Jump to where a symbol is defined:

```bash
gopls definition <file>:<line>:<column>
```

Example: Find where `UserService` is defined when cursor is on its usage.

### Find References

Find all references to a symbol across the codebase:

```bash
gopls references <file>:<line>:<column>
```

Use this to understand how widely a type, function, or method is used before modifying it.

### Find Implementations

Find all types that implement an interface:

```bash
gopls implementation <file>:<line>:<column>
```

Point at an interface definition to find all concrete implementations. This is critical for understanding dependency injection and polymorphism in Go codebases.

### List Symbols

List all symbols in a file or workspace:

```bash
gopls symbols <file>
```

Useful for getting a quick overview of types, functions, and methods in a file.

### Rename Symbol

Safely rename a symbol across the entire codebase:

```bash
gopls rename -w <file>:<line>:<column> <new-name>
```

The `-w` flag writes changes to disk. Always review the scope of a rename before executing.

### Workspace Symbols

Search for symbols across the workspace by name:

```bash
gopls workspace_symbol <query>
```

Find types, functions, or methods by name pattern across all packages.

## Reading LSP Diagnostics

After making edits to Go files, check LSP diagnostics to catch errors early:

1. Save the file to trigger gopls analysis
2. Check for diagnostic messages — they report type errors, unused imports, and other issues
3. Fix diagnostics before proceeding with further changes

## Common Patterns

### Mapping Interface Hierarchies

To understand an interface hierarchy:

1. Use `gopls symbols` on the file to find the interface definition
2. Use `gopls implementation` on the interface to find all implementors
3. Use `gopls references` on each method to understand usage patterns

### Tracing Call Chains

To trace how a function is called:

1. Use `gopls definition` to navigate to the function
2. Use `gopls references` to find all call sites
3. Repeat for each caller to build the full call chain

### Understanding DI Wiring

To understand dependency injection:

1. Find the interface with `gopls symbols` or `Grep`
2. Use `gopls implementation` to find concrete types
3. Use `gopls references` on constructors (`NewXxx`) to find where wiring happens
4. Trace to the DI container or main function

## Integration with Go Tools

gopls works alongside other Go tools:

- `go vet` — static analysis for common mistakes
- `golangci-lint` — comprehensive linting
- `go test -race` — race condition detection
- `go build` — compilation errors

Check the project's Makefile for how these tools are configured (`make lint`, `make vet`, `make test`).
