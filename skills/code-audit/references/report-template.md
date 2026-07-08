# Report Template

Use this structure for the Phase 4 report. Adapt section presence to the change — omit a dimension heading if it had no findings rather than writing "None" under every one. Keep prose tight; the reader wants findings, not padding.

## Structure

```markdown
# Code Review: <PR title / commit subject / "uncommitted changes on <branch>">

**Target:** <PR #123 / commit abc123 / local branch `feature-x` vs `master`>
**Scope:** <N files, +A/-D lines> — <one line on what the change does>

## Summary

<1–2 sentences: what the change accomplishes and the headline verdict. State the
recommendation here in a few words so a skimmer gets it immediately.>

## Critical Issues

<Anything that blocks merge. If none, write "None." — that's a good result, say it plainly.>

### 🔴 <Short title> — `path/to/file.rb:42`
<What's wrong and why it matters — the concrete failure it causes.>
```<lang>
// the problematic code, or a suggested fix
```
**Fix:** <concrete, actionable remedy.>

## Findings by Dimension

### Correctness
- **🟠 Major** — `file.py:88` — <issue and fix.>
- **🟡 Minor** — `file.py:120` — <issue.>

### Architecture
- **🟠 Major** — <issue, with the design tradeoff spelled out.>

### Performance
- **🟡 Minor** — `service.go:55` — <e.g. N+1: query inside the loop; batch with a single `IN`.>

### Security
- **🔴 Critical** — <if any; otherwise fold into Critical Issues above.>

<Use severity tags consistently: 🔴 Critical, 🟠 Major, 🟡 Minor, ⚪ Nit.
Order findings within each dimension by severity, worst first.>

## What's Good

<2–4 bullets on what the change does well — genuine, specific. This calibrates the
review and signals you actually read the code, not just hunted for faults.>

## Recommendation

**<Approve | Approve with nits | Request changes | Needs discussion>**

<One or two sentences justifying the call and naming the must-fix items (if any)
that gate approval. Be direct.>
```

## The recommendation call

Pick exactly one:

- **Approve** — solid; merge as-is. No blocking issues; at most trivial nits the author can take or leave.
- **Approve with nits** — good to merge; a few minor improvements suggested but none blocking.
- **Request changes** — one or more Critical/Major issues must be resolved before merge.
- **Needs discussion** — the change raises a design or requirements question that judgment alone can't settle; a human decision is needed before it can be approved or rejected.

## Example (a clean review)

```markdown
# Code Review: Add retry logic to the payment webhook handler

**Target:** PR #482 — `sm/webhook-retries` → `master`
**Scope:** 3 files, +117/-12 — retries failed webhook deliveries with exponential backoff.

## Summary
Adds bounded exponential-backoff retries to the webhook handler. Correct and well-tested;
one minor concern about jitter. **Approve with nits.**

## Critical Issues
None.

## Findings by Dimension

### Correctness
- **🟡 Minor** — `webhook.rb:64` — Backoff has no jitter, so many handlers retrying the
  same downstream outage will synchronize (thundering herd). Add randomized jitter to the delay.

### Performance
- **🟡 Minor** — `webhook.rb:71` — The retry sleep blocks the worker thread. Fine at current
  volume; if webhook traffic grows, move retries to a scheduled job so workers aren't held.

## What's Good
- Retry count is bounded and the max-attempts path is tested — no infinite-retry risk.
- Idempotency key is checked before re-processing, so retries can't double-charge.
- Clear, behavior-focused tests covering success, transient-failure-then-success, and exhaustion.

## Recommendation
**Approve with nits** — no blocking issues. Adding jitter (webhook.rb:64) is worth doing
before merge; the worker-thread note is a future scaling consideration, not a blocker.
```
