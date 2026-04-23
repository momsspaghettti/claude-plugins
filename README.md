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

### [gitlab-mr-review](plugins/gitlab-mr-review/)

Iterative, multi-agent review of GitLab Merge Requests. Produces a structured report with copy-paste-ready inline findings, detects regressions across runs, and integrates Jira/Slack MCPs when available. Requires `glab` CLI.

**Commands:**
- `/gitlab-mr-review:review <MR-URL>` — full multi-agent review with copy-paste-ready findings

**Features:**
- 9 specialized agents (preflight, context gatherer, 5 review focuses, validator, report composer)
- Read-only by design — no posting, no thread resolution
- Iterative: recognizes its own prior findings via hidden footer signature; flags regressions
- Optional Jira MCP integration for requirements extraction
- Optional Slack MCP integration for context harvesting
- Works on any self-hosted GitLab instance

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
