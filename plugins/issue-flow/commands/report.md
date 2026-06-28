---
description: One final comment on an issue — what was done, how it works, what remains (no preview-confirm)
argument-hint: <issue-number>
model: sonnet
---

Post **exactly one** structured comment on issue `#$ARGUMENTS` in the current repo. The comment is the work outcome and the source of truth for future sessions.

Before any `mcp__github__*` call, resolve `<OWNER>` and `<REPO>` from `git remote get-url origin` (format `https://github.com/<OWNER>/<REPO>.git`) — this file is template-shaped and must not hardcode a specific repo.

## Project config — read this first

This command ships in the shared `issue-flow` plugin, so its body is stack-agnostic. The project-specific values live in **`.claude/flow.config.md`** at the repo root. **Read that file before acting** and substitute its values wherever this command shows a placeholder literal:

- branch names → `DEV_BRANCH` / `PROD_BRANCH` (the literals `develop` / `main` in this file are placeholders)
- gate commands → `TEST_CMD`, `LINT_CMD`, `TYPECHECK_CMD` (literals like `uv run pytest`, `uv run mypy` are placeholders)
- e2e location → `E2E_PATH` (skip every e2e instruction if it is `none`)

If `.claude/flow.config.md` is missing, stop and ask the user to create it from the plugin's `templates/flow.config.example.md`.

This command is typically invoked automatically from `/issue-flow:work-on-issue` right after PR merge, so it runs **autonomously** — no preview, no confirmation prompt.

## Steps

### 1. Gather facts from git

```bash
git status
git log origin/develop --oneline -20
git log --grep="#$ARGUMENTS" --oneline -20
```

Find the commit(s) related to the task (via `Closes #$ARGUMENTS` in the squash commit, or via branch prefix). For them, gather:

- List of changes (`git show --stat <sha>`).
- Changed files with +/− line counts.

If `git status` shows uncommitted changes — that is a signal something is off (the PR should already be merged). Stop and ask the user before writing a report.

### 2. Gather facts about tests

```bash
uv run pytest
uv run pytest tests/e2e/ -v
uv run ruff check .
uv run mypy --strict src
```

A separate `tests/e2e/` run is needed to capture e2e coverage in the report (number passed and which tests were added/updated). If the PR diff has no new/changed files under `tests/e2e/` — note in the report "E2E not applicable: <reason>" referencing the heuristic from `/work-on-issue` (step 3.5).

If anything is red — **do not post the report**. Notify the user and stop. (In normal flow `/work-on-issue` has already run this to green before merge, so red tests here are an anomaly.)

### 3. Read the issue for AC context

```
mcp__github__issue_read(method=get, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS)
```

Map the changes to the **Acceptance criteria**: which are closed, which are not.

### 4. Build and publish the comment

Structure (English, 5 sections):

```markdown
## What was done
<2–5 bullets: key changes by substance, in terms of issue AC>

## How it works
<2–4 lines: brief description of behavior / architectural decision so a future session can restore context without reading the diff>

## Changed files
- `path/to/file.py` (+X, −Y) — <why>
- `path/to/other.py` (+X, −Y) — <why>

## Tests
- `uv run pytest`: ✅ <N passed>
- `uv run pytest tests/e2e/`: ✅ <M passed> — <added `test_<feature>_e2e.py` / updated `test_<existing>_e2e.py` / E2E not applicable: <reason>>
- `uv run ruff check .`: ✅
- `uv run mypy --strict src`: ✅
- AC coverage: <which AC are closed, which explicitly remain>
- Existing-test changes: <"none" OR list — `path/to/test_file.py::test_name` — deleted/rewritten/skipped, reason: <…>. Confirmed by user in `/work-on-issue` step 3.6.>

## Remaining
<either "nothing, AC fully closed", or an explicit list of what is not done and why — e.g., split off into follow-up issue #M>
```

Publish **immediately**, without preview and without asking the user:

```
mcp__github__add_issue_comment(
  owner=<OWNER>,
  repo=<REPO>,
  issue_number=$ARGUMENTS,
  body=<assembled comment>
)
```

Return to the user the URL of the posted comment.

## When to stop and ask

Only on real anomalies:

- uncommitted changes in `git status`;
- red `pytest`/`ruff`/`mypy`;
- no commits found related to the issue (nothing to describe);
- AC of the issue clearly not closed, but the PR is already merged (user decision needed — open follow-up or finish here).

In the normal case — publish immediately and return the URL.

## Forbidden

- Exactly **one** comment per invocation — do not split into several.
- Do not publish a report if `pytest`/`ruff`/`mypy` are red or there are out-of-scope uncommitted changes.
- Do not use `gh issue comment` — only `mcp__github__add_issue_comment`.
- Do not create files in `.claude/plans/`, `notes/` (for an active task). The report lives **only** in the issue comment.
- Do not show a preview before publishing — this is an autonomous command.
