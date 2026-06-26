---
description: Cut a release — open develop → main PR, wait for green CI, merge as merge-commit, and let project-status.yml move closed issues to Production
---

Promote everything currently on `develop` to `main` via a release PR, wait for CI, merge as a **merge commit** (not squash), and rely on `.github/workflows/project-status.yml` to move each linked issue's project card to **Production**.

Before any `mcp__github__*` call, resolve `<OWNER>` and `<REPO>` from `git remote get-url origin` (format `https://github.com/<OWNER>/<REPO>.git`) — this file is template-shaped and must not hardcode a specific repo.

## Project config — read this first

This command ships in the shared `issue-flow` plugin, so its body is stack-agnostic. The project-specific values live in **`.claude/flow.config.md`** at the repo root. **Read that file before acting** and substitute its values wherever this command shows a placeholder literal:

- branch names → `DEV_BRANCH` / `PROD_BRANCH` (the literals `develop` / `main` in this file are placeholders)
- CI check to wait on → `CI_CHECK_NAME`
- deploy verification → run step 5.5 only if `DEPLOY_VERIFY` is not `none`, dispatching `DEPLOY_VERIFY_SKILL`
- user-facing chat language → `USER_LANGUAGE`

If `.claude/flow.config.md` is missing, stop and ask the user to create it from the plugin's `templates/flow.config.example.md`.

Sibling commands are namespaced under the plugin: `/issue-flow:plan-issue`, `/issue-flow:work-on-issue`, `/issue-flow:report`, `/issue-flow:push-to-prod`.

## Mode

Autonomous, same spirit as `/work-on-issue`. Stop and ask only at a real fork:

- `develop` is not ahead of `main` (nothing to release);
- more than one open `develop → main` PR (unusual state, likely a stale draft);
- release-PR CI is red — fixes belong in a normal feature PR into `develop`, not direct edits on the release branch;
- merge fails due to a conflict with `main` (a product decision when it happens).

## Steps

### 1. Verify there is something to release

```bash
git fetch origin main develop
git log --oneline origin/main..origin/develop
```

If the log is empty — stop. Tell the user `develop is even with main; nothing to release.` and exit.

### 2. Collect `Closes #N` markers from the PRs merged into develop

A feature PR links its issue with `Closes #N` in the **PR body**, which GitHub turns into a native link. That keyword does **not** reliably reach any commit message on `develop` — a squash merge keeps only the PR title + `(#PR)` in the subject and drops the body's `Closes` line. Scraping commit text alone therefore silently misses those issues and leaves their cards stuck in **Develop** at release time. Derive the set from the **PR bodies**, which is where the link actually lives.

First, list the PR numbers merged into develop in this window — both merge-commit (`Merge pull request #N`) and squash (`… (#N)`) subjects carry the number:

```bash
git log origin/main..origin/develop --pretty=format:%s%n%b \
  | grep -oiE '(merge pull request #[0-9]+|\(#[0-9]+\))' \
  | grep -oE '[0-9]+' \
  | sort -un
```

Save these as `<PRS>`. It is a superset: a subject like `… (#87) (#91)` contributes both the issue `#87` and the PR `#91`. Non-PR numbers are harmless — they are skipped in the next step.

Then, for every number `n` in `<PRS>`, read the PR via MCP and union its closing refs:

```
mcp__github__pull_request_read(method=get, owner=<OWNER>, repo=<REPO>, pullNumber=<n>)
```

- If `n` is not a pull request (404 / it is an issue) — skip it.
- From the returned `body`, extract issue numbers with the **same** regex `.github/workflows/project-status.yml` uses (case-insensitive, word-boundary on `#`): `(close[sd]?|fix(es|ed)?|resolve[sd]?)[[:space:]]+#[0-9]+`.
- Union every match across all PR bodies.

Also union the legacy commit-text scrape as a belt-and-suspenders fallback — it catches a `Closes #N` written directly in a commit message that has no owning PR body:

```bash
git log origin/main..origin/develop --pretty=format:%B \
  | grep -oiE '(close[sd]?|fix(es|ed)?|resolve[sd]?)[[:space:]]+#[0-9]+' \
  | grep -oE '#[0-9]+' \
  | sort -u
```

