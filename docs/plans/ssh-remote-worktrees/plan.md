---
created: 2026-09-04
status: draft
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-001
  - REQ-EXECUTORS-SSH-WORKTREES-002
  - REQ-EXECUTORS-SSH-WORKTREES-003
  - REQ-EXECUTORS-SSH-WORKTREES-004
  - REQ-EXECUTORS-SSH-WORKTREES-005
  - REQ-EXECUTORS-SSH-WORKTREES-006
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
legacy_specs: []
---

# Implementation Plan: SSH Remote Worktrees

## Overview

Give SSH executor profiles two worktree materialization modes, `cache` and
`checkout`, executed by a new `agentctl materialize` subcommand on the target
instead of the login-shell prepare script. Build from the inside out: the shared
contract and the helper first, because every other slice consumes them; then
the profile keys; then the SSH executor step that runs the helper; then the
orchestrator inventory rows, reclamation and reset, the settings UI, and the
public docs. Tracked upstream as kdlbs/kandev#3386.

## Scope

### In scope

- `agentctl materialize`: per-host bare cache, source-checkout verification,
  worktree add and reuse, task-directory conflict detection, multi-repository
  rollback, redaction, and the stdin/stdout contract.
- Profile keys `ssh_materialization_mode` and `ssh_source_path_template` as
  authoritative, persisted launch metadata.
- SSH executor: materialization step, step reporting, prepare script as a
  post-materialization hook, worktree results on the instance, and the skip of
  the launch-time additional-repository reconstruction.
- `task_environment_repos` rows with remote worktree paths and branches, and
  reuse validation over them.
- Reclamation prune of worktree registrations; Reset Environment teardown of
  SSH worktrees through the reclaimer.
- SSH profile settings card, serialization, five-locale copy, task-create
  executor hint, one Playwright scenario.
- Public docs for the new modes.

### Out of scope

- Any change to `clone` mode or its prepare-script contract.
- The worktree-only add-branch action on SSH tasks.
- Cache eviction, disk accounting, or orphan sweeps.
- Windows targets and per-task user isolation.

## Technical approach

### Shared contract and helper (`internal/common/remotematerialize`, `internal/agentctl/materialize`, `cmd/agentctl`)

- `remotematerialize.Spec`, `Repository`, `Result`, `Step`, and `Worktree`
  types with JSON tags and `Validate()` (mode, absolute paths, `{name}` in the
  template is validated by the backend before the spec is built).
- `materialize.Run(ctx, spec, io)` in a new package: `classifyTaskDir`,
  `ensureCache` (flock via `syscall.Flock` on `<cache>.lock`, `git clone
  --bare --no-tags`, fetch refspec `+refs/heads/*:refs/heads/*`, `git fetch
  --prune`, optional `refs/pull/<N>/head`), `verifySource` (exists, working
  tree, normalized `origin` equality), `resolveBaseRef` (strict in cache mode;
  local-first with fallback warning in checkout mode), `addWorktree` (reuse a
  registered worktree with the expected common dir; reuse an existing task
  branch; otherwise `git worktree add -b <task_branch> <dir> <base_ref>`; move a
  lone `.kandev/` aside during add), `rollback` (remove worktrees created by
  this run and prune), and `redact` (spec `env` values).
- Every Git call goes through `subproc.NewGitCommand` with a lifecycle class,
  with `spec.env` applied to the command environment.
- `cmd/agentctl/materialize.go`: dispatch `os.Args[1] == "materialize"` next to
  the `git-credential` dispatch, read stdin to EOF, run, write the result JSON
  to stdout, exit non-zero when any step failed.

### Profile keys and metadata (`internal/orchestrator/executor`, lifecycle `executor_backend.go`)

- Add `ssh_materialization_mode` and `ssh_source_path_template` to
  `profileConfigAuthoritativeKeys`.
