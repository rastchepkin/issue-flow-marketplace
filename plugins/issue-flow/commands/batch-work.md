---
description: Batch-run /work-on-issue across multiple issues, isolating context per task via sub-agents
argument-hint: "<N1,N2,N3,...> [-no-merge] [-bypass | -ask] [-chain | -independent]"
model: haiku
---

Batch-execute `/issue-flow:work-on-issue` for the list of issue numbers in `$ARGUMENTS` (e.g., `12,15,18`). Each issue runs in an isolated sub-agent context so the main agent's window stays clean across the batch — the main agent only carries a one-line summary per finished task.

The batch is built so that **one issue needing a human never stalls the rest**. An issue that cannot finish on its own is *parked* (branch pushed, PR open, not merged) and the batch moves on — unless a later issue in the list depends on it, in which case parking would silently corrupt that later issue's starting point, so it escalates instead.

## Project config — read this first

This command ships in the shared `issue-flow` plugin. Read **`.claude/flow.config.md`** first; use `USER_LANGUAGE` for the chat output below (any "Russian" literal is a placeholder), and `AUTO_MERGE` for the wait behavior in Step B. Sibling commands are namespaced — the per-task driver is `/issue-flow:work-on-issue`. If `.claude/flow.config.md` is missing, stop and ask the user to create it from the plugin's `templates/flow.config.example.md`.

## Language of user-facing output

Everything this skill prints to the user — the per-issue plan, escalation prefaces, continue/abort prompts, the final batch log — is in the project's **`USER_LANGUAGE`**. This overrides the repo-wide English convention, which applies to committed artifacts (issue bodies, PR titles, branch names, commit messages, reports), not to this skill's ephemeral chat output.

Keep machine-level tokens verbatim (do not translate them): JSON field values (`status`, `branch_type`), branch names, PR/report URLs, step ids, issue numbers. Escalation `details` payloads are relayed **verbatim** in whatever language the sub-agent produced them — only the surrounding preface line you write is Russian.

## Parsing

`$ARGUMENTS` carries the issue list plus optional flags. Separate them first:

- **flags** — tokens starting with `-`. Recognized: `-no-merge`, `-bypass`, `-ask`, `-chain`, `-independent` (see Modes). `-bypass-low` is a deprecated alias for the default — accept it silently. An unknown `-token` → stop and ask.
- **issue list** — the remaining tokens. Split by `,`, strip whitespace, validate every token is a positive integer. On parse failure (empty list, non-numeric token) — stop and ask the user (in Russian).

Tasks run **strictly in order** as given. No parallelism: they share the same git working tree, and parallel branch checkouts would corrupt local state.

## Modes

- **default** — each issue is driven through `/issue-flow:work-on-issue`: branch → TDD → PR → green CI → merge → report. `-bypass` / `-ask` are forwarded verbatim to each sub-agent, changing its existing-test gate (see `/work-on-issue` Arguments). With no flag the gate runs its judge sub-agent, which auto-accepts low-risk test changes and escalates the rest.
- **`-no-merge` (overnight review mode)** — built for an unattended night run you review in the morning. Every issue stops at **PR open + green CI and is NOT merged**: no post-merge report, no deploy-verify, no auto-merge armed. So the batch never blocks waiting for you, `-no-merge` **implies `-bypass`**: existing-test changes are applied automatically and **recorded**. The morning log lists, per issue, the PR URL plus the plain-language test-change cards, so you do the test review and the merge yourself.
- **`-chain`** — treat every issue in the list as dependent on the previous one. Nothing is parked; any escalation reaches you immediately. Use it when you know the tasks build on each other.
- **`-independent`** — treat every issue as unrelated. Maximum throughput: nothing waits, everything that needs a decision parks. Use it when you know the tasks do not touch each other.
- With neither `-chain` nor `-independent`, dependence is **inferred** — see "Dependence graph" below.

## Dependence graph

Two issues are **related** when either holds:

- their preflight `touches` sets intersect (same file, or overlapping directory globs), or
- one references the other (`depends on #N`, `blocked by #N`, `#N` in the body/comments as a prerequisite), reported as `refs`.

For each issue `N` at position `i`, compute **`related_ahead(N)`**: does any issue *later in the list* relate to `N`? That single boolean drives two decisions:

```
 related_ahead(N) = true          related_ahead(N) = false
 ─────────────────────────        ─────────────────────────
 the next issue branches from     nothing downstream depends
 DEV_BRANCH and MUST contain      on N's code
 N's merged code
   │                                │
   ├ wait for the real merge        ├ arm auto-merge, do not wait
   └ escalation → ask the user      └ escalation → PARK, continue
      right now                        to the next issue
```

