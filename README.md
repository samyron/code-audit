# code-audit

A Claude Code skill that performs a thorough, human-quality code review of a change set
and produces a structured report ending in a clear **approve / request-changes**
recommendation.

It reviews across four dimensions — **correctness, architecture, performance, security** —
and works on:

- a **GitHub PR** (URL or number),
- a **commit SHA** or range, or
- **uncommitted local changes** on the current branch.

For remote branches it reviews from an isolated `git worktree` so it never disturbs
in-progress work. It reads project conventions (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`)
and PR discussion before judging, and stays out of the linter's lane — no formatting nits.

## Requirements

- `git` and the GitHub CLI (`gh`, authenticated) available on your PATH.

## Install

```
/plugin marketplace add samyron/code-audit
/plugin install code-audit@samyron-plugins
```

(Replace `samyron/code-audit` with the GitHub `owner/repo` you push this to. The
`@samyron-plugins` suffix is the marketplace name from `marketplace.json`.)

## Use

Once installed, trigger it naturally:

- "audit this PR https://github.com/org/repo/pull/123"
- "audit my branch"
- "review this commit abc123"
- "review my changes"

## Layout

```
code-audit/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # single-plugin marketplace manifest
└── skills/
    └── code-audit/
        ├── SKILL.md             # the review workflow
        └── references/
            ├── review-checklist.md   # per-dimension checklist
            └── report-template.md    # report structure + example
```
