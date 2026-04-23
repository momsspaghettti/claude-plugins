---
description: Review a GitLab Merge Request with multi-agent analysis, copy-paste-ready findings, and iterative regression detection across runs
argument-hint: <MR-URL> [optional hint] [--no-jira] [--no-slack] [--no-external] [--no-nits] [--min-severity=...] [--draft] [--save] [--gitlab-backend=glab|mcp] [--lang=ru|en|auto] [--dry-run]
---

# GitLab MR Review

You are orchestrating a read-only review of a GitLab Merge Request. The user
has invoked `/gitlab-mr-review:review` with the arguments below. Your job is
to run the 5-phase flow end-to-end, print progress, honor all interactive
gates, and deliver the final report.

**Arguments:** $ARGUMENTS

## Phase 0 — Argument parsing

Parse `$ARGUMENTS`:
- First token is the MR URL (required).
- Subsequent non-flag tokens concatenate into `user_hint`.
- Flags (prefix `--`):
  - `--no-jira`, `--no-slack`, `--no-external`, `--no-nits`, `--draft`, `--save`, `--dry-run` — boolean.
  - `--min-severity=<X>` — one of `must-fix`, `should-fix`, `nit`. Default `nit`.
  - `--gitlab-backend=<X>` — one of `glab`, `mcp`. Default `glab`.
  - `--lang=<X>` — one of `ru`, `en`, `auto`. Default `auto`.

If URL is missing or malformed, print an error with the expected format and exit.

Build a `flags` object. Announce: print a two-line banner summarizing which MR
is being reviewed and the active flag deltas from defaults.

## Phase 1 — Preflight

Dispatch the `mr-preflight` agent with `{ url, flags }`. Parse its JSON response.

- If `status == "error"`: print the message, exit.
- If `status == "gate"`: prompt the user per the gate message; abort or
  continue based on the answer.
- If `status == "early_exit"`: print the message, exit 0.
- If `status == "proceed"`: extract `parsed` and `prior_run`, continue.

Print: `🔎 [1/5] Preflight ✓` with brief details (host, project, iid, state).

## Phase 2 — Context gathering

Dispatch the `mr-context-gatherer` agent with `{ parsed, flags, prior_run, user_hint }`.
The agent returns the `mr_intent` artifact.

Print the context summary box (format per gitlab-mr-context-sources skill).

### Language confirmation

Based on `mr_intent.language`:
- If `--lang=ru` or `--lang=en` in flags — honor directly, skip prompt.
- Else if `confidence >= 0.85` — print `📋 Detected primary language: <name>. Proceed in <name>? [Y/n/en/ru]` and accept user input. Default Y on Enter.
- Else — print `📋 Language unclear (<A>~X%, <B>~Y%). Which language for output? [r] [e]` and require a choice.

Store the confirmed language in `output_language`.

### Low-confidence block

If `mr_intent.overall_confidence == "low"`:
- Print a block: `⚠️ Context is thin ... describe what this MR should do, or press Enter to continue with limited context:`
- Accept user input. If user types text, inject as `user_hint += "\n" + text` and proceed. If Enter alone, continue.

### Dry-run exit

If `--dry-run` in flags, print `🛑 Dry run — stopping after context summary.` and exit 0.

Print: `📚 [2/5] Context gathered ✓`.

## Phase 3 — Parallel reviewer fan-out

Dispatch the following agents in a single message with multiple Agent calls
(parallel execution per superpowers:dispatching-parallel-agents). Each
receives the shared input contract:

```
CONTEXT (authoritative):
<entire mr_intent YAML>

DIFF (the code to review):
<full unified diff>

YOUR FOCUS: <bugs | security | architecture | conventions | regression>

INSTRUCTIONS:
- Review the DIFF against CONTEXT.intent.requirements.
- If code contradicts CONTEXT.intent.non_goals, flag with high severity.
- Consider CONTEXT.sources.existing_threads — do NOT repeat findings that
  were already raised, unless a regression is detected.
- Every finding must have: focus, severity (must-fix|should-fix|nit), file,
  line_start, line_end, title, body, optional suggestion, optional
  relates_to_requirement, optional regression_of_thread_id, confidence 0-100.
- Output JSON: { "findings": [ ... ] }.
```

Built-in (always):
- `mr-bug-hunter`
- `mr-security-reviewer`
- `mr-architecture-reviewer`
- `mr-conventions-reviewer`
- `mr-regression-detector`

Optional chain — consult the gitlab-mr-backends skill to:
1. Read `<CWD>/.claude/gitlab-mr-review.local.md` if present; else use defaults.
2. For each configured `optional_reviewer`, evaluate its condition:
   - `always: true` → include.
   - `touches_files_matching: [globs]` → include if any diff file matches.
   - `command_available: <name>` → include if `command -v <name>` succeeds.
   - `tool_available: true` → include if any tool matches `tool_pattern`.
3. Dispatch each included optional reviewer with the same shared input.

Collect all `findings` arrays from every reviewer into `raw_findings`.

Print per-reviewer `✓ <name> (<count> raw findings)` as each completes.

## Phase 4 — Validation

Dispatch `mr-finding-validator` with `{ mr_intent, raw_findings, flags }`.
Parse its JSON response into `final_findings`, `dropped`, `dedup_summary`.

Print: `🔬 [4/5] Validated — <final_count> findings after filtering`.

## Phase 5 — Report composition

Dispatch `mr-report-composer` with:

```
{
  mr_intent,
  validator_output: { final_findings, dropped, dedup_summary },
  flags,
  language: output_language,
  runtime_stats: { started_at, completed_at, reviewers_run, optional_chain_run }
}
```

The composer emits the final report markdown directly to stdout. Capture it
for printing. If `--save` was passed, the composer also writes the file and
returns the written path; print the save confirmation.

Print: `📝 [5/5] Report composed ✓` followed by the report.

## Error handling

- If any single built-in reviewer fails, log the error, continue without it.
- If ≥2 built-in reviewers fail, abort with an error summary.
- If the optional chain's reviewer fails, log and continue.
- On Ctrl+C, exit cleanly — if `--save`, preserve partial context summary to
  the save location with a `.partial.md` suffix.

## Skills to consult

Every phase must consult the following skills as authoritative:
- `glab-mr-usage` — for every glab invocation (delegated to sub-agents).
- `gitlab-mr-backends` — for MCP detection and optional-chain evaluation.
- `gitlab-mr-context-sources` — for context structure and language detection.
- `gitlab-mr-finding-format` — for finding schema and markdown rules.
- `gitlab-mr-iteration` — for dedup and regression logic.

## Principles

- Fail fast on preflight errors with actionable instructions.
- Honor every flag exactly per spec §7.
- Never post to GitLab. Never resolve threads. Never modify source code.
- Preserve user control at every interactive gate.
