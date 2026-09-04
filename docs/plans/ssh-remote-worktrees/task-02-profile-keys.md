---
id: "02-profile-keys"
title: "Profile keys and launch metadata"
status: pending
wave: 1
depends_on: []
plan: "plan.md"
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-001
acceptance_criteria:
  - AC-EXECUTORS-SSH-WORKTREES-001.1
  - AC-EXECUTORS-SSH-WORKTREES-001.4
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
---

# Task 02: Profile keys and launch metadata

## Summary

Introduce `ssh_materialization_mode` and `ssh_source_path_template` as
authoritative profile keys, the matching lifecycle metadata constants, and
their persistence across resume, following the existing treatment of
`ssh_shell` and `ssh_reclaim_task_dir`.

## In scope

- `profileConfigAuthoritativeKeys` in
  `apps/backend/internal/orchestrator/executor/executor_state.go`.
- `MetadataKeySSHMaterializationMode`, `MetadataKeySSHSourcePathTemplate`,
  `MetadataKeySSHMaterialization`, `MetadataKeySSHMaterializedWorktrees` and
  the `persistentMetadataKeys` entries in
  `apps/backend/internal/agent/runtime/lifecycle/executor_backend.go`.
- Template validation on profile save (`{name}` present) returning
  `ErrInvalidExecutorConfig` from the executor-profile service.

## Out of scope

- Reading the keys inside the SSH executor (Task 03) and the UI (Task 06).

## Acceptance

- A task whose metadata carries `ssh_materialization_mode=cache` launches with
  the profile's value, including an empty profile value, exactly as the
  existing `ssh_shell` test proves for that key.
- Both keys survive a simulated resume through `persistentMetadataKeys`.
- Saving an SSH profile with mode `checkout` and a template without `{name}`
  is rejected with `ErrInvalidExecutorConfig`.

## Verification

```bash
cd apps/backend && go test ./internal/orchestrator/executor/ -run 'Materialization|Authoritative|ProfileConfig' -count=1
cd apps/backend && go test ./internal/agent/runtime/lifecycle/ -run 'PersistentMetadata|MetadataKeys' -count=1
cd apps/backend && go test ./internal/task/service/ -run 'ExecutorProfile.*Materialization' -count=1
```

## Files likely touched

- `apps/backend/internal/orchestrator/executor/executor_state.go`
- `apps/backend/internal/orchestrator/executor/executor_state_materialization_test.go` (new)
- `apps/backend/internal/agent/runtime/lifecycle/executor_backend.go`
- `apps/backend/internal/task/service/service_executor_profiles.go` (or the
  file that validates profile config) and its test

## Dependencies

None.

## Risks

- The authoritative-key list is pinned by `executor_state_reclaim_test.go`;
  extend that pattern rather than weakening it.

## Parallelism

`parallel-safe` with Task 01.

## Inputs

- System design: Profile configuration.
- Existing tests: `executor_state_test.go`, `executor_state_reclaim_test.go`.

## Results

Pending.
