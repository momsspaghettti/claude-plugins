---
name: glab-mr-usage
description: Comprehensive reference for using the glab CLI in read-only MR review workflows. Covers URL parsing, host resolution, auth preflight, reading MR metadata/diff/commits/threads/pipelines/approvals, JSON output schemas, pagination, and failure modes. Use when any agent needs to invoke glab against a GitLab Merge Request.
---

# glab CLI Usage for Read-Only MR Review

Single source of truth for invoking `glab` (v1.92.1) to review GitLab Merge
Requests across any self-hosted GitLab instance. All operations documented
here are READ-ONLY. Write commands are listed in Appendix A only so callers
recognize and avoid them.

## Table of Contents

1. [URL Parsing](#1-url-parsing)
2. [Host Resolution](#2-host-resolution)
3. [Auth Preflight](#3-auth-preflight)
4. [Reading MR Metadata](#4-reading-mr-metadata)
5. [Reading MR Description / Body](#5-reading-mr-description--body)
6. [Fetching the Unified Diff](#6-fetching-the-unified-diff)
7. [Listing MR Commits](#7-listing-mr-commits)
8. [Reading ALL Discussions / Threads](#8-reading-all-discussions--threads)
9. [Filtering Unresolved / Resolved Threads](#9-filtering-unresolved--resolved-threads)
10. [Reading MR Pipeline Status](#10-reading-mr-pipeline-status)
11. [Reading Approvals](#11-reading-approvals)
12. [Cross-Repo References](#12-cross-repo-references)
13. [Handling Special States](#13-handling-special-states)
14. [JSON Output Mode — Coverage Table](#14-json-output-mode--coverage-table)
15. [Rate Limits, Timeouts, Retries](#15-rate-limits-timeouts-retries)
16. [Pagination](#16-pagination)
17. [Appendix A — Write Commands (Not Used)](#appendix-a--write-commands-not-used)

---

## Conventions used in this document

Placeholders you will see throughout:

| Placeholder   | Meaning                                                           |
|---------------|-------------------------------------------------------------------|
| `<host>`      | GitLab hostname, e.g. `gitlab.com` or `gitlab.example.org`        |
| `<project>`   | Full project path, e.g. `group/subgroup/repo`                     |
| `<project-enc>` | URL-encoded `<project>`, i.e. slashes replaced with `%2F`       |
| `<iid>`       | Internal ID of the MR inside the project (the number in the URL) |
| `<group>`     | Top-level group                                                   |
| `<id>`        | Numeric global ID (issues, notes, discussions)                    |
| `<sha>`       | Commit SHA                                                        |

Golden rule for CLI calls: every glab command that targets a project must
carry both `--repo <project>` (or full URL) AND a host hint
(`GITLAB_HOST=<host>` or `--hostname <host>` where supported).
Do not rely on the current working directory unless the caller has explicitly
verified that the CWD git remote matches the MR's project.

---

## 1. URL Parsing

GitLab MR URLs have the canonical form:

```
https://<host>/<group>[/<subgroup>...]/<project>/-/merge_requests/<iid>
```

Examples:

```
https://gitlab.com/gitlab-org/cli/-/merge_requests/1234
https://gitlab.example.org/acme/backend/payments-service/-/merge_requests/42
https://gitlab.example.org/acme/platform/subteam/service/-/merge_requests/7
```

### Regex approach (pure text, no network)

A single Perl-compatible regex covers all nesting depths because the project
path is everything between the host and the literal `/-/merge_requests/`
sentinel:

```
^https?://(?P<host>[^/]+)/(?P<project>.+?)/-/merge_requests/(?P<iid>\d+)(?:[/?#].*)?$
```

Extraction rules:

- **host**: everything after `://` up to the first `/`. Lowercase it.
- **project**: everything between the host and the `/-/merge_requests/`
  sentinel. It is the full path (group + subgroups + repo), may contain `/`.
- **iid**: the run of digits immediately after `/-/merge_requests/`. Stop at
  the first non-digit — trailing `/diffs`, `?tab=...`, `#note_123` are not
  part of the iid.

### Shell one-liner (bash / zsh, POSIX ERE fallback)

```bash
url='https://gitlab.example.org/acme/backend/payments-service/-/merge_requests/42#note_9'
if [[ "$url" =~ ^https?://([^/]+)/(.+)/-/merge_requests/([0-9]+) ]]; then
  host="${BASH_REMATCH[1]}"
  project="${BASH_REMATCH[2]}"
  iid="${BASH_REMATCH[3]}"
fi
```

### Gotchas

- **Do NOT split on `/-/`** — paths may legitimately contain `-` segments.
  Always anchor on the literal `/-/merge_requests/`.
- **Trailing path segments**: `/diffs`, `/commits`, `/pipelines` after the iid
  are common. The regex above stops at `\d+` so they are discarded.
- **Query string and fragment**: e.g. `?tab=discussion`, `#note_456`. Strip
  after the iid.
- **`/merge_requests/new`**: not a review target. If `<iid>` is not purely
  numeric, reject the URL.
- **Port in host**: `gitlab.example.org:8443` — keep the port as part of the
  host string; glab accepts `host:port` in `GITLAB_HOST` and `--hostname`.
- **URL encoding**: in URLs in the wild, `<project>` has raw slashes. When
  passing to the GitLab REST API (section 12), you must URL-encode to
  `<project-enc>` (`/` → `%2F`). glab's own `--repo` flag accepts the raw form.
- **Case**: hostnames are case-insensitive, project paths are case-sensitive
  on most GitLab instances. Lowercase the host; preserve project path verbatim.

---

## 2. Host Resolution

glab targets multiple self-hosted instances. For a review tool that must work
against any host on demand, resolve the host explicitly per invocation — do
not rely on the user's current repo context.

### Precedence (observed)

glab determines the target host in this order:

1. Explicit flag: `--hostname <host>` (supported by `glab api`, `glab auth status`)
2. Environment variable: `GITLAB_HOST=<host>`
3. Per-repo config: hostname inferred from the git remote of the current
   directory, if CWD is inside a git repo
4. Global config: `~/.config/glab-cli/config.yml` `host:` key
5. Default: `gitlab.com`

Authentication tokens are stored per-host. Setting `GITLAB_HOST=<host>` also
selects the corresponding stored token.

### Recommended pattern — one-off command against a non-default host

Always export `GITLAB_HOST` for the subshell, and pass `--repo <project>` so glab
does not try to infer project from CWD:

```bash
GITLAB_HOST=<host> glab mr view <iid> --repo <project> --output json
```

This works for every `glab mr …`, `glab ci …`, `glab issue …` invocation.
The pair `GITLAB_HOST` + `--repo <project>` is the canonical form for this plugin.

### For `glab api` (REST / GraphQL)

`glab api` honors `--hostname` directly, which is slightly more explicit than
the env var and avoids any risk of leaking into child processes:

```bash
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>
```

Either form works; prefer `--hostname` for `glab api` calls, and
`GITLAB_HOST` for `glab mr …` style calls.

### Global token precedence

If `GITLAB_TOKEN`, `GITLAB_ACCESS_TOKEN`, or `OAUTH_TOKEN` are set in the
environment, they override the stored token for whichever host matches
`GITLAB_HOST`. For a multi-host review workflow this is usually undesirable —
unset them unless the caller specifically wants to inject a token.

### Inspecting what glab will use

```bash
# Show token and host currently configured (per-host)
glab config get host
glab config get --host <host> token   # per-host override
```

### Gotchas

- `GITLAB_HOST` value is bare host — `gitlab.example.org`, not
  `https://gitlab.example.org`. Protocol is derived from the stored per-host
  `api_protocol` (default `https`).
- A CWD with a GitLab git remote will override `GITLAB_HOST` for some
  commands if `--repo` is not provided. **Always pass `--repo <project>` for this
  plugin** to make the target explicit.
- `--repo` accepts `OWNER/REPO`, `GROUP/NAMESPACE/REPO`, full URL, or
  Git URL. Full HTTPS URLs are the safest unambiguous form:
  `--repo https://<host>/<project>.git` or simply the project path string.

---

## 3. Auth Preflight

Before any MR operation, assert that glab is authenticated for the target host.

### Minimal assertion

```bash
GITLAB_HOST=<host> glab auth status --hostname <host>
```

`glab auth status` flags:

- `--hostname <host>` — check a specific instance
- `-a, --all` — check every configured instance
- `-t, --show-token` — include the token in output (avoid in logs)

### Expected output — authenticated

Exit code `0`. Output is human-readable (no JSON mode); parse with string
matching. Example shape:

```
<host>
  ✓ Logged in to <host> as <username> (<source>)
  ✓ Git operations for <host> configured to use ssh protocol.
  ✓ API calls for <host> are made over https protocol.
  ✓ REST API Endpoint: https://<host>/api/v4/
  ✓ GraphQL Endpoint: https://<host>/api/graphql/
  ✓ Token: <redacted unless --show-token>
```

### Expected output — NOT authenticated

Exit code `1`. Output contains one or more of:

- `401 Unauthorized` on the `GET /api/v4/user` line
- `No token found (checked config file, keyring, and environment variables).`
- A final `ERROR … could not authenticate to one or more of the configured GitLab instances..`

Real captured example against an unauthenticated host:

```
<host>
  x <host>: API call failed: GET https://<host>/api/v4/user: 401 {message: 401 Unauthorized}
  ✓ Git operations for <host> configured to use ssh protocol.
  ✓ API calls for <host> are made over https protocol.
  ✓ REST API Endpoint: https://<host>/api/v4/
  ✓ GraphQL Endpoint: https://<host>/api/graphql/
  ! No token found (checked config file, keyring, and environment variables).

 ERROR
  X could not authenticate to one or more of the configured GitLab instances..
```

### Programmatic check

```bash
if ! GITLAB_HOST=<host> glab auth status --hostname <host> >/dev/null 2>&1; then
  echo "Not authenticated for <host>. Run: glab auth login --hostname <host>" >&2
  exit 2
fi
```

### Exit codes

- `0`: authenticated, token valid
- `1`: any failure (no token, token invalid, host unreachable, 401)

glab does not distinguish these via exit code — differentiate from stderr
text if needed (`grep -q "No token found"` vs `grep -q "401"`).

### Stronger assertion — token actually works

`auth status` verifies `GET /user`. A more thorough check that scopes are
sufficient for project reads:

```bash
glab api --hostname <host> projects/<project-enc> --silent
```

Exit 0 means the token can read the project.

### Gotchas

- `auth status` contacts the server — it is not a local-only check. Treat it
  as a network call.
- If `GITLAB_TOKEN` is set in the environment, `auth status` uses it and
  ignores the stored token for the host. Unset it to test stored credentials.

---

## 4. Reading MR Metadata

Primary command:

```bash
GITLAB_HOST=<host> glab mr view <iid> --repo <project> --output json
```

`glab mr view` flags of interest:

- `-F, --output text|json` — default `text`. Use `json` for machine parsing.
- `-c, --comments` — also fetch comments/activities (covered in section 8)
- `--resolved` / `--unresolved` — filter comments; implies `--comments`
- `-s, --system-logs` — include system activities

### JSON shape returned by `glab mr view --output json`

A single JSON object representing the GitLab MR. Key fields (non-exhaustive,
stable subset):

```json
{
  "id": 123456789,
  "iid": 42,
  "project_id": 1111,
  "title": "Add feature X",
  "description": "Full markdown body …",
  "state": "opened",
  "created_at": "2026-04-20T10:11:12.000Z",
  "updated_at": "2026-04-22T08:00:00.000Z",
  "merged_at": null,
  "closed_at": null,
  "target_branch": "main",
  "source_branch": "feature/x",
  "source_project_id": 1111,
  "target_project_id": 1111,
  "draft": false,
  "work_in_progress": false,
  "merge_status": "can_be_merged",
  "detailed_merge_status": "mergeable",
  "sha": "<sha>",
  "merge_commit_sha": null,
  "squash_commit_sha": null,
  "web_url": "https://<host>/<project>/-/merge_requests/42",
  "labels": ["backend", "needs-review"],
  "milestone": { "id": 10, "iid": 3, "title": "Sprint 42" },
  "author":   { "id": 7,  "username": "alice",  "name": "Alice"  },
  "assignees":[{ "id": 8,  "username": "bob",    "name": "Bob"    }],
  "reviewers":[{ "id": 9,  "username": "carol",  "name": "Carol"  }],
  "user_notes_count": 12,
  "upvotes": 1,
  "downvotes": 0,
  "has_conflicts": false,
  "blocking_discussions_resolved": true,
  "pipeline":       { "id": 987, "status": "success", "web_url": "…" },
  "head_pipeline":  { "id": 987, "status": "success", "sha": "<sha>" },
  "diff_refs":      { "base_sha": "<sha>", "head_sha": "<sha>", "start_sha": "<sha>" },
  "references":     { "short": "!42", "relative": "<project>!42", "full": "<project>!42" },
  "task_completion_status": { "count": 5, "completed_count": 3 }
}
```

Draft detection (see section 13 for detail):

- `draft` boolean (modern field) — `true` for drafts.
- `work_in_progress` boolean (legacy alias) — same value, kept for back-compat.
- Title also typically starts with `Draft:` — do not rely on title alone.

### jq recipes

```bash
# Title + state + draft flag
glab mr view <iid> --repo <project> --output json \
  | jq -r '[.iid, .state, (.draft|tostring), .title] | @tsv'

# Is this a draft?
glab mr view <iid> --repo <project> --output json | jq -r '.draft'

# Labels, comma-joined
glab mr view <iid> --repo <project> --output json | jq -r '.labels | join(",")'

# Source/target branches
glab mr view <iid> --repo <project> --output json | jq -r '.source_branch + " -> " + .target_branch'

# Reviewer usernames
glab mr view <iid> --repo <project> --output json | jq -r '.reviewers[].username'

# Approval status at a glance (approvals list: see section 11)
glab mr view <iid> --repo <project> --output json | jq '.blocking_discussions_resolved, .detailed_merge_status'
```

### Gotchas

- `state` values: `opened`, `closed`, `merged`, `locked`. It does NOT include
  `draft` as a state — draft is a separate boolean on an otherwise `opened` MR.
- `milestone`, `merged_at`, `closed_at`, `merge_commit_sha` may be `null`.
- `pipeline` field reflects the MR's *head pipeline* at view time and may be
  stale relative to the freshest pipeline — prefer `head_pipeline` or query
  pipelines directly (section 10).
- When the MR is cross-project (fork → upstream), `source_project_id` differs
  from `target_project_id`; the diff commands below still work.

---

## 5. Reading MR Description / Body

The description is the `description` field of the metadata JSON — there is
no separate "get body" command. Prefer extracting from `glab mr view --output json`
to avoid HTML rendering or ANSI color artifacts from the text view.

```bash
GITLAB_HOST=<host> glab mr view <iid> --repo <project> --output json \
  | jq -r '.description'
```

Human-readable variant (rendered markdown with Glamour):

```bash
GITLAB_HOST=<host> glab mr view <iid> --repo <project>
```

The description can contain:

- Markdown task lists: `- [ ] todo`, `- [x] done`. Task completion is also
  summarized at `.task_completion_status.count` / `.completed_count`.
- Related-issue references:
  - `#<id>` — issue in the same project
  - `<project>#<id>` — issue in another project
  - `!<iid>` — MR in the same project
  - `<project>!<iid>` — MR in another project
  - `Closes #<id>`, `Fixes #<id>`, `Related to !<iid>` — closing / linking keywords
- GitLab-flavored references: `@user`, `~label`, `%milestone`, `$snippet`
- Fenced code, tables, HTML `<details>` blocks, embedded images.

### Extracting references with jq + regex

```bash
glab mr view <iid> --repo <project> --output json \
  | jq -r '.description' \
  | grep -oE '([A-Za-z0-9_/.-]+)?[!#][0-9]+' \
  | sort -u
```

### Related issues — official shortcut

For the subset of references GitLab considers "related issues" (including
`Closes` autolinks), use:

```bash
GITLAB_HOST=<host> glab mr issues <iid> --repo <project>
```

This command does not support `--output json`. If JSON is required, fall
back to REST:

```bash
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/closes_issues
```

Returns an array of issue objects (same shape as `GET /projects/:id/issues/:iid`).

### Gotchas

- `description` may be literally `null` for MRs created without a body.
  Coalesce: `jq -r '.description // ""'`.
- Markdown is raw — no preprocessing. Line endings may be `\r\n` on some
  platforms; normalize if feeding to parsers.

---

## 6. Fetching the Unified Diff

```bash
GITLAB_HOST=<host> glab mr diff <iid> --repo <project> --raw
```

`glab mr diff` flags:

- `--raw` — emit raw unified diff without ANSI colors; pipe-safe. **Always
  use `--raw` for machine parsing.**
- `--color always|never|auto` — colorized output; not for parsing.

Output is a concatenated unified diff across all files changed in the MR,
in the standard `git diff` format:

```
diff --git a/path/to/file.go b/path/to/file.go
index <sha>..<sha> 100644
--- a/path/to/file.go
+++ b/path/to/file.go
@@ -10,7 +10,9 @@ func Foo() {
-    old
+    new1
+    new2
```

### Per-file diff

`glab mr diff` has no built-in file filter. Two options:

1. **Post-filter** the full diff with a diff parser or `csplit`-style split
   on `^diff --git ` markers.
2. **REST API** returns per-file records:

```bash
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/diffs --paginate --output ndjson
```

JSON element shape (one per file):

```json
{
  "old_path": "path/to/file.go",
  "new_path": "path/to/file.go",
  "a_mode": "100644",
  "b_mode": "100644",
  "new_file": false,
  "renamed_file": false,
  "deleted_file": false,
  "diff": "@@ -10,7 +10,9 @@ …\n-…\n+…\n"
}
```

Note: the `diff` field here is the *hunks only*, without the `diff --git` /
`index` / `---` / `+++` header lines that `glab mr diff` emits.

### Relationship to commits

`glab mr diff` returns the cumulative diff from the merge-base (target branch
tip at the time of MR creation / last rebase) to the MR head. It does not
include per-commit diffs. For per-commit diffs, combine section 7 with:

```bash
glab api --hostname <host> projects/<project-enc>/repository/commits/<sha>/diff
```

### Gotchas

- Very large diffs may be **truncated** by the GitLab server. The REST
  `/diffs` endpoint returns all files; the older
  `projects/:id/merge_requests/:iid/changes` endpoint has a
  `changes_count` that may be a string like `"1000+"` when truncated. Prefer
  `/diffs` with `--paginate`.
- Binary file changes show as `Binary files a/foo and b/foo differ` with no
  hunks.
- `--color always` injects ANSI escapes — never parse that output.
- File paths can contain spaces, Unicode, and quoted forms (`"path with
  \"quotes\""` per git's `core.quotepath`). Parse with a real diff library
  when precision matters.

---

## 7. Listing MR Commits

There is no dedicated `glab mr commits` command. Use the REST API:

```bash
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/commits --paginate --output ndjson
```

Element shape:

```json
{
  "id": "<sha>",
  "short_id": "abc1234",
  "created_at": "2026-04-21T15:00:00.000Z",
  "parent_ids": ["<sha>"],
  "title": "First line",
  "message": "First line\n\nBody paragraph.\n",
  "author_name": "Alice",
  "author_email": "alice@example.org",
  "authored_date": "2026-04-21T15:00:00.000Z",
  "committer_name": "Alice",
  "committer_email": "alice@example.org",
  "committed_date": "2026-04-21T15:00:00.000Z",
  "trailers": {},
  "web_url": "https://<host>/<project>/-/commit/<sha>"
}
```

### Bulk commit messages

```bash
# SHA + subject, TSV
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/commits \
  --paginate --output ndjson \
  | jq -r '[.short_id, .title] | @tsv'

# Full commit messages joined
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/commits \
  --paginate --output ndjson \
  | jq -r '.message' \
  | awk 'BEGIN{RS=""} {print; print "\n---"}'
```

### Gotchas

- Commits come in reverse chronological order (newest first). Reverse with
  `tac` or `jq -s 'reverse | .[]'` for human-readable chronological reading.
- After a force-push (rebase/squash), older commits disappear from this
  endpoint and the `diff_refs.base_sha` in the MR metadata moves.
- `trailers` (e.g. `Signed-off-by`) are parsed server-side and exposed as
  an object keyed by trailer name.

---

## 8. Reading ALL Discussions / Threads

Threads on an MR come in three kinds:

1. **General notes** — comments on the MR as a whole, not anchored to code.
2. **Diff notes (inline)** — anchored to a specific `(file, line)` of a diff.
3. **System notes** — auto-generated ("Alice marked this as draft",
   "assigned to Bob"). Usually filtered out for review.

### Primary (experimental but supported) command

```bash
GITLAB_HOST=<host> glab mr note list <iid> --repo <project> --output json
```

`glab mr note list` flags:

- `-F, --output text|json` — default `text`
- `--state all|resolved|unresolved` — default `all` (see section 9)
- `-t, --type all|general|diff|system` — default `all`
- `--file <path>` — show only diff notes for that file path

This command fetches the full discussions API under the hood, groups notes
into threads, and returns them.

### Output JSON shape

An array of discussion objects. Each discussion contains one or more notes —
notes[0] is the parent, notes[1..] are replies:

```json
[
  {
    "id": "<discussion-id>",
    "individual_note": false,
    "notes": [
      {
        "id": 3107030349,
        "type": "DiffNote",
        "body": "Please extract this into a helper.",
        "author": { "id": 9, "username": "carol", "name": "Carol" },
        "created_at": "2026-04-22T09:00:00.000Z",
        "updated_at": "2026-04-22T09:00:00.000Z",
        "system": false,
        "noteable_id": 123456789,
        "noteable_iid": 42,
        "noteable_type": "MergeRequest",
        "resolvable": true,
        "resolved": false,
        "resolved_by": null,
        "confidential": false,
        "position": {
          "base_sha": "<sha>",
          "start_sha": "<sha>",
          "head_sha": "<sha>",
          "old_path": "path/to/file.go",
          "new_path": "path/to/file.go",
          "position_type": "text",
          "old_line": null,
          "new_line": 42,
          "line_range": null
        }
      },
      {
        "id": 3107030350,
        "type": "DiffNote",
        "body": "Done in <sha>.",
        "author": { "id": 7, "username": "alice", "name": "Alice" },
        "resolvable": true,
        "resolved": true,
        "resolved_by": { "id": 9, "username": "carol" },
        "resolved_at": "2026-04-22T10:00:00.000Z",
        "system": false
      }
    ]
  }
]
```

### Telling inline from general from system

For each note inside `notes[]`:

| Note kind | `type`          | `system` | `position` present |
|-----------|-----------------|----------|--------------------|
| General   | `null` / `Note` | `false`  | absent             |
| Inline    | `"DiffNote"`    | `false`  | present (object)   |
| System    | `null` / `Note` | `true`   | absent             |

For a discussion, inspect `notes[0]`. If any note has `type == "DiffNote"`
or a non-null `position`, the whole thread is inline.

### Thread parent / reply structure

- `individual_note: true` — a standalone note that does not start a thread
  (you cannot reply to it in the UI).
- `individual_note: false` — a real thread. `notes[0]` is the opener;
  `notes[1..]` are replies in chronological order.
- Resolution is a property of **the discussion** (thread), tracked on the
  resolvable notes. `notes[0].resolved` is the authoritative state for
  resolvable diff threads.

### Fallback via REST

`glab mr note list` is marked EXPERIMENTAL. If it regresses or is removed,
the underlying REST endpoint is stable:

```bash
glab api --hostname <host> \
  projects/<project-enc>/merge_requests/<iid>/discussions \
  --paginate --output ndjson
```

Shape is identical to what `glab mr note list --output json` returns, except
the REST form emits one discussion per NDJSON line rather than a single
JSON array.

### Alternative — `glab mr view --comments`

```bash
GITLAB_HOST=<host> glab mr view <iid> --repo <project> --comments
```

Text-only, human-readable, no JSON mode. Prefer `glab mr note list` or the
REST fallback for programmatic access.

### jq recipes

```bash
# All open (unresolved) inline threads: file, line, body, author
glab mr note list <iid> --repo <project> --output json \
  | jq -r '
      .[] | .notes[0]
      | select(.type == "DiffNote" and .resolved == false)
      | [.position.new_path, (.position.new_line|tostring), .author.username, .body]
      | @tsv'

# Count threads by kind
glab mr note list <iid> --repo <project> --output json \
  | jq '{
      inline:   [.[] | select(.notes[0].type == "DiffNote")]         | length,
      general:  [.[] | select(.notes[0].type != "DiffNote" and (.notes[0].system|not))] | length,
      system:   [.[] | select(.notes[0].system == true)]             | length
    }'

# Unresolved threads only
glab mr note list <iid> --repo <project> --output json --state unresolved \
  | jq '.[] | {id, opener: .notes[0].body, file: .notes[0].position.new_path}'

# Walk a thread: parent + all replies
glab mr note list <iid> --repo <project> --output json \
  | jq -r '.[] | "--- \(.id) ---", (.notes[] | "  @\(.author.username): \(.body)")'
```

### Gotchas

- `--type system` is usually noise for code review; default filter is `all`.
- A resolvable diff note can be unresolved even after the line it references
  is gone — the `position` fields still point at the old SHA.
- `position.new_line: null` + `position.old_line: <n>` means the comment is
  on a removed line (the left side of the diff).
- `position_type` can also be `"image"` or `"file"` for non-text diffs.
- `confidential: true` notes are only visible to project members; the field
  is `true` on those notes in the JSON.

---

## 9. Filtering Unresolved / Resolved Threads

glab exposes this filter on `glab mr note list` via `--state`:

```bash
# Unresolved only
GITLAB_HOST=<host> glab mr note list <iid> --repo <project> \
  --output json --state unresolved

# Resolved only
GITLAB_HOST=<host> glab mr note list <iid> --repo <project> \
  --output json --state resolved

# Everything (default)
GITLAB_HOST=<host> glab mr note list <iid> --repo <project> \
  --output json --state all
```

`glab mr view` has an equivalent pair of flags that imply `--comments`:

```bash
glab mr view <iid> --repo <project> --unresolved
glab mr view <iid> --repo <project> --resolved
```

These are text-only (no JSON). Prefer `glab mr note list` for scripts.

### Semantics

- `--state resolved` keeps only discussions where every resolvable note is
  resolved.
- `--state unresolved` keeps discussions with at least one unresolved
  resolvable note.
- General (non-diff) notes are **not resolvable** — they never match
  `--state resolved` or `--state unresolved`, because their resolution state
  is "not applicable". To include them in a review checklist, fetch with
  `--state all` and partition client-side.

### jq-only partition (if you need both in one pass)

```bash
glab mr note list <iid> --repo <project> --output json \
  | jq '{
      unresolved: [.[] | select(.notes[0].resolvable == true and .notes[0].resolved == false)],
      resolved:   [.[] | select(.notes[0].resolvable == true and .notes[0].resolved == true)],
      general:    [.[] | select(.notes[0].resolvable != true and (.notes[0].system|not))]
    }'
```

### Gotcha

Resolution on a thread is tracked on notes, but the UI/REST treat it as a
thread property. Reading `notes[0].resolved` is reliable; reading it on a
random `notes[i]` is not — replies inherit the parent's resolution flag but
older GitLab versions have returned inconsistent values.

---

## 10. Reading MR Pipeline Status

Three entry points, from fastest/coarsest to richest:

### 10.1 From MR metadata

`glab mr view --output json` already contains `.head_pipeline`:

```bash
glab mr view <iid> --repo <project> --output json \
  | jq '.head_pipeline | {id, status, sha, ref, web_url, created_at, updated_at}'
```

`status` values: `created`, `waiting_for_resource`, `preparing`, `pending`,
`running`, `success`, `failed`, `canceled`, `skipped`, `manual`, `scheduled`.

### 10.2 `glab ci status` for the source branch

```bash
GITLAB_HOST=<host> glab ci status --repo <project> \
  --branch <source_branch> --output json
```

`glab ci status` flags:

- `-b, --branch <name>` — branch to query
- `-F, --output text|json` (text default). **JSON is not compatible with
  `--live` or `--compact`.**
- `-c, --compact`, `-l, --live` — human-only

Output is an array of job objects for the latest pipeline on that branch:

```json
[
  {
    "id": 98765,
    "name": "build",
    "stage": "build",
    "status": "success",
    "ref": "feature/x",
    "duration": 123,
    "queued_duration": 4,
    "created_at": "…",
    "started_at": "…",
    "finished_at": "…",
    "allow_failure": false,
    "web_url": "https://<host>/<project>/-/jobs/98765",
    "user": { "username": "alice" },
    "pipeline": { "id": 987, "status": "success", "sha": "<sha>", "ref": "feature/x" },
    "commit": { "id": "<sha>", "short_id": "abc1234", "title": "…" }
  }
]
```

### 10.3 `glab ci get` — single pipeline JSON

```bash
GITLAB_HOST=<host> glab ci get --repo <project> \
  --pipeline-id <pipeline-id> --output json --with-job-details
```

Returns a pipeline object with embedded jobs when `-d/--with-job-details` is set.

### jq recipes

```bash
# Job names + statuses for the MR's head pipeline
pipeline_id=$(glab mr view <iid> --repo <project> --output json | jq -r '.head_pipeline.id')
glab ci get --repo <project> --pipeline-id "$pipeline_id" --output json --with-job-details \
  | jq -r '.jobs[] | [.stage, .name, .status] | @tsv'

# Failed jobs only
glab ci status --repo <project> --branch <source_branch> --output json \
  | jq '.[] | select(.status == "failed") | {name, stage, web_url}'
```

### Gotchas

- `.pipeline` on the MR object can be an older pipeline if the latest one has
  not been associated yet. `.head_pipeline` is the one tied to the current
  `.sha`.
- MRs can have **merge-request pipelines** (source: `merge_request_event`)
  that are distinct from branch pipelines. The list at
  `GET /projects/:id/merge_requests/:iid/pipelines` returns all of them:

  ```bash
  glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/pipelines \
    --paginate --output ndjson
  ```

- `allow_failure: true` jobs that fail do not fail the pipeline — treat them
  separately when reporting.

---

## 11. Reading Approvals

Two shapes of information:

### 11.1 Eligible approvers (who *can* approve)

```bash
GITLAB_HOST=<host> glab mr approvers <iid> --repo <project> --output json
```

Returns the approval rules and eligible users. Exact shape depends on the
GitLab tier (Free/Premium/Ultimate) — on Free tier it is usually a simple
list of users; on higher tiers it includes rule groups.

### 11.2 Current approval state (who *has* approved)

There is no dedicated `glab mr` subcommand for this — use the REST endpoint:

```bash
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/approvals
```

Response shape (stable):

```json
{
  "id": 123456789,
  "iid": 42,
  "project_id": 1111,
  "title": "…",
  "description": "…",
  "state": "opened",
  "created_at": "…",
  "updated_at": "…",
  "merge_status": "can_be_merged",
  "approvals_required": 2,
  "approvals_left": 1,
  "approved_by": [
    { "user": { "id": 9, "username": "carol", "name": "Carol" } }
  ],
  "user_has_approved": false,
  "user_can_approve": true,
  "suggested_approvers": [ { "id": 8, "username": "bob" } ]
}
```

For a detailed rules-level view (Premium+):

```bash
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/approval_state
```

### jq recipes

```bash
# "2 of 2 required, approved by: alice, bob"
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>/approvals \
  | jq -r '
      "\(.approvals_required - .approvals_left) of \(.approvals_required) required, approved by: " +
      ((.approved_by | map(.user.username)) | join(", "))'
```

### Gotchas

- `approvals_required` is `0` on projects without enforced approval rules;
  `.approved_by` can still be non-empty (people clicked approve).
- `suggested_approvers` reflects GitLab's heuristic, not approval rules.
- The approvals endpoint returns an `AccessDenied` 403 for some guest tokens
  on public projects — if it fails, fall back to reading
  `glab mr view --output json | jq '.approvals_before_merge'` (which may be
  `null` for non-required MRs).

---

## 12. Cross-Repo References

A description often contains references to other MRs/issues. Each resolves
inside GitLab based on context:

| Reference form        | Meaning                                       |
|-----------------------|-----------------------------------------------|
| `#<id>`               | Issue in the **current** project              |
| `!<iid>`              | MR in the **current** project                 |
| `<project>#<id>`      | Issue in another project                      |
| `<project>!<iid>`     | MR in another project                         |
| `<group>/<project>#<id>` | Issue in another project (full path)      |

### Fetch another MR's brief metadata

```bash
GITLAB_HOST=<host> glab mr view <iid> --repo <other-project> --output json \
  | jq '{iid, title, state, draft, web_url, author: .author.username}'
```

### Fetch another issue's brief metadata

```bash
GITLAB_HOST=<host> glab issue view <id> --repo <other-project> --output json \
  | jq '{iid, title, state, web_url, author: .author.username, labels}'
```

`glab issue view` supports `-F/--output json`; its shape is analogous to the
MR object (no `source_branch`/`target_branch`, no `draft`, state is
`opened`/`closed`).

### Direct REST (when you only have the numeric ID and want to avoid CLI glue)

```bash
# Issue
glab api --hostname <host> projects/<project-enc>/issues/<iid>

# MR
glab api --hostname <host> projects/<project-enc>/merge_requests/<iid>
```

### Gotchas

- Cross-project references in the description keep their **relative** form
  (`<project>!<iid>`) — you must resolve `<project>` against the MR's host,
  since GitLab does not allow cross-*instance* references.
- `glab issue view` accepts a full URL argument, which can simplify resolution:
  `glab issue view https://<host>/<project>/-/issues/<id>` — glab parses the
  URL and derives host/project automatically. Still safer to pass
  `GITLAB_HOST` + `--repo` explicitly in scripts.
- On private projects, the reviewer's token may not have access to the
  referenced project — handle 404/403 gracefully and just surface the
  reference as-is.

---

## 13. Handling Special States

MR state flags to inspect (all from `glab mr view --output json`):

| Situation           | Fields to read                                                   |
|---------------------|------------------------------------------------------------------|
| Open, normal        | `state == "opened"` and `draft == false`                         |
| **Draft** (new)     | `state == "opened"` and `draft == true`                          |
| **Draft** (legacy)  | `work_in_progress == true` (deprecated alias of `draft`)         |
| Draft by title      | title starts with `Draft:` (fallback only; prefer the boolean)   |
| Closed (not merged) | `state == "closed"` and `merged_at == null`                      |
| Merged              | `state == "merged"` and `merged_at != null`                      |
| Locked              | `state == "locked"` (discussion locked)                          |
| Has conflicts       | `has_conflicts == true`                                          |
| Mergeable           | `detailed_merge_status == "mergeable"`                           |

### Draft detection — recommended

```bash
glab mr view <iid> --repo <project> --output json \
  | jq -r 'if .draft then "DRAFT" else "READY" end'
```

### `detailed_merge_status` values

Common ones you will see (non-exhaustive):

- `mergeable`
- `not_approved`
- `discussions_not_resolved`
- `draft_status` (the MR is a draft)
- `ci_must_pass`
- `ci_still_running`
- `broken_status` (conflicts)
- `need_rebase`
- `blocked_status`

This is the richest single field for "why can't this merge?" — surface it
directly to reviewers.

### Closed vs merged vs WIP — common confusions

- A merged MR still has `state == "merged"`, not `"closed"`.
- `work_in_progress` is kept for backwards compatibility; new code should
  read `draft`.
- An MR can be simultaneously `opened` + `draft` + `has_conflicts` +
  `blocking_discussions_resolved: false`. Do not treat these as mutually
  exclusive.

### Gotcha

- glab's **text** view (`glab mr view`, no `--output json`) renders "WIP"
  or "Draft" in the header but does not use stable labels — never parse the
  text view for state. Always use the JSON.

---

## 14. JSON Output Mode — Coverage Table

`-F json` / `--output json` support across the commands used by this plugin:

| Command                                    | JSON? | Notes                                               |
|--------------------------------------------|-------|-----------------------------------------------------|
| `glab mr view`                             | YES   | Single object. Full MR metadata (section 4).        |
| `glab mr list`                             | YES   | Array of MR objects. Supports `--page`, `--per-page`. |
| `glab mr note list`                        | YES   | Array of discussions (EXPERIMENTAL).                |
| `glab mr approvers`                        | YES   | Eligible-approvers list.                            |
| `glab mr diff`                             | NO    | Raw diff text only; use `--raw`.                    |
| `glab mr issues`                           | NO    | Text table; use REST `/closes_issues` for JSON.     |
| `glab mr approve` / `merge` / `close` …    | n/a   | Write commands — not used.                          |
| `glab ci status`                           | YES   | Not compatible with `--live` or `--compact`.        |
| `glab ci get`                              | YES   | Pair with `--with-job-details`.                     |
| `glab ci list`                             | YES   | Array of pipelines.                                 |
| `glab issue view`                          | YES   | Single issue object.                                |
| `glab auth status`                         | NO    | Text only; check exit code.                         |
| `glab api <endpoint>`                      | YES   | Default `json`, also `ndjson` for streams.          |

When an operation lacks a JSON mode, use `glab api` directly against the
equivalent REST endpoint and parse with `jq`.

### `ndjson` vs `json` for `glab api`

- Use `--output json` for small, bounded responses (single object / small
  array).
- Use `--output ndjson` with `--paginate` for unbounded lists (comments,
  commits, discussions, pipelines, diffs) — streams one record per line and
  does not require buffering the whole result set.

---

## 15. Rate Limits, Timeouts, Retries

### GitLab rate limits

Self-hosted instances typically enforce the "Authenticated API requests"
rate limit. Defaults on gitlab.com are 2000 req/min per user; self-managed
administrators may set this lower. Response headers to watch:

- `RateLimit-Limit: <n>`
- `RateLimit-Remaining: <n>`
- `RateLimit-Reset: <epoch-seconds>`
- `Retry-After: <seconds>` — only on 429 responses

Include these in retry logic:

```bash
glab api --hostname <host> --include projects/<project-enc>/merge_requests/<iid> 2>&1 \
  | sed -n '1,/^\r\?$/p'   # print header block only
```

The `-i, --include` flag on `glab api` prepends the HTTP response headers to
stdout.

### Timeouts

glab uses Go's `net/http` default client with no explicit HTTP timeout on
many calls — a stuck server can hang the command. Wrap invocations with an
OS-level timeout:

```bash
timeout 30s glab mr view <iid> --repo <project> --output json
```

Reasonable defaults for this plugin:

- Metadata / single-object reads: **15s**
- Discussions list, commits list: **30s**
- Full `--paginate` over a large MR: **60s**
- `glab mr diff` on a very large MR: **60s**

### Retry recommendation

- Retry on: `exit != 0` with stderr matching `429`, `5\d\d`, `timeout`,
  `connection reset`, `EOF`, `i/o timeout`.
- Do NOT retry on: `401`, `403`, `404` — they are deterministic and need
  human intervention.
- Backoff: 1s, 3s, 7s (exponential jitter). On 429 with `Retry-After`,
  sleep the indicated seconds instead.
- Max attempts: 3 for read operations.

---

## 16. Pagination

### glab flags

Many listing commands expose `-p/--page` and `-P/--per-page`:

| Command              | `-p`/`--page` | `-P`/`--per-page` | `--paginate` | Notes                                         |
|----------------------|---------------|--------------------|--------------|-----------------------------------------------|
| `glab mr list`       | yes (default 1)| yes (default 30)   | no           | Also `-A/--all` fetches all (caps internally).|
| `glab mr view`       | yes           | yes (default 20)   | no           | `--per-page` affects embedded comment list.   |
| `glab mr note list`  | no            | no                 | no           | Returns all discussions in one shot.          |
| `glab ci list`       | yes           | yes (default 30)   | no           |                                               |
| `glab issue view`    | yes           | yes                | no           | `--per-page` affects embedded comments.       |
| `glab api`           | n/a           | via query params   | `--paginate` | Use `--paginate --output ndjson`.             |

### Reading everything — `glab api --paginate`

```bash
# All commits on an MR
glab api --hostname <host> \
  projects/<project-enc>/merge_requests/<iid>/commits \
  --paginate --output ndjson

# All discussions
glab api --hostname <host> \
  projects/<project-enc>/merge_requests/<iid>/discussions \
  --paginate --output ndjson

# All diffs (per-file)
glab api --hostname <host> \
  projects/<project-enc>/merge_requests/<iid>/diffs \
  --paginate --output ndjson

# All pipelines associated with the MR
glab api --hostname <host> \
  projects/<project-enc>/merge_requests/<iid>/pipelines \
  --paginate --output ndjson
```

`--paginate` walks the `Link: <…>; rel="next"` header server-side until
exhausted. Combined with `--output ndjson`, memory stays bounded.

### Sizing requests

- Default GitLab REST page size is 20, max 100. Pass
  `--field per_page=100` on `glab api` for fewer round-trips.
- Keep `--per-page` ≤ 100; larger values are silently capped.

```bash
glab api --hostname <host> \
  projects/<project-enc>/merge_requests/<iid>/discussions \
  --field per_page=100 --paginate --output ndjson
```

### Gotchas

- `glab mr list -A/--all` is a convenience wrapper, not an NDJSON streamer —
  for very large lists, prefer `glab api projects/<project-enc>/merge_requests
  --paginate --output ndjson` with filters.
- Using `-p N` without `-P` uses the default page size (30), which may skip
  items if the caller expected a different size.
- Ordering across pages is consistent only when combined with `--order` /
  `--sort` — otherwise a new MR appearing during paging can cause a duplicate
  or missed item.

---

## Appendix A — Write Commands (Not Used)

The following `glab` commands exist and would mutate state. This plugin is
read-only — **none of these are invoked under any circumstance**. Listed so
reviewers and agents recognize them and can explicitly refuse.

| Command                   | Effect                                         | Status           |
|---------------------------|------------------------------------------------|------------------|
| `glab mr approve`         | Approves the MR                                | not used, read-only |
| `glab mr revoke`          | Revokes approval                               | not used, read-only |
| `glab mr merge` / `accept`| Merges the MR                                  | not used, read-only |
| `glab mr close`           | Closes the MR                                  | not used, read-only |
| `glab mr reopen`          | Reopens a closed MR                            | not used, read-only |
| `glab mr create`          | Creates a new MR                               | not used, read-only |
| `glab mr update`          | Edits MR title/description/labels/reviewers    | not used, read-only |
| `glab mr delete`          | Deletes the MR                                 | not used, read-only |
| `glab mr rebase`          | Triggers a rebase on the server                | not used, read-only |
| `glab mr checkout`        | Writes local branch / modifies working tree    | not used, read-only |
| `glab mr note` (no subcmd)| Posts a new comment                            | not used, read-only |
| `glab mr note resolve`    | Resolves a discussion                          | not used, read-only |
| `glab mr note reopen`     | Reopens a resolved discussion                  | not used, read-only |
| `glab mr subscribe` / `unsubscribe` | Changes subscription                 | not used, read-only |
| `glab mr todo`            | Adds a to-do                                   | not used, read-only |
| `glab ci run` / `retry` / `cancel` / `trigger` / `delete` | Mutate pipelines/jobs | not used, read-only |
| `glab api -X POST|PUT|PATCH|DELETE …` | Any non-GET HTTP verb              | not used, read-only |

**Guardrail for `glab api` callers in this plugin**: only invoke with the
default method (GET) or an explicit `-X GET`. Never pass `-X POST`, `-X PUT`,
`-X PATCH`, `-X DELETE`, `--field`, `--raw-field`, `--form`, or `--input` —
each of those either flips the default method to POST or is only meaningful
for writes.

---

## Appendix B — REST Fallbacks Summary

All are read-only `GET` endpoints. Use only when glab's native command is
missing JSON support or the feature altogether. Each is marked
"fallback; not needed if glab covers the case" where applicable.

| Need                                       | glab native          | REST fallback                                                                          |
|--------------------------------------------|----------------------|----------------------------------------------------------------------------------------|
| MR metadata                                | `glab mr view --output json` | `GET projects/<enc>/merge_requests/<iid>`                                            |
| MR list                                    | `glab mr list --output json` | `GET projects/<enc>/merge_requests`                                                  |
| Per-file diff                              | —                    | `GET projects/<enc>/merge_requests/<iid>/diffs` — **fallback; required**               |
| Commits                                    | —                    | `GET projects/<enc>/merge_requests/<iid>/commits` — **fallback; required**             |
| Discussions                                | `glab mr note list --output json` | `GET projects/<enc>/merge_requests/<iid>/discussions` — fallback; not needed   |
| Related/closing issues (JSON)              | `glab mr issues` (text only) | `GET projects/<enc>/merge_requests/<iid>/closes_issues` — **fallback; required** |
| Pipelines list for an MR                   | —                    | `GET projects/<enc>/merge_requests/<iid>/pipelines` — **fallback; required**           |
| Approval state                             | `glab mr approvers --output json` (eligibility) | `GET projects/<enc>/merge_requests/<iid>/approvals` — **fallback; required for state** |
| Approval rules breakdown (Premium+)        | —                    | `GET projects/<enc>/merge_requests/<iid>/approval_state` — **fallback; required**      |
| Single commit diff                         | —                    | `GET projects/<enc>/repository/commits/<sha>/diff` — **fallback; required**            |

Token used is whichever glab resolves via `GITLAB_HOST` + stored creds. No
separate `GITLAB_TOKEN` plumbing is required because `glab api` reuses the
same auth chain as the native commands.
