---
name: code-audit
description: This skill should be used when the user asks to "audit this PR", "audit my branch", "do a code audit", "review this PR", "do a code review", "review my branch", "review this commit", "review my changes", "take a look at these changes", or gives a GitHub PR URL/number, a commit SHA, or asks for feedback on uncommitted work. Performs a thorough, multi-dimension audit (correctness, architecture, performance, security) and produces a structured report ending in an approve / request-changes recommendation. This is the heavyweight review — prefer it over lightweight diff-only reviews when the user wants real scrutiny of a PR, branch, or commit.
---

# Code Audit

Perform a rigorous, human-quality code review of a change set and produce a structured report that leads with critical issues and ends with a clear recommendation. The goal is to catch the things that matter — bugs, design mistakes, performance cliffs, and security holes — not to relitigate style that a linter or formatter already owns.

Work through four phases in order: **Orient → Gather Context → Review → Report**.

## Phase 1: Orient — determine what is being reviewed

The review target arrives in one of three forms. Identify which, then materialize the diff.

**A GitHub PR** (a URL like `github.com/org/repo/pull/123`, or "PR 123", or "the PR for this branch"):
- Get metadata and the diff with `gh`:
  - `gh pr view <number-or-url> --json title,body,author,baseRefName,headRefName,state,files,additions,deletions`
  - `gh pr diff <number-or-url>` for the full patch.
- Read the discussion — it often points straight at the risky parts (see Phase 2).
- If the PR is on a branch not checked out locally, use a worktree (see below).

**A commit SHA** (or a range like `abc123..def456`):
- `git show <sha>` for a single commit, or `git diff <base>..<head>` for a range.
- `git log --oneline <range>` to see the commit sequence and intent.

**Uncommitted local work** (the default when no target is named — "review my changes"):
- `git status` to see the lay of the land.
- `git diff` for unstaged, `git diff --staged` for staged changes.
- Include untracked files that are part of the change: `git status --porcelain` then read the new files.
- Compare against the base branch for the full branch delta: `git diff $(git merge-base HEAD master)...HEAD` (substitute the actual default branch).

If the form is genuinely ambiguous (e.g. a bare number that could be a PR or something else), ask the user rather than guessing.

### Using a git worktree for remote branches

Never check out a remote branch in the user's working tree — it stomps on in-progress work. Instead create an isolated worktree:

```bash
git fetch origin
git worktree add /tmp/review-<branch-or-pr> <branch-or-sha>   # detached checkout is fine
```

Or, for a PR, `gh pr checkout <number>` **inside** the worktree directory. Do the review from there. When finished, clean up:

```bash
git worktree remove /tmp/review-<branch-or-pr>
```

Tell the user the worktree path in case cleanup is interrupted. If the diff is small and available purely as a patch (via `gh pr diff` or `git show`), a worktree may be unnecessary — only create one when it helps to build, run, or navigate the full code.

## Phase 2: Gather context before judging

A diff read in isolation produces shallow reviews. Load the surrounding context first.

**Project conventions and intent.** Read any `CLAUDE.md`, `AGENTS.md`, `AGENT.md`, `CONTRIBUTING.md`, `.cursorrules`, `README`, or `docs/` material relevant to the changed area. These encode the standards the change should be held to — architectural rules, naming, testing expectations, performance budgets. A "problem" that's actually a documented project convention is not a finding.

**The change's own story.** For a PR, read the description and **all review comments and discussion threads** (`gh pr view --comments`, or `gh api repos/{owner}/{repo}/pulls/{n}/comments` for inline review comments). Reviewers frequently flag concerns, and the author explains tradeoffs — this tells you where to look hard and what's already been litigated. For commits, read the messages.

**The code around the diff.** Open the changed files and read enough of the surrounding code — callers, callees, the data structures involved, existing tests — to understand how the change fits. Look for the patterns the codebase already uses so the review holds the change to the house style, not an abstract ideal.

## Phase 3: Review across four dimensions

Examine the change through each lens below. The full checklist of what to look for in each dimension lives in **`references/review-checklist.md`** — read it now and work through the relevant items. Not every dimension applies to every change (a docs PR has no SQL injection surface); focus effort where the change actually has risk.

The four dimensions:
- **Correctness** — does it do what it claims, including edge cases, error paths, and concurrency?
- **Architecture** — does it fit the system's design, with sensible boundaries and no needless coupling or complexity?
- **Performance** — right data structures and algorithms, no accidental O(n²), no N+1 queries, no unbounded growth?
- **Security** — input validation, authz/authn, injection, secrets, and the OWASP-style hazards?

For every issue, verify it against the actual code before reporting — trace the data flow or the call path enough to be confident it's real. A confident, correct finding is worth ten speculative ones. Assign each a severity (Critical / Major / Minor / Nit) as defined in the checklist.

**Stay out of the linter's lane.** Do not spend the review on formatting, import ordering, unused-variable nits, or anything an automated tool in the repo already enforces. Those add noise and bury the findings that matter. Comment on style only when it genuinely impairs correctness or readability.

## Phase 4: Report

Produce the report using the structure in **`references/report-template.md`**. The essential shape:

1. **Summary** — one or two sentences on what the change does and the overall verdict.
2. **Critical issues first** — anything that blocks merge (bugs, security holes, data loss), each with file:line, why it's wrong, and a concrete fix.
3. **Remaining findings by dimension**, ordered by severity within each.
4. **What's good** — briefly, genuinely. This calibrates the review and is worth stating.
5. **Recommendation** — a clear call: **Approve**, **Approve with nits**, **Request changes**, or **Needs discussion**.

Use `file:line` references so findings are clickable and actionable. Include short code snippets or suggested diffs where they make the fix obvious.

**Finding nothing major is a valid and good outcome.** If the change is solid, say so plainly and recommend approval — do not manufacture findings to look thorough. A review that invents problems is worse than one that confirms the code is ready.

If the user asked to post the review to a PR, offer to do so via `gh pr review` (with `--approve`, `--request-changes`, or `--comment`) or inline comments via `gh api` — but confirm before posting anything outward-facing.

## Reference files

- **`references/review-checklist.md`** — the detailed per-dimension checklist of what to look for. Read during Phase 3.
- **`references/report-template.md`** — the exact report structure and an example. Use during Phase 4.
