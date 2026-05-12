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
├── architecture.md        ← repo architecture notes (created/updated after reviews)
├── invariants.md          ← known invariants and contracts
├── risky-areas.md         ← historically buggy or complex areas
└── review-history.md      ← past review findings and patterns
```

Context files are optional. Create them only when stable knowledge is discovered during a review. Include them in subagent tasks when they exist.

## Workflow

### 1. Fetch PR metadata and diff

```bash
gh pr view <PR> -R <owner/repo> --json title,body,baseRefName,headRefName,additions,deletions,changedFiles
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

### 3. Detect docs-only PRs

If all changed files are documentation/content only, skip subagents. Report "docs-only, no code concerns" and stop.

### 4. Read existing repo context

Check `$REVIEW_DIR` for `architecture.md`, `invariants.md`, `risky-areas.md`, `review-history.md`. Read any that exist. Include their content in the subagent task prompts so reviewers have repo memory.

### 5. Spawn two subagent reviewers

Read the task prompt from the corresponding reference file, then build the `task` string with:
- Instructions from the reference file
- Repo clone path (`$REVIEW_DIR/repo`)
- PR diff and changed file list
- Any existing repo context files

Use `context: "fork"` so each subagent inherits tools (`exec`, `read`, `grep`) and can explore the repo independently. Each subagent has its entire context window dedicated to one specialty.

**Bug hunter** — read `references/bug-hunter-prompt.md`, append PR context, spawn.
**Security + architecture** — read `references/security-arch-prompt.md`, append PR context, spawn.

Call `sessions_yield()` after spawning both.

### 6. Merge, filter nits, and deduplicate

Collect findings from both subagents. Apply three filters in order:

**Nit filter (drop first):** Remove any finding that is:
- A style or naming preference
- A "consider using X" suggestion with no bug or security impact
- A formatting, import ordering, or convention comment
- Technically true but nobody would fix it before merging

**Verification filter:** For each remaining finding:
- Does it name an exact file and line?
- Does it describe a concrete break path?
- Would a senior engineer agree it is worth flagging?

**Dedup:** Remove duplicates where both agents flagged the same issue. Rank by severity.

### 7. Update repo context (only when new stable knowledge is discovered)

If the review revealed durable knowledge — architecture patterns, invariants, risky areas — update the corresponding files under `$REVIEW_DIR`. Append a brief entry to `review-history.md` with the PR number and key findings.

Do not add PR-specific noise. Only add knowledge that helps future reviews.

### 8. Report in chat

```
## PR Review: <title> (#<number>)

**Verdict:** APPROVE | NEEDS_CHANGES | NEEDS_DISCUSSION

### Findings
- [Critical/High/Medium/Low] `file:line` — what is wrong and why it breaks

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
