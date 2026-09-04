---
status: draft
system: executors
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-001
  - REQ-EXECUTORS-SSH-WORKTREES-002
  - REQ-EXECUTORS-SSH-WORKTREES-003
  - REQ-EXECUTORS-SSH-WORKTREES-004
  - REQ-EXECUTORS-SSH-WORKTREES-005
  - REQ-EXECUTORS-SSH-WORKTREES-006
created: 2026-09-04
owners:
  - kandev
---

# SSH Worktree Materialization System Design

## Purpose and boundaries

The SSH executor currently materializes the primary repository with a login-shell
prepare script that clones in place, and materializes additional repositories
after `agentctl` starts through the per-instance `/workspace/materialize-repository`
route. This design adds two worktree modes and moves their materialization into
the `agentctl` helper that Kandev already uploads to the host, so the work runs
as Go on the target instead of shell text interpreted by whichever login shell
the profile selects.

The executor system owns this design because it is environment construction
inside the SSH lifecycle. It consumes, and does not own:

- repository identity and clone locators (workspace system,
  [Local Workspace Repositories](../../workspaces/system-design/local-repositories.md));
- base-branch refresh policy
  ([Worktree Base Refresh](../../workspaces/system-design/worktree-base-refresh.md));
- task branch naming (`worktree.RenderTaskBranchName`, ADR 0032);
- task-owned worktree lifetime and the `task_environment_repos` inventory
  (ADR 2026-08-08);
- terminal-outcome reclamation
  ([Remote task-directory reclamation](remote-task-directory-reclamation.md)).

## Requirement mapping

