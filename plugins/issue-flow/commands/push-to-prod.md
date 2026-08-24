---
description: Cut a release — open develop → main PR listing the issues it ships, wait for green CI, merge as a merge-commit
model: sonnet
---

Promote everything currently on `develop` to `main` via a release PR, wait for CI, and merge as a **merge commit** (not squash).

The release PR is also the flow's answer to *"what is merged but not yet live?"* — while it is open, its issue list is exactly that set. There is no **Production** board column to keep in sync: issues reach **Done** when they merge into `DEV_BRANCH`, and what has shipped to `PROD_BRANCH` is read from the release PRs and from `git log`.

Before any `mcp__github__*` call, resolve `<OWNER>` and `<REPO>` from `git remote get-url origin` (format `https://github.com/<OWNER>/<REPO>.git`) — this file is template-shaped and must not hardcode a specific repo.

## Project config — read this first

This command ships in the shared `issue-flow` plugin, so its body is stack-agnostic. The project-specific values live in **`.claude/flow.config.md`** at the repo root. **Read that file before acting** and substitute its values wherever this command shows a placeholder literal:

- branch names → `DEV_BRANCH` / `PROD_BRANCH` (the literals `develop` / `main` in this file are placeholders)
- CI check to wait on → `CI_CHECK_NAME`
- merge strategy → `AUTO_MERGE` (step 4)
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

### 2. Collect the issues this release ships

The release PR body should list every issue going out, so the PR doubles as the release note and as the "merged but not yet live" list while it is open.

A feature PR links its issue with `Closes #N` in the **PR body**. That keyword does **not** reliably reach any commit message on `develop` — a squash merge keeps only the PR title + `(#PR)` in the subject and drops the body — so derive the set from the PR bodies, not from commit text.

List the PR numbers merged into develop in this window (both merge-commit `Merge pull request #N` and squash `… (#N)` subjects carry the number):

```bash
git log origin/main..origin/develop --pretty=format:%s%n%b |
  grep -oiE '(merge pull request #[0-9]+|\(#[0-9]+\))' |
  grep -oE '[0-9]+' |
  sort -un
```

Save these as `<PRS>` — a superset, since a subject like `… (#87) (#91)` contributes both the issue and the PR number. Then read each one and union the issues it closed:

```
mcp__github__pull_request_read(method=get, owner=<OWNER>, repo=<REPO>, pullNumber=<n>)
```

- If `n` is not a pull request (404 / it is an issue) — skip it.
- From the returned `body`, extract issue numbers matching `(close[sd]?|fix(es|ed)?|resolve[sd]?)\s+#[0-9]+`, case-insensitive.

Call the union `<ISSUES>`. This list is **descriptive**: a miss makes the release note less complete, nothing more — it does not strand a project card, because the cards already moved to **Done** when each feature PR merged into `develop`. If the set comes out empty, say so in one line and carry on; do not stop to ask.

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
- **Exactly one** → reuse as `<PR>`. Read its body and add any `#N` from `<ISSUES>` missing from its `## Ships` section (insert the section right after `## What & why` if absent), then persist via `mcp__github__update_pull_request(body=...)`. Skip to step 4.
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
Promote `develop` to `main`.

## Ships
- #A — <issue title>
- #B — <issue title>
...

## Checklist
- [ ] CI green
- [ ] Merge method: **Create a merge commit** (NOT squash) — preserves develop's history so the next release does not hit add/add conflicts
```

Write the list for humans — these issues are **already closed** (each was closed by its own feature PR merging into `develop`), so do **not** use `Closes #N` here: it would be a no-op that reads as if the release closed them. Save the PR number as `<PR>`.

### 4. Green CI

**This command is resumable.** Re-running it after a session dies is safe: step 3 finds the existing release PR instead of creating a second one, and a PR that merged while you were away is detected here (`state=closed`, `merged=true`) — in that case skip to step 5's local sync and carry on to steps 5.5–7. Never open a second release PR.

