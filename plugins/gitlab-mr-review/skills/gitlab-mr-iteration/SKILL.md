---
name: gitlab-mr-iteration
description: Iteration, deduplication, and regression-detection algorithms. Defines the footer-signature format for attributing prior findings, the two-step dedup algorithm (exact fid match + semantic similarity against human threads), regression detection by re-checking resolved-thread concerns against current diff, and delta-section composition. Use in regression-detector, validator, and report-composer agents whenever existing threads must be considered.
---

# Iteration, Dedup, and Regression Detection

## Footer signature

Format: `<!-- gitlab-mr-review:v1 fid=<fid> run=<YYYYMMDD-HHMMSS> -->`

Purpose: attribute prior findings across runs without any local state file. Every finding the plugin posts carries this hidden HTML comment as its last line. GitLab stores it in the note body but does not render it. On subsequent runs, the plugin reads all existing threads and scans each thread's first note body for this footer to identify which threads originated from a prior Claude review.

Parsing: apply the following regex to each thread's first note body:

```
<!--\s*gitlab-mr-review:v1\s+fid=([^\s]+)\s+run=([0-9]{8}-[0-9]{6})\s*-->
```

Capture group 1 is `fid`; capture group 2 is `run` (timestamp). Populate `existing_threads.by_thread[].claude_footer` with `{ fid, run }` when a match is found. If the note body contains no such footer, `claude_footer` is `null` and the thread is treated as human-authored.

## Dedup algorithm

The dedup algorithm runs in the validator agent (phase [4]). It processes each candidate finding from the merged `new_findings` list before any finding is included in the final report.

### Step 1 — exact match by fid

```
For each candidate from new_findings:

  # Step 1 — exact match against prior Claude finding
  If exists thread whose first note matches fid=<candidate.fid>:
    If thread.status == "unresolved":
      DROP candidate                                # still visible in MR
      Log: "dropped — already unresolved in thread #<id>"
    Else if thread.status == "resolved":
      If candidate still detected in current diff (same file, similar position):
        MARK candidate.regression_of_thread_id = thread.id
        KEEP candidate                              # will render regression notice
      Else:
        DROP candidate                              # was flagged, got fixed, no longer present
    Continue to next candidate
```

An exact fid match means the `fid` value in a parsed footer equals `candidate.id`. When the original thread is unresolved, the concern is still open and visible — re-posting would create a duplicate, so the candidate is dropped. When the thread is resolved and the concern reappears in the current diff, the candidate is kept but annotated as a regression.

### Step 2 — semantic match against human-authored threads

```
  # Step 2 — semantic dedup against human-authored threads
  For each thread in existing_threads (any status, non-Claude):
    If same_file(thread.file, candidate.file)
       AND within_range(thread.line, candidate.line_start..candidate.line_end, tolerance=5)
       AND semantic_similar(thread.body, candidate.body):        # LLM judgement
      DROP candidate
      Log: "dropped — overlaps with thread #<id> by <author>"
      Continue to next candidate

  KEEP candidate.
```

Predicates:
- `same_file(a, b)` — true if the file paths are equal after normalizing separators.
- `within_range(line, start..end, tolerance=5)` — true if `line` is within `[start - 5, end + 5]`.
- `semantic_similar(body_a, body_b)` — LLM judgement call using the similarity prompt template below.

Step 2 applies only to threads whose first note does not have a `claude_footer` (i.e., human-authored threads). This prevents Claude from re-raising a concern a human reviewer already raised, even if the thread was never resolved.

### Semantic-similarity prompt template

Use the following prompt verbatim when invoking the LLM to judge semantic similarity:

```
Candidate finding:
  file: <file>
  lines: <line_start>-<line_end>
  body: <body>

Existing thread:
  file: <file>
  line: <line>
  body: <body>

Are these raising the SAME concern? Answer YES if the underlying issue is the same
(even if wording differs); NO if they are distinct. Respond with just YES or NO.
```

