---
id: "06-web-settings"
title: "SSH profile materialization settings"
status: pending
wave: 2
depends_on:
  - "02-profile-keys"
plan: "plan.md"
requirements:
  - REQ-EXECUTORS-SSH-WORKTREES-001
acceptance_criteria:
  - AC-EXECUTORS-SSH-WORKTREES-001.2
system_design:
  - ../../specs/executors/system-design/ssh-worktree-materialization.md
---

# Task 06: SSH profile materialization settings

## Summary

Add the workspace-materialization card to the SSH profile page, serialize the
two keys, provide copy in all five locales, and adjust the task-create executor
hint for SSH profiles in worktree modes.

## In scope

- `ssh-workspace-materialization-card.tsx` with a mode select, a template
  input shown for `checkout`, and a validation message.
- `use-profile-runtime-form-state.ts`, `serialize-executor-config.ts`, and the
  profile page mount.
- `task-create-dialog-options.tsx` executor hint.
- Locale keys in `executors.json` and `task.json` for `en`, `pt-pt`,
  `zh-cn`, `zh-hk`, `zh-tw`.
- One Playwright scenario for the SSH profile page.

## Out of scope

- Any executor selection gating beyond the hint text; report other gates found.

## Acceptance

- Selecting `checkout` without a template shows the validation message and
  blocks save; entering a template and saving writes
  `ssh_materialization_mode=checkout` and `ssh_source_path_template` to the
  profile config, and reload shows both.
- Selecting `clone` removes both keys from the saved config.

## Verification

```bash
cd apps && pnpm install --frozen-lockfile
cd apps && pnpm --filter @kandev/web test -- serialize-executor-config ssh-workspace-materialization-card task-create-dialog-options
cd apps/web && pnpm run typecheck && pnpm run i18n:check && pnpm run lint
cd apps/web && pnpm e2e:raw -- e2e/settings/ssh-workspace-materialization.spec.ts
```

## Files likely touched

- `apps/web/components/settings/ssh-workspace-materialization-card.tsx` (new) and test
- `apps/web/components/settings/profile-edit/use-profile-runtime-form-state.ts`
- `apps/web/components/settings/profile-edit/serialize-executor-config.ts` and test
- `apps/web/app/settings/executors/[profileId]/page.tsx`
- `apps/web/components/task-create-dialog-options.tsx` and test
- `apps/web/src/locales/{en,pt-pt,zh-cn,zh-hk,zh-tw}/executors.json`, `task.json`
- `apps/web/e2e/settings/ssh-workspace-materialization.spec.ts` (new)

## Dependencies

Task 02 (key names).

## Risks

- Traditional Chinese copy must come from `pnpm run i18n:zh-hant`, not by hand.
- The e2e SSH profile fixture must exist without a live SSH host; reuse the
  pattern the reclamation toggle scenario uses.

## Parallelism

`parallel-safe` with Task 03.

## Inputs

- System design: Profile configuration, Components and responsibilities.
- Existing card: `ssh-task-dir-reclamation-card.tsx` and its test.

## Results

Pending.