- Add `MetadataKeySSHMaterializationMode`, `MetadataKeySSHSourcePathTemplate`,
  `MetadataKeySSHMaterialization`, and `MetadataKeySSHMaterializedWorktrees` to
  the lifecycle constants and the first three to `persistentMetadataKeys`.
- Validate the template shape (`{name}` present, no NUL) where the profile is
  saved, returning `ErrInvalidExecutorConfig`.

### SSH executor (`internal/agent/runtime/lifecycle/executor_ssh_materialize.go`)

- `sshMaterializationMode(metadata)` returns `clone` for empty or unknown
  values; unknown non-empty values fail the launch with a clear error.
- `buildSSHMaterializeSpec(req, taskDir, workdirRoot, remoteHome)` maps the
  launch request repository specs (primary first) to spec repositories,
  resolves `{owner}`/`{name}` from provider identity or the clone locator,
  expands `~` with the remote home already resolved for the task directory,
  and passes `sshRemoteContributionEnv(req, agentctlBin)` as `env`.
- `runSSHMaterialize(ctx, client, shell, agentctlBin, spec)` wraps
  `WrapLoginShell(shell, agentctlBin+" materialize")` in `runSSHCommandStdin`,
  parses the result, and reports steps through `report`.
- `CreateInstance`: call the step after `maybeUploadCredentials`; in worktree
  modes run only a stored profile prepare script (no default script) after
  materialization; store `ssh_materialization` and
  `ssh_materialized_worktrees` on the instance metadata.
- `manager_launch.go`: skip `materializeWorkspaceRepositories` when the
  instance metadata lists a worktree for every repository spec, and surface
  those worktrees as an `EnvPrepareResult` on the launch result.

### Orchestrator inventory (`internal/orchestrator/executor/executor_execute.go`)

- Map the SSH worktree results into `LaunchAgentResponse.Worktrees` so
  `environmentReposForLaunch` emits rows with `worktree_path` and
  `worktree_branch`; keep `WorktreeID` empty.
- Confirm `computeWorkspacePath` persists the remote task directory for SSH
  environments; if `resp.WorkspacePath` is empty for SSH, set it from the
  instance workspace path.
- Extend `validateReuseEnvironmentInventory` tests to cover rows without
  `worktree_id`.

### Reclamation and reset (`executor_ssh_reclaim.go`, `internal/task/service`)

- Reclaimer: capture `git rev-parse --git-common-dir` per discovered checkout
  before removal; after a confirmed removal run `git -C <common> worktree
  prune`; report prune failures as errors.
- `EnvironmentDestroyer.DestroySSHWorktrees(ctx, env)` implemented in the
  backendapp adapter with the reclaimer, resolving the executor record and
  profile from `env.ExecutorID`; `teardownEnvironmentResources` calls it for SSH
  environments whose metadata mode is a worktree mode.

### Web (`apps/web/components/settings`, task-create dialog)

- `ssh-workspace-materialization-card.tsx`: Radix `Select` for the mode and a
  text input for the template shown only for `checkout`, with help copy and
  `data-testid` hooks; validation message when `checkout` has no template.
- `use-profile-runtime-form-state.ts` and `serialize-executor-config.ts`:
  `sshMaterializationMode`, `sshSourcePathTemplate`, `setTextConfig` for both.
- `task-create-dialog-options.tsx`: extend `computeExecutorHint` so an SSH
  profile in a worktree mode is not described as unsuitable for multiple
  repositories.
- Locale files `executors.json` and `task.json` in `en`, `pt-pt`, `zh-cn`,
  `zh-hk`, `zh-tw`.

### Public docs

- `docs/public/executors.md`: SSH materialization modes, cache path, template
  placeholders, prepare-script ordering, reclamation prune, and reset.
- `docs/public/feature-status.md`: SSH executor row and the multi-repository
  row.

## Tests

