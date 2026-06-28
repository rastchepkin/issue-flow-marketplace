---
description: Discussion → GitHub issue (English, by template, no duplicates). Default: product review (UX + plan/AC sign-off); -auto creates straight away.
argument-hint: "[short task description] [-auto]"
model: sonnet
---

Turn the current discussion (or `$ARGUMENTS`) into a GitHub issue in the current repo. The issue is the single source of truth for the task; creating local plan files is **forbidden**.

Before any `mcp__github__*` call, resolve `<OWNER>` and `<REPO>` from `git remote get-url origin` (format `https://github.com/<OWNER>/<REPO>.git`) — this file is template-shaped and must not hardcode a specific repo.

## Arguments

`$ARGUMENTS` is the task description plus an optional flag. Parse it:

- **`-auto`** — autonomous mode: skip the UX step, the discussion, and the sign-off; if the context is clear, create the issue straight away (the behavior this command used to have by default). Use it for small/obvious tasks and unattended runs.
- **no flag (default)** — **product-review mode**: explore the UX when the change touches the frontend, discuss the change with the user, and present a product-change plan + AC for sign-off **before** creating the issue. Use it for anything a manager should approve first.

Strip the flag from the text; the remaining words are the task description.

## Project config — read this first

This command ships in the shared `issue-flow` plugin, so its body is stack-agnostic. The project-specific values live in **`.claude/flow.config.md`** at the repo root. **Read that file before acting** and substitute its values wherever this command shows a placeholder literal:

- branch names → `DEV_BRANCH` / `PROD_BRANCH` (the literals `develop` / `main` in this file are placeholders)
- the bug-report Environment section → list this project's actual runtime, not `Python, Django, Node`
- user-facing chat language → `USER_LANGUAGE`

If `.claude/flow.config.md` is missing, stop and ask the user to create it from the plugin's `templates/flow.config.example.md`.

Sibling commands are namespaced under the plugin: `/issue-flow:plan-issue`, `/issue-flow:work-on-issue`, `/issue-flow:report`, `/issue-flow:push-to-prod`, `/issue-flow:batch-work`, `/issue-flow:ux-explore`.

## Mode

Behavior depends on the flag parsed in Arguments.

**`-auto` — autonomous.** If the discussion gives enough context, create the issue without preview and without asking for confirmation. Stop and ask **only** at a real fork:

- the task sounds vague and 1–3 clarifying questions are needed to formulate AC;
- a potential duplicate is found — user decision required (new / comment on existing / drop topic);
- ambiguity of type (feature vs bug) that cannot be resolved from context.

**default — product review.** Do **not** create the issue silently. First run the product-review pre-flow (step 0 below): UX exploration when the frontend is touched, a short discussion, and an explicit plan + AC sign-off. Only after the user approves do you proceed to steps 2–5. The duplicate/type forks above still apply (raise them during the discussion). All pre-flow output is in `USER_LANGUAGE`.

## Steps

### 0. Product-review pre-flow (default mode only; skip entirely under `-auto`)

Goal: let a non-engineer (a manager) shape and approve the change before it becomes an issue.

**0.1. UX exploration when the frontend is involved.** Decide whether the task plausibly changes something the user sees or interacts with — a new/changed screen, control, flow, state, or visual. If yes **and** the project actually has a frontend (its `FE_*` gates in `.claude/flow.config.md` are not `none`), run **`/issue-flow:ux-explore`** on the task first and carry its outcome (chosen UI element, placement, states, flow) into the discussion and into the issue's "Proposed solution". For backend-only / config / docs tasks, skip UX.

**0.2. Discuss the change.** Talk it through with the user in `USER_LANGUAGE`: confirm the goal, scope, edge cases, and what is explicitly out of scope. Ask the 1–3 clarifying questions you would otherwise defer. Surface duplicate/overlap findings here if step 3's context-gathering (which you may do early) turns any up.

**0.3. Present the plan + AC for sign-off.** Before creating anything, post a compact, **non-engineer-readable** summary in `USER_LANGUAGE` and **wait for approval**:

```markdown
Согласуем перед заведением задачи:

ЧТО МЕНЯЕМ (для пользователя)
<2–4 строки простыми словами: что человек увидит/сможет делать по-новому>

UX (если был /ux-explore)
<1–2 строки: какой элемент/экран/флоу выбрали; иначе строку опускаем>

КРИТЕРИИ ПРИЁМКИ (AC)
- [ ] <измеримый критерий 1>
- [ ] <измеримый критерий 2>

ВНЕ ЗАДАЧИ
<что сознательно не делаем сейчас>

—————
Ок заводить задачу с этим планом? Или что поправить?
```

On "ок" → proceed to step 2 with the agreed plan/AC as the basis for the body. On edits → revise and re-confirm. The English issue body (steps 4–5) must match what was approved here.

### 1. Clarify scope (only if needed)

In **default mode** the clarifying happened in step 0.2 — skip this step.

In **`-auto` mode**: if the discussion and `$ARGUMENTS` make the task and AC clear — skip this step; if vague — ask **1–3 targeted questions** before creating the issue. No more.

