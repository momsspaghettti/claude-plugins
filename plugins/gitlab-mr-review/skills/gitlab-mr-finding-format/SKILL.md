---
name: gitlab-mr-finding-format
description: Exact schema for review findings, copy-paste-ready GitLab markdown rendering rules, severity taxonomy, suggestion block conditions, stable finding-ID derivation, GitLab permalink construction, and the 5-section report structure. Use in reviewer, validator, and report-composer agents to produce consistent outputs that paste cleanly into GitLab inline comments.
---

# Finding Format

## Severity ladder

| Level | Meaning | Inclusion threshold |
|-------|---------|---------------------|
| `must-fix` | Bug, security hole, data corruption, broken contract, violates stated requirement | always |
| `should-fix` | Architecture smell, maintainability, missing test, subtle correctness | confidence >= 80 |
| `nit` | Style, naming, minor readability | confidence >= 80 AND `--no-nits` not set |

Command flag `--min-severity=must-fix|should-fix|nit` raises the bar. When `--min-severity=must-fix`, only `must-fix` findings are included. When `--min-severity=should-fix`, `must-fix` and `should-fix` are included; `nit` is suppressed.

## Finding JSON schema

Each reviewer outputs findings as a JSON object. The schema is:

```json
{
  "id": "<stable-fid>",
  "focus": "<agent name>",
  "severity": "must-fix | should-fix | nit",
  "file": "relative/path/from/repo/root",
  "line_start": 87,
  "line_end": 94,
  "column_hint": null,
  "title": "<one-line summary>",
  "body": "<markdown paragraph(s) explaining the issue and why>",
  "suggestion": "<optional code replacement>",
  "suggestion_fits_inline": true,
  "relates_to_requirement": "<requirement id> | null",
  "regression_of_thread_id": "<thread id> | null",
  "confidence": 95
}
```

Fields explained:

- `id` — stable finding identifier (see derivation below). Used by the dedup algorithm to match prior Claude findings across runs.
- `focus` — the name of the reviewer agent that produced this finding (e.g., `bugs`, `security`, `architecture`).
- `severity` — one of the three levels defined in the severity ladder.
- `file` — path relative to repository root (not prefixed with `a/` or `b/`).
- `line_start` / `line_end` — 1-based line numbers of the code range the finding addresses.
- `column_hint` — optional column number; `null` for most findings.
- `title` — one-line summary suitable for use as a GitLab comment heading.
- `body` — markdown prose explaining the issue, its impact, and why it matters.
- `suggestion` — optional code replacement text. If non-empty and `suggestion_fits_inline` is true, rendered as a GitLab suggestion block.
- `suggestion_fits_inline` — true only when the suggestion replaces exactly `line_start..line_end` and the block semantics are valid in GitLab.
- `relates_to_requirement` — requirement ID string (e.g., `R2`) if the finding directly relates to a stated requirement; `null` otherwise.
- `regression_of_thread_id` — GitLab discussion ID of the thread where this concern was previously raised and resolved; `null` for new findings.
- `confidence` — integer 0–100 representing reviewer confidence in the finding.

### Stable fid derivation

The `id` field is a deterministic, stable finding identifier computed as follows:

1. Normalize `title_keywords`: lowercase the title, strip all non-alphanumeric characters, take the first 5 words, join with a single space.
2. Concatenate components with pipe separator: `<focus>|<file>|<line_start>|<title_keywords>`
3. Compute the SHA-1 hash of the concatenated string (UTF-8 encoded).
4. Use the first 8 hex characters as the `fid`.

Worked example:

- Input: `focus="bugs"`, `file="internal/auth/handler.go"`, `line_start=87`, `title="Nil pointer dereference when user is missing"`
- `title_keywords` = `"nil pointer dereference when user"` (first 5 words after lowercasing and stripping non-alphanumeric)
- Concatenated string: `"bugs|internal/auth/handler.go|87|nil pointer dereference when user"`
- SHA-1: `a3f2c1b8...` → `fid = "a3f2c1b8"`

This algorithm ensures that the same finding at the same location always gets the same `fid`, even across different runs, enabling exact-match dedup.

## Copy-paste-ready GitLab markdown

The following template produces a markdown block suitable for pasting as a GitLab inline comment. Each block corresponds to a single finding.

```markdown
**[<severity>] <title>**

<body paragraph(s)>

Relates to requirement: *<requirement text>*                   # if non-null

```suggestion:-0+<line_count>
<suggestion lines>
```

<!-- gitlab-mr-review:v1 fid=<fid> run=<YYYYMMDD-HHMMSS> -->
```

The `run` timestamp in the footer uses UTC time formatted as `YYYYMMDD-HHMMSS` (e.g., `20260423-091530`).

### Conditional blocks

#### Suggestion block

