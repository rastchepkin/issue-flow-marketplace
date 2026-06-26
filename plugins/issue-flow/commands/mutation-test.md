---
description: On-demand mutation testing (mutmut / StrykerJS) over a project's critical logic — reports a mutation score and surviving mutants, never a CI gate
argument-hint: [backend | frontend | <path-or-glob>]
---

Run **mutation testing** on demand and report the result. Mutation testing injects deliberate logic bugs (`>`→`>=`, `and`→`or`, `return x`→`return None`, …), reruns the suite per mutant, and reports a **mutation score** = killed / total. **Surviving** mutants pinpoint logic that no assertion actually checks — they are the output that matters.

This command is **manual and on-demand only**. It is deliberately **not** part of CI, the PR gate, or the TDD loop — mutation runs are slow (full suite × N mutants). It **commits nothing**, touches no workflow, and changes no source. The result is a chat report plus (for the frontend) a local HTML report. A surviving mutant is a **finding**, not a failure — the maintainer decides whether to add a test. Do **not** open a PR, edit tests, or "fix" anything from this command.

## Project config — read this first

The mutation targets and tools are per-project and live in **`.claude/flow.config.md`** (read it first):

- `MUTATION_BACKEND_TOOL` (e.g. `mutmut`, or `none`) and `MUTATION_BACKEND_PATHS` — the backend critical set to mutate (also typically configured in the project's `pyproject.toml` `[tool.mutmut]`).
- `MUTATION_BACKEND_RUNNER` (optional) — a narrower test runner override (default: the project's configured full-suite runner).
- `MUTATION_FRONTEND_TOOL` (e.g. `stryker`, or `none`) and `MUTATION_FRONTEND_GLOB` — the frontend critical set (also in the project's `stryker.conf.json`).

If neither tool is configured, tell the user mutation testing isn't set up for this project and stop.

## Argument

`$ARGUMENTS` selects the scope (default `backend`):

- **`backend`** (or empty) — the backend tool over `MUTATION_BACKEND_PATHS` (its config default).
- **`frontend`** — the frontend tool over `MUTATION_FRONTEND_GLOB` (its config default).
- **An explicit path or glob** — narrow to one module; route to the backend tool if it looks like a backend source path, to the frontend tool if it looks like a frontend path/glob.

## Steps

### 1. Resolve scope and tool

From `$ARGUMENTS`: empty/`backend` → backend tool + default paths; `frontend` → frontend tool + default glob; an explicit path → that tool with a narrowed scope flag. Tell the user (one line, their language) what scope is about to run and that it is slow. **Do not** ask for confirmation — proceed.

### 2a. Backend — mutmut (when `MUTATION_BACKEND_TOOL: mutmut`)

Run from the repo root. **On Windows force UTF-8 I/O**, else mutmut crashes printing its emoji banner to a cp1252 console (`UnicodeEncodeError`):

```bash
# default critical set (from [tool.mutmut] / MUTATION_BACKEND_PATHS)
PYTHONUTF8=1 uv run mutmut run

# explicit narrower scope
PYTHONUTF8=1 uv run mutmut run --paths-to-mutate <path>
```

Notes:

- The configured `runner` is usually the **full** suite (honest but slow). For a fast, focused score, override with `MUTATION_BACKEND_RUNNER` or just the covering tests:

  ```bash
  PYTHONUTF8=1 uv run mutmut run --paths-to-mutate <path> --runner "<MUTATION_BACKEND_RUNNER>"
  ```

- `mutmut run`'s **exit code is non-zero when mutants survive** — expected, not an error. Read the score from the output, never from `$?`.
- List survivors:

  ```bash
  PYTHONUTF8=1 uv run mutmut results          # summary + survived/timeout/suspicious IDs
  PYTHONUTF8=1 uv run mutmut show <id>        # file:line + the exact mutation, per surviving ID
  ```

- `.mutmut-cache` is git-ignored — a re-run reuses it; `rm -f .mutmut-cache` only to force a clean baseline.

### 2b. Frontend — Stryker (when `MUTATION_FRONTEND_TOOL: stryker`)

```bash
cd frontend
npx stryker run                                  # default scope (from stryker.conf.json / MUTATION_FRONTEND_GLOB)
npx stryker run --mutate "<glob>"                # explicit narrower scope
```

Notes:

- Stryker uses the configured test runner and the repo's test config automatically.
- Output: a `clear-text` summary table in stdout (per-file score + survivors) and an HTML report (git-ignored). The `.stryker-tmp/` working dir is git-ignored.

### 3. Report to the user

Print a **concise** report in the user's `USER_LANGUAGE`. Include:

- the **scope** that ran and the tool;
- the **mutation score** (killed / total, %), and timeouts/suspicious if any;
- the **surviving mutants** as a short list of `file:line — <mutation>` (cap ~15; if more, say how many and group by file);
- a one-line read: which survivors look like a genuine assertion gap worth a test vs. equivalent/cosmetic mutants;
- a reminder that this changed nothing and added no gate — the maintainer decides.

Suggested shape (translate to `USER_LANGUAGE`):

```markdown
Mutation testing — <scope> (<tool>)
Score: <killed>/<total> killed (<pct>%)<, timeouts: K, suspicious: M if any>

Surviving mutants (no assertion catches the bug):
- <file>:<line> — <what mutated, e.g. `>` → `>=`>
- …

Read: <which survivors are a real assertion gap vs. equivalent>.
This changed nothing and added no gate — deciding whether to add a test is up to you.
```

## Slowness & narrowing

Mutation testing is **O(mutants × suite time)**. To keep a run usable: narrow `--paths-to-mutate` / `--mutate` to one module; override the runner to only the covering tests; run one scope at a time.

## Forbidden

- Do **not** add mutation testing to CI, the PR gate, pre-commit, or the TDD loop.
- Do **not** commit anything, edit tests, or open a PR — read-only beyond git-ignored caches/reports.
- Do **not** treat a non-zero `mutmut run` exit code or surviving mutants as a build failure — survivors are the deliverable, read by a human.
