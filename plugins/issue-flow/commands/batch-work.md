---
description: Batch-run /work-on-issue across multiple issues, isolating context per task via sub-agents
argument-hint: "<N1,N2,N3,...>"
model: haiku
---

Batch-execute `/issue-flow:work-on-issue` for the list of issue numbers in `$ARGUMENTS` (e.g., `12,15,18`). Each issue runs in an isolated sub-agent context so the main agent's window stays clean across the batch — the main agent only carries a one-line summary per finished task.

## Project config — read this first

This command ships in the shared `issue-flow` plugin. Read **`.claude/flow.config.md`** first; use `USER_LANGUAGE` for the chat output below (any "Russian" literal is a placeholder). Sibling commands are namespaced — the per-task driver is `/issue-flow:work-on-issue`. If `.claude/flow.config.md` is missing, stop and ask the user to create it from the plugin's `templates/flow.config.example.md`.

## Language of user-facing output

Everything this skill prints to the user — the per-issue plan, escalation prefaces, continue/abort prompts, the final batch log — is in the project's **`USER_LANGUAGE`**. This overrides the repo-wide English convention, which applies to committed artifacts (issue bodies, PR titles, branch names, commit messages, reports), not to this skill's ephemeral chat output.

Keep machine-level tokens verbatim (do not translate them): JSON field values (`status`, `branch_type`), branch names, PR/report URLs, step ids, issue numbers. Escalation `details` payloads are relayed **verbatim** in whatever language the sub-agent produced them — only the surrounding preface line you write is Russian.

## Parsing

Split `$ARGUMENTS` by `,` and strip whitespace from each token. Validate every token is a positive integer. On parse failure (empty list, non-numeric token) — stop and ask the user (in Russian).

Tasks run **strictly in order** as given. No parallelism: they share the same git working tree, and parallel branch checkouts would corrupt local state.

## Per-task loop

For each issue number `N` in order:

### Step A. Pre-flight — surface the plan to the user

Direct `/work-on-issue` produces a 3–6 line plan (task, AC, approach, affected files) at its step 1 as a visible notification. In batch mode that notification is otherwise swallowed by the sub-agent, so kick off a tiny **read-only** preflight sub-agent first to recover that visibility.

Use the `Agent` tool with `subagent_type=claude` and `model=haiku` (this preflight is read-only analysis — a cheap model is enough). Prompt (self-contained):

```
You are running a read-only pre-flight for issue #N in the current repo. Do NOT branch, edit, commit, or run tests — analysis only.

1. Resolve <OWNER>/<REPO> from `git remote get-url origin` (format https://github.com/<OWNER>/<REPO>.git).
2. Read the issue body + all comments:
   mcp__github__issue_read(method=get, owner=<OWNER>, repo=<REPO>, issue_number=N)
   mcp__github__issue_read(method=get_comments, owner=<OWNER>, repo=<REPO>, issue_number=N)
3. Skim the repo just enough to name the likely affected files (Glob/Grep/Read are fine, no edits).
4. Produce the same 3–6 line plan that step 1 of the `/issue-flow:work-on-issue` command describes: task, AC, approach, affected files. **Write the `plan` text in Russian** (it is shown to the user). From labels infer branch type (`feat` for `type:feature`, `fix` for `type:bug`).
5. Return exactly one JSON object as your final message, nothing else:

   {"status":"plan","issue":N,"branch_type":"feat|fix","plan":"<3-6 line markdown body, \\n-escaped>"}

   If the issue cannot be read or is closed/locked:
   {"status":"failed","issue":N,"reason":"<short>","details":"<short>"}
```

Parse the JSON. On `status=plan`, print to the user **in Russian**, then proceed to Step B without waiting:

```
### Issue #N — план
<plan, unescaped>
```

On `status=failed` from preflight — append a Russian batch-log line `#N — ПРОВАЛ (префлайт): <reason>` and ask the user (in Russian) whether to continue with remaining issues (default: continue). Skip Steps B and C for this N.

### Step B. Spawn the work sub-agent

