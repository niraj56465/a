---
name: codex-code-review-openclaw
description: >
  Review GitHub pull requests using specialized subagents for high accuracy
  and minimal false positives. Spawn parallel reviewers (bug hunter + security/architecture)
  that each trace code paths beyond the diff, then merge findings into one report.
  Report in chat only — never push to GitHub or post GitHub comments.
  Use when asked to review a PR, pull request, code change, or diff.
version: 4.0.0
metadata:
  openclaw:
    emoji: "🔍"
    requires:
      bins:
        - gh
        - git
        - grep
---

# PR Code Review

Orchestrate two specialized subagent reviewers in parallel, merge their findings, report in chat.

## Local repo and context storage

All repos and review context live under:

```
~/.openclaw/workspace/.review-context/<owner>-<repo>/
├── repo/                  ← persistent git clone
├── prs/<number>.json      ← last reviewed head SHA, prior findings, dismissed findings
├── architecture.md        ← repo architecture notes (created/updated after reviews)
├── invariants.md          ← known invariants and contracts
├── risky-areas.md         ← historically buggy or complex areas
└── review-history.md      ← past review findings and patterns
```

Context files are optional. Create them only when stable knowledge is discovered during a review. Include them in subagent tasks when they exist. PR state files are used to avoid repeating old findings on re-review.

## Workflow

### 1. Fetch PR metadata and diff

```bash
gh pr view <PR> -R <owner/repo> --json title,body,baseRefName,headRefName,headRefOid,additions,deletions,changedFiles,files
gh pr diff <PR> -R <owner/repo>
```

### 2. Prepare local repo

Set `REVIEW_DIR=~/.openclaw/workspace/.review-context/<owner>-<repo>`.

**First time (no clone exists):**
```bash
mkdir -p $REVIEW_DIR
gh repo clone <owner/repo> $REVIEW_DIR/repo -- --depth=50
cd $REVIEW_DIR/repo
gh pr checkout <N>
```

**Subsequent reviews (clone exists):**
```bash
cd $REVIEW_DIR/repo
git checkout main
git clean -fd
git fetch origin
gh pr checkout <N>
```

If `gh pr checkout` fails, delete `$REVIEW_DIR/repo` and re-clone fresh.

### 3. Load prior PR state and feedback memory

If `$REVIEW_DIR/prs/<N>.json` exists, read it before reviewing. Use it to avoid repeated findings.

Suggested state shape:

```json
{
  "pr": 123,
  "lastReviewedHeadSha": "abc123",
  "reportedFindings": [],
  "dismissedFindings": [],
  "acceptedFindings": []
}
```

If `lastReviewedHeadSha` exists and differs from the current `headRefOid`, treat this as a re-review: focus on `git diff <lastReviewedHeadSha>..<headRefOid>` and only carry forward old findings if they are still clearly unresolved.

Also scan `review-history.md` for repeated false-positive patterns and project preferences. Suppress findings matching dismissed patterns unless there is new, stronger evidence.

### 4. Detect docs-only PRs

If all changed files are documentation/content only, skip subagents. Report "docs-only, no code concerns" and stop.

### 5. Read existing repo context

Check `$REVIEW_DIR` for `architecture.md`, `invariants.md`, `risky-areas.md`, `review-history.md`. Read any that exist. Include their content in the subagent task prompts so reviewers have repo memory.

### 6. Build the review context pack

Before spawning reviewers, prepare one compact context pack and pass the same pack to both subagents. Include:

- PR title, body, base/head branch, additions/deletions, and changed file list
- PR diff, or incremental diff for re-reviews
- Repo clone path (`$REVIEW_DIR/repo`)
- Relevant existing repo context files
- Prior PR state: previous findings, dismissed findings, accepted findings
- Full changed file contents, unless a file is huge/generated; for huge files, include enclosing functions/classes/sections only
- Upstream dependency context: directly referenced functions, types, constants, configs, and likely callers/callees
- Related tests and CODEOWNERS when present
- PR delta brief: what this PR changes, where changed files fit in the architecture, affected execution flow, and design details that may look suspicious but are intentional
- System constraints when relevant: timeout caps, output/body limits, concurrency limits, rate limits, memory limits, default parameter differences, fallback behavior
- Risk signals: auth/security files, migrations, dependency files, CI/config files, public API/type/schema changes
- Common false-positive patterns from repo memory plus these defaults: host vs container paths, idempotent safety nets, intentional fallthrough, best-effort error suppression, logging style inconsistencies, embedded shell/python in infrastructure code

Keep the context pack focused. Do not paste unrelated files. Tell subagents to inspect the local repo when they need more context. Context quality matters more than agent count.

### 7. Spawn two subagent reviewers

Read the task prompt from the corresponding reference file, then build the `task` string with:
- Instructions from the reference file
- The review context pack from step 6

Use `context: "fork"` so each subagent inherits tools (`exec`, `read`, `grep`) and can explore the repo independently. Each subagent has its entire context window dedicated to one specialty.

**Bug + contract reviewer** — read `references/bug-hunter-prompt.md`, append the context pack, spawn.
**Security + production-risk reviewer** — read `references/security-arch-prompt.md`, append the context pack, spawn.

