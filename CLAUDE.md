# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

A collection of Claude Code plugins providing specialized development workflows. Each plugin is a self-contained unit under `plugins/<plugin-name>/` with its own agents, commands, skills, and configuration.

The repository has no build system, test suite, or CI/CD — it is a pure content repository of markdown-based plugin definitions.

## Plugin Structure Convention

Every plugin follows this layout:

```
plugins/<plugin-name>/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest (name, version, description, author, keywords)
├── .lsp.json                # Optional LSP server configuration
├── README.md                # Plugin documentation
├── commands/                # Slash commands (markdown with YAML frontmatter)
│   └── <command>.md         # Invoked as /plugin-name:command-name
├── agents/                  # Subagent definitions (markdown with YAML frontmatter)
│   └── <agent>.md           # Defines: name, description, tools, skills, model, color
└── skills/                  # Injectable knowledge references
    └── <skill-name>/
        └── SKILL.md         # Defines: name, description + content body
```

The top-level `.claude-plugin/marketplace.json` is the plugin registry — it lists all plugins with their source paths.

## Key Conventions

- **Commands** use YAML frontmatter (`description`, `argument-hint`) followed by the full prompt/workflow instructions. Arguments are referenced via `$ARGUMENTS`.
- **Agents** use YAML frontmatter (`name`, `description`, `tools`, `skills`, `model`, `color`) followed by their system prompt. The `description` field controls when the agent is triggered. Agents specify `model: inherit` to use the parent model.
- **Skills** use YAML frontmatter (`name`, `description`) followed by reference knowledge. The `description` field controls when the skill is injected. Skills are referenced by name in agent frontmatter.
- **`plugin.json`** must include `name`, `version`, `description`, `author`, and `keywords`.

## Running Locally

```bash
claude --plugin-dir ./plugins/<plugin-name>
```

## Current Plugins

- **go-backend-dev** — Go backend development workflows with gopls integration, 4 specialized agents (explorer, architect, reviewer, test-designer), and 3 skills (gopls-usage, go-expertise, go-review). Entry points: `/go-backend-dev:feature <description>` for feature development, `/go-backend-dev:tests <what to test>` for test writing.