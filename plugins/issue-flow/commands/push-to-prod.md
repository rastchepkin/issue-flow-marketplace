---
description: Cut a release — open develop → main PR listing the issues it ships, stop for their manual prod steps, wait for green CI, merge as a merge-commit
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
- manual prod steps label → `PROD_ACTION_LABEL` (step 2.5; default `prod:action-required`, blank disables the gate)
- deploy verification → run step 5.5 only if `DEPLOY_VERIFY` is not `none`, dispatching `DEPLOY_VERIFY_SKILL`
- user-facing chat language → `USER_LANGUAGE`

If `.claude/flow.config.md` is missing, stop and ask the user to create it from the plugin's `templates/flow.config.example.md`.

Sibling commands are namespaced under the plugin: `/issue-flow:plan-issue`, `/issue-flow:work-on-issue`, `/issue-flow:report`, `/issue-flow:push-to-prod`.

## Mode

Autonomous, same spirit as `/work-on-issue`. Stop and ask only at a real fork:

- `develop` is not ahead of `main` (nothing to release);
- more than one open `develop → main` PR (unusual state, likely a stale draft);
- **the release carries manual prod steps that are not done yet** (step 2.5) — this is a planned stop, not an error: the user does them, confirms, and the release continues;
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

Call the union `<ISSUES>`. For the release note this list is **descriptive**: a miss makes the note less complete, nothing more — it does not strand a project card, because the cards already moved to **Done** when each feature PR merged into `develop`. If the set comes out empty, say so in one line and carry on; do not stop to ask. (For manual prod steps a miss *does* matter — step 2.5 has its own safety net for that.)

### 2.5. Collect manual prod steps — the pre-deploy gate

**Skip this step if `PROD_ACTION_LABEL` is blank in `.claude/flow.config.md`.**

Some tasks need a change outside the code before their code can run in production: an env var to set, a secret to add, an infra/hosting setting, a one-off script. `/plan-issue` and `/work-on-issue` record those on the issue (they live **only** there; nobody is expected to remember them) as a `## Prod checklist` section in the issue body plus the `PROD_ACTION_LABEL` label:

```markdown
## Prod checklist
Before deploy:
- [ ] Set env `STRIPE_WEBHOOK_SECRET` on the prod app (hosting panel → Environment)
After deploy:
- [ ] Run `manage.py backfill_invoices` once on prod
```

Gather them for this release:

1. **From the release's issues.** For every `#N` in `<ISSUES>`, read the issue (`mcp__github__issue_read(method=get, …)` — you need its title for step 3 anyway). Take its `## Prod checklist` items if it carries `PROD_ACTION_LABEL` **or** has that section. Keep each item's `Before deploy` / `After deploy` bucket and its checked state. An item with no bucket counts as **Before** — the safe direction.
2. **Safety net — labelled issues the release list missed.** `<ISSUES>` is derived from PR bodies and can miss one. Search for every issue that still carries the label:

   ```
   mcp__github__search_issues(query="repo:<OWNER>/<REPO> label:<PROD_ACTION_LABEL>", perPage=50)
   ```

   - **open** ones are not merged yet — they are not in this release; ignore them.
   - **closed** ones not in `<ISSUES>` → either the derivation missed them, or they shipped in an earlier release with an `After deploy` step never confirmed. Find the issue's merged feature PR (`mcp__github__search_pull_requests(query="repo:<OWNER>/<REPO> is:merged base:<DEV_BRANCH> \"#<N>\" in:body")`) and test its `merge_commit_sha` with `git merge-base --is-ancestor <sha> origin/main`. Not in `main` yet → it **is** in this release: add it to `<ISSUES>` and to the checklist. Already in `main` → carry its unchecked items as **leftovers from a previous release** and show them to the user in the same message, separately. If no PR is found, treat it as part of this release (the safe direction).

Call the result `<PROD_STEPS>`. Values are never written into an issue or a PR — only the *name* of the variable/secret and *where* to set it. If you see an actual secret value in an issue body, do not copy it anywhere and tell the user to rotate it.

- **`<PROD_STEPS>` empty** → nothing to gate; carry on to step 3 silently.
- **otherwise** → step 3 writes them into the release PR body, and step 4 must not start while any **Before deploy** item is unchecked (step 4 opens with that check).

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
- **Exactly one** → reuse as `<PR>`. Read its body and add any `#N` from `<ISSUES>` missing from its `## Ships` section (insert the section right after `## What & why` if absent), and any `<PROD_STEPS>` item missing from its `## Before deploy` / `## After deploy` sections. **Never uncheck** an item that is already `[x]` in the PR body — that tick is the user's confirmation and survives re-runs. Persist via `mcp__github__update_pull_request(body=...)`. Skip to step 4.
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

## Before deploy
<!-- only if <PROD_STEPS> has Before items; omit the section otherwise -->
- [ ] #A — Set env `FOO_API_KEY` on the prod app
...