Call `sessions_yield()` after spawning both.

### 8. Parent-agent validation, filtering, and deduplication

Collect findings from both subagents. Never forward subagent findings directly. Subagents maximize recall; the parent reviewer maximizes precision with full repo context. Apply these filters in order:

**Nit filter (drop first):** Remove any finding that is:
- A style or naming preference
- A "consider using X" suggestion with no bug or security impact
- A formatting, import ordering, or convention comment
- Technically true but nobody would fix it before merging

**Verification filter:** For each remaining finding:
- Does it name an exact file and line?
- Does it describe a concrete break path?
- Does it include evidence from the diff or repo, not speculation?
- Does it survive a second-pass check against the actual diff/repo?
- Is it new, or still unresolved from a prior review?
- Is the scenario realistic given the architecture brief, system constraints, and runtime behavior?
- Is it contradicted by guards, fallbacks, idempotency, or trust-boundary context elsewhere in the repo?
- Would a senior engineer agree it is worth flagging?

Drop any finding that cannot be expressed with this schema:

```
- [Severity] `file:line` — concise issue
  Evidence: exact code/change that creates the problem
  Break path: input/event → code path → failure/impact
  Suggested fix: concrete fix direction
  Confidence: High/Medium/Low
```

**Dedup:** Remove duplicates where both agents flagged the same issue. Rank by severity.

**Output cap:** Report at most 5 findings by default. Include more only for Critical/High issues. Prefer a short high-signal report over a complete noisy report.

## Calibration blocks

Use these while filtering subagent output:

**Severity**
- Critical: security breach, data loss, production outage/crash. Requires concrete proof.
- High: realistic incorrect behavior for normal users or common operations.
- Medium: maintainability risk, unusual edge case, or plausible but conditional bug.
- Low: minor issue worth noting only if it affects correctness or review clarity.
- Drop: style-only, speculative, already handled, or not worth fixing before merge.

**Runtime verification**
- What actually happens at runtime?
- Who controls the input: trusted operator/config or untrusted user/API data?
- Which exact path executes, including fallback paths?
- Do guards, validation, idempotency, or try/catch already handle it?
- Is the scenario realistic?

**System constraint checks**
- Timeout caps in wrappers, watchers, gateways, clients, or jobs
- Output/body size limits and truncation behavior
- Concurrency, rate, memory, and queue limits
- Default parameter mismatches when replacing one function/path with another
- Fallback behavior on script failure vs transport failure vs validation failure

**Common false positives**
- Host vs container path differences caused by bind mounts
- Idempotent safety-net operations that intentionally run twice
- Intentional fallthrough or boundary checks
- Best-effort `|| true`, ignored setup errors, or non-critical empty catches
- Logger style inconsistency without behavioral impact
- Embedded shell/python in infrastructure code without concrete maintenance/runtime risk

Include dismissed findings only when they were non-obvious or likely to be questioned.

### 9. Update repo context and PR state

If the review revealed durable knowledge — architecture patterns, invariants, risky areas — update the corresponding files under `$REVIEW_DIR`. Append a brief entry to `review-history.md` with the PR number and key findings.

Update `architecture.md` when the review discovers durable repo architecture knowledge: execution flow, subsystem boundaries, host/container/runtime boundaries, hard system constraints, integration points, or design decisions future reviewers may mistake for bugs.

Update `invariants.md` for durable contracts or rules. Update `risky-areas.md` for historically fragile code paths. Update `review-history.md` for PR-specific findings, accepted findings, dismissed findings, and false-positive patterns.

Do not add PR-specific noise to `architecture.md`, `invariants.md`, or `risky-areas.md`. If unsure whether knowledge is durable, write it to `review-history.md` instead.

Update `$REVIEW_DIR/prs/<N>.json` with the current `headRefOid`, reported findings, and any known accepted/dismissed findings. If the user later says a finding was wrong/noisy, record that as a dismissed finding so future reviews suppress similar noise.

### 10. Report in chat

```
## PR Review: <title> (#<number>)

**Verdict:** APPROVE | NEEDS_CHANGES | NEEDS_DISCUSSION

### Findings
- [Critical/High/Medium/Low] `file:line` — what is wrong and why it breaks
  Evidence: exact changed code or repo fact
  Break path: concrete failure path
  Suggested fix: concrete fix direction
  Confidence: High/Medium/Low

### What's Good
- (1-2 lines)

### Suggested Fixes
- (concrete, not vague)
```

If no real issues survive the merge: "No issues found. APPROVE."

## Hard rules

- Never push commits, branches, or any content to GitHub.
- Never post PR reviews, comments, or reactions on GitHub.
- Never modify files inside the cloned repo.
- Repo context files live in `$REVIEW_DIR`, never inside the repo.
- Report only in chat.
- Zero false positives over catching everything.

## References

- `references/bug-hunter-prompt.md` — task prompt for the bug hunter subagent
- `references/security-arch-prompt.md` — task prompt for the security/architecture subagent
- `references/examples.md` — concrete good vs bad finding examples