| Requirement | Design section |
| --- | --- |
| `REQ-EXECUTORS-SSH-WORKTREES-001` | [Profile configuration](#profile-configuration), [Launch control flow](#launch-control-flow) |
| `REQ-EXECUTORS-SSH-WORKTREES-002` | [Host repository cache](#host-repository-cache), [Materializer](#agentctl-materializer) |
| `REQ-EXECUTORS-SSH-WORKTREES-003` | [Source checkouts](#source-checkouts), [Materializer](#agentctl-materializer) |
| `REQ-EXECUTORS-SSH-WORKTREES-004` | [Task directory states](#task-directory-states), [Launch control flow](#launch-control-flow), [Persistence](#persistence) |
| `REQ-EXECUTORS-SSH-WORKTREES-005` | [Reclamation and reset](#reclamation-and-reset) |
| `REQ-EXECUTORS-SSH-WORKTREES-006` | [Observability](#observability) |

## Components and responsibilities

- **`SSHExecutor` (`apps/backend/internal/agent/runtime/lifecycle`).** Resolves
  the mode from launch metadata, builds the materialization spec, runs
  `agentctl materialize` over the existing SSH connection before the user's
  prepare script, reports one preparation step per phase, and records the
  materialized worktrees on the `ExecutorInstance`. New code lives in
  `executor_ssh_materialize.go`; `executor_ssh.go` and
  `executor_ssh_operations.go` are near the file-size limits and only gain the
  call site.
- **`internal/common/remotematerialize`.** The spec and result contract shared
  by the backend (producer) and `agentctl` (consumer). Cross-tier code must live
  under `internal/common`.
- **`agentctl materialize` (`apps/backend/cmd/agentctl/materialize.go`,
  `apps/backend/internal/agentctl/materialize`).** A one-shot subcommand,
  dispatched before flag parsing like `git-credential`, that reads the spec from
  stdin, performs cache, source, and worktree operations with
  `subproc.NewGitCommand`, and prints the result to stdout.
- **Orchestrator (`internal/orchestrator/executor`).** Treats the new profile
  keys as authoritative, builds `task_environment_repos` rows from the reported
  worktrees, and skips the launch-time additional-repository reconstruction
  when the executor already materialized every repository.
- **`SSHTaskDirReclaimer` (`executor_ssh_reclaim.go`).** Probes worktrees the
  same way as clones, captures each worktree's common directory before removal,
  and prunes the registration afterwards.
- **`EnvironmentDestroyer` (`internal/task/service`).** Gains an SSH worktree
  teardown used by Reset Environment for worktree-mode environments.
- **Web settings (`apps/web/components/settings`).** A workspace-materialization
  card on the SSH profile page and its serialization into profile config.
- **Public docs.** `docs/public/executors.md` and `docs/public/feature-status.md`.

## Data and contracts

### Profile configuration

Two executor-profile config keys, both added to `profileConfigAuthoritativeKeys`
in `internal/orchestrator/executor/executor_state.go` for the same reason as
`ssh_workdir_root` and `ssh_shell`: they redirect where Kandev writes on the
host, so a task must not be able to supply them (AC-001.4).

| Key | Values | Meaning |
| --- | --- | --- |
| `ssh_materialization_mode` | `""`, `clone`, `cache`, `checkout` | Empty and `clone` keep the current prepare-script path (AC-001.1). |
| `ssh_source_path_template` | path template | Required for `checkout`. Placeholders `{owner}` and `{name}`; a leading `~` expands against the remote home. Must contain `{name}`. |

Matching lifecycle constants `MetadataKeySSHMaterializationMode` and
`MetadataKeySSHSourcePathTemplate` join `persistentMetadataKeys` so resume,
reclamation, and reset read the same values after a backend restart. The
instance also records `ssh_materialization` (the mode that actually produced
the workspace) and `ssh_materialized_worktrees` (see below).

### Host repository cache

One bare repository per repository identity per host:

```text
<workdir_root>/repos/<host>/<owner>/<name>.git
```

The segments come from the repository's clone locator with credentials and the
`.git` suffix removed and each segment passed through the same sanitizer the
host cache uses for path segments. A locator that is a filesystem path or has
no host is refused for `cache` mode; SSH hosts have no access to the control
host's filesystem. The bare repository is created with
`git clone --bare --no-tags` and its fetch refspec set to
`+refs/heads/*:refs/heads/*`, so a refresh is one `git fetch --prune`. Pull
request heads are fetched on demand as `refs/pull/<N>/head` when the task
repository names a pull request.

A lock file `<cache>.lock` taken with `flock` (Go `syscall.Flock`, so no
`flock(1)` dependency on macOS) serializes creation and refresh across
concurrent launches on one host (AC-002.3). Worktree creation needs no extra
lock: Git serializes worktree registration inside the common directory, and
task directories are distinct.

### Source checkouts

For `checkout` mode the backend resolves one absolute source path per attached
repository from the template and the repository's provider owner and name
(falling back to the last two path segments of the clone locator). `{owner}`
and `{name}` values are rejected when they contain a path separator or are
`.`/`..`. The materializer then verifies, in order: the path exists, it is a Git
working tree (`git rev-parse --is-inside-work-tree`), and its `origin` URL
identifies the same repository as the task's clone locator after the same
normalization Kandev applies when it reconciles checkout origins. Any failure
stops the launch with the repository, path, and failing check in the error
(AC-003.3).

The source checkout is opened only through `git -C <source>` commands that read
refs, fetch into `refs/remotes/origin/*`, or add, prune, and remove worktrees.
No command changes its `HEAD`, index, working tree, stashes, or local branches
(AC-003.2, AC-003.5).

### Materialization spec and result

`remotematerialize.Spec` (stdin, JSON):

| Field | Meaning |
| --- | --- |
| `mode` | `cache` or `checkout` |
| `task_dir` | Absolute remote task directory |
| `cache_root` | Absolute `<workdir_root>/repos` (cache mode) |
| `repositories[]` | `position`, `repository_id`, `task_repository_id`, `clone_url`, `owner`, `name`, `base_branch`, `checkout_branch`, `pr_number`, `task_branch`, `pull_before_worktree`, `source_path` (checkout mode), `subdir` (empty for the primary, the sanitized repository directory name otherwise) |
| `env` | The same `KEY=value` pairs the prepare script receives today; applied to every Git subprocess and used for redaction |
| `timeout_seconds` | Bounded by the preparation timeout |

`remotematerialize.Result` (stdout, JSON): `steps[]` (`name`, `status`,
`detail`, `error`) and `worktrees[]` mirroring `lifecycle.RepoWorktreeResult`
(`repository_id`, `task_repository_id`, `worktree_path`, `worktree_branch`,
`base_branch`, `requested_base_branch`, `base_branch_fallback_warning`,
`reused`, `materialization`). Every `error` and `detail` string is redacted
against the spec `env` values before it is written, so the backend never has
to parse Git output for secrets. Nothing in the spec or result is written to
disk on either side.

### Task directory states

The materializer classifies the task directory before it acts:

| State | Action |
| --- | --- |
| Missing, empty, or only `.kandev/` | Materialize. A lone `.kandev/` is moved aside during `git worktree add` and restored afterwards, because Git refuses a non-empty target. |
| Worktree whose common directory is the expected cache or source | Reuse without refresh, as the clone path reuses a matching checkout today. |
| Plain clone with a matching `origin` (created by `clone` mode) | Reuse and report `materialization=clone` (AC-001.3). |
| Worktree of a different common directory, checkout with a different `origin`, or other content | Fail with a conflict error naming the task directory and what was found; nothing is deleted or converted (AC-004.7). |

Additional repositories use the same table on `<task_dir>/<subdir>`.

### Instance and environment records

`ExecutorInstance.Metadata["ssh_materialized_worktrees"]` carries the result
`worktrees[]`. The lifecycle launch path copies it into an `EnvPrepareResult`
so `environmentReposForLaunch` in `executor_execute.go` builds one
`task_environment_repos` row per worktree with `worktree_path` and
`worktree_branch` set (AC-004.3), exactly as a host Worktree launch does.
`WorktreeID` stays empty for remote worktrees; consumers already treat it as
optional. `validateReuseEnvironmentInventory` compares the canonical inventory
by repository and branch slug, which the rows now carry.

## Launch control flow

`SSHExecutor.CreateInstance` keeps its step order. The new work sits between
`maybeUploadCredentials` and `runPrepareScript`:

1. Resolve the mode. Empty or `clone` runs the unchanged prepare-script path.
2. Build the spec: absolute task directory and cache root from
   `expandRemoteHome(workdirRoot)`, one entry per repository spec in the
   launch request (primary first), the task branch already computed on the
   control host, the environment from `sshRemoteContributionEnv`, and the
   resolved source paths in `checkout` mode.
3. Run `WrapLoginShell(shell, "<agentctl> materialize")` through
   `runSSHCommandStdin` with the spec on stdin. The login shell supplies `PATH`
   for `git`; `agentctl` reads stdin to EOF itself, so the shell never parses
   the spec. The command runs under the preparation timeout.
4. Parse the result. Report each `steps[]` entry through `report`, so the UI
   shows the mode, the cache step, and one step per repository (AC-006.1). A
   failed result fails the launch with the failed step's redacted error
   (AC-006.2) before any controller starts.
5. When the profile has a prepare script, run it as today with the task
   directory as the working directory, after materialization (AC-004.6). The
   managed default prepare script is not used in worktree modes.
6. Continue with `verifyPrimaryCheckout`, the session directory, and
   `agentctl` start unchanged. The `.kandev/sessions/<id>` directory is created
   after materialization, which is what keeps the task directory empty for the
   first `git worktree add`.
7. In `manager_launch.go`, skip `materializeWorkspaceRepositories` when the
   instance reports worktrees for every repository spec; the instance
   materialized them before `agentctl` started.

Remote contributions (`materializeSSHRemoteContribution`) run after
preparation as today. A worktree shares its `origin` configuration with the
cache or source, so the contribution script sees the same remote.

The materializer works in cache mode as: lock, create or refresh the cache,
resolve the base ref as `refs/heads/<base>` in the cache (strict; a missing or
unfetched branch fails the repository, AC-002.2), then `git worktree add`
either reusing an existing task branch or creating it from the base ref. In
checkout mode: verify the source, fetch
`+refs/heads/<base>:refs/remotes/origin/<base>` when `pull_before_worktree`
is set, choose `refs/remotes/origin/<base>` when it contains the local `<base>`
or no local `<base>` exists, otherwise the local `<base>` with a fallback
warning (AC-003.4), then `git worktree add` from the source. A multi-repository
failure removes the worktrees this run created and prunes their registrations
before returning, so a retry starts from the same state.

## Failure and recovery

- **Cache refresh fails** (network, credentials, missing branch): the launch
  stops, the error names the repository and the redacted Git line. Nothing
  else changes on the host.
- **Source checkout invalid**: the launch stops before any controller starts.
- **Conflict in the task directory**: the launch stops; the directory is left
  as found so the user can inspect or remove it.
- **Timeout**: the SSH command is cancelled by the preparation context; a
  partially created cache holds the lock only for the process lifetime, and the
  next launch retries creation under the lock.
- **Backend restart or ordinary stop**: worktrees stay in place; resume follows
  the existing SSH resume path and reuses the task directory.
- **Older cached helper**: the content-hash upload replaces an `agentctl` that
  lacks the subcommand before materialization runs.

## Reclamation and reset

Reclamation keeps its trigger, ownership resolution, and safety probes. The
reclaimer changes in three places (AC-005.1, AC-005.2):

- `discoverCheckouts` matches `.git` files as well as directories, which `find
  -name .git` already does; `probeCheckout` runs unchanged inside a worktree.
- Before removal, the reclaimer records `git rev-parse --git-common-dir` for the
  task directory and each child checkout.
- After `rm -rf` succeeds, it runs `git -C <common-dir> worktree prune` for each
  recorded common directory. Prune only drops registrations whose directory is
  gone, so it can never touch another task's worktree, the cache contents, or
  the source checkout's own state. A prune failure is reported as a reclamation
  error, not hidden.

Reset Environment for a worktree-mode environment adds
`EnvironmentDestroyer.DestroySSHWorktrees(ctx, env)`. It resolves the executor
record and profile for `env.ExecutorID`, the task directory from
`env.WorkspacePath` (the SSH executor persists the remote task directory there
on success), and runs the reclaimer with the same probes. A dirty or unpushed
worktree makes reset refuse with the probe reason (AC-005.3 with
`REQ-SSH-TASKDIR-RECLAMATION-002`); the next launch re-materializes from the
current cache or source. The cache and source checkout are never targets.

## Persistence

- Profile keys are plain `executor_profiles.config` entries; absent keys mean
  `clone`, so existing rows need no migration.
- `task_environment_repos` rows for remote worktrees carry remote absolute
  paths in `worktree_path`. Nothing else reads that column as a host path for
  SSH environments; the host worktree store filters by `worktree_id`.
- `task_environments.workspace_path` holds the remote task directory for SSH
  environments; reset depends on it.
- `executors_running.metadata` carries the mode and worktree list through the
  persistent metadata keys.

## Security

- The spec travels on stdin and the result on stdout of an SSH exec channel;
  neither is written to the remote filesystem or passed as process arguments.
- Cache paths are built only from sanitized identity segments under
  `<workdir_root>/repos`; template output is validated as an absolute path
  after `~` expansion, and `{owner}`/`{name}` values cannot contain separators.
- Git credentials reach the materializer exactly as they reach the prepare
  script today: the managed broker helper configuration or the profile token,
  through `env`. No new transport (AC-002.4).
- The reclaimer's path guard still requires the task directory to be one
  segment under `<workdir_root>/tasks/`; prune targets are the recorded common
  directories, never user-supplied paths.
- The host's trust statements for SSH stay in force: the SSH user can read the
  cache and every worktree.

## Observability

- Preparation steps: `Materializing workspace (cache|checkout)`,
  `Refreshing repository cache <name>` or `Verifying source checkout <name>`,
  and `Creating worktree <name>` with `reused` in the detail when applicable.
- Structured backend logs `ssh.materialize.started|completed|failed` with mode,
  task directory, repository count, duration, and the failed step name; no
  URLs with credentials.
- The materializer redacts every spec `env` value from step output before
  printing, and the backend applies `redactSSHScriptOutput` again as defence
  in depth.

## Testing

- Materializer unit tests with real Git repositories in temp directories:
  cache creation, refresh under a held lock, task-branch reuse, checkout-mode
  source verification and read-only guarantees (assert `HEAD`, index, stash,
  and branch list unchanged), the task-directory state table, multi-repository
  rollback, and redaction.
- SSH executor tests with the in-process fake SSH server: spec on stdin, step
  reporting, prepare-script ordering, conflict and failure propagation, and the
  launch-time reconstruction skip.
- Orchestrator tests: authoritative keys, environment rows from worktree
  results, reuse validation.
- Reclaimer tests with the fake runner: common-directory capture and prune
  ordering.
- Web unit tests for serialization and the card; one Playwright scenario for
  the SSH profile page.

## Related decisions

- [ADR 0032: Configurable worktree branch names](../../../decisions/0032-configurable-worktree-branch-names.md)
- [ADR 2026-08-08: Keep worktree ownership at the task lifecycle](../../../decisions/2026-08-08-task-owned-worktree-lifetime.md)
- [ADR 0025: Runtime cleanup uses `executors_running`](../../../decisions/0025-runtime-cleanup-uses-executors-running.md)
- [ADR 2026-07-20: Provider-neutral remote repositories](../../../decisions/2026-07-20-provider-neutral-remote-repositories.md)
