---
description: Autonomously drive an issue to merge into develop and publish a report — no per-step confirmations
argument-hint: "<issue-number> [-bypass_low | -bypass]"
model: opus
---

Take issue `#$ARGUMENTS` in the current repo and drive it to completion **autonomously**: branch → TDD → PR → green CI → merge to `develop` → automatic `/report`. The issue is the single source of truth.

Before any `mcp__github__*` call, resolve `<OWNER>` and `<REPO>` from `git remote get-url origin` (format `https://github.com/<OWNER>/<REPO>.git`) — this file is template-shaped and must not hardcode a specific repo.

## Arguments

`$ARGUMENTS` carries the issue number plus optional flags. Parse it once, up front:

- **`<N>`** — the single positive integer in `$ARGUMENTS`. **Everywhere below, `$ARGUMENTS` used as an issue number means `<N>`** (flags stripped) — in `#$ARGUMENTS`, `issue_number=$ARGUMENTS`, the branch name, `Closes #$ARGUMENTS`, and the `/report $ARGUMENTS` hand-off. If no integer is present, stop and ask.
- **flags** — tokens starting with `-`. Only these are recognized; they relax the **existing-test gate** (step 3.6) and nothing else:
  - **`-bypass_low`** — auto-accept existing-test changes when **every** changed test group is **Low** risk; still stop and ask if any group is **Medium** or **High**.
  - **`-bypass`** — auto-accept **all** existing-test changes regardless of risk; never pause at the gate.
  - none (default) — current behavior: any edit/deletion of an existing test stops and asks.

The bypass flags affect **only** the existing-test gate. Every other "stop and ask" fork in the Mode section (merge conflict, unfixable CI, AC ambiguity, scope expansion, security finding) still escalates exactly as before — `-bypass` does **not** silence those.

## Project config — read this first

This command ships in the shared `issue-flow` plugin, so its body is stack-agnostic. The project-specific values live in **`.claude/flow.config.md`** at the repo root. **Read that file before acting** and substitute its values wherever this command shows a placeholder literal:

- branch names → `DEV_BRANCH` / `PROD_BRANCH` (the literals `develop` / `main` in this file are placeholders)
- gate commands → `TEST_CMD`, `LINT_CMD`, `FORMAT_CMD`, `TYPECHECK_CMD`, and the `FE_*` gates (literals like `uv run pytest`, `uv run mypy`, `npm run …` are placeholders)
- e2e location → `E2E_PATH` (skip every e2e instruction if it is `none`)
- CI check to wait on → `CI_CHECK_NAME`
- review steps → `CODE_REVIEW_SKILL` / `SECURITY_REVIEW` (skip a step whose value is `none`)
- deploy verification → run step 8.5 only if `DEPLOY_VERIFY` is not `none`, dispatching `DEPLOY_VERIFY_SKILL`
- user-facing chat language → `USER_LANGUAGE` (any "(currently Russian)" note below is a placeholder)

If `.claude/flow.config.md` is missing, stop and ask the user to create it from the plugin's `templates/flow.config.example.md`.

Sibling commands are namespaced under the plugin: `/issue-flow:plan-issue`, `/issue-flow:work-on-issue`, `/issue-flow:report`, `/issue-flow:push-to-prod`, `/issue-flow:batch-work`, `/issue-flow:ux-explore`.

## Mode

This is an **autonomous flow** in the spirit of a Replit-style agent. By default **do not ask** the user and **do not wait** for confirmations between steps. Stop and ask **only** at a real fork:

- contradiction/ambiguity in AC that cannot be resolved by reading the issue;
- merge conflict with `develop` that requires a product decision;
- red tests/linter that could not be fixed in 2–3 iterations;
- request for an action outside issue scope (new DB migrations, removal of public API, CI/infra changes);
- CI check failed for a reason that cannot be fixed by a local patch (e.g., secrets/environment);
- **modification or deletion of existing tests** — a change to guardrails, gated at step 3.6. By default an explicit confirmation is required; the `-bypass_low` / `-bypass` flags relax this gate (see Arguments and step 3.6). Adding new tests is fine, no confirmation needed.
- **`/security-review` flagged a security problem** (see step 5.6) — never auto-fix-and-merge a security finding; surface it to the user and stop.

