---
description: Verify a merge actually reached the running app on Dokploy — compare the deployed git SHA against the merge commit, optionally drive the live UI, diagnose build failures (read-only)
argument-hint: <merge-sha> [dev|prod]
model: haiku
---

Confirm that the commit just merged is **live on the running app**, not merely merged on GitHub. The reliable, change-agnostic signal is **the deployed git SHA vs. the merge commit SHA** — the app's health endpoint reports the running image's SHA, so a match means the new image is actually serving traffic.

This command assumes a **Dokploy**-hosted backend whose health endpoint exposes the running git SHA. It is meant to run as an **isolated subagent**: deploy-log polling is noisy and must not pollute the parent flow's context. It returns a single compact **verdict** to the caller.

## Project config — read this first

All targets are per-project and live in **`.claude/flow.config.md`** (never hardcoded here). **Read it first** and resolve, for the selected env (`dev` = `DEV_BRANCH` env, default; `prod` = `PROD_BRANCH` env):

- `DEPLOY_<ENV>_APP_ID` — the Dokploy applicationId for this env.
- `DEPLOY_<ENV>_HEALTH_URL` — the health URL (e.g. `…/api/health/`; keep any trailing slash exactly).
- `DEPLOY_<ENV>_BASE_URL` — the app base URL (for the optional Playwright check).
- `HEALTH_SHA_FIELD` — the JSON field in the health response carrying the running SHA (default `git_sha`).
- `DOKPLOY_PROJECT_NAME` — used only to re-resolve app ids if they drift.
- `LOGS_ENDPOINT_PATH` + `LOGS_TOKEN_ENV` — optional first-party redacted-logs endpoint and the **name** of the env var holding its token (referenced by name, never expanded). `none` → skip, fall back to Dokploy logs.
- Playwright keys (`PLAYWRIGHT_ENABLED`, `PLAYWRIGHT_DIR`, `PLAYWRIGHT_CONFIG`, `PLAYWRIGHT_SPEC`, `PLAYWRIGHT_AUTH_HELPER`, `PLAYWRIGHT_BASE_URL_ENV`, `PLAYWRIGHT_ROLES`, `PLAYWRIGHT_CREDS_ENV`) — for the optional UI feature-check; if `PLAYWRIGHT_ENABLED` is not `true`, skip step 3.5.

If `.claude/flow.config.md` is missing or lacks the `DEPLOY_*` keys, stop and tell the caller the project isn't configured for deploy verification.

`$ARGUMENTS`:
- **First token** — the merge commit SHA to verify (full 40-char preferred; a 7-char short SHA also works, the comparison is prefix-based).
- **Second token (optional)** — env: `dev` (default) or `prod`.

## Hard constraints

- **Never** call the denied Dokploy env-reading tools: `mcp__dokploy__application-one`, `mcp__dokploy__application-saveEnvironment`, `mcp__dokploy__domain-byApplicationId`, and any analogous `*-one` / `*-saveEnvironment` tool. They echo runtime secrets. Everything needed comes from `deployment-*` and `application-readLogs` (build/deploy logs) — both read-only.
- **Never write to Dokploy** — no `application-redeploy`, no `settings-clean*`, no redeploys. Every failure (including `no space left on device`) is **surfaced** to the caller, never auto-fixed.
- **Never print secrets** or whole env/log dumps — quote only the diagnostic line(s). Reference any token/credential env var **by name**, never expand it.
- **Read-only HTTP** to the health URL — no auth header (public liveness).

## Steps

### 1. Find the deployment for the merge SHA

```
mcp__dokploy__deployment-all(applicationId=<DEPLOY_<ENV>_APP_ID>)
```

Each record carries `description: "Commit: <full-sha>"`, a `status` (`running` | `done` | `error`), `finishedAt`, `errorMessage`, `logPath`. Match the record whose `Commit:` value **starts with** the merge SHA.

- **No matching deployment yet** → Dokploy may not have picked up the push. Wait ~30s and re-list, up to ~5 min. If still absent → verdict `no-deploy` (push did not trigger a build — likely webhook/branch-filter; surface to caller).

