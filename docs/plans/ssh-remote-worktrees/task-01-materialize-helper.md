---
id: "01-materialize-helper"
title: "Materialization contract and agentctl helper"
status: pending
wave: 1
depends_on: []
plan: "plan.md"
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-002
  - REQ-EXECUTORS-SSH-WORKTREES-003
  - REQ-EXECUTORS-SSH-WORKTREES-004
acceptance_criteria:
  - AC-EXECUTORS-SSH-WORKTREES-002.1
  - AC-EXECUTORS-SSH-WORKTREES-002.2
  - AC-EXECUTORS-SSH-WORKTREES-002.3
  - AC-EXECUTORS-SSH-WORKTREES-002.5
  - AC-EXECUTORS-SSH-WORKTREES-002.6
  - AC-EXECUTORS-SSH-WORKTREES-003.1
  - AC-EXECUTORS-SSH-WORKTREES-003.2
  - AC-EXECUTORS-SSH-WORKTREES-003.3
  - AC-EXECUTORS-SSH-WORKTREES-003.4
  - AC-EXECUTORS-SSH-WORKTREES-003.5
  - AC-EXECUTORS-SSH-WORKTREES-004.1
  - AC-EXECUTORS-SSH-WORKTREES-004.4
  - AC-EXECUTORS-SSH-WORKTREES-004.5
  - AC-EXECUTORS-SSH-WORKTREES-004.7
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
---

# Task 01: Materialization contract and agentctl helper

## Summary

Add the shared `remotematerialize` spec and result types and the
`agentctl materialize` subcommand that creates or refreshes a per-host bare
cache, verifies a source checkout, and adds or reuses task worktrees. This is
the only work order that touches Git on the target; everything else feeds it a
spec and consumes its result.

## In scope

- `apps/backend/internal/common/remotematerialize`: `Spec`, `Repository`,
  `Result`, `Step`, `Worktree`, JSON tags, `Validate()`.
- `apps/backend/internal/agentctl/materialize`: task-directory classification,
  cache create and refresh under `flock`, source verification, base-ref
  resolution for both modes, worktree add and reuse, the `.kandev/` move-aside,
  multi-repository rollback with prune, and redaction of `env` values.
- `apps/backend/cmd/agentctl/materialize.go`: subcommand dispatch, stdin to
  EOF, result on stdout, non-zero exit on any failed step.

## Out of scope

- Building the spec or choosing the mode (Task 03).
- Environment rows, reclamation, reset, UI, and docs.

## Acceptance

- With a real bare origin in a temp directory, `cache` mode creates the cache
  on first run, refreshes it on the second, and produces a worktree whose
  `git rev-parse --git-common-dir` is the cache; a second run against the same
  task directory reports `reused`.
- `checkout` mode creates a worktree that shares the source's object store and
  leaves the source's `HEAD`, index, working tree, stash list, and branch list
  byte-identical; a missing path, non-repository, or foreign `origin` fails
  with the repository, path, and check named, and nothing is created.
- The task-directory state table is enforced: a plain clone with a matching
  origin is reused, a foreign worktree or checkout fails without deletion, and
  a directory holding only `.kandev/` is materialized with `.kandev/` intact.

## Verification

```bash
cd apps/backend && go test ./internal/common/remotematerialize/ ./internal/agentctl/materialize/ ./cmd/agentctl/ -count=1 -race
cd apps/backend && go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.9.0 run ./internal/common/remotematerialize/... ./internal/agentctl/materialize/... ./cmd/agentctl/...
```

## Files likely touched

- `apps/backend/internal/common/remotematerialize/spec.go`, `spec_test.go`
- `apps/backend/internal/agentctl/materialize/run.go`, `classify.go`,
  `cache.go`, `source.go`, `worktree.go`, `redact.go`, and one test file
  per source file
- `apps/backend/cmd/agentctl/materialize.go`, `materialize_test.go`,
  `main.go` (dispatch only)

## Dependencies

None.

## Risks

- `git worktree add` behavior with a pre-existing empty directory and the
  `.kandev/` move-aside must be tested against the Git version in CI.
- Origin normalization must match the rules the backend uses when it
  reconciles checkout origins; reuse that helper rather than a new one.

## Parallelism

`parallel-safe` with Task 02 (disjoint packages).

## Inputs

- System design: Host repository cache, Source checkouts, Materialization spec
  and result, Task directory states.
- Existing patterns: `cmd/agentctl/github_credential.go` for subcommand
  dispatch, `internal/agentctl/server/api/workspace_materialize.go` for Git
  invocation through `subproc`, `internal/repoclone/origin.go` for origin
  normalization.

## Results

Pending.
