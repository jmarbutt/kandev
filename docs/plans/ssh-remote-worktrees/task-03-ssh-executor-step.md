---
id: "03-ssh-executor-step"
title: "SSH executor materialization step"
status: pending
wave: 2
depends_on:
  - "01-materialize-helper"
  - "02-profile-keys"
plan: "plan.md"
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-001
  - REQ-EXECUTORS-SSH-WORKTREES-002
  - REQ-EXECUTORS-SSH-WORKTREES-004
  - REQ-EXECUTORS-SSH-WORKTREES-006
acceptance_criteria:
  - AC-EXECUTORS-SSH-WORKTREES-001.1
  - AC-EXECUTORS-SSH-WORKTREES-001.3
  - AC-EXECUTORS-SSH-WORKTREES-002.4
  - AC-EXECUTORS-SSH-WORKTREES-002.5
  - AC-EXECUTORS-SSH-WORKTREES-004.2
  - AC-EXECUTORS-SSH-WORKTREES-004.5
  - AC-EXECUTORS-SSH-WORKTREES-004.6
  - AC-EXECUTORS-SSH-WORKTREES-006.1
  - AC-EXECUTORS-SSH-WORKTREES-006.2
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
---

# Task 03: SSH executor materialization step

## Summary

Run `agentctl materialize` from `SSHExecutor.CreateInstance` in the worktree
modes, feed it the spec on stdin, report its steps, run a stored prepare script
afterwards, record the worktrees on the instance, and stop the launch path from
re-materializing additional repositories after `agentctl` starts.

## In scope

- `executor_ssh_materialize.go`: mode resolution, spec construction (repository
  specs, owner and name, template expansion, `env`), command execution, result
  parsing, step reporting, instance metadata.
- The call site in `CreateInstance` between `maybeUploadCredentials` and
  `runPrepareScript`; worktree modes skip the default prepare script and run
  only a stored profile script.
- `manager_launch.go`: skip `materializeWorkspaceRepositories` when the
  instance reports a worktree for every repository spec; expose the worktrees
  on the launch result as an `EnvPrepareResult`.

## Out of scope

- Git operations on the target (Task 01) and environment rows (Task 04).

## Acceptance

- With the fake SSH server, a `cache` launch sends exactly one
  `<shell> -lc '<agentctl> materialize'` command with the JSON spec on stdin
  before any prepare-script command, and the spec's `env` equals what the
  prepare script would have received.
- A failed result step fails the launch before any `nohup` command with the
  step's redacted error, and a successful result produces preparation steps
  named for the mode, the cache or source step, and each repository.
- An empty or `clone` mode sends no `materialize` command and behaves as the
  existing lifecycle test asserts.

## Verification

```bash
cd apps/backend && go test ./internal/agent/runtime/lifecycle/ -run 'SSHMaterializ|SSHExecutorCreateInstance|SSHExecutorRunPrepareScript' -count=1 -race
cd apps/backend && go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.9.0 run ./internal/agent/runtime/lifecycle/...
```

## Files likely touched

- `apps/backend/internal/agent/runtime/lifecycle/executor_ssh_materialize.go` (new)
- `apps/backend/internal/agent/runtime/lifecycle/executor_ssh_materialize_remote_test.go` (new)
- `apps/backend/internal/agent/runtime/lifecycle/executor_ssh.go` (call site only)
- `apps/backend/internal/agent/runtime/lifecycle/executor_ssh_scripts.go`
  (prepare script as post-hook in worktree modes)
- `apps/backend/internal/agent/runtime/lifecycle/manager_launch.go`

## Dependencies

Task 01 (spec types and result shape), Task 02 (metadata constants).

## Risks

- The repository spec type on the launch request may lack owner and name;
  extend it from the task repository rows rather than parsing the locator
  only.
- File-size limits on `executor_ssh.go`; keep the call site to a few lines.

## Parallelism

`sequential` after Tasks 01 and 02; `parallel-safe` with Task 06.

## Inputs

- System design: Launch control flow, Instance and environment records,
  Observability.
- Existing tests: `executor_ssh_lifecycle_test.go`,
  `executor_ssh_scripts_remote_test.go`, `ssh_fake_server_test.go`.

## Results

Pending.