If the app id ever drifts, re-resolve it via `mcp__dokploy__project-all` (project `DOKPLOY_PROJECT_NAME` → the env's application). `project-all` returns only names/ids — **no env** — so it is safe. Do **not** re-resolve via `domain-byApplicationId` / `application-one` (they embed the `env` block — secrets).

### 2. Poll to a terminal state

Re-list `deployment-all` at **~30–45s intervals** (no faster) until the matched record's `status` leaves `running`:

- `done` → step 3.
- `error` → step 4.
- **Stuck `running` > 15 min** → verdict `timeout` with `deploymentId` and `logPath`.

### 3. Confirm the deployed SHA

"Build succeeded" ≠ "new image is serving". Give the container a moment to swap, then GET `DEPLOY_<ENV>_HEALTH_URL` (`-L` follows a trailing-slash redirect, `-f` still fails on a real 4xx/5xx):

```bash
curl -fsSL <DEPLOY_<ENV>_HEALTH_URL>
```

Parse the JSON `<HEALTH_SHA_FIELD>`.

- field **starts with** the merge SHA (or vice-versa for short SHAs) → **verdict `deployed`**. Proceed to the optional Playwright check.
- field == `"unknown"` (or absent) → the running image has no SHA baked in (the `GIT_SHA`/`SOURCE_COMMIT` build-arg wiring did not take effect). Verdict `sha-unknown` (deploy succeeded but unverifiable — surface; do not loop).
- field is a **different** real SHA → running image is stale or a newer deploy raced. Re-poll `deployment-all` once: if a newer deployment for this SHA is `running`, resume step 2; else verdict `sha-mismatch` with both SHAs.
- `curl` fails (non-2xx / refused) → app not answering. Re-check after ~30s up to ~3 min (boot / migrations). If still failing:
  - **HTTP 5xx** (app up but erroring) → read redacted runtime logs (**step 3.6**), return verdict `health-unreachable` with the status **and** the log excerpt.
  - **refused / non-5xx** (never came up) → verdict `health-unreachable` with the status (no runtime logs to read).

Only on **verdict `deployed`** proceed to step 3.5; otherwise return the SHA verdict (a UI check against an unconfirmed image is meaningless).

### 3.5. Optional Playwright feature-check (UI-visible AC)

Skip entirely unless `PLAYWRIGHT_ENABLED: true`. The SHA check proves code deployed, not that a user-visible feature renders. For issues whose AC is **UI-visible**, drive the live app and assert the feature; for **backend-only** AC, skip and note why.

- **UI-visible (run):** AC describes something a user sees/does — a screen, button/badge/field, visibility/role rule, navigation change.
- **Backend-only (skip):** API contract with no UI surface, migration, serializer/validation, background task, server-only permission → record `playwright: skipped — backend-only AC (<reason>)`.
- **Mixed:** run for the UI-visible AC; note the backend-only ones as not covered.

Translate the chosen AC into a concrete assertion, parameterized by the **role** it targets:

1. **Role.** Pick from `PLAYWRIGHT_ROLES`; pre-created roles' credentials are in the env vars named by `PLAYWRIGHT_CREDS_ENV` (referenced by name, never expanded). Use the auth helper at `PLAYWRIGHT_AUTH_HELPER`.
2. **Navigate** to the screen named in the AC.
3. **Assert** the AC's visible token (`getByRole`/`getByText`/`getByLabel`, a visibility toggle, a role-gated element's presence/absence). Prefer role/label selectors if the UI is i18n-translated.
4. Write the assertion into the spec at `PLAYWRIGHT_SPEC` (use it as the template — replace its body; do not edit the auth helper per run).

Run it against the live env:

```bash
cd <PLAYWRIGHT_DIR>
<PLAYWRIGHT_BASE_URL_ENV>=<DEPLOY_<ENV>_BASE_URL> \
  npx playwright test --config <PLAYWRIGHT_CONFIG> <PLAYWRIGHT_SPEC>
```

- **passes** → `playwright: passed — <asserted token>`.
- **fails** → `playwright: failed — <expected vs seen>` (image deployed but UI doesn't show the feature; surface, do not auto-fix; quote only the assertion's expectation, never secrets/page dumps).
- **skipped** → `playwright: skipped — backend-only AC (<reason>)` or `… — no base URL/creds`.

