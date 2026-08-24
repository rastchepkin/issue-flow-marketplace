# GitHub setup for the issue-flow plugin

Everything you have to configure **on GitHub** before the flow runs end to end, in the order it
should be done. Written for a person doing it by hand.

> If you'd rather not do it by hand: open Claude Code in the target repo and say *"install the
> issue-flow plugin from `<marketplace-url>` and set it up here, follow APPLY.md"*. [APPLY.md](../APPLY.md)
> is the same material written as a runbook for the agent — it will ask you for what it can't infer.

Substitute `<OWNER>/<REPO>` throughout, and `develop`/`main` if your branches are named differently.
Everything here needs the `gh` CLI authenticated (`gh auth login`) with admin on the repo.

---

## What you end up with

```
 Issues                                   Board (GitHub's default 3 columns)
 ────────────────────────                 ──────────────────────────────────
 open, status:backlog        ····filtered out····▶  (hidden from the Todo view)
 open                        ───────────────────▶   Todo
 open, work started          ───────────────────▶   In Progress
 closed by "Closes #N"       ───────────────────▶   Done
 closed as "not planned"     ····auto-archived···▶  (off the board, still searchable)

 Merged into develop but not live  →  the open develop → main release PR
 Live in production                →  git log main / GitHub Releases
```

No custom Actions, no bot token for the board, no columns to keep in sync. Only three of the five
states are on the board; two are a label and a native close reason.

---

## 1. Two long-lived branches

The flow assumes a GitFlow split: an integration branch (`DEV_BRANCH`) and a production branch
(`PROD_BRANCH`). Feature branches always come off `develop`; `main` only ever receives release PRs.

```bash
git checkout main
git pull
git checkout -b develop
git push -u origin develop
```

Leave `main` as the repo's default branch.

## 2. Labels

Four labels, all used by the commands:

```bash
gh label create "type:feature"   --color 1d76db --force
gh label create "type:bug"       --color d73a4a --force
gh label create "status:backlog" --color ededed --force --description "Needed, but not scheduled yet"
gh label create "needs-decision" --color fbca04 --force --description "Parked - waiting on a human decision"
```

| Label | Set by | Cleared by | Means |
|---|---|---|---|
| `type:feature` / `type:bug` | the issue template | — | picks the branch prefix (`feat/` / `fix/`) |
| `status:backlog` | `/plan-issue` (flag `-backlog`, or the sign-off answer; default under `-auto`) | `/work-on-issue` when work starts | needed, but not now |
| `needs-decision` | `/batch-work` when it parks a PR | you, when you decide | a PR is open and waiting on a human |

## 3. Repo files

Copy from the marketplace's `templates/github/` into your repo's `.github/`:

```
.github/ISSUE_TEMPLATE/feature_request.yml
.github/ISSUE_TEMPLATE/bug_report.yml     ← edit the "Environment" hint to your real runtime
.github/ISSUE_TEMPLATE/config.yml
.github/pull_request_template.md          ← edit the checklist to your gate commands
.github/workflows/ci.yml                  ← from ci.example.yml, see next step
```

## 4. CI

The flow will not merge anything without a green check, so this is the load-bearing piece.

Start from `templates/github/workflows/ci.example.yml`, delete the stacks you don't use, and make the
commands match exactly what you'll put in `.claude/flow.config.md` (`TEST_CMD`, `LINT_CMD`,
`FORMAT_CMD`, `TYPECHECK_CMD`, `FE_*`).

Three things must line up, or the flow waits forever on a check that never reports:

```
 the JOB name in ci.yml   ──┬──▶  CI_CHECK_NAME in .claude/flow.config.md
                            └──▶  the "required status check" in branch protection (step 5)
```

The example names the job `CI`, which matches the default `CI_CHECK_NAME: CI`. If you use a matrix,
the checks are reported as `CI (3.12)`, `CI (20)`, … — then *those* are the names to require.

Open a throwaway PR into `develop` and confirm the check appears with the name you expect before
moving on. `gh pr checks <PR>` lists them.

## 5. Branch protection + auto-merge

This pair is what lets the flow survive a session that dies or goes dormant while CI runs: GitHub
performs the merge itself once the checks pass.

**5a. Allow auto-merge on the repo:**

```bash
gh api -X PATCH repos/<OWNER>/<REPO> -F allow_auto_merge=true
```

**5b. Require status checks on `develop`** (and on `main` if you use `/push-to-prod`):

```bash
gh api -X PUT repos/<OWNER>/<REPO>/branches/develop/protection --input - <<'JSON'
{
  "required_status_checks": { "strict": true, "contexts": ["CI"] },
  "enforce_admins": false,
  "required_pull_request_reviews": null,
  "restrictions": null
}
JSON
```

> ⚠️ **5b is a safety property, not a nicety.** `gh pr merge --auto` defers the merge *only* because a
> required check is pending. On a branch with **no** required checks, the same command merges
> **immediately** — so turning on `AUTO_MERGE` without this does not speed the flow up, it deletes the
> CI gate. The commands warn if they notice a PR merging while CI is still pending, but detection is
> not prevention.

Verify before you enable it:

```bash
gh api repos/<OWNER>/<REPO>/branches/develop/protection/required_status_checks --jq '.contexts'
```

An empty list or a `404` means it is not configured — keep `AUTO_MERGE: false` until this prints your
check name.

> Branch protection on **private** repos needs a paid plan. On a private free repo, either use
> [rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets)
> if available to you, or leave `AUTO_MERGE: false` — the flow then polls and merges itself, which
> works fine as long as the session stays alive.

