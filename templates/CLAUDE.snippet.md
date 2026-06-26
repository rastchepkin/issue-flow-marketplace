<!--
  Merge these sections into the target repo's CLAUDE.md (create CLAUDE.md if absent).
  The moving parts (branch names, gate commands, e2e path, CI check, deploy hooks) are NOT repeated
  here — they live in `.claude/flow.config.md`, which the issue-flow commands read at runtime.
  Replace <USER_LANGUAGE> with the project's chat language.
-->

## Issue-driven workflow (issue-flow plugin)

Any non-trivial task goes through a GitHub Issue. The issue (body + comments) is the single source of
truth. The flow is provided by the `issue-flow` plugin; project-specific values live in
`.claude/flow.config.md`.

1. **Discussion → issue.** `/issue-flow:plan-issue` — clarifies scope, dedups, drafts the body per
   `feature_request.yml` / `bug_report.yml`, creates the issue via `mcp__github__issue_write`.
2. **Work the issue.** `/issue-flow:work-on-issue <N>` — reads the issue, branches from `DEV_BRANCH`,
   runs the TDD loop, opens a PR with a mandatory `Closes #N`, waits for green CI, merges, auto-reports.
3. **Wrap-up.** `/issue-flow:report` — one final comment on the issue (auto-invoked by work-on-issue).
4. **Release.** `/issue-flow:push-to-prod` — opens a `DEV_BRANCH → PROD_BRANCH` PR with aggregated
   `Closes #N` markers, waits for CI, merges as a **merge commit** (not squash).

Creating local plan files (`.claude/plans/*`, `notes/*` for active tasks, any `*-plan.md` at the repo
root) is **forbidden**. The plan lives in the issue.

## Project config

Branch names, the green-before-PR gate commands, the e2e path, the CI check name, the review steps,
and any deploy verification all live in **`.claude/flow.config.md`**. That file is the single place
that couples the (otherwise stack-agnostic, shared) plugin commands to this project. Keep it current.

## Language convention

All artifacts read by the agent are **English**: issue bodies, PR titles/bodies, `/report` comments,
code, branch names, commit messages, docstrings, file/dir names, this `CLAUDE.md`. Chat with the user
runs in `<USER_LANGUAGE>` (also set as `USER_LANGUAGE` in `.claude/flow.config.md`).

## GitFlow

- Task branches are created **from `DEV_BRANCH`**, not `PROD_BRANCH`.
- Branch name: `feat/<N>-<kebab-summary>` (feature) or `fix/<N>-<kebab-summary>` (bug), `<N>` = issue number.
- PR opened **against `DEV_BRANCH`**; body must contain `Closes #<N>` (triggers `project-status.yml`).
- Release to `PROD_BRANCH` via a separate `DEV_BRANCH → PROD_BRANCH` PR (`/issue-flow:push-to-prod`),
  merged as a **merge commit**, not squash — keeps `DEV_BRANCH` history reachable so the next release
  doesn't hit add/add conflicts.
- **Never** `git push --force` to `PROD_BRANCH`/`DEV_BRANCH`. **Never** `--no-verify` without explicit request.

## MCP and GitHub operations

- For all issue/PR operations use `mcp__github__*` (see `.mcp.json`). Do not use `gh` for issue/PR ops —
  MCP returns structured responses and one source of truth.
- `gh` is allowed for what MCP doesn't cover (local auth, repo variables/secrets setup).

## Infrastructure we rely on

- `.github/ISSUE_TEMPLATE/feature_request.yml` / `bug_report.yml` — issue body structure.
- `.github/pull_request_template.md` — PR body structure (`Closes #`, gate checklist).
- `.github/workflows/project-status.yml` — auto-moves project cards by `Closes #N`. No-op until the
  `PROJECT_ID` / `STATUS_FIELD_ID` / `OPTION_*` repo variables and `PROJECTS_TOKEN` secret are set.
- `.mcp.json` — GitHub MCP wired.
- `.claude/flow.config.md` — per-project values for the issue-flow commands.
- `.claude/settings.json` — pre-approves the read/git/GitHub-MCP calls the commands make.
