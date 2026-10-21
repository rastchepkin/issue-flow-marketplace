---
description: Autonomously drive an issue to merge into develop and publish a report — no per-step confirmations
argument-hint: "<issue-number> [-ask | -bypass]"
model: opus
---

Take issue `#$ARGUMENTS` in the current repo and drive it to completion **autonomously**: branch → spec tests (isolated author) → TDD → PR → green CI → merge to `develop` → automatic `/report`. The issue is the single source of truth.

Before any `mcp__github__*` call, resolve `<OWNER>` and `<REPO>` from `git remote get-url origin` (format `https://github.com/<OWNER>/<REPO>.git`) — this file is template-shaped and must not hardcode a specific repo.

## Arguments

`$ARGUMENTS` carries the issue number plus optional flags. Parse it once, up front:

- **`<N>`** — the single positive integer in `$ARGUMENTS`. **Everywhere below, `$ARGUMENTS` used as an issue number means `<N>`** (flags stripped) — in `#$ARGUMENTS`, `issue_number=$ARGUMENTS`, the branch name, `Closes #$ARGUMENTS`, and the `/report $ARGUMENTS` hand-off. If no integer is present, stop and ask.
- **flags** — tokens starting with `-`. Only these are recognized; they change the **existing-test gate** (step 3.6) and nothing else:
  - **none (default)** — the gate runs the **test-change judge** (an isolated sub-agent). `low` risk is auto-accepted and recorded; `medium`/`high` stops and asks.
  - **`-ask`** — skip the judge; **any** edit or deletion of an existing test stops and asks. The strict pre-judge behavior, for work on critical code.
  - **`-bypass`** — auto-accept **all** existing-test changes regardless of the judge's verdict; never pause at the gate. For fully unattended runs.
  - **`-bypass-low`** — deprecated alias for the default (the judge already auto-accepts exactly the low-risk changes this flag used to cover). Accept it silently; do not error.

The flags affect **only** the existing-test gate. Every other "stop and ask" fork in the Mode section (merge conflict, unfixable CI, AC ambiguity, scope expansion, security finding) still escalates exactly as before — `-bypass` does **not** silence those, and the step-3.6 **deny-list** overrides `-bypass` as documented in that step.

## Project config — read this first

This command ships in the shared `issue-flow` plugin, so its body is stack-agnostic. The project-specific values live in **`.claude/flow.config.md`** at the repo root. **Read that file before acting** and substitute its values wherever this command shows a placeholder literal:

- branch names → `DEV_BRANCH` / `PROD_BRANCH` (the literals `develop` / `main` in this file are placeholders)
- gate commands → `TEST_CMD`, `LINT_CMD`, `FORMAT_CMD`, `TYPECHECK_CMD`, and the `FE_*` gates (literals like `uv run pytest` are placeholders)
- e2e location → `E2E_PATH` (skip every e2e instruction if it is `none`)
- CI check to wait on → `CI_CHECK_NAME`
- review steps → `CODE_REVIEW_SKILL` / `SECURITY_REVIEW` (skip a step whose value is `none`)
- test-change judge → `TEST_JUDGE`, `TEST_JUDGE_MODEL`, `TEST_PROTECTED_PATHS` (step 3.6)
- spec-driven tests → `SPEC_TESTS` (`on` | `confirm` | `existing` | `off`), `SPEC_AUTHOR_MODEL` (steps 3.0, 3.9), `SPEC_MUTATION` (step 3.8, which also needs the `MUTATION_*` keys)
- backlog label → `BACKLOG_LABEL` (step 1.5)
- manual prod steps label → `PROD_ACTION_LABEL` (step 3.7; blank disables it)
- merge strategy → `AUTO_MERGE` (step 6)
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
- **modification or deletion of existing tests judged `medium`/`high` risk**, or hitting the deny-list — gated at step 3.6. Adding new tests is always fine, no confirmation needed.
- **a spec test (step 3.0) looks wrong to you**, or cannot be made green without contradicting another one — that is an AC/spec ambiguity, not a test problem. Never edit a spec test to get past it (see step 3).
- **the security review flagged a security problem** (see step 5.6) — never auto-fix-and-merge a security finding; surface it to the user and stop. Only an **actual finding** is the fork: a **clean** security review is **not** a stop point — it is a pass-through, keep going to CI/merge in the same turn without pausing.

In all other cases — proceed to the end without pauses.

## Resume — this command is idempotent

A session can die, be interrupted, or go dormant mid-flow (common in cloud/web environments during the CI wait). Re-invoking `/issue-flow:work-on-issue $ARGUMENTS` must therefore **never start over** — step 2 detects how far the previous run got and re-enters at the right step. There is no separate "finish" command: the same invocation both starts and resumes.

## Steps

### 1. Read the issue

```
mcp__github__issue_read(method=get, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS)
mcp__github__issue_read(method=get_comments, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS)
```

Study **body + all comments** — they may contain scope refinement, AC changes, or intermediate decisions. From labels infer `<type>` for the branch: `feat` (label `type:feature`) or `fix` (label `type:bug`).

**Check the issue is actually open.** GitHub records *why* an issue closed in `state_reason`:

- `state: open` → proceed normally.
- `state: closed`, `state_reason: not_planned` → the issue was **cancelled**: someone decided it is no longer worth doing. Stop and ask the user whether to reopen it before doing anything else; quote the closing comment if there is one. Never silently work a cancelled issue — that is the one case where the board and the issue agree the work should not happen.
- `state: closed`, `state_reason: completed` → already done. Say so and stop, unless the user is explicitly asking for follow-up work (then the right move is a new issue, not this one).

While reading the comments, note whether a **`/report` comment already exists** (a comment whose body starts with `## What was done`) — step 2 needs that signal.

Send the user **one short message** (3–6 lines): task, AC, approach, affected files. This is a notification, **not** a confirmation request — go straight to step 1.5. On a **resume** (step 2 finds existing work), replace it with one line saying where you are picking up.

### 1.5. Move the issue to "In Progress" and clear the backlog label (config-gated, best-effort)

Right away — **before** the branch and PR exist — flip the issue's project card to **In Progress** so the board reflects that work has started.

This is the **only** board move the flow performs by hand. The other two are GitHub's own built-in project workflows and need no code from us: *Item added to project → Todo*, and *Item closed → Done* (the PR's `Closes #N` closes the issue on merge, which fires it). "Work has started" has no native trigger, which is why this one step exists.