In all other cases — proceed to the end without pauses.

## Steps

### 1. Read the issue

```
mcp__github__issue_read(method=get, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS)
mcp__github__issue_read(method=get_comments, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS)
```

Study **body + all comments** — they may contain scope refinement, AC changes, or intermediate decisions. From labels infer `<type>` for the branch: `feat` (label `type:feature`) or `fix` (label `type:bug`).

Send the user **one short message** (3–6 lines): task, AC, approach, affected files. This is a notification, **not** a confirmation request — go straight to step 2.

### 2. Branch from `develop`

```bash
git fetch origin develop
git checkout develop
git pull --ff-only origin develop
git checkout -b <type>/$ARGUMENTS-<kebab-summary>
```

`<kebab-summary>` — a short 2–4 word English summary. **Never** branch from `main`, **never** work directly on `develop`/`main`.

### 3. TDD loop

For each AC:

1. **Red.** Failing test reflecting the criterion. `uv run pytest` fails for the expected reason.
2. **Green.** Minimal implementation. `uv run pytest` green.
3. **Refactor.** Cleanup without changing behavior. `uv run pytest` green again.

After all AC — full run:

```bash
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy --strict src
```

All four must be green before PR. If something is red — try to fix (up to 2–3 iterations). If you cannot — stop and ask.

### 3.5. E2E check

After green unit-TDD, assess e2e applicability.

**Applicable** if the task changes:

- public API contract (new/changed DRF endpoint, serializer change, URL routing change);
- request → view → ORM → response chain (new models, new DRF permissions/authentication, new middleware);
- frontend flow visible to the user (new screen, change to an existing one, change to API interaction).

**Not applicable** (note in PR body as one line "E2E not applicable: <reason>"):

- internal refactor without user-facing effect;
- config/types/docs-only changes;
- migrations with no new user-visible or API surface.

**If applicable:**

- Backend API e2e: test via `rest_framework.test.APIClient` against the real URL conf (with pytest-django migrations), no mocks. File: `backend/apps/<app>/tests/test_<feature>_api.py`.
- Frontend UI e2e (once a separate suite exists): test of the screen's interaction with a mocked API via MSW or against a local backend. File: `frontend/src/<feature>/__tests__/<feature>.e2e.test.tsx`.
2. The test must go through real HTTP POST via `send_update(...)` → pass through `build_app(...)` → touch DB/state → assert the effect via `fake_bot_session.captured_texts()` and/or `e2e_db_session`.
3. Run: `uv run pytest tests/e2e/ -x` — must be green. Then full `uv run pytest` again to be safe.
4. Commit as a separate `test:` commit (or together with the `feat:` commit — your call, but e2e must be present in the PR diff).

### 3.6. Gate on existing test changes

Tests are guardrails. Any **edit or deletion of existing** tests (unit or e2e) must be **explicitly confirmed by the user** before commit. Adding new tests passes without confirmation.

After TDD and e2e iterations (steps 3 and 3.5), **before** committing test changes, compute the diff:

```bash
git diff develop... --stat -- '**/test_*.py' '**/*.test.*' '**/*.test.tsx' '**/tests/**' 'frontend/**/__tests__/**'
```

If the diff contains **deleted** test files, **renamed** or **modified** existing test cases (including `pytest.mark.skip`/`xfail` on a previously passing test), this gate engages. First **classify each changed test group's risk** (`Низко` / `Средне` / `Высоко`) using the same judgment the report template below asks for. Then decide how to proceed **based on the flags parsed in Arguments**:

- **default (no flag)** — stop and post the report below; **wait for explicit confirmation** before committing the test changes.
- **`-bypass_low`** — if **every** changed group is `Низко`, auto-accept (commit without waiting) and post the same cards as a **non-blocking** note prefixed `Изменения тестов приняты автоматически (риск низкий, флаг -bypass_low):`. If **any** group is `Средне`/`Высоко`, fall back to the default: post the report and **wait**.
- **`-bypass`** — auto-accept **all** changed groups regardless of risk; commit without waiting and post the same cards as a **non-blocking** note prefixed `Изменения тестов приняты автоматически (флаг -bypass):` so the change stays visible for later review.

