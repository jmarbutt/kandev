---
id: "05-reclaim-and-reset"
title: "Reclamation prune and reset teardown"
status: pending
wave: 3
depends_on:
  - "03-ssh-executor-step"
plan: "plan.md"
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-005
acceptance_criteria:
  - AC-EXECUTORS-SSH-WORKTREES-005.1
  - AC-EXECUTORS-SSH-WORKTREES-005.2
  - AC-EXECUTORS-SSH-WORKTREES-005.3
  - AC-EXECUTORS-SSH-WORKTREES-005.4
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
---

# Task 05: Reclamation prune and reset teardown

## Summary

Make the SSH reclaimer capture each checkout's common directory and prune its
registration after removal, and let Reset Environment tear down SSH worktrees
through the same guarded path.

## In scope

- `executor_ssh_reclaim.go`: common-directory capture before removal, prune
  after a confirmed removal, prune failure reported as an error.
- `EnvironmentDestroyer.DestroySSHWorktrees` in `internal/task/service`, its
  backendapp implementation, and the SSH branch of
  `teardownEnvironmentResources` for worktree-mode environments.

## Out of scope

- Cache removal or any change to the source checkout beyond registrations.
- The reclamation trigger, ownership resolution, and safety probes.

## Acceptance

- With the fake runner, reclaiming a worktree task directory issues the
  common-directory query for the root and each child checkout before
  `rm -rf`, and `git worktree prune` for each recorded common directory after
  the removal is confirmed; a skipped probe issues no prune.
- Reset of an SSH worktree-mode environment calls `DestroySSHWorktrees`, which
  refuses when a probe reports unsafe state and otherwise removes and prunes.

## Verification

```bash
cd apps/backend && go test ./internal/agent/runtime/lifecycle/ -run 'Reclaim' -count=1 -race
cd apps/backend && go test ./internal/task/service/ -run 'ResetTaskEnvironment|SSHWorktree' -count=1 -race
```

## Files likely touched

- `apps/backend/internal/agent/runtime/lifecycle/executor_ssh_reclaim.go`
- `apps/backend/internal/agent/runtime/lifecycle/executor_ssh_reclaim_worktree_test.go` (new)
- `apps/backend/internal/task/service/service_task_environments.go`
- `apps/backend/internal/backendapp/ssh_reclaim.go`

## Dependencies

Task 03 (mode and worktree metadata on the instance).

## Risks

- `executor_ssh_reclaim_test.go` asserts remote commands verbatim; new
  commands need their own assertions without changing the existing ones.

## Parallelism

`parallel-safe` with Task 04.

## Inputs

- System design: Reclamation and reset.
- Remote task-directory reclamation system design.

## Results

Pending.
