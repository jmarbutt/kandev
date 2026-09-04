---
status: draft
system: executors
created: 2026-09-04
owners:
  - kandev
---

# SSH Worktree Materialization Requirements

## Overview

An SSH executor turns a machine the user already owns into the place where agent
work runs while Kandev stays on a separate control host. Today every task on an
SSH host is a fresh full clone in its task directory. Large repositories take a
long time to start, every task carries its own object store, a checkout that
already exists on the host cannot be used, and the task cannot use capabilities
that need worktrees.

This capability adds two worktree-based materialization modes to the SSH
executor. The executor system owns it because it is environment construction
inside the SSH lifecycle defined by `REQ-EXECUTORS-SSH-EXECUTOR-001`. Repository
identity, base-branch refresh policy, branch naming, and task-owned worktree
lifetime remain with the [workspace system](../../workspaces/README.md) and the
[task system](../../tasks/README.md); this requirement references those
contracts and does not redefine them.

## Terminology

- **Materialization mode:** The per-profile strategy the SSH executor uses to
  create a task workspace on the host. The modes are `clone` (the existing
  per-task clone), `cache`, and `checkout`.
- **Host repository cache:** A Kandev-owned bare repository for one attached
  repository, kept under `<workdir_root>/repos/` on the SSH host.
- **Source checkout:** A user-owned Git working tree on the SSH host that
  serves as the worktree source for one attached repository in `checkout` mode.
- **Source path template:** The per-profile path pattern that locates the source
  checkout of each attached repository on the host. It accepts the placeholders
  `{owner}` and `{name}`, for example `~/repos/{owner}/{name}`.
- **Task worktree:** A Git worktree that Kandev creates for one task repository
  inside the task directory `<workdir_root>/tasks/<task-dir-name>`.
- **Worktree modes:** `cache` and `checkout` together.

## Requirements

### REQ-EXECUTORS-SSH-WORKTREES-001: Materialization mode selection

**Intent:** Let a user choose how an SSH profile materializes task workspaces
without changing existing profiles.

**User story:** As a developer with an always-on build machine, I want an SSH
profile to create tasks as worktrees, so that tasks start in seconds and share
one object store.

#### Acceptance criteria

- **AC-EXECUTORS-SSH-WORKTREES-001.1:** When an SSH executor profile does not
  select a materialization mode, the system shall materialize task workspaces
  exactly as it did before this requirement.
- **AC-EXECUTORS-SSH-WORKTREES-001.2:** The SSH profile editor shall offer the
  `clone`, `cache`, and `checkout` modes, shall require a source path template
  when `checkout` is selected, and shall refuse to save `checkout` without one
  or with a template that contains no `{name}` placeholder.
- **AC-EXECUTORS-SSH-WORKTREES-001.3:** When the mode of a profile changes, the
  system shall keep every existing task environment in the materialization it
  was created with, and a later session of such a task shall reuse that
  workspace without re-materializing it.
- **AC-EXECUTORS-SSH-WORKTREES-001.4:** When a task supplies a materialization
  mode or source path template in its own metadata, the system shall use the
  values from the executor profile instead.

### REQ-EXECUTORS-SSH-WORKTREES-002: Per-host repository cache

**Intent:** Share one object store per repository per host so that task
creation does not repeat a full clone.

#### Acceptance criteria

- **AC-EXECUTORS-SSH-WORKTREES-002.1:** When a `cache` task launches and no
  cache exists on that host for an attached repository, the system shall create
  a Kandev-owned cache for that repository before it creates the task worktree.
- **AC-EXECUTORS-SSH-WORKTREES-002.2:** When a cache exists, the system shall
  refresh the selected base branch in it before it creates a new task worktree.
  The refresh follows the required-materialization contract of
  `REQ-WORKSPACES-WORKTREE-BASE-REFRESH-001`: when the base branch cannot be
  obtained, the launch shall stop with a credential-safe error that names the
  repository.
- **AC-EXECUTORS-SSH-WORKTREES-002.3:** When two launches on one host use the
  same cache at the same time, each launch shall observe either a complete
  refresh or its own refresh, and neither launch shall corrupt the cache.
- **AC-EXECUTORS-SSH-WORKTREES-002.4:** The cache shall obtain remote access
  through the Task Git access policy that applies to the task, and the system
  shall add no other credential transport for it.
- **AC-EXECUTORS-SSH-WORKTREES-002.5:** The system shall never start an agent,
  terminal, or prepare script inside a cache; the agent working directory shall
  always be the task worktree.
- **AC-EXECUTORS-SSH-WORKTREES-002.6:** The cache shall remain on the host after
  its tasks are archived, deleted, or reclaimed.

### REQ-EXECUTORS-SSH-WORKTREES-003: Existing remote checkout as worktree source

**Intent:** Let tasks work from the checkout that already lives on the host
without Kandev ever modifying that checkout.

**User story:** As a developer, I want Kandev tasks on my dev box to branch from
the repository I already have there, so that no second copy is cloned and my
own checkout stays exactly as I left it.

#### Acceptance criteria

- **AC-EXECUTORS-SSH-WORKTREES-003.1:** When a `checkout` task launches, the
  system shall resolve the source checkout of each attached repository from the
  profile's source path template and shall create that repository's task
  worktree from it, so that the worktree shares the checkout's object store.
