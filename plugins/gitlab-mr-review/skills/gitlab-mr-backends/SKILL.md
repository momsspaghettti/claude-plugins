---
name: gitlab-mr-backends
description: Backend selection and MCP detection patterns for the gitlab-mr-review plugin. Defines glab as the sole GitLab backend, agnostic MCP-pattern matching for optional Jira and Slack sources, graceful degradation when MCPs are absent, and parsing of the optional-reviewer chain from .claude/gitlab-mr-review.local.md. Use when any agent needs to decide which backend to call for external data.
---

# Backend Strategy

## GitLab — glab only

`glab` is the sole backend for all GitLab operations. The host is extracted from the MR URL at review time — `gitlab.com` is never hardcoded. All invocations follow the canonical form: `GITLAB_HOST=<host> glab <subcommand> --repo <project> ...` so that the target instance and project are always explicit, independent of the current working directory's git remote.

GitLab MCP tools (tools matching `mcp__*gitlab*__*`) are ignored by default. An opt-in escape hatch `--gitlab-backend=mcp` is recognized but currently unsupported — if it is passed, the plugin must fail fast with a clear error message directing the user back to the glab-based flow.

### Preflight requirements

Before any MR data fetch, the orchestrator must verify both of the following:

1. `command -v glab` — confirms glab is installed and on `PATH`. If this fails, emit an actionable error: "glab is not installed. Install from https://gitlab.com/gitlab-org/cli and ensure it is on PATH."
2. `GITLAB_HOST=<host> glab auth status --hostname <host>` — confirms the stored token for `<host>` is valid. If this fails (exit code 1), emit: "Not authenticated for `<host>`. Run: `glab auth login --hostname <host>`" and abort. Do not attempt to proceed with a potentially unauthenticated session.

Both checks are lightweight network or local operations and must complete before any substantive glab call is made.

### Escape hatch

The flag `--gitlab-backend=mcp` is opt-in and currently unsupported. If a caller passes it, the plugin must reject it immediately with a message such as: "GitLab MCP backend is not yet supported. Remove `--gitlab-backend=mcp` to use the default glab backend." This prevents silent fallthrough to undefined behavior.

## Jira — optional, agnostic detection

### Tool-pattern matching

Jira integration is detected at runtime by scanning the set of available tool names. The plugin matches tools using the following patterns (glob-style):

- `mcp__*jira*__*`
- `mcp__*atlassian*__*`

If any tool name matches either pattern, a Jira MCP is considered present. No server name is hardcoded — the detection is purely pattern-based against whatever tools the runtime exposes.

### Operations of interest

When a Jira MCP is detected, the relevant operations to invoke are:

- **Fetch issue**: look for tool names matching `get_issue`, `getIssue`, or `issue_get` — call with the ticket key (e.g., `PROJ-123`).
- **Fetch comments**: look for tool names matching `get_comments` — call with the same key to retrieve discussion context.

Use the first matching tool name found for each operation. Do not assume a fixed tool name.

### Ticket extraction

Extract Jira ticket keys using the regex `[A-Z]{2,}-\d+`. Search scope is the union of: MR title, MR description, source branch name, and all commit messages. Collect all matches and deduplicate keys before fetching. This ensures that even tickets mentioned only in a commit subject or a branch name like `feature/PROJ-42-add-login` are captured.

### Graceful degradation

If no Jira MCP tools are found, set `sources.jira = "not_available"` in `mr_intent` and continue silently. Never prompt the user to manually provide Jira data unless a future `--context-jira` flag is explicitly added. If the user passed `--no-jira` or `--no-external`, skip Jira entirely regardless of MCP availability.

## Slack — optional, agnostic detection

### Tool-pattern matching

Slack integration is detected at runtime by scanning available tool names for the pattern `mcp__*slack*__*`. If any match is found, a Slack MCP is considered present. No specific server name is assumed.

### Slack URL patterns

Harvest Slack links matching the following URL forms:

- Standard: `https://<workspace>.slack.com/archives/<channel>/p<timestamp>`
- Enterprise: `https://<workspace>.enterprise.slack.com/archives/<channel>/p<timestamp>` (or similar enterprise variants)

The `<timestamp>` component in Slack URLs uses a packed decimal format (e.g., `p1714000000123456`). When invoking Slack MCP tools, the channel and timestamp must be extracted from the URL.

### Harvest scope

Search for Slack URLs in: MR description, Jira ticket descriptions (if Jira was fetched), and commit messages. Collect all unique URLs found across all sources before fetching threads.

### Operations of interest

When a Slack MCP is present, invoke tools matching:

- `read_thread` / `get_thread_messages` — retrieve full thread given channel and thread timestamp.
- `get_message` — retrieve a single message given channel and timestamp, used when only a single message URL is referenced rather than a thread.

Use the first matching tool name found for each operation type.

### Graceful degradation

If no Slack MCP tools are found, set `sources.slack = "not_available"` in `mr_intent` and continue. Never block the review on the absence of Slack context. If the user passed `--no-slack` or `--no-external`, skip Slack entirely.

## Optional reviewer chain

### Parse `<CWD>/.claude/gitlab-mr-review.local.md` YAML frontmatter

At the start of the review, the orchestrator checks for the file `<CWD>/.claude/gitlab-mr-review.local.md`. If it exists, parse its YAML frontmatter to extract `optional_reviewers`. If it does not exist, apply plugin defaults (see below). Errors in parsing (malformed YAML) should be reported as a warning, and defaults applied.

### Supported condition keys

| Key | Value type | Meaning |
|-----|-----------|---------|
| `always` | boolean | Always include when `true` |
| `touches_files_matching` | list of globs | Include only if any changed file matches any glob |
| `command_available` | string | Include only if `command -v <name>` succeeds |
| `tool_available` | boolean or string | Include only if the MCP tool matching `tool_pattern` is present |

Conditions are evaluated against the current MR's changed files and available runtime tools. A reviewer entry is included in the fan-out only if its condition evaluates to true.

### Default chain

When `<CWD>/.claude/gitlab-mr-review.local.md` is absent, the following defaults apply:

```yaml
optional_reviewers:
  - type: agent
    subagent_type: "go-backend-dev:go-reviewer"
    condition:
      touches_files_matching: ["*.go"]
  - type: agent
    subagent_type: "superpowers:code-reviewer"
    condition:
      always: true
```

The `go-backend-dev:go-reviewer` agent is included only when the MR touches `.go` files. The `superpowers:code-reviewer` agent is always included as a baseline reviewer.

### Invocation mechanics

Depending on the `type` field of each reviewer entry:

- **`agent`**: invoke via the Agent tool, passing `subagent_type` as the agent identifier. The agent receives the same `mr_intent` and diff context as built-in reviewers.
- **`cli`**: invoke via Bash, running the specified `command`. Parse stdout as JSON findings in the same schema as built-in reviewer output.
- **`mcp`**: match the `tool_pattern` against available tool names at runtime; invoke the first matching tool. If no tool matches, skip this reviewer (equivalent to graceful degradation — do not fail).

All optional reviewer findings flow through the same validator (dedup + confidence threshold) as built-in reviewers.

## Principles

- Never hardcode gitlab.com.
- Never block the review on a missing MCP — degrade gracefully.
- User control via flags — honor `--no-jira`, `--no-slack`, `--no-external`, `--gitlab-backend`.
- MCP detection is runtime, by tool-name pattern; never by hardcoded server name.