#### 4a. `AUTO_MERGE: true` — hand the merge to GitHub

Arm auto-merge with the **merge-commit** method and let GitHub do it when the checks pass, so the release does not depend on this session staying alive:

```bash
gh pr merge <PR> --auto --merge
```

`--merge`, never `--squash` — see step 5 for why. The same precondition as in `/work-on-issue` step 6a applies: `--auto` only defers if `PROD_BRANCH` has **required status checks** in branch protection; with none, GitHub merges immediately and the CI gate is not real. Read the PR once right after arming: if it is already merged while `CI_CHECK_NAME` is still pending, say so plainly to the user and continue.

Then poll `get_status` inline at ~30–45 second intervals for **at most ~10 minutes**:

- **Merged in the window** → step 5's post-merge sync.
- **Still pending** → end the turn cleanly: the release PR is armed and will merge itself on green. Tell the user that re-running `/issue-flow:push-to-prod` afterwards finishes the wrap-up (step 6–7).
- **A check failed** → auto-merge stays armed but will not fire. Stop. Release-PR fixes go through a normal feature PR into `develop` (then re-run `/push-to-prod`). Do not push directly to `develop` from this skill.

#### 4b. `AUTO_MERGE: false` — poll, then merge in step 5

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

Skip the merge call itself if step 4a already got the PR merged — go straight to the local sync below.

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
  model="haiku",
  description="Verify production deploy",
  prompt="Run the <DEPLOY_VERIFY_SKILL> command for merge SHA <SHA> on the production environment. Follow it exactly and return its single compact verdict block. Do not print secrets."
)
```

Surface the returned **verdict** in the step-7 wrap-up. A non-success verdict does not roll back the release (the merge stands); it is operational feedback that the production deploy needs attention. Best-effort: a subagent error must not block step 6.

### 6. The release record

There is **nothing to reconcile on the project board.** Each issue in `<ISSUES>` reached **Done** when its own feature PR merged into `DEV_BRANCH`; a release does not change an issue's status, it changes where the code runs. The durable record of what shipped is the merged release PR itself — it carries the issue list, the merge commit, and the date.

So this step is just a check that the record is readable:

- the merged release PR's `## Ships` list is populated (if step 2 came up empty, note that in the wrap-up rather than silently shipping an unlabelled release);
- `git log --oneline -1 <PROD_BRANCH>` is the merge commit anyone can diff against.

If the user wants a tagged, browsable release on top of that, `gh release create <tag> --generate-notes --target <PROD_BRANCH>` does it — **only when they ask**. Do not tag on your own: version naming is a product decision.

### 7. Wrap-up

Return to the user in one message:

- URL of the merged release PR;
- the `<ISSUES>` set as `Shipped: #A, #B, …` (or `none if the list came up empty` — say so plainly, it means the release note is incomplete);
- the deploy verdict from step 5.5 (one line: live in production, or the deploy issue to look at) — omit this line if `DEPLOY_VERIFY` is `none`;
- one-line note that `PROD_BRANCH` and `DEV_BRANCH` are synced locally.

## Forbidden

- Never `merge_method=squash` for release PRs — and never `gh pr merge --auto --squash` either; the release path is always a merge commit.
- Never open a second release PR when one is already open — reuse it (step 3) so a resumed run cannot fork the release.
- Never `git push --force` to `main` or `develop`.
- Never `--no-verify` or pre-commit bypass without an explicit user request.
- Never push fixes for failing release-PR CI directly to `develop` from this skill — route them through a normal feature PR into `develop`.
- Do not skip waiting for green CI before merge.
- Do not use `gh` CLI for PR operations — only `mcp__github__*`.
- Do not create local plan files. The release lives in the PR body.
- Do not put `Closes #N` in a release PR body — those issues are already closed, and the line would read as if the release closed them.
- Do not tag or publish a GitHub Release unless the user asks — the version number is theirs to choose.
