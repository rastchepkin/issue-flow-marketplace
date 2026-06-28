# APPLY.md — installing the issue-flow plugin into this project

You are an agent installing the `issue-flow` plugin into the repo you're running in. Goal: the
commands `/issue-flow:plan-issue`, `/issue-flow:work-on-issue`, `/issue-flow:report`,
`/issue-flow:push-to-prod` run end-to-end against this repo's GitHub, driven by a per-project
`.claude/flow.config.md`. Work through the phases in order. Where a value isn't inferable from the
repo, **ask the user** (questions are batched in Phase 1) — don't invent project facts.

> The command bodies are stack-agnostic and read `.claude/flow.config.md` at runtime. So the bulk
> of "adapting to this project" is **writing that one config file correctly** — not editing commands.

---

## Phase 0 — Preconditions

1. **Git + GitHub remote.** `git remote get-url origin` must be a GitHub repo. No GitHub remote →
   stop and tell the user; the whole flow is GitHub-issue/PR based.
2. **Two long-lived branches.** This flow assumes a GitFlow split (integration + production). Detect
   candidates with `git branch -a` and `git remote show origin` (its "HEAD branch" is the default,
   usually the prod branch). Common pairs: `develop`/`main`, `dev`/`main`. Confirm in Phase 1.
3. **Existing files.** Note whether `.claude/`, `.mcp.json`, `CLAUDE.md`, `.github/` already exist —
   you'll merge, not overwrite.

## Phase 1 — Gather the per-project values (ask the user once)

These become `.claude/flow.config.md`. Ask them together, with best-guess defaults filled in:

| Value | For | Default guess |
|---|---|---|
| `DEV_BRANCH` | integration branch; PRs target it; work-on-issue branches from it | the non-prod long-lived branch (often `develop`) |
| `PROD_BRANCH` | release target; push-to-prod merges into it | the default branch (often `main`) |
| Gate commands | the exact lint/type/test/build commands the flow runs | infer from the repo (Phase 2) and confirm |
| Separate frontend suite? | whether to keep the `FE_*` gates | presence of `frontend/`, a `package.json` with a `build` script |
| `E2E_PATH` | where e2e tests live, or `none` | detect a `tests/e2e/`-like dir |
| `CI_CHECK_NAME` | the PR check the flow waits on before merge | the check name in the repo's CI workflow (often `CI`) |
| `CODE_REVIEW_SKILL` / `SECURITY_REVIEW` | review steps in work-on-issue, or `none` | `code-review:code-review` / `/security-review` if those skills exist, else `none` |
| `DEPLOY_VERIFY` | whether to poll a deploy after merge | default **`none`** unless the user has a deploy to verify |
| `USER_LANGUAGE` | language of the commands' user-facing chat | ask; default English |

## Phase 2 — Detect the stack's gate commands

Find this repo's real green-before-PR commands (don't keep the Python/uv defaults unless they're right):

- **Python + uv**: `pyproject.toml` + `uv.lock` → `uv run pytest`, `uv run ruff check .`, `uv run mypy`.
- **Python + poetry/pip**: `poetry run pytest` / `pytest`, `ruff`/`flake8`, `mypy`.
- **Node/TS**: `package.json` scripts → `npm run lint`, `npm run typecheck`, `npm test`, `npm run build` (or `pnpm`/`yarn`).
- **Go**: `go test ./...`, `go vet ./...`, `golangci-lint run`.
- **Rust**: `cargo test`, `cargo clippy`, `cargo fmt --check`.

## Phase 3 — Install the plugin

1. **Add the marketplace** (from the git repo where this kit was published):
   ```
   /plugin marketplace add <owner>/<marketplace-repo>      # GitHub shorthand
   # or: /plugin marketplace add https://gitlab.com/<...>.git
   ```