### 2. Classify and pick template

Determine type:

- **feature** — new capability, enhancement, refactor → template `.github/ISSUE_TEMPLATE/feature_request.yml`, label `type:feature`, title starts with `[feat] `.
- **bug** — something is not working → template `.github/ISSUE_TEMPLATE/bug_report.yml`, label `type:bug`, title starts with `[bug] `.

### 3. Gather context and dedup

Before drafting the body, build a picture of "what is already done / in progress / planned" — issues and PRs in this repo carry the plan and the final `/report` comment with the result, which is exactly the context needed for the new plan.

**3.1. Open issues** — current active work:

```
mcp__github__list_issues(owner=<OWNER>, repo=<REPO>, state=open, perPage=50)
```

**3.2. Similar closed issues** — what has already been done on this topic:

```
mcp__github__search_issues(query="<keywords> repo:<OWNER>/<REPO> is:closed", perPage=20)
```

**3.3. Deep read of top candidates** — for 2–3 most relevant issues (open OR closed) pull full body and all comments:

```
mcp__github__issue_read(owner=<OWNER>, repo=<REPO>, issue_number=<N>, method=get)
mcp__github__issue_read(owner=<OWNER>, repo=<REPO>, issue_number=<N>, method=get_comments)
```

In closed issues, look for the final `/report` comment: which AC were actually completed, which files/modules were touched, what was left out of scope.

**3.4. Open PRs** — what is currently in progress (so as not to plan on top of someone else's branch):

```
mcp__github__list_pull_requests(owner=<OWNER>, repo=<REPO>, state=open, perPage=20)
```

**3.5. Recently merged PRs** — what just landed in `develop`:

```
mcp__github__list_pull_requests(owner=<OWNER>, repo=<REPO>, state=closed, sort=updated, direction=desc, perPage=10)
```

If needed — `mcp__github__pull_request_read` on the 1–2 most relevant to see the actual diff/description.

**3.6. Collision resolution:**

- Exact duplicate of an open issue → show to user, ask: create new, comment on existing, or drop.
- Closed issue on the same topic → does not block creating a new one, but its result (`/report`) **must** be reflected in step 4.
- Topic is actively in flight in an open PR → warn the user and ask whether a separate issue is really needed.

### 4. Draft the body (English)

**Use the context gathered in step 3.** The new issue body must explicitly build on what is already done / in progress:

- If the task **extends a closed issue** → reference it in "Problem / Why" (`extends #N: <X> was done, <Y> remains`). In "Implementation plan" reuse files/modules mentioned in its `/report` comment.
- If the task **overlaps with an open PR** → mention the PR in "Proposed solution" and explicitly draw the boundary (what is done here vs there).
- **Formulate AC with awareness of what is closed** — do not duplicate criteria that `/report` already confirmed as done.
- If context shows that **suitable utilities/layers already exist** — fix in "Proposed solution" "reuse `<module>`", so `/work-on-issue` does not produce duplicates.
- If the task **changes a contract or removes behavior** (meaning `/work-on-issue` may need to edit/delete existing tests) — add an "Affected existing tests (preliminary)" section listing: which test and why it is expected to change. This gives `/work-on-issue` step 3.6 material for confirmation and makes review easier.

**For feature** — sections per `feature_request.yml`:

```markdown
## Problem / Why
<context and motivation>

## Proposed solution
<what and how we do it, which files/modules are touched>

## Acceptance criteria
- [ ] <measurable criterion 1>
- [ ] <measurable criterion 2>

## Implementation plan (draft)
<optional: 3–6 plan bullets — memory for the next session>

## Affected existing tests (preliminary)
<optional, if the task changes a contract / removes behavior>
- `path/to/test_file.py::test_name` — expected to delete/rewrite. Reason: <…>
- …
```

**For bug** — sections per `bug_report.yml`:

```markdown
## Steps to reproduce
1.
2.
3.

## Expected behavior
...

## Actual behavior
...

## Environment
Python, Django, Node, OS, prod/local
```

Title — short, to the point, English, prefixed with `[feat]` or `[bug]`.

### 5. Create

Create the issue. In **default mode** the plan/AC were already approved in step 0.3, so create it now (do not ask a second time) with a body that matches what was signed off. In **`-auto` mode** create it **immediately**, without preview and without asking the user (if steps 1 and 3 did not reveal a fork):

```
mcp__github__issue_write(
  method=create,
  owner=<OWNER>,
  repo=<REPO>,
  title=<title>,
  body=<body>,
  labels=["type:feature"]  // or ["type:bug"]
)
```

Return the user the number and URL of the created issue in one line.

## Forbidden

- Do not create files in `.claude/plans/`, `notes/` (for an active task), or `*-plan.md` at the repo root. The plan lives **only** in the issue body.
- Do not use `gh issue create` — only `mcp__github__issue_write`.
- Under `-auto`, do not show a preview before creation when the discussion is clear — that mode is autonomous. (Default mode is the opposite: the step 0.3 sign-off is required before creating.)
