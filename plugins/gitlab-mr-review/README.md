# gitlab-mr-review

Iterative, multi-agent review of GitLab Merge Requests delivered as copy-paste-ready inline findings. Detects regressions across runs. Integrates optional Jira and Slack MCPs for richer context. Works on any self-hosted GitLab instance.

## Why

Claude Code's built-in `/code-review` covers GitHub but not GitLab (see upstream [issue #26932](https://github.com/anthropics/claude-code/issues/26932)). Existing community plugins each cover slices — this plugin covers the full iterative workflow: context gathering, parallel multi-agent review, dedup against existing threads, regression detection, and a structured report.

## Prerequisites

- `glab` CLI v1.92.1 or newer: https://docs.gitlab.com/cli/
- `glab` authenticated for the target host: `glab auth login --hostname <host>`
- Optional: any Jira MCP server for `[A-Z]{2,}-\d+` ticket enrichment
- Optional: any Slack MCP server for thread-link enrichment

## Install

Via your marketplace of choice, or clone this repository and add it as a local plugin directory per your Claude Code setup.

## Usage

```
/gitlab-mr-review:review <MR-URL> [optional hint]
    [--no-jira] [--no-slack] [--no-external]
    [--no-nits] [--min-severity=must-fix|should-fix|nit]
    [--draft]
    [--save]
    [--gitlab-backend=glab|mcp]
    [--lang=ru|en|auto]
    [--dry-run]
```

### Examples

Review a standard MR:
```
/gitlab-mr-review:review https://gitlab.example.com/group/project/-/merge_requests/205
```

Focus the reviewers on a specific area:
```
/gitlab-mr-review:review https://gitlab.example.com/group/project/-/merge_requests/205 "focus on the cache invalidation flow"
```

Save the report to disk:
```
/gitlab-mr-review:review <URL> --save
```

Skip external services, only use MR diff and threads:
```
/gitlab-mr-review:review <URL> --no-external
```

Dry run (gather context and summarize, but don't review):
```
/gitlab-mr-review:review <URL> --dry-run
```

## How it works

1. **Preflight** (fast, Haiku): parses URL, verifies `glab` is installed and authenticated, checks MR state, detects a recent prior review.
2. **Context gathering** (Sonnet): pulls MR metadata, diff, commits, existing threads; extracts `[A-Z]{2,}-\d+` ticket keys and invokes Jira MCP if present; harvests Slack links; reads CLAUDE.md in touched directories; detects the MR's primary language.
3. **Parallel review** (Opus for bugs/security/architecture/regression, Sonnet for conventions): five built-in reviewers plus configurable optional reviewers, all reading the same diff and context artifact.
4. **Validation**: dedupes findings across reviewers and against existing MR threads; filters to confidence ≥ 80; detects regressions of previously-resolved threads.
5. **Report**: 5-section markdown — MR intent, implementation quality, delta since last review (when applicable), copy-paste-ready findings, meta.

## Iteration

Re-run the command on the same MR after the author pushes changes. The plugin recognizes its prior findings via a hidden HTML-comment footer signature:

```html
<!-- gitlab-mr-review:v1 fid=<hash> run=<timestamp> -->
```

When you paste a finding into GitLab, the footer comes with it. On the next run, the plugin:
- Skips findings already present (and unresolved) — no repetition.
- Flags regressions (thread marked resolved but the issue has returned).
- Shows a delta section with progress since your last review.

## Optional reviewer chain

Create `<CWD>/.claude/gitlab-mr-review.local.md` (YAML frontmatter) to customize the optional reviewer chain:

```yaml
---
optional_reviewers:
  - type: agent
    subagent_type: "go-backend-dev:go-reviewer"
    condition:
      touches_files_matching: ["*.go"]
  - type: agent
    subagent_type: "superpowers:code-reviewer"
    condition:
      always: true
---

# Project-specific review notes

Optional free-form notes that the plugin does not parse.
```

Supported condition keys: `always`, `touches_files_matching`, `command_available`, `tool_available`.

When the file is absent, the plugin applies sensible defaults (go-reviewer for `*.go` files, superpowers:code-reviewer always).

## Non-goals

- Does not post comments to GitLab (read-only).
- Does not resolve threads or reply to other reviewers.
- Does not modify code, create MRs, or approve.
- Does not send notifications.
- Does not work on GitHub (use Claude Code's built-in `/code-review` for GitHub).

## Troubleshooting

### `glab` not installed
Install from https://docs.gitlab.com/cli/ and verify with `glab --version`.

### `glab` unauthenticated for a host
Run `glab auth login --hostname <host>` where `<host>` matches the MR URL.

### Very large diff (>2000 lines or >100 files)
The plugin prompts; you can narrow with `--focus=<path>` (if implemented) or proceed anyway.

### Report too verbose
Use `--min-severity=should-fix` or `--no-nits` to suppress low-severity findings.

### Wrong output language
Override with `--lang=ru` or `--lang=en`.

## License

Same as the parent `ivan-claude-plugins` marketplace.