**Clear the backlog label first.** Read `BACKLOG_LABEL` from `.claude/flow.config.md` (default `status:backlog`). That label means "needed, but not now"; taking the issue into work makes it false. Removing it is what keeps a board view filtered on `-label:<BACKLOG_LABEL>` honest:

```
mcp__github__issue_write(method=update, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS, labels=[<the issue's current labels, minus BACKLOG_LABEL>])
```

Skip silently if the label is absent, if `BACKLOG_LABEL` is blank, or if the update fails — best-effort, never blocking.

Then the board move. Read `PROJECT_ID`, `STATUS_FIELD_ID`, and `OPTION_IN_PROGRESS` from `.claude/flow.config.md` (the `## Project board` block).

- **Graceful skip:** if any of the three is missing or blank, print one line — `Project board not configured (PROJECT_ID/STATUS_FIELD_ID/OPTION_IN_PROGRESS) — skipping In Progress move.` — and go straight to step 2. **Never** let this block the flow.

Otherwise run the GraphQL via `gh api graphql` (a project-board mutation, not a PR/issue op, so it is outside the "only `mcp__github__*`" rule — a deliberate scoped exception, like the `gh`-based code review in step 5.5). Resolve the issue node id, ensure the issue is on the board, then set the status field.

```bash
set -euo pipefail

ISSUE_NODE_ID=$(gh api graphql \
  -f query='query($owner:String!,$repo:String!,$num:Int!){repository(owner:$owner,name:$repo){issue(number:$num){id}}}' \
  -f owner=<OWNER> -f repo=<REPO> -F num=$ARGUMENTS \
  --jq '.data.repository.issue.id')

ITEM_ID=$(gh api graphql \
  -f query='mutation($p:ID!,$c:ID!){addProjectV2ItemById(input:{projectId:$p,contentId:$c}){item{id}}}' \
  -f p=<PROJECT_ID> -f c="$ISSUE_NODE_ID" \
  --jq '.data.addProjectV2ItemById.item.id')

gh api graphql \
  -f query='mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){updateProjectV2ItemFieldValue(input:{projectId:$p,itemId:$i,fieldId:$f,value:{singleSelectOptionId:$o}}){projectV2Item{id}}}' \
  -f p=<PROJECT_ID> -f i="$ITEM_ID" -f f=<STATUS_FIELD_ID> -f o=<OPTION_IN_PROGRESS> \
  > /dev/null

echo "Issue #$ARGUMENTS moved to In Progress."
```

`addProjectV2ItemById` is idempotent — re-running on an issue already on the board just returns its existing item id, so resuming `/work-on-issue` is safe.

This step is **best-effort**: if the call fails (e.g. the local `gh` token lacks `project` scope, or the option id is stale), print a one-line warning and **continue to step 2** anyway — a board hiccup must not stop the actual work.

### 2. Detect where to resume, then branch from `develop`

**First, look for existing work on this issue.** Never assume a clean slate:

```bash
git fetch origin <DEV_BRANCH>
git ls-remote --heads origin | grep -E "refs/heads/(feat|fix)/$ARGUMENTS-" || true
```

If a remote branch matches, read its PR (and whether it merged):

```
mcp__github__list_pull_requests(owner=<OWNER>, repo=<REPO>, head=<OWNER>:<branch>, state=all)
```

Combine that with the report-comment signal from step 1 and re-enter at the matching step:

