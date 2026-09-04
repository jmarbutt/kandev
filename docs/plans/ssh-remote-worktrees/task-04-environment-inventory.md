---
id: "04-environment-inventory"
title: "Environment inventory from remote worktrees"
status: pending
wave: 3
depends_on:
  - "03-ssh-executor-step"
plan: "plan.md"
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-004
acceptance_criteria:
  - AC-EXECUTORS-SSH-WORKTREES-004.3
  - AC-EXECUTORS-SSH-WORKTREES-004.4
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
---

# Task 04: Environment inventory from remote worktrees

## Summary

Turn the worktrees an SSH launch reports into `task_environment_repos` rows
with remote paths and branches, persist the remote task directory as the
environment workspace path, and keep reuse validation working for rows without
a host worktree ID.

## In scope

- Mapping instance worktrees to `LaunchAgentResponse.Worktrees` in
  `executor_execute.go` for SSH launches.
- `computeWorkspacePath` and `persistTaskEnvironment` behavior for SSH
  environments.
- `validateReuseEnvironmentInventory` coverage for rows with empty
  `worktree_id`.

## Out of scope

- The SSH executor and the helper (Tasks 01 and 03).

## Acceptance

- A launch response carrying two SSH worktrees persists two
  `task_environment_repos` rows with the remote `worktree_path` and
  `worktree_branch` and an empty `worktree_id`, and the environment is
  `ready` with `workspace_path` equal to the remote task directory.
- A second launch of that task passes `validateReuseEnvironmentInventory`.

## Verification

```bash
cd apps/backend && go test ./internal/orchestrator/executor/ -run 'RemoteWorktree|EnvironmentReposForLaunch|ReuseEnvironmentInventory' -count=1 -race
```

## Files likely touched

- `apps/backend/internal/orchestrator/executor/executor_execute.go`
- `apps/backend/internal/orchestrator/executor/executor_environment_reuse.go`
- `apps/backend/internal/orchestrator/executor/executor_execute_remote_worktrees_test.go` (new)

## Dependencies

Task 03.

## Risks

- Storage inventory or Changes-panel code may assume `worktree_path` is a host
  path; confirm the SSH branch of those readers ignores it or treats it as
  remote.

## Parallelism

`parallel-safe` with Task 05 (disjoint packages).

## Inputs

- System design: Instance and environment records, Persistence.
- ADR 2026-08-08 (task-owned worktree lifetime).

## Results

Pending.