Use the `Agent` tool with `subagent_type=claude` and `model=opus` (this sub-agent runs the full `/work-on-issue` coding flow — give it the strongest model, since the orchestrator itself runs cheap). The sub-agent prompt must be **self-contained** (sub-agent has no memory of this conversation). Pass it:

```
You are executing /work-on-issue for issue #N in the current repo. A read-only preflight has already shown the user the plan below — do NOT re-emit it.

Preflight plan (already visible to the user):
<plan from Step A, unescaped>
Branch type: <branch_type from Step A>

1. Follow the `/issue-flow:work-on-issue` command's steps exactly (invoke it, or follow its documented flow). Skip the user-facing notification in step 1 (the parent already showed the plan); still read the issue + comments yourself to ground the work.
2. Override its interactive behavior: **do NOT ask the user for confirmation** at any step. Instead, if any step in /work-on-issue would normally pause for a user decision (existing-test gate at step 3.6, merge conflict on develop, repeatedly red CI you cannot fix in 2–3 iterations, AC ambiguity, scope expansion, or any other "stop and ask" branch listed in its Mode section) — **stop and return** a structured escalation to the parent agent, then exit.
3. When returning, respond with **exactly one** JSON object as the final message, nothing else:

   For success:
   {"status":"done","issue":N,"pr_url":"<url>","report_url":"<url>","ac":"closed | partial: <details>"}

   For escalation:
   {"status":"escalation","issue":N,"branch":"<branch-name>","step":"<step-id, e.g., 3.6>","reason":"<short label>","details":"<verbatim payload to show the user, e.g., the test-change report from step 3.6>"}

   For failure (unrecoverable, not requiring user input):
   {"status":"failed","issue":N,"reason":"<short>","details":"<short>"}

4. Do not narrate progress to the parent. Do all work silently and end with the single JSON object.
```

### Step C. Handle the sub-agent result

Parse the JSON object the sub-agent returned.

- **`status=done`** → append one line to the in-memory batch log: `#N — <pr_url> — AC <ac>`. Continue to the next N.
- **`status=escalation`** → relay `details` to the user verbatim (in the language the sub-agent produced it), plus one Russian preface line: `Issue #N остановлен на шаге <step> в ветке <branch>: <reason>. Жду твоё решение.` Wait for the user's reply.
  After the user replies, spawn a **fresh** sub-agent (again `subagent_type=claude`, `model=opus` — it resumes the coding flow) with a continuation prompt:

  ```
  You are resuming /work-on-issue for issue #N.

  Previous sub-agent stopped at step <step> on branch <branch-name>. Reason: <reason>.
  Context payload it returned:
  <details>

  User's decision:
  <user reply, verbatim>

  Resume from step <step> applying the user's decision. Follow the same rules as a fresh `/issue-flow:work-on-issue` run, with the same "do not ask, escalate via JSON" override. Return the same JSON shapes on completion or further escalation.
  ```

  Loop step C until the sub-agent returns `done` or `failed` for this N.
- **`status=failed`** → append `#N — ПРОВАЛ: <reason>` to the batch log. Ask the user (in Russian): continue batch with remaining N's, or abort? Default: continue.

### Step D. Context hygiene between tasks

The main agent must **not** restate, summarize, or narrate the sub-agent's work after parsing its JSON. The only carry-over to the next iteration is the batch log lines. This is the whole point of the isolation — keep the main window thin.

## Wrap-up

After the last N (or user-requested abort), print the batch log (one line per issue) under a Russian heading. Then stop. No additional summary.

## Forbidden

- Do not run issues in parallel. Sequential only — shared working tree.
- Do not auto-resolve escalations. The existing-test gate (step 3.6) in particular must always reach the user — that is its entire purpose.
- Do not skip `/work-on-issue`'s green-CI wait — sub-agent must wait for green CI before merge, same as a direct invocation.
- Do not retain or echo sub-agent narration in the main window. Only the JSON object's parsed fields, and only what the batch log needs.
- Do not create local plan files. The batch log lives only in the chat output.