- **AC-EXECUTORS-SSH-WORKTREES-003.2:** The system shall not change the source
  checkout's current branch, index, working tree, stashes, local branches, or
  remotes when it creates, reuses, resets, or removes a task worktree. Adding or
  removing the task worktree's registration is the only permitted change.
- **AC-EXECUTORS-SSH-WORKTREES-003.3:** When a resolved source checkout path
  does not exist, is not a Git working tree, or has an `origin` that does not
  identify the attached repository, the launch shall fail before any remote
  controller starts with an error that names the repository, the resolved path,
  and the failing check.
- **AC-EXECUTORS-SSH-WORKTREES-003.4:** When pull-before-worktree is enabled for
  the repository, the system shall refresh only the remote-tracking ref of the
  base branch in the source checkout, and shall apply the local-first fallback
  and warning behavior of `REQ-WORKSPACES-WORKTREE-BASE-REFRESH-001` when the
  refresh fails or diverges.
- **AC-EXECUTORS-SSH-WORKTREES-003.5:** When a source checkout is on a branch
  other than the task base branch, or has uncommitted changes, the system shall
  still create the task worktree from the base branch and shall not report the
  checkout's own state as an error.

### REQ-EXECUTORS-SSH-WORKTREES-004: Worktree task environments on SSH hosts

**Intent:** Give SSH tasks in the worktree modes the same workspace shape and
capabilities as Worktree tasks on the Kandev host.

#### Acceptance criteria

- **AC-EXECUTORS-SSH-WORKTREES-004.1:** In a worktree mode, the system shall
  materialize the primary repository as a worktree at the task directory root
  and every additional attached repository as a worktree in a direct child
  directory, so that `git rev-parse --show-toplevel` inside the primary
  worktree returns the task directory.
- **AC-EXECUTORS-SSH-WORKTREES-004.2:** The task branch of each worktree shall
  be the branch that the workspace branch template rules
  (`REQ-WORKSPACES-WORKTREE-BRANCH-TEMPLATES-001`) would produce for a
  Worktree-executor task with the same inputs.
- **AC-EXECUTORS-SSH-WORKTREES-004.3:** The system shall record every task
  worktree on the task environment so that Changes, Review, and pull-request
  surfaces work per repository and a later session of the task reuses the same
  worktrees.
- **AC-EXECUTORS-SSH-WORKTREES-004.4:** A task with two or more attached
  repositories shall be accepted on an SSH profile in a worktree mode and shall
  receive one worktree per attached repository.
- **AC-EXECUTORS-SSH-WORKTREES-004.5:** Materialization in a worktree mode shall
  produce the same workspace whether the profile shell is `sh`, `bash`, or
  `zsh`, and shall not depend on any login-shell startup file.
- **AC-EXECUTORS-SSH-WORKTREES-004.6:** In a worktree mode, a stored profile
  prepare script shall run after materialization with the task directory as its
  working directory, and shall not replace materialization.
- **AC-EXECUTORS-SSH-WORKTREES-004.7:** When the task directory already holds a
  workspace that was not created by the selected mode or from the selected
  source, the launch shall fail before any controller starts with an error that
  names the conflict, and the system shall not delete or convert that workspace.

### REQ-EXECUTORS-SSH-WORKTREES-005: Worktree cleanup and retention

**Intent:** Reclaim task worktrees under the same guarantees as task
directories while never touching the shared cache or the user's checkout.

#### Acceptance criteria

- **AC-EXECUTORS-SSH-WORKTREES-005.1:** When reclamation removes a task
  directory that holds task worktrees, the system shall also remove their
  worktree registrations from the cache or source checkout, so that
  `git worktree list` there no longer names the removed task.
- **AC-EXECUTORS-SSH-WORKTREES-005.2:** Reclamation of a worktree task shall
  apply every safety and ownership rule of `REQ-SSH-TASKDIR-RECLAMATION-002`
  and `REQ-SSH-TASKDIR-RECLAMATION-003` to each task worktree, and shall never
  remove the cache or the source checkout.
- **AC-EXECUTORS-SSH-WORKTREES-005.3:** When the user resets the environment of a
  worktree task, the system shall remove and re-create the task worktrees from
  the current cache or source checkout and shall keep the cache and source
  checkout unchanged.
- **AC-EXECUTORS-SSH-WORKTREES-005.4:** An ordinary stop, a backend restart, and
  a reclamation that is skipped for safety shall leave task worktrees in place.

### REQ-EXECUTORS-SSH-WORKTREES-006: Materialization diagnostics

**Intent:** Make a failed materialization explain itself.

#### Acceptance criteria

- **AC-EXECUTORS-SSH-WORKTREES-006.1:** Preparation progress shall show the
  selected mode, the cache creation or refresh step, and one step per repository
  worktree.
- **AC-EXECUTORS-SSH-WORKTREES-006.2:** When a materialization step fails, the
  failed step shall carry the last Git error line with credentials redacted,
  instead of only a generic message.

## Out of scope

- Adding a branch to a running SSH task through the worktree-only add-branch
  action; that compatibility path stays with the Worktree executor.
- Eviction, garbage collection, or disk accounting for host repository caches,
  and any sweep of caches whose tasks no longer exist.
- Any change to `clone` mode behavior, including its prepare-script contract.
- Copying uncommitted host changes to the remote.
- Windows SSH targets.
- Per-task user isolation on shared hosts.