| Acceptance criteria | Evidence |
| --- | --- |
| AC-001.1, AC-001.3 | `executor_ssh_materialize_remote_test.go`: empty mode runs the prepare-script path; existing clone reused under `cache` |
| AC-001.2 | `ssh-workspace-materialization-card.test.tsx`, `serialize-executor-config.test.ts`, Playwright scenario |
| AC-001.4 | `executor_state_materialization_test.go`: authoritative overwrite |
| AC-002.1, AC-002.2, AC-002.6 | `materialize/cache_test.go`: create, refresh, strict missing-branch failure, cache survives worktree removal |
| AC-002.3 | `materialize/cache_test.go`: refresh blocks on a held lock |
| AC-002.4 | `executor_ssh_materialize_remote_test.go`: spec `env` equals the prepare-script environment |
| AC-002.5 | `materialize/run_test.go`: worktree paths never equal cache paths; `executor_ssh_materialize_remote_test.go`: instance workspace path is the task directory |
| AC-003.1 to AC-003.5 | `materialize/source_test.go`: shared objects, unchanged source state, invalid path/origin errors, local-first base selection, dirty source tolerated |
| AC-004.1, AC-004.4 | `materialize/run_test.go`: primary at root, additional in child directories |
| AC-004.2 | `executor_ssh_materialize_remote_test.go`: spec task branch equals `MetadataKeyWorktreeBranch` |
| AC-004.3 | `executor_execute_remote_worktrees_test.go`: rows carry path and branch |
| AC-004.5 | `materialize/run_test.go` runs without any shell; `executor_ssh_materialize_remote_test.go` asserts the command shape under `bash` and `zsh` |
| AC-004.6 | `executor_ssh_materialize_remote_test.go`: prepare script command follows the materialize command |
| AC-004.7 | `materialize/classify_test.go`: conflict states fail without deletion |
| AC-005.1, AC-005.2, AC-005.4 | `executor_ssh_reclaim_worktree_test.go`: prune after removal, skip leaves worktrees |
| AC-005.3 | `service_task_environments_ssh_test.go`: reset calls `DestroySSHWorktrees`, refuses when probes fail |
| AC-006.1, AC-006.2 | `executor_ssh_materialize_remote_test.go`: step names and redacted error |

## E2E tests

- `apps/web/e2e/settings/ssh-workspace-materialization.spec.ts` (default web
  project): open an SSH profile, select `checkout` without a template and see
  the validation message, enter a template, save, reload, and see the saved
  values. Covers AC-001.2.

## Work orders

- [ ] [Task 01: Materialization contract and agentctl helper](task-01-materialize-helper.md)
- [ ] [Task 02: Profile keys and launch metadata](task-02-profile-keys.md)
- [ ] [Task 03: SSH executor materialization step](task-03-ssh-executor-step.md)
- [ ] [Task 04: Environment inventory from remote worktrees](task-04-environment-inventory.md)
- [ ] [Task 05: Reclamation prune and reset teardown](task-05-reclaim-and-reset.md)
- [ ] [Task 06: SSH profile materialization settings](task-06-web-settings.md)
- [ ] [Task 07: Public documentation](task-07-public-docs.md)

## Verification results

Pending.

## Risks

- `git worktree add` refuses a non-empty directory; the `.kandev/` move-aside
  must be exercised by a test that pre-creates the runtime directory.
- The launch request's repository spec type may not carry owner and name; if it
  does not, Task 03 extends it from the task repository rows before building
  the spec.
- `task_environments.workspace_path` may be empty for SSH environments today;
  Task 04 verifies and fixes that before Task 05 relies on it.
- Backend file-size limits: `executor_ssh.go` and `executor_ssh_operations.go`
  are near their caps; new logic must go into the new files named above.
- Contributor process: kdlbs/kandev#3386 must have maintainer discussion before
  a pull request opens.

## Open questions

- Whether the task-create dialog should also stop hiding the second-repository
  control for SSH profiles in worktree modes, or only change the hint text.
  Task 06 changes the hint and reports what else gates the control.
