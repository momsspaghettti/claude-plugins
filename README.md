# Claude Code Plugins

A collection of Claude Code plugins for specialized development workflows.

## Plugins

### [go-backend-dev](plugins/go-backend-dev/)

Structured workflows for developing features and writing tests in Go backend services. Specialized agents for codebase exploration, architecture design, test strategy, and quality review — optimized for Go idioms, gopls semantic navigation, and Makefile-driven workflows.

**Commands:**
- `/go-backend-dev:feature <description>` — 8-phase feature development workflow
- `/go-backend-dev:tests <what to test>` — guided test writing workflow

**Features:**
- gopls integration for semantic code navigation
- Go-specialized agents (explorer, architect, reviewer, test designer)
- 8-phase workflows with mandatory plan approval before implementation
- Go-specific code review (concurrency, performance, security, idioms)
- Test strategy planning (table tests, TestSuite, mock boundaries)
- Makefile-aware build system integration

## Installation

Install plugins from this repository via Claude Code:

```bash
claude plugins install <plugin-name>
```

Or for local development:

```bash
claude --plugin-dir ./plugins/<plugin-name>
```

## License

MIT
