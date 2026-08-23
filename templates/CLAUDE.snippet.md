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

`/issue-flow:batch-work <N1,N2,…>` runs step 2 over several issues, one at a time, each in its own
sub-agent. An issue that needs a human decision is **parked** (branch pushed, PR open, labelled
`needs-decision`) and the batch continues — unless a later issue in the list depends on it, in which
case it escalates instead.

### Existing tests are gated

Adding tests is free. **Editing or deleting an existing test** goes through the step-3.6 gate: a
deny-list (deleted test file, a new `skip`/`xfail` on a green test, tests outside the modules the
issue touches, anything under `TEST_PROTECTED_PATHS`) always reaches a human, and everything else is
scored by an isolated judge sub-agent against the issue's AC, the test diff, and the source diff.
`low` is auto-accepted; `medium`/`high` stops and asks. Either way the change is recorded in the PR
body's "Test changes" section and in the `/report` comment — nothing lands unrecorded.

### The flow is resumable

Sessions die, and cloud/web sessions go dormant during a CI wait. Every command detects existing
branch / PR / report state and re-enters where it left off, so recovering is just re-running the same
command — never a restart, and never a second branch or PR. With `AUTO_MERGE: true` the PR is armed
with GitHub's auto-merge and lands on green CI without an agent alive; a resumed
`/issue-flow:work-on-issue <N>` then only publishes the report.

### Backlog

An issue that is needed but not scheduled carries the `status:backlog` label (`BACKLOG_LABEL`).
`/issue-flow:plan-issue` applies it; `/issue-flow:work-on-issue` removes it the moment work starts.
The board's active-queue view filters on `-label:status:backlog`.

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
- `gh` is allowed for what MCP doesn't cover (local auth, repo variables/secrets setup) plus three
  scoped exceptions the commands document inline: the project-board GraphQL mutation, the code-review
  skill's own PR comment, and arming auto-merge (`gh pr merge --auto`, which MCP cannot do).

## Infrastructure we rely on

- `.github/ISSUE_TEMPLATE/feature_request.yml` / `bug_report.yml` — issue body structure.
- `.github/pull_request_template.md` — PR body structure (`Closes #`, gate checklist).
- `.github/workflows/project-status.yml` — auto-moves project cards by `Closes #N`. No-op until the
  `PROJECT_ID` / `STATUS_FIELD_ID` / `OPTION_*` repo variables and `PROJECTS_TOKEN` secret are set.
- `.mcp.json` — GitHub MCP wired.
- `.claude/flow.config.md` — per-project values for the issue-flow commands.
- `.claude/settings.json` — pre-approves the read/git/GitHub-MCP calls the commands make.
