---
name: gitlab-mr-context-sources
description: Instructions for extracting and synthesizing MR context from every available source — MR metadata, diff, commits, existing threads, Jira tickets (if MCP present), Slack links (if MCP present), CLAUDE.md files, and linked MRs/issues. Produces the structured mr_intent artifact consumed by every reviewer. Use in the context-gatherer agent to drive per-source extraction logic.
---

# Context Sources

## Pipeline overview

The context-gatherer agent processes sources in a fixed sequential pipeline. Each source populates fields in the `mr_intent` artifact.

| # | Source | How | Skip if |
|---|--------|-----|---------|
| 1 | MR metadata | `glab mr view <iid> --output json` | never |
| 2 | MR diff | `glab mr diff <iid>` | never |
| 3 | MR commits | included in metadata or via dedicated glab call | never |
| 4 | Existing threads | `glab mr view <iid> --comments` with resolution filters | never; empty OK |
| 5 | Jira tickets | Extract `[A-Z]{2,}-\d+` from title ∪ description ∪ branch ∪ commits → Jira MCP | `--no-jira`, no MCP, no matches |
| 6 | Slack links | Extract URLs from description ∪ Jira ∪ commits → Slack MCP | `--no-slack`, no MCP, no matches |
| 7 | CLAUDE.md | Read root + each touched directory | no CLAUDE.md found (logged) |
| 8 | Linked MRs/issues | 1-level lookup for `!<iid>` / `#<id>` references | no references |

## The mr_intent artifact

Produced in-memory by the gatherer and passed to every downstream agent as shared input. The schema below is authoritative — downstream agents parse this exact structure.

```yaml
mr:
  url: "<full URL>"
  host: "<hostname>"
  project: "<group>/<subgroup>/<project>"
  iid: 205
  title: "<MR title>"
  state: "opened"               # opened | closed | merged | draft
  author: "<username>"
  source_branch: "<branch>"
  target_branch: "<branch>"
  labels: ["..."]

intent:
  summary: "One paragraph: what this MR is supposed to do, synthesized from all sources."
  requirements:
    - "R1: ..."
    - "R2: ..."
  non_goals:
    - "..."
  affected_areas:
    - "internal/service/user/..."

sources:
  mr_description:
    content: "<raw>"
    length: 1250
  jira:
    available: true
    tickets:
      - key: "PROJ-123"
        title: "..."
        description: "..."
        acceptance_criteria: ["..."]
        status: "In Progress"
        comments_sampled: 3
  slack:
    available: false
    threads: []
  commits:
    count: 7
    messages: ["..."]
  existing_threads:
    total: 12
    resolved: 7
    unresolved: 5
    by_thread:
      - id: "abc123"
        file: "path/to/file"
        line: 142
        status: "resolved"
        notes: [{ author: "...", body: "...", created_at: "..." }]
        claude_footer: { fid: "bug-001", run: "20260420-140512" }  # parsed if present
  claude_md:
    - path: "CLAUDE.md"
      summary: "<key points, <500 chars>"
    - path: "internal/service/user/CLAUDE.md"
      summary: "..."
  linked:
    mrs: []
    issues: []

language:
  detected: "en"
  confidence: 0.92
  samples:
    - { source: "description", text: "..." }

overall_confidence: "high"    # high | medium | low
overall_confidence_reasons:
  - "Jira ticket present with AC"
  - "Description >500 chars"

user_hint: "<raw text after URL in command, if any>"
```

## Per-source extraction

### 1. MR metadata and description

Invoke `GITLAB_HOST=<host> glab mr view <iid> -R <project> --output json`. Parse the JSON object and populate:

- `mr.url` from `.web_url`
- `mr.host` from URL parsing (not from the JSON — derive from the input URL)
- `mr.project` from URL parsing
- `mr.iid` from `.iid`
- `mr.title` from `.title`
- `mr.state` from `.state` (check `.draft` boolean to override state to `"draft"` if true)
- `mr.author` from `.author.username`
- `mr.source_branch` / `mr.target_branch` from the corresponding fields
- `mr.labels` from `.labels`
- `sources.mr_description.content` from `.description // ""`
- `sources.mr_description.length` as character count of the description