| Observed state | Re-enter at |
|---|---|
| no remote branch, no PR | **step 2** below — create the branch |
| remote branch, no PR | checkout the branch; **step 3.0** if the spec applies and the branch has no `test(spec):` commit yet (a spec already written to the issue by 3.0a is reused, not redrafted), else **step 3** — finish the work |
| PR **open**, no code-review comment on it | **step 5.5** |
| PR **open**, code-review comment present | **step 5.6** (the security review always re-runs on resume — it leaves no artifact to detect, and re-running a hard gate is the safe direction) |
| PR **merged**, no `/report` comment on the issue | **step 8** |
| PR **merged**, `/report` comment present | nothing to do — tell the user in one line and exit |
| issue **cancelled** (closed as `not_planned`) but a branch/PR exists | stop and ask. The work was abandoned after it started, so the open PR is now litter: offer to close the PR (leaving the branch — deleting it is the user's call). Do not merge and do not report. |

Detect the code-review comment via `mcp__github__pull_request_read(method=get_comments, …)`. When resuming onto an existing branch, `git checkout <branch> && git pull --ff-only origin <branch>` first, and do **not** re-run steps the table says are already done.

**Clean slate — create the branch:**

```bash
git checkout <DEV_BRANCH>
git pull --ff-only origin <DEV_BRANCH>
git checkout -b <type>/$ARGUMENTS-<kebab-summary>
```

`<kebab-summary>` — a short 2–4 word English summary. **Never** branch from `main`, **never** work directly on `develop`/`main`.

### 3.0. Spec tests — written by an isolated author, then frozen

**Why.** When the agent that writes the code also writes the tests, the tests tend to confirm the code it is about to write rather than the requirement. Mutation testing does not fix this: it only measures assertions against code that exists, and is blind to requirements nobody tested. So the tests that define "done" are written **first**, by a **separate sub-agent**, from the issue's requirements, risks, edge cases and invariants, and the implementer may not change them.

**When it runs.** Whenever `SPEC_TESTS` is not `off`. The value decides what happens when the issue has **no spec yet**, meaning no `AC-<n>:` IDs and no `## Verification spec` (typical for issues written before this flow, or by hand):

| `SPEC_TESTS` | Issue has a spec | Issue has no spec |
|---|---|---|
| `on` (**default**, also when the key is absent) | use it | **draft it** (3.0a), write it to the issue, **continue** |
| `confirm` | use it | draft it (3.0a), write it to the issue, **stop** and ask the user to approve or edit it, then continue |
| `existing` | use it | plain TDD loop (step 3), no spec tests |
| `off` | plain TDD loop | plain TDD loop |

#### 3.0a. Draft the spec when the issue has none

The spec decides what gets tested. If the implementer wrote it, it would shape the requirements around the code it plans to write, which is the same circularity as writing its own tests. So the draft comes from a **separate** sub-agent (`subagent_type=claude`, `model=<SPEC_AUTHOR_MODEL>`). Give it the issue and the repo, and nothing of your own:

```
You are the SPEC WRITER for issue #<N> in this repo. Another agent will turn your spec into tests, and a third will implement the change. Neither may change your output. Do not write tests or code.

The issue (body + all comments, verbatim):
<…>

1. Decide if a spec applies. It does NOT apply to docs-only, config-only, or pure refactors with no behavior change. Then return {"applicable":false,"why":"…"} and stop.
2. Read the code the issue touches, AND its callers and the tests that cover it. In an existing codebase the main risk is breaking adjacent behavior, and only the code shows you where that is.
3. Write:
   - AC: number the issue's existing acceptance criteria `AC-1…` in their current order. Keep the wording. Only fix a criterion that cannot be observed from outside, and say that you did. If the issue has no AC, derive them from the issue text only. Never add scope.
   - RISK-n [high|medium|low]: how this change could ship broken or be abused. Wrong permission, race, partial failure, data loss, and especially regressions of the existing behavior you found in step 2 (name the caller/behavior).
   - EDGE-n: exact boundaries (empty/one/max, == at a limit, duplicates, unicode, timezone, concurrent, retry).
   - INV-n: rules that must hold for every input or sequence.
   - Interface: the public names tests may rely on, marked (existing) or (new).
   - Mark anything not automatable `(manual: <reason>)`.
   Keep it proportional. A small task gets 2–4 items in total.
4. If the issue is too ambiguous to state an AC, do not guess. Put the question under "unclear".

Return exactly one JSON object, nothing else:
{"applicable":true,
 "ac":["AC-1: …", …],
 "spec_markdown":"Risks\n- RISK-1 [high]: …\nEdge cases\n- EDGE-1: …\nInvariants\n- INV-1: …\nInterface (what tests may rely on)\n- `…` (existing)",
 "reworded_ac":[{"id":"AC-2","was":"…","now":"…","why":"…"}],
 "unclear":[{"question":"<either/or question>"}]}
```

Act on it:

- `applicable: false` → plain TDD loop (step 3). Record `Spec not applicable: <why>` for the PR body.
- `unclear` is non-empty → AC-ambiguity fork (Mode). Ask the user, then re-run the writer with the answer added to the issue.
- Otherwise **write it to the issue**. Prefix the existing AC lines with their IDs in place, and append `## Verification spec` with `spec_markdown` plus a first line `_Auto-drafted by an isolated sub-agent in /work-on-issue; not reviewed by a human._`. Use `mcp__github__issue_write(method=update, …)` and leave every other part of the body exactly as it was. Re-read the issue, which is now the spec the author works from.
- `SPEC_TESTS: on` → tell the user one line in `USER_LANGUAGE` (`Спецификации в задаче не было — составил черновик (<k> пунктов), записал в #N, продолжаю. Поправить можно в issue.`). List any `reworded_ac`, and continue to the author below.
- `SPEC_TESTS: confirm` → show the spec in plain words (the step-0.3 "ЧТО МОЖЕТ СЛОМАТЬСЯ И ЧТО ПРОВЕРИМ" shape from `/plan-issue`) plus `reworded_ac`, and wait. On edits, update the issue and re-show. On "ок", remove the "not reviewed" line and continue.

A spec drafted here is weaker than one a human signed off in `/plan-issue`. That is why it is written to the issue, where anyone can read and fix it, and why the PR body and the report say where the spec came from.

#### 3.0b. Write the spec tests

**IDs.** The traceable items are `AC-<n>`, `RISK-<n>`, `EDGE-<n>`, `INV-<n>`, as declared in the issue. An item annotated `(manual: <reason>)` is not automatable and gets no test. Every spec test carries a tag comment in the test language's comment syntax, naming the issue and one or more IDs:

```
# spec: #<N>/AC-1, #<N>/EDGE-2
```

The issue number in the tag keeps IDs from different issues apart, since every issue has its own `AC-1`.

**Isolation.** Dispatch **one** sub-agent (`subagent_type=claude`, `model=<SPEC_AUTHOR_MODEL>`, default `opus`). Give it the issue (body + all comments) and the repo. Give it **nothing** of your own: no approach, no plan, no file list, not the step-1 notification. On a resume where the branch already has source changes but no spec commit, also pass `git diff --name-only origin/<DEV_BRANCH>...HEAD` and tell it not to open those files. Note in the PR body that independence was reduced.

```
Agent(subagent_type="claude", model="<SPEC_AUTHOR_MODEL>", description="Write spec tests",
      prompt="<the prompt below>")
```

```
You are the TEST AUTHOR for issue #<N> in this repo. A different agent will implement the change after you, and it is not allowed to edit your tests. Your tests are the definition of "done". Write them from the issue, not from a guess about the implementation.

The issue (body + all comments, verbatim):
<…>

Rules:
1. Cover every ID in the issue: AC-*, RISK-*, EDGE-*, INV-*, except items marked `(manual: …)`.
   - AC-*: the observable behavior the criterion states.
   - RISK-*: a test that fails if the risk materializes. Write it as the attack or the failure, not as the happy path.
   - EDGE-*: the exact boundary: both sides of it, empty, max, duplicate, unicode, concurrent, whatever the item names.
   - INV-*: a property that must hold across many inputs or sequences. Use the project's property-based library (Hypothesis, fast-check, …) if it is already a dependency. Otherwise use a table of adversarial cases. Do not add dependencies.
2. Tag every test with a comment `spec: #<N>/<ID>[, #<N>/<ID>…]`. A test with no tag is not a spec test. Do not write one.
3. Assert the specific outcome: exact values, error types, state after the call. A test that only checks "no exception", "is not None" or "called once" is not acceptable unless that IS the requirement.
4. Test through the interface the issue names (its `Interface` list, or the modules in "Proposed solution"). Follow the existing test layout, fixtures and naming. Put spec tests in their own new test files (e.g. `test_<feature>_spec.*`), never in existing test files.
5. Do not write or edit any non-test file. Do not create stubs of the code under test.
6. Run the new tests. Each one must FAIL for the right reason:
   - acceptable: the assertion fails, or the symbol is missing BECAUSE the issue declares it as new;
   - not acceptable: syntax errors, broken fixtures, wrong import paths for code that already exists, a typo'd existing API. Fix those.
   A test that already PASSES means the behavior already exists or the test is vacuous. Decide which. If vacuous, rewrite it. If the behavior exists (common for RISK/INV that the old code already guards), keep it as a regression guard and list it under `already_green`.
7. If an ID is too vague to test, do not invent a meaning. List it under `unclear` with the question that would resolve it.
8. Commit only your test files, with the subject `test(spec): #<N> <short summary>`. Do not push.

Return exactly one JSON object as your final message, nothing else:
{"commit":"<sha>","files":["<path>",…],
 "tests":[{"id":"AC-1","test":"<file>::<name>","red_reason":"<assertion | missing new symbol `X`>"}],
 "already_green":[{"id":"RISK-2","test":"<file>::<name>","why":"<behavior already exists>"}],
 "unclear":[{"id":"EDGE-3","question":"<either/or question>"}]}
```

**Act on the result.**

- `unclear` is non-empty → that is an AC-ambiguity fork (Mode). Stop and ask the user, one either/or question per ID, with your recommendation. Once they answer, update the issue body with the answer and re-run the author for those IDs only.
- Otherwise check it yourself: the commit exists and touches only test files, and `TEST_CMD` on those files fails, and only in the tests it listed as red. Save the commit as `<SPEC_SHA>` and the file list as `<SPEC_FILES>`. From here on these files are **frozen** (steps 3 and 3.6).
- Tell the user one line: `Спецификационные тесты: <k> тестов на <m> требований, заморожены (<SPEC_SHA short>).` (in `USER_LANGUAGE`).

### 3. TDD loop

**With spec tests (step 3.0 ran):** the Red step is already done for every traced ID. Your job is to make the spec tests green.

1. **Green.** Implement until the spec tests pass. You may add **your own** tests for internals, in **other** files. Those are normal tests, not spec tests: no `spec:` tag.
2. **Refactor.** `TEST_CMD` green again.
3. **Never edit, delete, skip, or move a file in `<SPEC_FILES>`.** If a spec test seems wrong (it contradicts another spec test, the issue, or an existing behavior the issue does not change), stop and ask the user, as the Mode section says. Quote the test and the ID, and give the either/or choice: fix the code, or change the spec. If the spec changes, the user updates the issue, and the **author** re-runs for that ID. You never rewrite the test.

**Without spec tests:** for each AC:

1. **Red.** Failing test reflecting the criterion. `TEST_CMD` fails for the expected reason.
2. **Green.** Minimal implementation. `TEST_CMD` green.
3. **Refactor.** Cleanup without changing behavior. `TEST_CMD` green again.

After all AC — a full run of every gate configured in `.claude/flow.config.md`:

```bash
<TEST_CMD>
<LINT_CMD>
<FORMAT_CMD>
<TYPECHECK_CMD>
# plus the FE_* gates if the change touches the frontend
```

All configured gates must be green before PR. If something is red — try to fix (up to 2–3 iterations). If you cannot — stop and ask.

### 3.5. E2E check

Skip this step entirely if `E2E_PATH` is `none`. Otherwise, after green unit-TDD, assess e2e applicability.

**Applicable** if the task changes:

- public API contract (new/changed endpoint, serializer change, URL routing change);
- request → view → storage → response chain (new models, new permissions/authentication, new middleware);
- frontend flow visible to the user (new screen, change to an existing one, change to API interaction).

**Not applicable** (note in the PR body as one line "E2E not applicable: <reason>"):

- internal refactor without user-facing effect;
- config/types/docs-only changes;
- migrations with no new user-visible or API surface.

**If applicable:** the test must exercise the real stack end to end — real routing, real storage, no mocks of the layer under test — live under `E2E_PATH`, and be green before the PR. Run the e2e suite, then the full `TEST_CMD` again to be safe. Commit as a separate `test:` commit (or together with the `feat:` commit — your call, but e2e must be present in the PR diff).

### 3.6. Gate on existing test changes — deny-list, then judge

Tests are guardrails. **Adding** new tests never needs approval. **Editing or deleting an existing** test does, because it can silently remove coverage — but requiring a human for every such edit turns the gate into a rubber stamp at any real throughput. So the gate is two layers: a cheap deterministic **deny-list** that always escalates, and an isolated **judge sub-agent** for everything else.

Compute the test diff first (adapt the pathspecs to this project's test layout):

```bash
git diff <DEV_BRANCH>... --stat -- '**/test_*.py' '**/*_test.go' '**/*.test.*' '**/tests/**' '**/__tests__/**' '**/conftest.py'
git diff <DEV_BRANCH>...      -- '**/test_*.py' '**/*_test.go' '**/*.test.*' '**/tests/**' '**/__tests__/**' '**/conftest.py'
```

If it contains **only additions** — new test files, or new test cases in existing files — the gate does not engage, **except for the spec-lock rule below**, which is checked regardless. Spec files are additions relative to `DEV_BRANCH`, so the additions-only shortcut would miss them. Otherwise commit normally and go to step 4.

**Spec-lock check** (only when step 3.0 ran):

```bash
git log --format='%h %s' <SPEC_SHA>..HEAD -- <SPEC_FILES> | grep -v ' test(spec): ' || true
```

Any output means a commit other than the spec author's touched a frozen spec file. That is a deny-list hit. The same applies to uncommitted changes to those files (`git status --porcelain -- <SPEC_FILES>`).

#### 3.6a. Deny-list — always escalates to the user

These are cheap to detect and too expensive to get wrong, so they **never** depend on a model's judgment and are **not** relaxed by `-bypass`:

- an **entire test file deleted**;
- `skip` / `xfail` / `.skip` / `.only` / `t.Skip` added to a test that was previously passing;
- a changed test file that lies **outside the modules this issue's source diff touches** — including shared fixtures, `conftest.py`, and test helpers. A test being edited far from the code being changed is the classic silent-coverage-loss shape;
- any changed test matching `TEST_PROTECTED_PATHS` from `.claude/flow.config.md` (auth, permissions, billing, migrations — whatever this project declares untouchable);
- the **spec-lock** check above produced output: a frozen spec test was changed by anyone other than the spec author. The only fix is to revert that change. If the test really is wrong, the spec has to change on the issue, which is a user decision.

Any hit → force that group's risk to `high` and go straight to the user report in 3.6c, naming **which** deny-list rule fired.

#### 3.6b. The judge sub-agent

For every remaining changed group, dispatch **one isolated sub-agent** (`subagent_type=claude`, `model=<TEST_JUDGE_MODEL>`, default `opus`). If `TEST_JUDGE` is not `on`, skip the judge and treat the gate as `-ask`.

The judge must be **blind to the implementer's reasoning** — pass it facts only. Told why the change is fine, it will agree; the whole point of a separate agent is that it derives necessity itself. Give it the issue, the test diff, and the **source** diff — the source diff is what separates "the assertion was adapted to a new signature" from "the assertion got weaker":

```
Agent(
  subagent_type="claude",
  model="<TEST_JUDGE_MODEL>",
  description="Judge test changes",
  prompt="<the prompt below>"
)
```

```
You are an independent reviewer of TEST changes on a feature branch. You are read-only: do not edit, commit, or push. Running the test suite to check a claim is allowed.

You are NOT told why the author made these changes, and you must not ask. Derive from the evidence alone whether each change is a necessary consequence of the task, or an erosion of a guardrail.

Evidence:
1. The issue (the task's contract):
<issue body + all comments, verbatim>
2. The diff of TEST files:
<test diff, verbatim>
3. The diff of SOURCE files (non-test) on the same branch:
<source diff, verbatim>

Group the test changes the way a reader would — by file or by feature, not per micro-assert. For each group, judge three axes:

A. ac_fit — does this test change follow necessarily from the issue's acceptance criteria?
   in-scope        the AC require exactly this change
   out-of-scope    the change is not required by any AC (gold-plating or drift)
   contradicts-ac  it encodes behavior the issue does not ask for, or contradicts a decision recorded in the issue

B. coverage — what happens to the guarantee the test used to give?
   preserved  the same property is still asserted, mechanically adapted (rename, signature, moved import)
   narrowed   a case, branch, or assertion was dropped; a matcher got looser
   removed    the property is no longer checked anywhere

C. blast_radius — does the touched test guard behavior beyond this issue?
   this-feature-only   it only covers what this issue changes
   touches-adjacent    it is also the guardrail for behavior this issue does not touch

Additional rule — behavior-preserving source changes. If the source diff is a pure refactor (rename / move / extract, no semantic change) but a test's assertions change MEANING, that is at least `medium` regardless of the axes above: a refactor by definition should not need different expectations.

Risk = the worst of the three axes. `preserved` + `in-scope` + `this-feature-only` is `low`. Anything `removed`, `contradicts-ac`, or `touches-adjacent` is at least `medium`. **When you are unsure, return `medium`, never `low`.** Being wrong toward escalation costs a question; being wrong toward acceptance costs a silent hole in the suite.

Return exactly one JSON object as your final message, nothing else:

{"verdict":"low|medium|high",
 "groups":[
   {"name":"<plain-language group name, in <USER_LANGUAGE>>",
    "files":["<path>", ...],
    "ac_fit":"in-scope|out-of-scope|contradicts-ac",
    "coverage":"preserved|narrowed|removed",
    "blast_radius":"this-feature-only|touches-adjacent",
    "risk":"low|medium|high",
    "was":"<what the test checked before — one line, in <USER_LANGUAGE>, no jargon>",
    "now":"<what it checks instead — one line, in <USER_LANGUAGE>, no jargon>",
    "why":"<half a line: why this risk level; on medium/high, what would de-risk it>"}
 ]}

"verdict" is the worst risk across all groups.
```

#### 3.6c. Act on the verdict

| Situation | What happens |
|---|---|
| deny-list hit | **always** report and wait, even under `-bypass` |
| `-ask` flag | report and wait; the judge is not consulted |
| verdict `low` | **auto-accept**: commit without waiting, emit the cards as a non-blocking note prefixed `Изменения тестов приняты автоматически (судья: риск низкий):` |
| verdict `medium`/`high`, no `-bypass` | report and **wait** for explicit confirmation before committing |
| verdict `medium`/`high`, `-bypass` | auto-accept, emit the cards prefixed `Изменения тестов приняты автоматически (флаг -bypass, судья: <verdict>):` |

Auto-acceptance never applies when a test change implies the **AC itself is wrong** — that is an AC ambiguity fork (Mode), not a test-gate decision, and it escalates regardless of `-bypass`. Every accepted change, auto or confirmed, is still listed in the PR body's "Test changes" section (step 5) and in the `/report` comment, so nothing lands unrecorded.

The report (and the non-blocking note) is a **user-facing chat message**, not a repo artifact, so write it in the **user's language** (currently Russian) and for a **non-engineer reader**:

- **No jargon.** Forbidden words: "guardrail", "blob", "re-pin", "alias", "back-compat", "contract", "mirror". Say what they mean in plain words ("проверка формата", "версия резюме", "совместимость со старым импортом").
- **One card per changed group** — render the judge's `name` / `was` / `now` / `why` verbatim; do not re-summarize them.
- **Risk in plain words** — `Низко` / `Средне` / `Высоко`. On `Средне`/`Высоко`, name what you'll do to de-risk.
- **Open questions go last, separately** — only the decisions that actually block you, each as a simple either/or **with your recommendation**. Do not interleave them into the cards.

Template (fill in, keep this shape and language):

```markdown
Меняю N групп тестов. Судья: <вердикт>. По каждой — что проверяла, что меняю, опасно ли.

ТЕСТ 1 — <name из вердикта судьи>
 Было: <was>
 Меняю: <now>
 Опасно? <Низко/Средне/Высоко> — <why>

ТЕСТ 2 — …
 …

—————
Нужны твои решения, без них не начну:
1) <вопрос как выбор А/Б> → Совет: <вариант и в полстроки почему>
2) …