`-chain` forces `related_ahead` true for every issue except the last; `-independent` forces it false for all.

> ⚠ **`-no-merge` and related issues do not mix.** Under `-no-merge` nothing lands on `DEV_BRANCH`, so a later related issue necessarily branches from a `DEV_BRANCH` that is missing its predecessor's code — it will duplicate work or conflict at review time. If the computed graph contains any related pair and `-no-merge` is active, say so to the user **before** starting Step B (in Russian: which issues overlap, and that the later ones will be built without the earlier code), and offer the two clean options: drop `-no-merge` for this run, or split the batch so each `-no-merge` run holds only unrelated issues. Default if they do not answer: proceed, and repeat the warning in the final log next to the affected issues. Within `-no-merge`, `related_ahead` still governs escalate-vs-park as computed.

## Per-task loop

For each issue number `N` in order:

### Step A. Pre-flight — surface the plan and collect the dependence signal

Direct `/work-on-issue` produces a 3–6 line plan at its step 1 as a visible notification. In batch mode that notification is otherwise swallowed by the sub-agent, so kick off a tiny **read-only** preflight sub-agent first to recover that visibility — and to get the `touches`/`refs` the dependence graph needs.

Use the `Agent` tool with `subagent_type=claude` and `model=haiku` (read-only analysis — a cheap model is enough). Prompt (self-contained):