## 6. A token for the GitHub MCP server

The commands talk to GitHub through the `github` MCP server, not the `gh` CLI. It needs a token with
read/write on issues, pull requests, and contents — a classic PAT with `repo`, or a fine-grained token
scoped to this repo.

Put it in the environment variable your `.mcp.json` references (merge
`templates/github.mcp.json` into the repo's `.mcp.json`), and reference it as `${VAR}` — never inline
the token into a committed file.

Confirm the server connects before relying on it: `/mcp` in Claude Code, or ask it to list open issues.

## 7. The project board

Optional — the flow works without it, and skips the board step with a warning if it isn't configured.

**7a. Create a Project** (Projects → New project → Board). Keep the **default Status field: Todo /
In Progress / Done.** Do not add Develop, Production, Backlog, or Cancelled columns — those states are
covered by the release PR, the `status:backlog` label, and the native close reason respectively.

**7b. Turn on three built-in workflows** (Project → ⋯ → Workflows). All three ship with GitHub; none
requires code or a token:

| Workflow | Configure as | Gives you |
|---|---|---|
| Item added to project | set Status → **Todo** | new issues land in Todo |
| Item closed | set Status → **Done** | `Closes #N` on merge closes the issue → card goes to Done |
| Auto-archive items | filter `is:issue is:closed reason:"not-planned"` | cancelled work leaves the board without being deleted |

Also turn on **Auto-add to project** (filter `is:issue is:open`) so you don't have to add cards by hand.

> The *Item closed* workflow does not branch on *why* an issue closed, so a cancelled issue may flash
> through Done before Auto-archive removes it. The end state is correct either way. If that bothers
> you, drop the *Item closed* workflow and move cards to Done by hand.

**7c. Add the Todo view filter** so parked work doesn't clutter the active queue: on the board view,
set the filter to

```
-label:status:backlog
```

**7d. Read the three IDs** the `/work-on-issue` "In Progress" move needs:

```bash
gh api graphql -f query='
  query($login:String!,$number:Int!){
    user(login:$login){ projectV2(number:$number){
      id
      fields(first:20){ nodes{ ... on ProjectV2SingleSelectField{ id name options{ id name } } } }
    } } }' -F login="<your-gh-login>" -F number=<project-number>
```

Use `organization(login:...)` instead of `user(login:...)` for an org project. The project number is
the one in its URL. From the output take:

- `projectV2.id` → `PROJECT_ID` (starts `PVT_`)
- the `Status` field's `id` → `STATUS_FIELD_ID` (starts `PVTSSF_`)
- that field's **In Progress** option `id` → `OPTION_IN_PROGRESS`

Your `gh` token needs the `project` scope to read this: `gh auth refresh -s project`.

## 8. `.claude/flow.config.md`

Copy `templates/flow.config.example.md` to `.claude/flow.config.md` in the repo and fill it in. This
is the only per-project file the plugin reads, and plugin updates never touch it.

The keys that depend on the steps above:

```
- DEV_BRANCH: develop            ← step 1
- PROD_BRANCH: main              ← step 1
- CI_CHECK_NAME: CI              ← step 4, must equal the job name
- AUTO_MERGE: true               ← step 5, ONLY after 5b verifies
- BACKLOG_LABEL: status:backlog  ← step 2
- PROJECT_ID: PVT_...            ← step 7d
- STATUS_FIELD_ID: PVTSSF_...    ← step 7d
- OPTION_IN_PROGRESS: ...        ← step 7d
```

## 9. Check it works

Run one small real task end to end:

1. `/issue-flow:plan-issue` a trivial change → the issue is created with a `type:*` label and lands in
   **Todo** (or gets `status:backlog` if you said so at the sign-off).
2. `/issue-flow:work-on-issue <N>` → the card moves to **In Progress**, a `feat/<N>-…` branch is cut
   from `develop`, a PR opens with `Closes #N` in the body, and the CI check appears on it.
3. With `AUTO_MERGE: true`: the PR shows **auto-merge enabled** and does **not** merge while CI is
   pending. If it merges instantly, step 5b is missing — set `AUTO_MERGE: false` and fix it.
4. On merge: the issue closes and its card moves to **Done**, and one `/report` comment is posted.
5. Interrupt a run while CI is pending, then re-run `/issue-flow:work-on-issue <N>` — it must resume
   (report only, if the PR merged meanwhile), not create a second branch or PR.
6. Close some throwaway issue with `gh issue close <N> --reason "not planned"` and confirm the card
   leaves the board while the issue stays findable under `is:closed reason:"not planned"`.

---

## When something doesn't work

| Symptom | Cause | Fix |
|---|---|---|
| The flow waits forever on CI | `CI_CHECK_NAME` ≠ the job name reported on the PR | `gh pr checks <PR>` and copy the exact name |
| The PR merged before CI ran | no required status checks on the base branch | step 5b, then re-verify |
| `gh pr merge --auto` errors | auto-merge not allowed on the repo | step 5a |
| Card never leaves **In Progress** | the PR body has no `Closes #N`, so the issue never closed | fix the PR body; the flow treats this line as mandatory |
| Card never reaches **Todo** | *Auto-add to project* / *Item added* workflows are off | step 7b |
| Board step skipped with a warning | one of the three IDs in flow.config is blank | step 7d |
| `/work-on-issue` refuses to start | the issue is closed as *not planned* — it was cancelled | reopen it deliberately, or plan a new one |
| `security-review` fails on `origin/HEAD` | the ref isn't set in a fresh clone | the command sets it itself; if it still fails, `git remote set-head origin develop` |