Подтверди изменения тестов — или скажи, что поправить.
```

On the **waiting** path: after "yes" — commit test changes as a **separate** `test:` commit (or several, if logically distinct), so guardrail edits read at a glance in `git log`. On an **auto-accepted** path: commit the same way **without** waiting, right after emitting the non-blocking note.

If the user refuses (waiting path only) — reconsider the approach: maybe the new behavior should coexist with the old, or the AC is formulated incorrectly.

### 3.7. Record manual prod steps (config-gated, non-blocking)

**Skip if `PROD_ACTION_LABEL` is blank in `.claude/flow.config.md`.**

Some changes cannot go live by merging code alone: someone has to set an env var, add a secret, change a hosting/infra setting, or run a one-off command. On `DEV_BRANCH` that happens while the work is fresh; by the time `/issue-flow:push-to-prod` ships it, nobody remembers. So record it **now, on the issue** — `push-to-prod` step 2.5 collects these and will not deploy until they are done.

Scan the branch's diff against `DEV_BRANCH` (`git diff origin/<DEV_BRANCH>...HEAD`) for signals. Typical ones (adapt to the stack):

- a **new** env var read — `os.environ[…]`, `os.getenv`, `env(…)`, `process.env.X`, `import.meta.env.X`, a new settings field bound to an env var;
- a new key in `.env.example` / `.env.sample`, a settings schema, or a `docker-compose*`/`Dockerfile`/deploy manifest;
- a new secret referenced in `.github/workflows/*` (`secrets.X`, `vars.X`);
- a new external service, webhook URL, OAuth redirect, cron/scheduled job, bucket or queue the hosting has to know about;
- a data migration or one-off script that must be run by hand after the deploy;
- anything the **issue body or comments** already state as a manual step (e.g. `/plan-issue` filled the section in).

Also count anything you yourself asked the user to do outside the code during this run ("set X in the dev environment").

**Do not flag** a variable that already has a safe default and is optional, or one that already existed before this branch.

If you found anything, merge it into the issue body's `## Prod checklist` section (create it at the end of the body if absent), keeping the two buckets:

```markdown
## Prod checklist
Before deploy:
- [ ] Set env `FOO_API_KEY` on the prod app — required by `<module>`, no default
After deploy:
- [ ] Run `manage.py backfill_foo` once on prod
```

- **Before deploy** — the new code breaks or misbehaves without it (most env vars, secrets, infra settings). When unsure, it goes here.
- **After deploy** — needs the new code to be live first (data backfills, one-off commands, re-registering a webhook to a new endpoint).
- Name the variable/secret and **where** it is set, never its value.
- Merge, do not duplicate: an item already present (checked or not) stays as it is. Never rewrite other parts of the issue body.

Then add `PROD_ACTION_LABEL` to the issue (`mcp__github__issue_write(method=update, …)` with the body and `labels=[<current labels> + PROD_ACTION_LABEL]`). If the label does not exist in the repo, create it once via `gh label create <PROD_ACTION_LABEL> --color b60205 --description "Needs a manual step in production before/after release"` and retry (the same scoped label exception `/plan-issue` uses).

This step is **not a stop point** — the work continues. Tell the user in one line what was recorded, and if an item is also needed on the `DEV_BRANCH` environment for this merge to work there, say so explicitly: `Для dev-окружения тоже нужно: <item>.` Nothing found → no message.

### 3.8. Harden the spec tests against surviving mutants (config-gated, non-blocking)

**Skip unless** step 3.0 ran, `SPEC_MUTATION` is `on`, and the matching `MUTATION_BACKEND_TOOL` / `MUTATION_FRONTEND_TOOL` for the changed source is not `none`.

A green spec suite can still let real bugs through. A **surviving mutant** in the code this branch changed is concrete evidence of that: a deliberate bug that no test catches. The fix belongs to the **test author**, not to you. You would write the test that fits your own code, which is the failure this whole design exists to avoid.

Dispatch one sub-agent (`subagent_type=claude`, `model=<SPEC_AUTHOR_MODEL>`). Again pass no reasoning of your own, only the facts:

```
You are the TEST AUTHOR for issue #<N>, hardening the spec tests you (or an earlier author) wrote. Read-only for all non-test files.

The issue (body + all comments, verbatim): <…>
The spec test files: <SPEC_FILES>

1. Run the `/issue-flow:mutation-test diff` procedure (read its command file and follow its `diff` scope), restricted to the spec tests plus the tests that cover the changed modules. Keep only survivors on lines this branch changed (`git diff origin/<DEV_BRANCH>...HEAD`).
2. For each survivor, decide:
   - killable: there is a requirement (an ID in the issue) that the mutated behavior violates. Add a test that kills it, tagged `spec: #<N>/<ID>`, in a spec file (a new one if needed);
   - equivalent: the mutation does not change observable behavior. Say why in one line;
   - spec_gap: the mutated behavior is real, but no ID in the issue requires it. Do NOT invent a requirement. Report it.
3. One round only. Re-run the mutation on the touched files once to confirm the kills.
4. A new test that FAILS against the current code has found a real bug. Keep it, do not weaken it, and list it under `failing`.
5. Commit only test files, subject `test(spec): #<N> kill surviving mutants`. Do not push.

Return exactly one JSON object, nothing else:
{"commit":"<sha or null>","score_before":"<killed>/<total>","score_after":"<killed>/<total>",
 "killed":[{"id":"RISK-1","mutant":"<file>:<line> <mutation>","test":"<file>::<name>"}],
 "equivalent":[{"mutant":"…","why":"…"}],
 "spec_gap":[{"mutant":"…","behavior":"<plain words>"}],
 "failing":[{"id":"…","test":"…","what_breaks":"<plain words>"}]}
```

Act on it:

- Add the commit's new files to `<SPEC_FILES>`. They are frozen too.
- `failing` is non-empty → the **implementation** is wrong. Fix the code (never the test) until it passes, re-run the gates, and record it in the PR body.
- `spec_gap` → do not block. List the items in the PR body's Traceability section. Either the code does something nobody asked for, or the spec is missing a requirement. The user decides later.
- A tool error or timeout → one line in the PR body (`Mutation hardening skipped: <reason>`) and continue. This step never blocks the flow.

### 3.9. Traceability check (deterministic, blocking)

**Skip unless** step 3.0 ran. This check uses no model judgment. It is plain text matching, so it cannot be argued with.

Write the issue body to a temp file outside the repo, then:

```bash
N=<N>
# IDs the issue declares, minus the manual ones
grep -E '\b(AC|RISK|EDGE|INV)-[0-9]+' issue.md | grep -v '(manual:' \
  | grep -oE '\b(AC|RISK|EDGE|INV)-[0-9]+' | sort -u > declared.txt
# IDs the tests claim
git grep -hoE "spec: .*" -- <test pathspecs from step 3.6> \
  | grep -oE "#$N/(AC|RISK|EDGE|INV)-[0-9]+" | sed "s|#$N/||" | sort -u > tagged.txt
comm -23 declared.txt tagged.txt   # uncovered: declared but no test
comm -13 declared.txt tagged.txt   # orphans: a tag for an ID the issue does not declare
```

- **Uncovered** IDs → re-dispatch the step-3.0 author for exactly those IDs (with the same isolation, but it may now read the code). Then repeat this check. If an ID still has no test, stop and ask.
- **Orphans** → a test claims a requirement that does not exist. Ask the author to re-tag it or remove the tag. Do not edit it yourself.
- Both empty → build the matrix for the PR body (step 5). For each ID: its tests (`git grep -n "#$N/<ID>"`) and their status from the last `TEST_CMD` run. Every tagged test must be green, and manual IDs are listed as `manual: <reason>`.

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
  base=<DEV_BRANCH>,
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
- Existing tests modified/deleted (gate per step 3.6 — state the judge's verdict and how it was accepted):
  - `path/to/test_file.py::test_name` — deleted / rewritten / skipped. Judge: low|medium|high (<ac_fit>, <coverage>, <blast_radius>). Accepted: automatically / confirmed by user / `-bypass`. Reason: <…>
  - …

## Traceability
<only if step 3.0 ran; omit otherwise>
Spec source: from the issue (`/plan-issue`, human sign-off) | auto-drafted at step 3.0a, not reviewed by a human | auto-drafted and approved by the user (`confirm`).
Spec tests by an isolated author: `<SPEC_SHA short>` (+ hardening `<sha>` if step 3.8 committed). Frozen: not edited by the implementer.

| ID | Requirement / risk (short) | Tests | Status |
|---|---|---|---|
| AC-1 | <…> | `path::test_name` | ✅ |
| RISK-1 | <…> | `path::test_name`, `path::test_other` | ✅ |
| EDGE-2 | <…> | — | manual: <reason> |

Mutation (changed lines): <score_before> → <score_after>; equivalent: <n>. <or "Mutation hardening skipped: <reason>" / "not configured">
Spec gaps (behavior no requirement covers): <list from step 3.8, or "none">

## Prod steps
<only if step 3.7 recorded anything; omit otherwise>
Manual steps are tracked in the issue's `## Prod checklist`; `/push-to-prod` gates the release on them.
- Before deploy: <item>
- After deploy: <item>

## Checklist
- [x] TDD: red → green → refactor completed
- [x] Spec tests written by an isolated author, untouched by the implementer, every spec ID traced OR no spec on this issue
- [x] All configured gates green (tests, lint, format, typecheck, e2e)
- [x] E2E test added/updated OR noted "E2E not applicable: <reason>"
- [x] Existing-test changes judged and recorded OR existing tests not touched
- [ ] docs/README updated if needed
```

`Closes #$ARGUMENTS` is **mandatory** in the body. It is what makes GitHub close the issue when the PR merges, which in turn fires the built-in *Item closed → Done* project workflow. Without it the issue stays open and its card never leaves **In Progress**, and `/issue-flow:push-to-prod` cannot tell which issues a release contains.

Save the PR number (`<PR>`) for the next steps.

### 5.5 + 5.6. Reviews — two isolated sub-agents, dispatched in parallel

Code review and security review are independent read-only passes over the same diff, and both produce a lot of output that must not land in this flow's context. Run **both as sub-agents, in a single message, so they execute concurrently** — that also keeps the pre-CI phase short, which matters for step 6.

First make `origin/HEAD` resolve. The security review diffs against `origin/HEAD...`, and in fresh/cloud containers that ref is often absent (clone doesn't set it), which makes it fail with `fatal: ambiguous argument 'origin/HEAD...'`. Point it at the integration branch **before** dispatching, so both sub-agents see exactly this branch's changes rather than the whole `develop..main` delta:

```bash
git remote set-head origin <DEV_BRANCH> 2>/dev/null || git remote set-head origin --auto 2>/dev/null || true
```

Then dispatch. Skip either sub-agent whose skill is `none` in `.claude/flow.config.md` (`CODE_REVIEW_SKILL` / `SECURITY_REVIEW`); if `CODE_REVIEW_SKILL` is `none`, do a plain self-review of the diff inline instead.

**On a resume** (step 2 routed you here with a code-review comment already on the PR): dispatch **only** the security sub-agent. The code review already ran and its one round of fixes is on the branch — re-running it would be a second round, which this step forbids. The security review always re-runs, because it leaves no artifact to detect and a stale pass is not a pass.

```
Agent(subagent_type="claude", model="sonnet", description="Code review PR",
      prompt="<code-review prompt below>")
Agent(subagent_type="claude", model="opus", description="Security review branch",
      prompt="<security prompt below>")
```

**Each sub-agent reconciles its own findings against the issue and returns only the conclusion.** This is what makes the isolation worth anything: if the parent had to read the raw findings in order to filter them, the review output would land in its context anyway.

Code-review sub-agent prompt:

```
Run the `<CODE_REVIEW_SKILL>` skill on PR #<PR> in the current repo. It runs a multi-agent review and posts its result as a PR comment via `gh` (a deliberate, scoped exception to this repo's "only mcp__github__*" rule).

Then reconcile EVERY finding against the issue — the plan is the source of truth. Read it yourself:
  mcp__github__issue_read(method=get, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS)
  mcp__github__issue_read(method=get_comments, owner=<OWNER>, repo=<REPO>, issue_number=$ARGUMENTS)

A finding is `adequate` when it is a real, in-scope defect or a violation of CLAUDE.md / the issue's AC.
A finding is `rejected` when it: asks for behavior not in the AC (that is follow-up); contradicts an explicit decision in the issue body/comments; is pre-existing on lines this PR did not touch; or is a stylistic nitpick a linter/typechecker/CI already covers.

Do not fix anything. Return exactly one JSON object as your final message, nothing else:
{"review":"code","findings":[{"file":"<path>","line":<n>,"severity":"high|medium|low","summary":"<one sentence>","fix":"<one-line suggested fix>","in_scope":"adequate|rejected","why":"<half a line, required when rejected>"}],"rejected_count":<n>}
```

Security sub-agent prompt:

```
Run the `<SECURITY_REVIEW>` review on the pending changes of the current branch in this repo. Read-only: do not edit or commit.

Return exactly one JSON object as your final message, nothing else:
{"review":"security","verdict":"clean|findings","findings":[{"file":"<path>","line":<n>,"severity":"critical|high|medium|low","risk":"<what an attacker gains — one sentence, plain words>","fix":"<one-line suggested fix>"}]}

`verdict` is "clean" only when there is no finding at all.
```

**Act on the two results:**

1. **Security first — it is a hard gate.**
   - `verdict: clean` → continue **immediately** into the code-review fixes and then step 6, in the **same** turn. Do **not** pause, and do **not** post a standalone "security review passed" message and then end your turn — that counts as a wrongful stop. At most note it in one line on the way.
   - `verdict: findings` → **stop. Do not merge, do not arm auto-merge, do not run `/report`.** Escalate to the user in chat and end the autonomous flow there. The escalation is a **user-facing chat message** in the user's language (currently Russian), plain words, no jargon: for each finding — what the risk is, where, and a one-line suggested fix. Make clear the PR is open but **deliberately not merged**, and that resuming means re-running `/issue-flow:work-on-issue $ARGUMENTS`. Leave the branch and PR as-is — do not close them. A security finding is a **real fork**: the agent does not silently auto-fix security issues and then merge.

2. **Code review — exactly one round of fixes.** Apply fixes for the `adequate` findings only. Keep the TDD discipline: if a fix changes behavior, add or adjust a test first (the step-3.6 gate on **existing**-test edits still applies). Re-run the relevant local gates. Commit as a focused `fix:`/`refactor:` commit and `git push` to the same branch — the push re-triggers CI, which step 6 waits on. If there were no findings, or all were rejected, note that in one line and proceed; no commit needed. **Do not re-run the review after applying the fixes.**

**Never poll for a review result.** Both sub-agents return their verdict **synchronously** — the moment the Agent call returns, you have it. Do **not** spawn a background `Bash`/`sleep` loop, and do **not** create a "wait until the agents post their result" task: there is nothing to wait for, and a background wait **dies when the container is reclaimed**, leaving the flow hanging forever.

### 6. Green CI, then merge

The CI check to wait on is named `CI_CHECK_NAME` (from `.claude/flow.config.md`; default `CI`), triggered on `pull_request`. A PR is **never** merged before it passes — even if everything is green locally.

Which path you take depends on `AUTO_MERGE` in `.claude/flow.config.md`.

#### 6a. `AUTO_MERGE: true` — hand the merge to GitHub (recommended)

Sitting in a poll loop is what makes this flow fragile: in cloud/web sessions the turn goes dormant during the wait, and the merge then never happens until a human pokes the session. GitHub can do the merge itself the moment the required checks pass, with no agent alive:

```bash
gh pr merge <PR> --auto --squash
```

(A `gh` call — like the code review and the board mutation, a deliberate, scoped exception to the "only `mcp__github__*`" rule, because the MCP merge tool cannot arm auto-merge.)

> ⚠️ **Precondition, and it is not optional.** `--auto` only defers the merge if the base branch has **required status checks** configured in branch protection. With none configured, GitHub merges **immediately** — silently bypassing the CI gate. So right after arming it, read the PR once: if it is **already merged** while `CI_CHECK_NAME` is still pending or queued, that misconfiguration just fired. Say so plainly to the user — `auto-merge merged the PR before CI ran — <DEV_BRANCH> has no required status checks in branch protection; fix that before the next run` — and carry on to step 7. The merge cannot be undone, but the user must know the gate is not real.

With auto-merge armed, poll `get_status` inline at ~30–45 second intervals for **at most ~10 minutes**, then take whichever branch applies:

- **Merged within the window** → continue to step 7 in the same turn. This is the common case; nothing is lost.
- **Still pending after ~10 minutes** → **end the turn cleanly.** Tell the user the PR is armed and will merge itself when CI goes green, and that re-running `/issue-flow:work-on-issue $ARGUMENTS` at any later point picks up at the report (step 2's resume table routes there). Do **not** keep polling, and do **not** offload the wait to a background task.
- **A check fails** → auto-merge stays armed but will not fire. Read the failing job's logs and fix locally (up to 2–3 iterations: commit, push, re-poll). If you cannot — stop and ask the user.

#### 6b. `AUTO_MERGE: false` — poll, then merge via MCP

Poll status via MCP at ~30–45 second intervals (no faster) until all checks complete. Run this poll **inline** — repeated MCP calls within your own turn. **Never** offload the wait to a background `Bash`/`sleep` task: such waits die on container reclaim and leave the flow hanging.

```
mcp__github__pull_request_read(method=get_status, owner=<OWNER>, repo=<REPO>, pullNumber=<PR>)
```

- **All checks success** → merge now:

  ```
  mcp__github__merge_pull_request(owner=<OWNER>, repo=<REPO>, pullNumber=<PR>, merge_method=squash)
  ```

- **At least one failure** → read the failing job's logs, fix locally (up to 2–3 iterations: commit, push, wait for CI again). If you cannot — stop and ask the user.
- **Stuck > 15 minutes in pending/queued** → stop and notify the user. Re-running the command later resumes here.

### 7. After the merge

If the merge failed due to a conflict with `develop` — stop and ask the user (this is a product fork).

Sync `develop` locally:

```bash
git checkout <DEV_BRANCH>
git pull --ff-only origin <DEV_BRANCH>
```

Do **not** delete the local feature branch automatically — leave it to the user.

The merge closes the issue via `Closes #N`, and GitHub's built-in *Item closed → Done* workflow moves the card. Nothing to do here. Which issues are in `DEV_BRANCH` but not yet released is answered by the open `develop → main` release PR, not by a board column — `/issue-flow:push-to-prod` builds that list.

### 8. Auto-report

Immediately after merge, execute the `/issue-flow:report $ARGUMENTS` steps without pauses and without preview-confirm: gather facts from git/tests, map to issue AC, publish **one** comment via `mcp__github__add_issue_comment`. Details in the `/issue-flow:report` command.

This is also the step a **resumed** run lands on when the PR merged while the session was away.

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
- briefly (1–2 lines): what is closed, what remains (if anything — that is follow-up, not the current task);
- if the issue has a `## Prod checklist`: one line — `Ручные шаги для прода записаны в #N — /push-to-prod напомнит.` plus anything still needed on the `DEV_BRANCH` environment.

If the run instead ended at step 6a's hand-off (PR armed for auto-merge, CI still pending), the wrap-up is that one status line plus the note that re-running the command finishes the report — nothing else.

## Forbidden

- Never `git push --force` to `develop`/`main`.
- Never `--no-verify`, `--no-gpg-sign`, or pre-commit bypass without explicit user request.
- Never `base=main` for feature/bugfix branches.
- Never open a PR without `Closes #N` in the body — the issue would never close and the card would never reach **Done**.
- Never work an issue closed as `not_planned` without the user reopening it first.
- Never merge a PR before green CI — including via `--auto` on a branch with no required status checks (see the warning in step 6a).
- Never merge a PR, or arm auto-merge, while the security review (step 5.6) has an unresolved finding — escalate to the user and stop.
- Never let the step-3.6 deny-list be relaxed by `-bypass`.
- Never write a secret **value** into the issue, the PR, or a commit — the prod checklist names the variable and where to set it, nothing more.
- Never pass the implementer's reasoning to the test judge — it must judge from the issue and the diffs alone.
- Never pass your approach, plan, or file list to the spec writer or the spec-test author (steps 3.0a, 3.0b, 3.8, 3.9). They work from the issue alone.
- Never write or edit the verification spec yourself. If it is missing, the 3.0a writer drafts it. If it is wrong, the user fixes it on the issue.
- Never edit, delete, skip, or move a frozen spec-test file, and never author a `test(spec):` commit yourself. That subject is reserved for the spec author, and the spec-lock check trusts it.
- Do not create files in `.claude/plans/`, `notes/` (for an active task), or `*-plan.md` at the repo root.
- Do not use `gh` CLI for PR/issue operations — only `mcp__github__*`. The scoped exceptions are the project-board mutation (step 1.5), the code-review skill's own PR comment (step 5.5), arming auto-merge (step 6a), and creating a missing `PROD_ACTION_LABEL` (step 3.7).