The overall verdict stays `deployed` — the Playwright line is an **additional** signal alongside it.

### 3.6. Read redacted runtime logs on a post-deploy 5xx

Reached **only** when the deploy is `done` but the running app returns a **5xx** (health 500 in step 3, or a 5xx during the Playwright check). The failure is in the *running code*, not the build; the next signal is runtime stdout.

**Only do this if the project exposes redacted logs** (`LOGS_ENDPOINT_PATH` ≠ `none`, and the app masks PII at the logging layer — confirm in flow.config / project docs). Prefer the first-party endpoint; it must redact at read time:

```bash
curl -s -H "X-Logs-Token: $<LOGS_TOKEN_ENV>" "<DEPLOY_<ENV>_BASE_URL><LOGS_ENDPOINT_PATH>?tail=200&level=ERROR"
```

(The token env var is referenced by name only — never print or expand it.) If unavailable, fall back to Dokploy build/runtime logs:

```
mcp__dokploy__application-readLogs(applicationId=<DEPLOY_<ENV>_APP_ID>, tail=200)
```

Scan the tail for the traceback / error region tied to the 500:

- Quote only the **relevant lines** (exception type, failing view/path, last few frames) — never the whole tail. If a line still looks email/phone/name-shaped, drop it.
- Carry a 3–6 line excerpt into the verdict's `runtime_log`.
- Diagnostic only — never auto-fixes, never changes the verdict.

If the project does **not** expose redacted logs, skip this step and set `runtime_log: n/a (no redacted log source)`.

### 4. Diagnose a failed build/deploy

On `status == "error"`, read the **build/deploy log** (PII-free) for the matched deployment:

```
mcp__dokploy__application-readLogs(applicationId=<DEPLOY_<ENV>_APP_ID>, tail=200)
```

Verdict `build-failed` with a 3–6 line quote of the error region — surface, never auto-fix:

- **`no space left on device`** (or `ENOSPC`) — the recurring heavy-image disk failure. Note it in `detail` so a human can prune the Dokploy builder/unused images/volumes; this command does **not** clean or redeploy.
- **Anything else** (migration error, dep resolution, OOM, syntax/import error) → quote the error region.

## Verdict (return to the caller)

Return exactly one compact block (no log dumps):

```
verify-deploy: <env> @ <merge-sha-short>
verdict: deployed | sha-mismatch | sha-unknown | health-unreachable | build-failed | no-deploy | timeout
deployed_sha: <running SHA from health, or n/a>
playwright: passed — <token> | failed — <expected vs seen> | skipped — <reason> | n/a (verdict != deployed or PLAYWRIGHT_ENABLED != true)
runtime_log: <scrubbed 3–6 line excerpt on a post-deploy 5xx> | n/a
detail: <one line — for failures the error line or SHA pair; for deployed, "live SHA matches merge">
```

Mapping for the caller:

- `deployed` + `playwright: passed` → success, merge is live **and** the UI feature renders.
- `deployed` + `playwright: failed` → image live but feature doesn't render — **surface**.
- `deployed` + `playwright: skipped` → success for a backend-only AC (or no UI target/creds).
- `sha-mismatch` / `sha-unknown` / `health-unreachable` / `build-failed` / `no-deploy` / `timeout` → **surface**; deploy not confirmed live. For `build-failed` + `no space left on device`, the fix is a manual Dokploy cleanup.

## Forbidden

- Never call `mcp__dokploy__*-one`, `mcp__dokploy__*-saveEnvironment`, or `domain-byApplicationId` (embed the `env` block — secrets). Re-resolve ids via `project-all` only.
- Read runtime **app** logs **only** on a post-deploy 5xx (step 3.6), and quote only the scrubbed error region.
- Never auto-fix anything other than surfacing the `no space left on device` pattern.
- Never print/expand the credential or logs-token env vars — reference by name only; the verdict reports pass/fail and the asserted token, never an email/password.
- Never run the Playwright check before the SHA verdict is `deployed`.