```
You are running a read-only pre-flight for issue #N in the current repo. Do NOT branch, edit, commit, or run tests — analysis only.

1. Resolve <OWNER>/<REPO> from `git remote get-url origin` (format https://github.com/<OWNER>/<REPO>.git).
2. Read the issue body + all comments:
   mcp__github__issue_read(method=get, owner=<OWNER>, repo=<REPO>, issue_number=N)
   mcp__github__issue_read(method=get_comments, owner=<OWNER>, repo=<REPO>, issue_number=N)
3. Skim the repo just enough to name the likely affected files (Glob/Grep/Read are fine, no edits).
4. Produce the same 3–6 line plan that step 1 of the `/issue-flow:work-on-issue` command describes: task, AC, approach, affected files. **Write the `plan` text in Russian** (it is shown to the user). From labels infer branch type (`feat` for `type:feature`, `fix` for `type:bug`).
5. Return exactly one JSON object as your final message, nothing else:

   {"status":"plan","issue":N,"branch_type":"feat|fix",
    "touches":["<repo-relative path or glob this issue will likely modify>", ...],
    "refs":[<issue numbers this issue's body/comments name as a prerequisite or dependency>],
    "plan":"<3-6 line markdown body, \\n-escaped>"}

   `touches` drives the batch's dependence graph: be specific enough to be useful (a directory glob like `backend/apps/orders/**` is fine, `**` is not) and inclusive rather than minimal — a missed overlap costs a merge conflict later.

   If the issue cannot be read or is closed/locked:
   {"status":"failed","issue":N,"reason":"<short>","details":"<short>"}
```

Parse the JSON. Keep `touches` and `refs` for the dependence graph. On `status=plan`, print to the user **in Russian**, then proceed to Step B without waiting:

```
### Issue #N — план
<plan, unescaped>
```

On `status=failed` from preflight — append a Russian log line `#N — ПРОВАЛ (префлайт): <reason>` and ask the user (in Russian) whether to continue with remaining issues (default: continue). Skip Steps B and C for this N.

> Preflight every issue in the list **before** starting Step B on the first one. `related_ahead` needs the whole graph, and preflight is cheap and read-only, so the up-front pass costs little and makes the wait/park decisions correct from the first task.

### Step B. Spawn the work sub-agent

Use the `Agent` tool with `subagent_type=claude` and `model=opus` (this sub-agent runs the full `/work-on-issue` coding flow — give it the strongest model, since the orchestrator itself runs cheap). The sub-agent prompt must be **self-contained** (the sub-agent has no memory of this conversation). Build it from the template below, substituting the **mode block** per the active flags and this issue's `related_ahead`.

```
You are executing /work-on-issue for issue #N in the current repo. A read-only preflight has already shown the user the plan below — do NOT re-emit it.

Preflight plan (already visible to the user):
<plan from Step A, unescaped>
Branch type: <branch_type from Step A>

1. Follow the `/issue-flow:work-on-issue` command's steps exactly (invoke it, or follow its documented flow), as if invoked `/work-on-issue N <FLAGS>` where <FLAGS> = the test-gate flags forwarded by the batch (`-bypass` / `-ask`, or none). Honor those flags at its existing-test gate (step 3.6) — including its deny-list, which `-bypass` does NOT relax. Skip the user-facing notification in step 1 (the parent already showed the plan); still read the issue + comments yourself to ground the work.
2. Override its interactive behavior: **do NOT ask the user for confirmation** at any step. Whenever a step would normally pause for a user decision that the forwarded flags do NOT auto-resolve — merge conflict on DEV_BRANCH, repeatedly red CI you cannot fix in 2–3 iterations, AC ambiguity, scope expansion, a security finding, or a test change the gate escalates — do NOT proceed past it. Instead take the <BLOCKED BEHAVIOR> below and exit.
   <BLOCKED BEHAVIOR — substitute one:>
   • related_ahead = true  → return a structured `escalation` to the parent immediately (a later task in this batch depends on your merged code, so it cannot start without a decision here).
   • related_ahead = false → PARK the task: make sure the work so far is not lost and is reviewable, then return `parked`. Concretely: commit what you have, `git push` the branch, and ensure a PR exists (open a DRAFT PR if you had not reached step 5 yet) whose body starts with `Closes #N` and states in one line why it is parked. Do NOT merge it and do NOT arm auto-merge. Add the label `needs-decision` to the PR if the label exists in the repo; skip silently if it does not.
   <MERGE BEHAVIOR — substitute one:>
   • related_ahead = true  → drive to the very end: wait for green CI, merge to DEV_BRANCH, auto-report, deploy-verify, per /work-on-issue steps 6–8.5. The next task branches from your merge, so it must actually be on DEV_BRANCH before you return.
   • related_ahead = false, AUTO_MERGE true  → follow /work-on-issue step 6a: arm auto-merge (`gh pr merge <PR> --auto --squash`), including its required-status-checks safety read. Then do NOT poll and do NOT wait: return `armed` right away. Skip steps 7–8.5 — the parent handles the post-merge sweep, and the report is produced later by a resumed `/work-on-issue N`. Skipping step 7.5 (forwarding `Closes #N` to an open release PR) is safe: `/issue-flow:push-to-prod` re-derives that set from the merged PR bodies.
   • related_ahead = false, AUTO_MERGE false → wait for green CI and merge normally per step 6b, then steps 7–8.5. (Without auto-merge there is no way to hand the merge off, so the wait is unavoidable.)
   • -no-merge → STOP once the PR is open AND CI is green. Do NOT merge, do NOT arm auto-merge, do NOT run steps 7, 7.5, 8, or 8.5. Treat existing-test changes as `-bypass` (apply automatically, never escalate, deny-list excepted) and capture the plain-language test-change cards from step 3.6 verbatim to return.
3. When returning, respond with **exactly one** JSON object as the final message, nothing else:
   {"status":"done","issue":N,"pr_url":"<url>","report_url":"<url>","ac":"closed | partial: <details>","test_changes":"<step-3.6 cards verbatim, or 'нет изменений существующих тестов'>"}
   {"status":"armed","issue":N,"pr_url":"<url>","ac":"closed | partial: <details>","test_changes":"<as above>"}
   {"status":"done_no_merge","issue":N,"pr_url":"<url>","ac":"closed | partial: <details>","test_changes":"<as above>"}
   {"status":"parked","issue":N,"pr_url":"<url>","branch":"<branch-name>","step":"<step-id, e.g. 3.6>","reason":"<short label>","security":true|false,"details":"<verbatim payload to show the user>"}
   {"status":"escalation","issue":N,"branch":"<branch-name>","step":"<step-id>","reason":"<short label>","security":true|false,"details":"<verbatim payload to show the user>"}
   {"status":"failed","issue":N,"reason":"<short>","details":"<short>"}

   Set `"security": true` when the blocker is a security-review finding — the parent surfaces those separately.
4. Do not narrate progress to the parent. Do all work silently and end with the single JSON object.
```

### Step C. Handle the sub-agent result

Parse the JSON object the sub-agent returned.

- **`status=done`** → batch log, section *Смержено*: `#N — <pr_url> — AC <ac>`. Continue.
- **`status=armed`** → remember `<pr_url>` for the Step E sweep; provisional log line `#N — <pr_url> — auto-merge включён`. Continue immediately, without waiting.
- **`status=done_no_merge`** → batch log, section *Ждёт решения*: header `#N — <pr_url> — НЕ смёржен, ждёт ревью — AC <ac>`, with the `test_changes` cards indented underneath. Continue.
- **`status=parked`** → batch log, section *Ждёт решения* (or its *Безопасность* subsection when `security` is true): header `#N — <pr_url> — припаркован на шаге <step>: <reason>`, with `details` relayed **verbatim** underneath. Print one Russian line to the user now so the parking is visible as it happens, then **continue to the next issue without waiting**.
- **`status=escalation`** → relay `details` to the user verbatim, plus one Russian preface line: `Issue #N остановлен на шаге <step> в ветке <branch>: <reason>. От него зависят следующие задачи в списке, поэтому жду твоё решение.` Wait for the user's reply.
  After the user replies, spawn a **fresh** sub-agent (`subagent_type=claude`, `model=opus`) with a continuation prompt:

  ```
  You are resuming /work-on-issue for issue #N.

  Previous sub-agent stopped at step <step> on branch <branch-name>. Reason: <reason>.
  Context payload it returned:
  <details>

  User's decision:
  <user reply, verbatim>

  Resume from step <step> applying the user's decision. Follow the same rules as a fresh `/issue-flow:work-on-issue` run — including its step-2 resume detection — with the same "do not ask, escalate or park via JSON" override and the same MERGE BEHAVIOR block as before. Return the same JSON shapes on completion or further escalation.
  ```

  Loop Step C until this N returns `done`, `parked`, or `failed`.
- **`status=failed`** → batch log, section *Провалено*: `#N — ПРОВАЛ: <reason>`. Ask the user (in Russian): continue with the remaining N's, or abort? Default: continue.

### Step D. Context hygiene between tasks

The main agent must **not** restate, summarize, or narrate the sub-agent's work after parsing its JSON. The only carry-over to the next iteration is the batch log lines and the dependence graph. This is the whole point of the isolation — keep the main window thin.

### Step E. Post-batch sweep over the armed PRs

After the last N, every `armed` PR has had the whole rest of the batch to merge itself. Read each one once — **a single pass, not a poll loop**:

```
mcp__github__pull_request_read(method=get, owner=<OWNER>, repo=<REPO>, pullNumber=<PR>)
mcp__github__pull_request_read(method=get_status, owner=<OWNER>, repo=<REPO>, pullNumber=<PR>)
```

Classify each into the final log:

- **merged** → section *Смержено*, with the note `отчёт не опубликован — запусти /issue-flow:work-on-issue <N>` (the work sub-agent skipped step 8; a resumed run posts the report).
- **open, CI still pending** → section *Ждёт CI*: `#N — <pr_url> — смержится сам, когда CI позеленеет`.
- **open, a check failed** → section *Ждёт решения*: `#N — <pr_url> — CI красный, auto-merge не сработает: <failed check>`.

Do not fix red CI here — the batch is over and each fix belongs to its own `/work-on-issue N` run.

## Wrap-up

Print the batch log under a Russian heading, in four sections, omitting any that are empty:

```markdown
## Итоги прогона

### Смержено
…

### Ждёт CI (auto-merge включён)
…

### Ждёт решения
#### ⚠ Безопасность
…
#### Остальное
…

### Провалено
…
```

Then one closing line stating what is required of the user: which PRs need a decision, and that merged issues without a report finish with `/issue-flow:work-on-issue <N>`. Under `-no-merge`, make the heading say the PRs are open and waiting for review/merge (e.g. `## Итоги ночного прогона — PR открыты, ждут ревью`) and end with the reminder that nothing was merged: review each PR's test changes, merge the good ones, then run `/issue-flow:verify-deploy` if the project uses it. No additional summary after that.

## Forbidden

- Do not run issues in parallel. Sequential only — shared working tree.
- Do not start a task whose predecessor it depends on has not actually merged. `related_ahead` exists precisely to prevent branching from a `DEV_BRANCH` that is missing code the task needs.
- Do not park an issue that a later issue in the list depends on — escalate instead.
- Do not auto-resolve escalations **the active flags do not explicitly cover**, and never relax the test gate's deny-list — `-bypass` does not reach it.
- Do not skip `/work-on-issue`'s CI gate. A task either waits for green CI, or hands the merge to GitHub's auto-merge (which fires only on green) — never merges without one of the two.
- In `-no-merge` mode, do not merge any PR and do not arm auto-merge — leave everything open for the user's morning review.
- Do not turn Step E into a poll loop. One read per armed PR, then report what you saw.
- Do not retain or echo sub-agent narration in the main window. Only the JSON object's parsed fields, and only what the batch log needs.
- Do not create local plan files. The batch log lives only in the chat output.