The union of the PR-body refs and this fallback is `<CLOSES>`. Do **not** use commit-text alone — that is the bug this step replaces (see #103).

If the set is empty — warn the user that the project-board move will be a no-op and ask whether to proceed anyway. Default: proceed.

### 3. Find or create the release PR

```
mcp__github__list_pull_requests(
  owner=<OWNER>,
  repo=<REPO>,
  base=main,
  head=<OWNER>:develop,
  state=open
)
```

- **More than one** → stop and ask.
- **Exactly one** → reuse as `<PR>`. Read its body. For every `#N` in `<CLOSES>` whose `Closes #N` line is missing from the body, append it under a stable `## Closes` section (insert the section right after `## What & why` if absent) and persist via `mcp__github__update_pull_request(body=...)`. Skip to step 4.
- **Zero** → create one:

```
mcp__github__create_pull_request(
  owner=<OWNER>,
  repo=<REPO>,
  base=main,
  head=develop,
  title="release: develop → main (YYYY-MM-DD)",
  body=<see below>
)
```

PR body:

```markdown
## What & why
Promote `develop` to `main`. Linked issues are listed under `## Closes`.

## Closes
Closes #A
Closes #B
...

## Checklist
- [ ] CI green
- [ ] Merge method: **Create a merge commit** (NOT squash) — preserves develop's history so the next release does not hit add/add conflicts
```

One `Closes #N` per line — `.github/workflows/project-status.yml`'s regex matches per line, and a table or bullet list is fine for human readers but does not move project cards on its own. Save the PR number as `<PR>`.

### 4. Wait for green CI

Poll at ~30–45 second intervals (no faster):

```
mcp__github__pull_request_read(
  method=get_status,
  owner=<OWNER>,
  repo=<REPO>,
  pullNumber=<PR>
)
```

- **All checks success** → step 5.
- **At least one failure** → stop. Release-PR fixes go through a normal feature PR into `develop` (then re-run `/push-to-prod`). Do not push directly to `develop` from this skill.
- **Stuck > 15 minutes in pending/queued** → notify the user.

### 5. Merge as a merge-commit

```
mcp__github__merge_pull_request(
  owner=<OWNER>,
  repo=<REPO>,
  pullNumber=<PR>,
  merge_method=merge
)
```

**Not** `squash`. A squash collapses develop's history into a single new SHA on `main`; the next release then sees add/add conflicts on every file touched in the previous release because the original commits are unreachable from `main`.

If merge fails due to a conflict with `main` — stop and ask (this is a product fork).

Sync local refs after the merge:

```bash
git checkout main
git pull --ff-only origin main
git checkout develop
git pull --ff-only origin develop
```

### 5.5. Verify the release reached the running app (config-gated)

**Skip this step entirely if `DEPLOY_VERIFY` is `none` in `.claude/flow.config.md`** (the common case).

Otherwise: a merge commit on `PROD_BRANCH` triggers a production deploy whose success the merge does not prove. Confirm the promoted code is **live** with the project-local `DEPLOY_VERIFY_SKILL`, run as an **isolated subagent** (deploy-log polling is noisy).

Resolve the merge commit SHA (the merge commit now on `PROD_BRANCH`):

```bash
git rev-parse <PROD_BRANCH>
```

Then dispatch the subagent, pointing it at the `DEPLOY_VERIFY_SKILL` for the production environment:

```
Agent(
  subagent_type="general-purpose",
  description="Verify production deploy",
  prompt="Run the <DEPLOY_VERIFY_SKILL> command for merge SHA <SHA> on the production environment. Follow it exactly and return its single compact verdict block. Do not print secrets."
)
```

Surface the returned **verdict** in the step-7 wrap-up. A non-success verdict does not roll back the release (the merge stands); it is operational feedback that the production deploy needs attention. Best-effort: a subagent error must not block step 6.

### 6. Confirm Production status updates

`.github/workflows/project-status.yml` runs on PR-closed (merged) with `base=main` and moves each linked issue's project card to **Production**, extracting numbers from the merged PR body via the same regex used in step 2.

Verification is best-effort:

- If the workflow's project-board variables (`PROJECT_ID`, `STATUS_FIELD_ID`, `OPTION_DEVELOP`, `OPTION_PRODUCTION`) are unset, the workflow exits early and the move is silently disabled — there is nothing to verify.
- Otherwise the move runs in seconds on the merged PR; the run is visible under Actions → "Project status sync".

Do not block on this — either it succeeds quickly or the project board is not configured.

### 7. Wrap-up

Return to the user in one message:

- URL of the merged release PR;
- the `<CLOSES>` set as `Moved to Production: #A, #B, …` (or `none` if empty);
- the deploy verdict from step 5.5 (one line: live in production, or the deploy issue to look at) — omit this line if `DEPLOY_VERIFY` is `none`;
- one-line note that `PROD_BRANCH` and `DEV_BRANCH` are synced locally.

## Forbidden

- Never `merge_method=squash` for release PRs.
- Never `git push --force` to `main` or `develop`.
- Never `--no-verify` or pre-commit bypass without an explicit user request.
- Never push fixes for failing release-PR CI directly to `develop` from this skill — route them through a normal feature PR into `develop`.
- Do not skip waiting for green CI before merge.
- Do not use `gh` CLI for PR operations — only `mcp__github__*`.
- Do not create local plan files. The release lives in the PR body.