When changes are auto-accepted (either bypass path), do **not** wait for a reply — but still emit the cards (so the user can review after the fact) and still list every modified/deleted test in the PR body's "Test changes" section (step 5). Auto-acceptance never applies to a brand-new behavior that contradicts the issue AC — if a test change implies the AC itself is wrong, that is an AC ambiguity fork (Mode), not a test-gate decision, and it escalates regardless of `-bypass`.

The report (and the non-blocking note) is a **user-facing chat message**, not a repo artifact, so write it in the **user's language** (currently Russian) and for a **non-engineer reader**:

- **No jargon.** Forbidden words: "guardrail", "blob", "re-pin", "alias", "back-compat", "contract", "mirror". Say what they mean in plain words ("проверка формата", "версия резюме", "совместимость со старым импортом").
- **One card per changed test group** (group by file or by feature, not one card per micro-assert). Each card has exactly three lines: what it checked, what changes, and how risky that is.
- **Risk in plain words** — `Низко` / `Средне` / `Высоко` with a half-line why. On `Средне`/`Высоко`, name what you'll do to de-risk (e.g. "перепроверю формат отдельным тестом").
- **Open questions go last, separately** — only the decisions that actually block you, each as a simple either/or **with your recommendation**. Do not interleave them into the cards.

Template (fill in, keep this shape and language):

```markdown
Меняю N групп тестов. По каждой — что проверяла, что меняю, опасно ли.

ТЕСТ 1 — <человеческое имя группы, напр. «проверка резюме в API»>
 Было: <что проверял тест, 1 строка простыми словами>
 Меняю: <что станет вместо этого, 1 строка>
 Опасно? <Низко/Средне/Высоко> — <короткое почему; на Средне/Высоко — что сделаю, чтобы подстраховаться>

ТЕСТ 2 — …
 …

—————
Нужны твои решения, без них не начну:
1) <вопрос как выбор А/Б> → Совет: <вариант и в полстроки почему>
2) …

Подтверди изменения тестов — или скажи, что поправить.
```

On the **default / waiting** path: wait for explicit confirmation, and after "yes" — commit test changes as a **separate** `test:` commit (or several, if logically distinct), so guardrail edits read at a glance in `git log`. On an **auto-accepted** path (`-bypass`, or `-bypass_low` with all-`Низко` groups): commit the same way **without** waiting, right after emitting the non-blocking note.

If the user refuses (waiting path only) — reconsider the approach: maybe the new behavior should coexist with the old, or the AC is formulated incorrectly.

Entirely **new** tests (new files or new functions in existing files) — this gate does not apply, they go in a regular `test:`/`feat:` commit.

### 4. Commit and push

Conventional-style in English (`feat:`, `fix:`, `refactor:`, `test:`, `docs:`, `chore:`). One logically coherent commit per change, not "WIP".

```bash
git add <specific files>
git commit -m "<type>: <short summary>"
git push -u origin <type>/$ARGUMENTS-<kebab-summary>
```

Do not use `git add -A` / `git add .` — uncommitted out-of-scope changes (e.g., local `.claude/settings.json`) may slip in.

### 5. PR via MCP

```
mcp__github__create_pull_request(
  owner=<OWNER>,
  repo=<REPO>,
  base=develop,
  head=<type>/$ARGUMENTS-<kebab-summary>,
  title=<English title, to the point>,
  body=<see below>
)
```

PR body — English, per `.github/pull_request_template.md`:

```markdown
## What & why
Closes #$ARGUMENTS

<2–5 lines: what changed and why, referencing AC of the issue>

## Test changes
<one of:>
- Only new tests added; existing tests not touched.
- Existing tests modified/deleted (gate per step 3.6 — confirmed by user, or auto-accepted via `-bypass_low`/`-bypass`; state which and the risk):
  - `path/to/test_file.py::test_name` — deleted / rewritten / skipped. Risk: low/medium/high. Reason: <…>
  - …

## Checklist
- [x] TDD: red → green → refactor completed
- [x] `pytest` green (including `tests/e2e/`)
- [x] `ruff check .` clean
- [x] `mypy --strict` clean
- [x] E2E test added/updated OR noted "E2E not applicable: <reason>"
- [x] Existing-test changes confirmed by user / auto-accepted via `-bypass`/`-bypass_low` OR existing tests not touched
- [ ] docs/README updated if needed
```