A `YES` response causes the candidate to be dropped. The prompt is designed to be minimal and binary to reduce ambiguity and cost.

## Regression detection

Regression detection runs as a dedicated fan-out agent (`mr-regression-detector`) in phase [3], in parallel with the standard reviewer agents. It receives `mr_intent.sources.existing_threads` filtered to threads with `status == "resolved"` plus the current unified diff.

Mechanics: for each resolved thread in the filtered list, the agent extracts the "concern" that the thread originally raised and then judges whether that concern still applies to the current code. If it does, a finding is emitted with `regression_of_thread_id` set to the thread's ID and severity bumped to at least `should-fix` (or higher if the original finding was `must-fix`).

Concern-extraction prompt template:

```
The following is a resolved GitLab review thread on an MR.

Thread file: <file>
Thread line: <line>
Thread notes:
<first note body>

In one sentence, what specific code concern was this thread raising?
Respond with just the concern description, no preamble.
```

Applies-to-current-code prompt template:

```
Original concern: <extracted concern>

Current diff for file <file> (context around line <line>):
<relevant diff hunk>

Does this concern still apply to the current version of the code?
Answer YES if the problematic code is still present or has reappeared.
Answer NO if the code has been fixed or the concern no longer applies.
Respond with just YES or NO.
```

A `YES` response causes the agent to emit a finding with `regression_of_thread_id = thread.id` and `severity = max("should-fix", original_severity)`. These regression findings then flow through the validator like any other finding from the reviewer fan-out.

## Delta section composition

The delta section (section 3 of the report) is only emitted when at least one thread in `existing_threads` has a parseable `gitlab-mr-review:v1` footer. If no footers are found, the section is omitted entirely and the review behaves as if it is the first run.

When footers are present:

- Derive the previous review timestamp from the most recent `run=` field across all parsed footers (highest `YYYYMMDD-HHMMSS` value).
- Count new commits since that timestamp by comparing commit `committed_date` values against the run timestamp.
- Count previous findings by category:
  - **Resolved**: threads with a Claude footer and `status == "resolved"`, not flagged as regression.
  - **Regressing**: findings in the current run with `regression_of_thread_id` set.
  - **Carried over unresolved**: threads with a Claude footer and `status == "unresolved"` that were dropped (not re-posted).
  - **No longer applicable**: threads with a Claude footer and `status == "resolved"` where the concern no longer appears in the current diff (dropped without regression).
- Count new findings: any finding in the current run whose `fid` does not match any existing thread footer.

## Edge cases and their handling

| Case | Behaviour |
|------|-----------|
| Prior findings posted without footer (user edited it out) | Not recognized; treated as regular human threads. Semantic dedup still applies. |
| `fid` collision | Possible false-drop; logged so user sees. |
| Thread resolved by system activity | Treated as resolved. Regression detector may re-raise if concern persists. |
| Two concurrent runs on same MR | Unsupported. Future TODO: lock file in `.claude/reviews/`. Not required for MVP. |
| MR force-pushed, SHA-based permalinks stale | `fid` does not depend on SHA — unaffected. Permalink URLs in bodies may 404 — acceptable. |
| User never posted prior findings | No footers found → delta section omitted → behaves as first review. |
| `--save` files intended as fallback identity | Not used. Single source of truth is GitLab threads + footer. |

## Summary of iteration behaviour

Re-running the plugin on the same MR works without any local state file. GitLab threads are the sole source of truth across runs. Footer signatures embedded in prior posted comments establish finding identity: the plugin can reconstruct which concerns it raised, which were resolved, and which are regressing purely by reading the thread state at review time. Dedup ensures new runs do not flood the MR with duplicate comments. Regression detection closes the loop by catching concerns that were resolved but have since reappeared in the code. The delta section gives reviewers a concise summary of what changed since the last Claude review without requiring any out-of-band coordination.