## After deploy
<!-- only if <PROD_STEPS> has After items; omit the section otherwise -->
- [ ] #B — Run `manage.py backfill_invoices` once
...

## Checklist
- [ ] CI green
- [ ] Merge method: **Create a merge commit** (NOT squash) — preserves develop's history so the next release does not hit add/add conflicts
```

Write the list for humans — these issues are **already closed** (each was closed by its own feature PR merging into `develop`), so do **not** use `Closes #N` here: it would be a no-op that reads as if the release closed them. Save the PR number as `<PR>`.

### 4. Green CI

#### 4.0. Manual prod steps first — stop until they are done

Re-read the release PR body. If its `## Before deploy` section has **any unchecked item**, do **not** arm auto-merge and do **not** merge. A merge into `PROD_BRANCH` is what triggers the production deploy, and with `AUTO_MERGE: true` GitHub would merge on green CI before the user had a chance to act. So this check comes **before** 4a/4b, never after.

If an earlier run already armed auto-merge on this PR (a new feature with prod steps landed in `develop` since then), disarm it first — `gh pr merge <PR> --disable-auto` — and say so in the message. Re-arming happens in 4a once the steps are done.

Message the user in `USER_LANGUAGE`, plain words, grouped by issue, and end the turn:

```markdown
Перед релизом нужно сделать руками в проде:

#A — <issue title>
- [ ] <пункт>

#B — <issue title>
- [ ] <пункт>

После деплоя (напомню в конце):
- #B — <пункт>

<если есть хвосты прошлых релизов:>
Не подтверждено с прошлых релизов:
- #C — <пункт>

—————
Release PR: <url>. Когда сделаешь — скажи «сделал» (или какие пункты), и я продолжу релиз.
```

On the user's confirmation:

- tick the confirmed items `[x]` in the release PR body (`mcp__github__update_pull_request`) **and** in each issue's `## Prod checklist` (`mcp__github__issue_write(method=update, body=…)` — edit only those checkbox lines, never rewrite the rest of the body);
- if some items are confirmed as "not needed anymore", strike them (`- [x] ~~…~~ — not needed: <reason>`) rather than deleting them;
- once every **Before deploy** item is checked → continue to 4a/4b in the same turn.

The ticks are the durable state: a re-run after the session died finds them in the PR body and does not ask again. The user may also tick the boxes on GitHub directly — that counts as confirmation. With no unchecked **Before deploy** item, this sub-step is a silent pass-through.

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

**Manual prod steps (skip if `<PROD_STEPS>` is empty).** The deploy is what the **After deploy** items were waiting for — list them in the step-7 wrap-up as the user's next action. Then clear `PROD_ACTION_LABEL` from every issue whose `## Prod checklist` is now fully checked (`mcp__github__issue_write(method=update, labels=[<current labels minus PROD_ACTION_LABEL>])`). An issue that still has an unchecked **After deploy** item **keeps** the label — that is what lets the next `/push-to-prod` (step 2.5's safety net) remind about it if it is never confirmed. If the user confirms the After items later in the same session, tick them in the release PR and the issue, and clear the label then.

If the user wants a tagged, browsable release on top of that, `gh release create <tag> --generate-notes --target <PROD_BRANCH>` does it — **only when they ask**. Do not tag on your own: version naming is a product decision.

### 7. Wrap-up

Return to the user in one message:

- URL of the merged release PR;
- the `<ISSUES>` set as `Shipped: #A, #B, …` (or `none if the list came up empty` — say so plainly, it means the release note is incomplete);
- the deploy verdict from step 5.5 (one line: live in production, or the deploy issue to look at) — omit this line if `DEPLOY_VERIFY` is `none`;
- **if `<PROD_STEPS>` has After-deploy items:** `Теперь сделай после деплоя:` + the list (`#N — <item>`), and ask to say «сделал» when done — the labels stay on those issues until then;
- one-line note that `PROD_BRANCH` and `DEV_BRANCH` are synced locally.

## Forbidden

- Never `merge_method=squash` for release PRs — and never `gh pr merge --auto --squash` either; the release path is always a merge commit.
- Never open a second release PR when one is already open — reuse it (step 3) so a resumed run cannot fork the release.
- Never `git push --force` to `main` or `develop`.
- Never `--no-verify` or pre-commit bypass without an explicit user request.
- Never push fixes for failing release-PR CI directly to `develop` from this skill — route them through a normal feature PR into `develop`.
- Do not skip waiting for green CI before merge.
- Never merge, or arm auto-merge on, a release PR while its `## Before deploy` section has an unchecked item — and never tick an item the user did not confirm.
- Never write a secret **value** into a PR or issue — only the variable's name and where to set it.
- Do not use `gh` CLI for PR operations — only `mcp__github__*`.
- Do not create local plan files. The release lives in the PR body.
- Do not put `Closes #N` in a release PR body — those issues are already closed, and the line would read as if the release closed them.
- Do not tag or publish a GitHub Release unless the user asks — the version number is theirs to choose.