`Closes #$ARGUMENTS` is **mandatory** in the body — without it `.github/workflows/project-status.yml` will not move the project card.

Save the PR number (`<PR>`) for the next steps.

### 5.5. Code review — one round of fixes

After the PR is open, run a code review on it and reconcile the findings against the issue plan. **Exactly one** round of fixes — do **not** loop review → fix → review.

1. Invoke the `code-review:code-review` skill on the just-opened `<PR>`. It runs a multi-agent review of the PR, filters to high-confidence findings, and posts the result as a PR comment via `gh` — read that comment for the findings. (This is a deliberate, user-chosen exception to the repo's "only `mcp__github__*`" rule, scoped to this review step.)
2. **Reconcile each finding against the issue (body + comments) — the plan is the source of truth.** A finding is **adequate** (→ fix it) when it is a real, in-scope defect or a violation of `CLAUDE.md` / the issue's AC. A finding is **rejected** (→ skip, note why) when it:
   - asks for behavior or scope **not** in the issue AC (that is follow-up, not this task);
   - contradicts an explicit decision recorded in the issue body/comments;
   - is a pre-existing issue on lines this PR did not touch;
   - is a stylistic nitpick a linter/typechecker/CI already covers.
3. Apply fixes for the **adequate** findings only. Keep the TDD discipline: if a fix changes behavior, add/adjust a test first (the step 3.6 gate on **existing**-test edits still applies). Re-run the relevant local gates (`uv run pytest`, `ruff`, `mypy`, and the frontend gates if FE changed).
4. Commit the fixes as a focused `fix:`/`refactor:` commit and `git push` to the same branch. The push re-triggers CI, which step 6 waits on.
5. If there were **no** findings, or all findings were rejected — note that in one line and proceed; no commit needed.

This is **one** round. Do not re-run `code-review:code-review` after applying the fixes — go straight to the security gate.

### 5.6. Security review — hard gate before merge

After the code-review fixes are pushed, run `/security-review` on the branch's pending changes. This is a **blocking gate**: a clean review is required to reach merge.

- **No security problems found** → proceed to step 6.
- **Any security problem found** → **stop. Do not wait for CI, do not merge, do not run `/report`, do not wrap up the issue.** Escalate to the user in chat and end the autonomous flow there.

The escalation is a **user-facing chat message** — write it in the **user's language** (currently Russian), plain words, no jargon. For each finding: what is the risk, where (file/line), and a one-line suggested fix. Make clear the PR is open but **deliberately not merged** pending the user's decision, and that resuming means re-running `/work-on-issue $ARGUMENTS` (or telling you how to handle each finding). Leave the branch and PR as-is — do not close them.

A security finding is a **real fork** (see Mode): the agent does not silently auto-fix security issues and then merge — the user must see them first.

### 6. Wait for green CI

The CI check to wait on is named `CI_CHECK_NAME` (from `.claude/flow.config.md`; default `CI`), triggered on `pull_request`. **Always** wait for it to pass **before** merge — even if everything is green locally.

Poll status via MCP at ~30–45 second intervals (no faster) until all checks complete:

```
mcp__github__pull_request_read(
  method=get_status,
  owner=<OWNER>,
  repo=<REPO>,
  pullNumber=<PR>
)
```

Possible outcomes:

- **All checks success** → go to step 7.
- **At least one failure** → read the failing job's logs, try to fix locally (up to 2–3 iterations: commit, push, wait for CI again). If you cannot — stop and ask the user.
- **Stuck > 15 minutes in pending/queued** — stop, notify the user.

### 7. Merge PR

After green CI — squash merge via MCP:

```
mcp__github__merge_pull_request(
  owner=<OWNER>,
  repo=<REPO>,
  pullNumber=<PR>,
  merge_method=squash
)
```

If merge fails due to a conflict with `develop` — stop and ask the user (this is a product fork).

After successful merge, sync `develop` locally:

```bash
git checkout develop
git pull --ff-only origin develop
```

Do **not** delete the local feature branch automatically — leave it to the user.

### 7.5. Forward `Closes #N` to the open release PR (if any)

`.github/workflows/project-status.yml` moves an issue's project card to **Production** only when a PR with `base=main` is merged **and** its body contains a `Closes #N` marker for that issue. Squash-style release PRs that list issues as a table or bullet list (without the `Closes` keyword) are silently ignored — exactly the trap that left a backlog of merged issues stuck in `Develop` until manual cleanup.

To prevent that, after merging this feature into `develop`, also append a `Closes #$ARGUMENTS` line to the open `develop → main` release PR (if one already exists):

```
mcp__github__list_pull_requests(
  owner=<OWNER>,
  repo=<REPO>,
  base=main,
  head=<OWNER>:develop,
  state=open
)
```

- **Zero open release PRs** → skip silently. There is no release PR yet — the next one created must include `Closes` markers itself (see `Release PR conventions` below).
- **Exactly one** → read its body. If the body already contains `Closes #$ARGUMENTS` (case-insensitive, with a `#` word boundary), skip. Otherwise append `Closes #$ARGUMENTS` on its own line under a stable `## Closes` section (insert the section right after `## What & why` if absent) and persist via `mcp__github__update_pull_request(body=...)`.
- **More than one** → stop and ask the user (unusual state, likely a stale PR).

This step is best-effort: a failure must not block the report in step 8.

**Release PR conventions (for whoever cuts the release):**

- **Merge method**: `Create a merge commit`, **not** squash. A squash collapses develop's history into a single new SHA on `main`; the next release then sees add/add conflicts on every file touched in the previous release because the original commits are unreachable from `main`.
- **Body**: every issue being released must appear as `Closes #N` on its own line (a table or bullet list is fine for human readers, but the `Closes` lines are what the workflow regex matches).

### 8. Auto-report

Immediately after merge, execute the `/issue-flow:report $ARGUMENTS` steps without pauses and without preview-confirm: gather facts from git/tests, map to issue AC, publish **one** comment via `mcp__github__add_issue_comment`. Details in the `/issue-flow:report` command.

### 8.5. Verify the merge reached the running app (config-gated)

**Skip this step entirely if `DEPLOY_VERIFY` is `none` in `.claude/flow.config.md`** (the common case) — there is nothing to poll.

Otherwise: merging into `DEV_BRANCH` triggers a deploy whose success a green GitHub merge does **not** prove (builds are slow and can fail). Confirm it with the project-local `DEPLOY_VERIFY_SKILL`, run as an **isolated subagent** (deploy-log polling is noisy — keep it out of this flow's context).

Resolve the merge commit SHA first:

```bash
git rev-parse HEAD   # on DEV_BRANCH, after the step-7 sync — this is the merge commit
```

Then dispatch the subagent, pointing it at the `DEPLOY_VERIFY_SKILL` for `DEV_BRANCH`:

```
Agent(
  subagent_type="general-purpose",
  model="haiku",
  description="Verify deploy",
  prompt="Run the <DEPLOY_VERIFY_SKILL> command for merge SHA <SHA> on the DEV_BRANCH environment. Follow it exactly and return its single compact verdict block. Do not print secrets."
)
```

Surface the returned **verdict** to the user (one line) in step 9:

- a success verdict (e.g. `deployed`) → the merge is live; nothing more to do.
- any other verdict → report it plainly as a deploy issue needing attention. This does **not** reopen the issue or fail the task — the code is merged and reported; the deploy verdict is operational feedback. Best-effort: a subagent error must not block step 9.

### 9. Wrap-up

Return to the user in one message:

- URL of the merged PR;
- URL of the posted report comment;
- the deploy verdict from step 8.5 (one line: live on `DEV_BRANCH`, or the deploy issue to look at) — omit this line if `DEPLOY_VERIFY` is `none`;
- briefly (1–2 lines): what is closed, what remains (if anything — that is follow-up, not the current task).

## Forbidden

- Never `git push --force` to `develop`/`main`.
- Never `--no-verify`, `--no-gpg-sign`, or pre-commit bypass without explicit user request.
- Never `base=main` for feature/bugfix branches.
- Never merge a PR before green CI — even if everything is green locally.
- Never merge a PR while `/security-review` (step 5.6) has an unresolved finding — escalate to the user and stop.
- Do not create files in `.claude/plans/`, `notes/` (for an active task), or `*-plan.md` at the repo root.
- Do not use `gh` CLI for PR/issue operations — only `mcp__github__*`.