Extract linked references (`!<iid>`, `#<id>`) from the description for source 8. Also extract Jira ticket keys and Slack URLs from the description for sources 5 and 6.

### 2. Unified diff

Invoke `GITLAB_HOST=<host> glab mr diff <iid> -R <project> --raw`. The output is a standard unified diff across all changed files. Parse `diff --git a/<path> b/<path>` headers to enumerate changed file paths — these drive `intent.affected_areas` and CLAUDE.md discovery (source 7). Store the full raw diff string for passing to reviewer agents. Count lines added (`+` prefix) and removed (`-` prefix, excluding `---` headers) to populate the "Files: N changed (+X -Y)" summary in the context print.

### 3. Commits

Invoke `glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/commits --paginate --output ndjson`. Each NDJSON line is a commit object. Extract:

- `sources.commits.count` — total number of commit objects received
- `sources.commits.messages` — array of `.message` strings (full commit messages, not just titles)

Store commit SHAs, author names, and timestamps for use in language detection sampling and Jira/Slack URL extraction. Commits arrive newest-first; reverse for chronological display.

### 4. Existing threads

Invoke `GITLAB_HOST=<host> glab mr note list <iid> -R <project> --output json --state all`. For each discussion object in the returned array:

- Map to `sources.existing_threads.by_thread[]` with: `id`, `file` (from `notes[0].position.new_path` if inline, else null), `line` (from `notes[0].position.new_line`), `status` (`"resolved"` if `notes[0].resolved == true`, else `"unresolved"`), and `notes` array with `author`, `body`, `created_at` from each note.
- Scan `notes[0].body` for the footer pattern `<!-- gitlab-mr-review:v1 fid=<fid> run=<run> -->` using regex. If found, parse and populate `claude_footer: { fid, run }`.

Compute totals: `existing_threads.total`, `existing_threads.resolved`, `existing_threads.unresolved`.

### 5. Jira tickets

Apply the regex `[A-Z]{2,}-\d+` across MR title, MR description, source branch name, and all commit messages. Deduplicate the resulting set of ticket keys. Skip if the user passed `--no-jira` or `--no-external`, or if no Jira MCP tools are detected (see gitlab-mr-backends skill for detection logic).

For each unique key, invoke the matching `get_issue` / `getIssue` / `issue_get` tool. Parse the response to populate `sources.jira.tickets[]` with: `key`, `title`, `description`, `acceptance_criteria` (extracted from the issue body as a list), `status`, and `comments_sampled` (count of comments fetched via `get_comments`). Set `sources.jira.available = true`. Extract any Slack URLs from Jira descriptions for source 6.

### 6. Slack links

Search for Slack URLs matching `https://<workspace>.slack.com/archives/<channel>/p<timestamp>` (and enterprise variants) across MR description, Jira ticket descriptions (from source 5), and commit messages. Collect unique URLs. Skip if `--no-slack` or `--no-external`, or if no Slack MCP tools are detected.

For each URL, extract `channel` and `timestamp` components. Invoke the matching `read_thread` / `get_thread_messages` tool. Map results to `sources.slack.threads[]`. Set `sources.slack.available = true`. If no Slack MCP is found, set `sources.slack.available = false` and `sources.slack.threads = []`.

### 7. CLAUDE.md discovery

Walk the repository to find CLAUDE.md files. Locations to check: the repository root, plus each directory that contains a changed file (derived from the diff in source 2). For example, if `internal/service/user/auth.go` was changed, check for `internal/service/user/CLAUDE.md`.

For each CLAUDE.md found, read its content and summarize to fewer than 500 characters. Append to `sources.claude_md[]` with the file path relative to repo root and the summary. If no CLAUDE.md files are found anywhere, log a note but do not fail.

### 8. Linked MRs/issues

Scan the MR description for cross-reference patterns:
- `!<iid>` — MR in the same project
- `#<id>` — issue in the same project
- `<project>!<iid>` — MR in another project
- `<project>#<id>` — issue in another project