Emit a suggestion fenced block (` ```suggestion:-0+<line_count>`) only when ALL of the following are true:

1. `suggestion` field is non-empty.
2. `suggestion_fits_inline` is `true`.
3. The suggestion replaces exactly the range `line_start..line_end`.

When these conditions are met, `line_count` is `line_end - line_start + 1`. GitLab will render the suggestion as an inline diff with an "Apply suggestion" button.

If any condition is not met (suggestion is empty, does not fit inline, or spans a different range), render the suggestion as a regular fenced code block using the file's language identifier (e.g., ` ```go`) without suggestion semantics.

#### Requirement link

If `relates_to_requirement` is non-null, emit a single italic line immediately after the body paragraphs:

```
Relates to requirement: *<requirement text>*
```

Where `<requirement text>` is the full requirement string from `intent.requirements` identified by the requirement ID. Omit this line entirely if `relates_to_requirement` is `null`.

#### Regression notice

If `regression_of_thread_id` is non-null, prepend a single line before the severity/title heading:

```
⚠️ *Regression: previously raised and resolved in [this thread](<url>) — now reappearing.*
```

Where `<url>` is the GitLab permalink to the original thread. This line must be the first line of the comment block.

#### Footer signature

Always emit as the last line of every finding comment:

```
<!-- gitlab-mr-review:v1 fid=<fid> run=<YYYYMMDD-HHMMSS> -->
```

GitLab preserves this HTML comment in the note body but does not render it visibly. The plugin scans for this footer on subsequent runs to identify its own prior findings. It must be present on every finding regardless of severity, suggestion presence, or any other condition.

## GitLab permalinks

Permalink format for anchoring a finding to a specific line range in the codebase at the MR's head commit:

```
https://<host>/<project>/-/blob/<head_sha>/<file-path>#L<start>-<end>
```

Where:
- `<host>` and `<project>` are extracted from the MR URL.
- `<head_sha>` is obtained from `glab mr view --output json | jq -r '.diff_refs.head_sha'` (or `.sha`).
- `<file-path>` is the relative file path from the repository root.
- `L<start>-<end>` uses 1-based line numbers (e.g., `L87-L94` or `L87` for a single line).

These permalinks are used in the terminal card for the `📎 <file>:<line>` clickable reference and optionally in finding bodies for "see also" cross-references.

## Terminal card rendering

Each finding is displayed in the terminal using the following compact card format, followed by a `---` separator and a fenced region containing the copy-paste markdown block:

```
╭─ [N/TOTAL] <severity> · <focus> · confidence <n>
│ 📄 <file>:<line_start>-<line_end>
│ <title>
│ <body first 3 lines>
│ → relates to <requirement id> | regression of thread #<id>
│ 💡 suggestion available (<N> lines, fits inline | no)
╰──────────────────────────────────────────────────────
```

The `→ relates to` and `💡 suggestion available` lines are omitted when not applicable. The copy-paste markdown block is rendered in a fenced region immediately after `╰──` so the user can select it cleanly.

## Full report skeleton

```
# MR Review — <host>/<project>!<iid>

> <MR title>

## 1. What this MR does
<mr_intent.summary as prose>
Declared requirements: R1, R2, R3.
Non-goals: ...

## 2. Implementation quality
Verdict: partial | complete | over-scoped | off-target

Requirements coverage:
  R1 ✅ covered
  R2 ⚠️ partial — <reason>
  R3 ❌ not addressed

Architectural observations:
- <bullet>
- <bullet>

## 3. Delta since last review       # only if prior Claude review detected
Previous review: <timestamp>
New commits since then: <n>
Previous findings status: ✅ N resolved, ❌ N regressing, 🔄 N carried over, ➡️ N no longer applicable
New findings this run: <n>

## 4. Findings to post
> Copy each block below as a separate inline comment on the indicated file:line.

### Finding 1 · <severity> · <focus>
📎 `<file>` **L<start>–L<end>**

<copy-paste-ready markdown block>

---

### Finding 2 · ...

## 5. Meta
- Backend: glab v<version>
- Context: MR + <Jira keys> + CLAUDE.md × N
- Language: <name> (auto, confidence <n>)
- Reviewers: <comma list>
- Optional chain: <comma list>
- Dropped: N findings below confidence threshold
- Skipped: N findings already raised in threads #<id>, #<id>
- Runtime: <mm:ss>
```

Section 3 (Delta) is only emitted when at least one prior Claude finding footer was detected in `existing_threads`. If no footers were found, omit the section entirely.

## Output destinations

Terminal output is always produced regardless of flags. The full report (sections 1–5) is printed to stdout with formatting suitable for terminal rendering.

When the user passes `--save`, the same full report is also written to a file:

```
<CWD>/.claude/reviews/<project-slug>-<iid>-<YYYYMMDD-HHMMSS>.md
```

Where `<project-slug>` is the project path with forward slashes replaced by hyphens (e.g., `group/subgroup/repo` becomes `group-subgroup-repo`), `<iid>` is the MR's internal ID, and the timestamp is UTC in `YYYYMMDD-HHMMSS` format.

Ensure the `.claude/reviews/` directory exists before writing; create it if absent. Terminal output is identical in both cases — `--save` only adds the file write.