2. **Install the plugin** (scope `project` so it's committed for the whole team, or `user` for just you):
   ```
   /plugin install issue-flow@issue-flow-marketplace --scope project
   ```
3. Confirm the four commands appear (`/issue-flow:plan-issue`, …). If you edited the marketplace in
   the same session, run `/reload-plugins`.

> Alternative without a marketplace (local testing only): `claude --plugin-dir <path>/plugins/issue-flow`.

## Phase 4 — Write `.claude/flow.config.md`

Copy `templates/flow.config.example.md` → `.claude/flow.config.md` and fill in every key from Phase 1–2.
Delete the `FE_*` block if there's no frontend; set `E2E_PATH: none` if there's no e2e suite; set
`DEPLOY_VERIFY: none` unless wiring a deploy check. **This file is the only per-project coupling** —
keep the `KEY: value` names exactly as written (the commands look them up by key).

## Phase 5 — Install the GitHub infra + supporting files

Copy from `templates/` into the repo (merge where a file exists):

1. **GitHub** → `.github/`
   - `github/ISSUE_TEMPLATE/{feature_request,bug_report,config}.yml`
   - `github/pull_request_template.md` — adjust the checklist gate names to this repo's commands
   - `github/workflows/project-status.yml`
   - In `bug_report.yml`, change the **Environment** description (`Python, Django, Node, OS`) to this
     project's real runtime.
2. **MCP** → merge `github.mcp.json`'s `github` server into the repo's `.mcp.json` (create if absent).
   Pick an env-var name for the PAT and reference it as `${...}` — never inline the token.
3. **Settings** → merge `settings.example.json` into `.claude/settings.json`. Trim the `Bash(...)`
   tooling allow-entries to the Phase-2 commands; keep the git + `mcp__github__*` entries.
4. **CLAUDE.md** → merge `CLAUDE.snippet.md`'s sections into the repo's `CLAUDE.md` (create if absent).
   The snippet points at `.claude/flow.config.md` for the moving parts, so little to substitute.

## Phase 6 — GitHub-side wiring (needs `gh` + admin on the repo)

1. **Issue labels** (the templates apply `type:feature` / `type:bug`):
   ```bash
   gh label create "type:feature" --color 1d76db --force
   gh label create "type:bug"     --color d73a4a --force
   ```
2. **PAT for the GitHub MCP server** — token with issue/PR/contents read-write (classic: `repo` +
   `read:org`; or fine-grained scoped to this repo). Put it in the env var referenced in `.mcp.json`.
   Verify the MCP server connects before relying on it.
3. **CI named `CI_CHECK_NAME`.** The flow waits on a PR check by that name (set in flow.config). The
   repo must have a CI workflow running on `pull_request` for `DEV_BRANCH` and `PROD_BRANCH` that runs
   the Phase-2 gate commands. If none exists, create one (the source repo's `ci.yml` is a Python+Node
   example to adapt, not copy).
4. **Project board automation (optional).** `project-status.yml` is a **no-op until configured** —
   safe to install and ignore. To enable, set the repo variables `PROJECT_ID`, `STATUS_FIELD_ID`,
   `OPTION_DEVELOP`, `OPTION_PRODUCTION` and the secret `PROJECTS_TOKEN` (classic PAT, scopes `repo` +
   `project`). Discover the IDs with:
   ```bash
   gh api graphql -f query='
     query($login:String!,$number:Int!){
       user(login:$login){ projectV2(number:$number){
         id
         fields(first:20){ nodes{ ... on ProjectV2SingleSelectField{ id name options{ id name } } } }
       } } }' -F login="<gh-login>" -F number=<project-number>
   gh variable set PROJECT_ID        --body "PVT_..."
   gh variable set STATUS_FIELD_ID   --body "PVTSSF_..."
   gh variable set OPTION_DEVELOP    --body "<develop-option-id>"
   gh variable set OPTION_PRODUCTION --body "<production-option-id>"
   gh secret set PROJECTS_TOKEN
   ```
   (Use `organization(login:...)` instead of `user(...)` for an org project. Board columns should
   include `Develop` and `Production` to match the option lookups.)

## Phase 7 — Deploy verification (only if `DEPLOY_VERIFY` ≠ `none`)

The plugin ships `/issue-flow:verify-deploy` for **Dokploy**-hosted backends whose health endpoint
exposes the running git SHA. To enable it: set `DEPLOY_VERIFY: dokploy`, `DEPLOY_VERIFY_SKILL:
/issue-flow:verify-deploy`, and fill the `DEPLOY_*` target keys (app ids, health URLs, base URLs) plus
the optional `LOGS_*` / `PLAYWRIGHT_*` keys in `.claude/flow.config.md` — those targets are per-project
and never live in the marketplace. For a **non-Dokploy** platform, write a project-local
`.claude/commands/verify-deploy.md` instead and point `DEPLOY_VERIFY_SKILL` at it. Otherwise leave
`DEPLOY_VERIFY: none` and the deploy steps self-skip.

`/issue-flow:mutation-test` (manual-only mutmut/Stryker) is also shipped; set the `MUTATION_*` keys in
flow.config to point it at this project's critical modules, or leave the tools `none` to disable it.

## Phase 8 — Verify the install

1. The four commands are listed and namespaced (`/issue-flow:…`).
2. `.claude/flow.config.md` exists and every key matches this repo (branches, gate commands).
3. The GitHub MCP server connects (try a read, e.g. list issues).
4. Dry-run: `/issue-flow:plan-issue` a tiny change → `/issue-flow:work-on-issue <N>` → confirm it
   reads flow.config, branches from `DEV_BRANCH`, opens a PR with `Closes #N`, waits for the CI check,
   and `/issue-flow:report` posts one comment. Fix any mismatch surfaced here in flow.config (not in
   the command files — those are shared).

---

## Keeping projects in sync (tell the user)

- The commands live in the marketplace repo. To change the flow: edit there, `git push`, then in each
  project run **`/plugin marketplace update issue-flow-marketplace`** (commands auto-version by commit
  SHA, so the latest commit is picked up). `/reload-plugins` applies it in the current session.
- `.claude/flow.config.md` is per-project and never touched by updates — no merge conflicts.
- If you add a new config key to a command, also add it to `templates/flow.config.example.md` and tell
  existing projects to add it to their `.claude/flow.config.md`.

### The contract that makes the flow work

- **Issue = source of truth.** No local plan files. Plan lives in the issue body; result in the `/report` comment.
- **`Closes #N` in the PR body** is mandatory — it's what `project-status.yml` keys on and what push-to-prod aggregates.
- **Branch from `DEV_BRANCH`, PR into `DEV_BRANCH`, release `DEV_BRANCH → PROD_BRANCH`** as a **merge commit** (never squash the release).
- **Autonomous by default**; stop only at real forks (ambiguous AC, merge conflict, unfixable red CI, security finding, existing-test edits). Flags tune this: `/work-on-issue -bypass_low|-bypass` relax the existing-test gate, `/batch-work -no-merge` runs an unattended night batch that opens PRs without merging, and `/plan-issue` (without `-auto`) instead runs a product review (UX + plan/AC sign-off) before creating the issue.
- **English for all agent-read artifacts**; user-facing chat in `USER_LANGUAGE`.