For each unique reference found, perform a one-level lookup using `glab mr view` or `glab issue view` (with `--output json`). Map results to `linked.mrs[]` or `linked.issues[]` respectively with at minimum: `iid`, `title`, `state`, `url`. Handle 404/403 gracefully — surface the reference as-is without failing the review. Do not recurse into linked items' own references (one level only).

## Deriving intent

### Synthesizing summary, requirements, non_goals, affected_areas

The `intent` block is synthesized from all available sources using the following priority order:

- **requirements**: prefer explicit acceptance criteria from Jira tickets (if present) over bullet-list items in the MR description over analysis of commit messages. Label each requirement `R1`, `R2`, etc.
- **non_goals**: explicit "non-goals", "out of scope", or "won't do" statements in the MR description take highest precedence. If none are stated, leave the list empty rather than inferring.
- **summary**: a single paragraph synthesizing purpose from Jira ticket title + description, MR title + description, and key commit messages. Prefer Jira context when available as it typically carries more structured intent.
- **affected_areas**: derived from changed file paths grouped by directory. Use the diff's file list to enumerate top-level packages or modules touched.

### Language detection

Language detection determines the output language for the review. The algorithm:

1. Concatenate samples: MR title + first 500 characters of description + first 5 commit messages + (if Jira present) Jira ticket title + first 300 characters of Jira description.
2. Detect the primary language and compute a confidence score between 0.0 and 1.0, using either a heuristic approach (Cyrillic vs Latin character ratio, common-word scoring for known languages) or a quick LLM judgement call. Implementation chooses the most reliable available method.
3. If confidence is 0.85 or higher: prompt the user with `"Detected primary language: <name>. Proceed in <name>? [Y/n/en/ru]"`.
4. If confidence is below 0.85: prompt with `"Language unclear (<lang1> ~X%, <lang2> ~Y%). Which language for output? [r] [e]"`.
5. Store the user's choice in orchestrator state for the current run. This value is never written to disk — it persists only for the duration of the review session.

Populate `language.detected`, `language.confidence`, and `language.samples` in `mr_intent` from this process.

### Confidence scoring

```
high   = MR description >= 200 chars
         AND at least one of: Jira ticket with body >= 200 chars
                              OR commits (>=3) with descriptive messages (>=20 chars each)
         AND affected_areas identifiable

medium = MR description 50..200 chars
         OR Jira ticket present but description empty
         OR only commit messages to rely on (but meaningful)

low    = MR description <50 chars
         AND no Jira ticket
         AND commits non-descriptive ("wip", "fix", "asdf"-style)
```

Set `overall_confidence` and populate `overall_confidence_reasons` with a list of the matching criteria that determined the level.

## The context summary print

Printed between context gathering and the reviewer fan-out phase:

```
╭────── MR Context Summary ──────────────────────────────╮
│ MR:       <title (truncated)>                          │
│ Files:    N changed (+X -Y)                            │
│ Jira:     <key> (<status>, AC: N criteria)  OR  —      │
│ Slack:    N thread(s)  OR  —                           │
│ Threads:  N total (X resolved, Y unresolved)           │
│ Prior:    Claude reviewed <when>  OR  —                │
│ Lang:     <name> (auto, confidence <n>)                │
│ Conf:     high | medium | low                          │
│                                                        │
│ Intent:   "<mr_intent.summary first 2 lines>"          │
╰────────────────────────────────────────────────────────╯
```

The "Prior: Claude reviewed `<when>`" line is populated from the most recent `run=` timestamp found in any parsed `claude_footer` across existing threads. If no footers are found, display `—`.

## Blocking vs informing

If `overall_confidence == "low"`: the orchestrator blocks before launching the reviewer fan-out and prompts the user: `"Context confidence is low — intent is unclear. Describe what this MR is supposed to do, or press Enter to continue with limited context."` If the user provides a description, it is stored in `user_hint` and used to strengthen `intent.summary`.

If `overall_confidence == "medium"` or `"high"`: the context summary is printed and the review proceeds immediately. The confidence level is surfaced as a self-check item in the Meta section (section 5) of the final report, but it does not block execution.
