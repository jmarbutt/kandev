---
id: "07-public-docs"
title: "Public documentation"
status: pending
wave: 4
depends_on:
  - "03-ssh-executor-step"
  - "05-reclaim-and-reset"
  - "06-web-settings"
plan: "plan.md"
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-001
  - REQ-EXECUTORS-SSH-WORKTREES-002
  - REQ-EXECUTORS-SSH-WORKTREES-003
  - REQ-EXECUTORS-SSH-WORKTREES-005
acceptance_criteria:
  - AC-EXECUTORS-SSH-WORKTREES-001.1
  - AC-EXECUTORS-SSH-WORKTREES-002.6
  - AC-EXECUTORS-SSH-WORKTREES-003.2
  - AC-EXECUTORS-SSH-WORKTREES-005.1
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
---

# Task 07: Public documentation

## Summary

Document the SSH materialization modes, the cache location, the source path
template, prepare-script ordering, reclamation prune, and reset behavior in
the public executor docs and the feature status page.

## In scope

- `docs/public/executors.md` SSH sections.
- `docs/public/feature-status.md` SSH executor and multi-repository rows.

## Out of scope

- New pages or `meta.json` changes.

## Acceptance

- The executors page states the three modes, the default, the cache path
  pattern, the template placeholders, that the user's checkout is never
  modified beyond worktree registrations, and what reclamation and reset do to
  worktrees.
- The public docs validators pass.

## Verification

```bash
node --test scripts/validate-public-docs.test.mjs
node scripts/validate-public-docs.mjs
```

## Files likely touched

- `docs/public/executors.md`
- `docs/public/feature-status.md`

## Dependencies

Tasks 03, 05, and 06 (final behavior and setting names).

## Risks

None.

## Parallelism

`sequential`

## Inputs

- Requirements document and system design.
- `docs/public/README.md` for the docs contract.

## Results

Pending.
