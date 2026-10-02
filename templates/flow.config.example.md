<!--
  Copy this file to `.claude/flow.config.md` in the target repo and fill in real values.
  The issue-flow plugin commands READ this file at runtime — it is the single place that
  couples the (otherwise identical) commands to this project. It is NOT part of the plugin,
  so editing it never conflicts with `/plugin marketplace update`.

  Keep the `KEY: value` lines exactly as named — the commands look them up by these keys.
-->

# issue-flow project config

## Branches
- DEV_BRANCH: develop        <!-- integration branch: PRs target it, work-on-issue branches from it -->
- PROD_BRANCH: main          <!-- release target: push-to-prod merges DEV_BRANCH → here -->

## Green-before-PR gate commands
<!-- The exact commands work-on-issue / report run and the PR checklist references.
     List ONLY the ones this project has; delete the rest. -->
- TEST_CMD: uv run pytest
- LINT_CMD: uv run ruff check .
- FORMAT_CMD: uv run ruff format --check .
- TYPECHECK_CMD: uv run mypy

## Frontend gates (delete this whole block if there is no separate frontend suite)
- FE_LINT_CMD: npm run lint
- FE_TYPECHECK_CMD: npm run typecheck
- FE_TEST_CMD: npm run test
- FE_BUILD_CMD: npm run build

## E2E
- E2E_PATH: tests/e2e/       <!-- where e2e tests live; set to `none` if the project has no e2e suite -->

## CI
- CI_CHECK_NAME: CI          <!-- the PR check name work-on-issue / push-to-prod wait on before merge -->

## Merge
<!-- AUTO_MERGE: true hands the merge to GitHub (`gh pr merge --auto`), so the flow no longer needs
     the agent alive while CI runs — the session can die or go dormant and the PR still lands.
     ⚠ Set it to true ONLY after BOTH of these are configured on GitHub:
        1. Settings → General → Pull Requests → "Allow auto-merge" is ON;
        2. DEV_BRANCH (and PROD_BRANCH, for push-to-prod) have REQUIRED STATUS CHECKS in branch
           protection, including CI_CHECK_NAME.
     Without (2), `--auto` merges IMMEDIATELY and the CI gate is silently bypassed. -->
- AUTO_MERGE: false

## Project board (optional — used by /issue-flow:work-on-issue step 1.5 to move the card to "In Progress")
<!-- The board uses GitHub's DEFAULT three columns: Todo / In Progress / Done. Two of the three
     moves are GitHub's own built-in project workflows and need nothing here:
        Todo  ← "Item added to project"
        Done  ← "Item closed"  (the PR's `Closes #N` closes the issue on merge)
     "Work has started" has no native trigger, so work-on-issue step 1.5 sets In Progress itself —
     that is what these three keys are for. Discover them with the `gh api graphql` query in
     docs/GITHUB-SETUP.md. Leave any of the three BLANK to disable the move; work-on-issue then
     skips step 1.5 with a warning and proceeds.

     Deliberately absent: a Develop and a Production column. "Merged but not yet live" is the open
     develop→main release PR, and "shipped" is git history / GitHub Releases — neither needs a column
     kept in sync by a bespoke workflow. Cancelled work is not a column either: close the issue as
     "not planned" (`gh issue close N --reason "not planned"`) and let the built-in Auto-archive
     workflow take the card off the board. The issue itself is never deleted. -->
- PROJECT_ID:                <!-- e.g. PVT_... ; same value as the PROJECT_ID repo variable -->
- STATUS_FIELD_ID:           <!-- e.g. PVTSSF_... ; same value as the STATUS_FIELD_ID repo variable -->
- OPTION_IN_PROGRESS:        <!-- single-select option id of the "In Progress" column -->

## Backlog
<!-- Label meaning "needed, but not now". /plan-issue applies it (flag -backlog, or the КОГДА answer
     in its sign-off; it is the default under -auto); /work-on-issue removes it at step 1.5 when the
     issue is taken into work. Filter a board view on `-label:<BACKLOG_LABEL>` to see just the active
     queue. Leave blank to disable backlog marking entirely. -->
- BACKLOG_LABEL: status:backlog

## Manual prod steps
<!-- Label meaning "this issue needs a manual step outside the code to go live" — an env var, a secret,
     a hosting/infra setting, a one-off script. The steps themselves live in the issue body's
     `## Prod checklist` (Before deploy / After deploy checkboxes). /plan-issue and /work-on-issue
     step 3.7 record them; /push-to-prod step 2.5 collects them into the release PR and will NOT merge
     (or arm auto-merge) until every "Before deploy" item is confirmed. The label is cleared once the
     issue's checklist is fully ticked. Leave blank to disable the whole mechanism. -->
- PROD_ACTION_LABEL: prod:action-required

## Test-change judge (/issue-flow:work-on-issue step 3.6)
<!-- Editing or deleting an EXISTING test can silently remove coverage. With TEST_JUDGE: on, an
     isolated read-only sub-agent judges each change against the issue's AC, the test diff and the
     source diff, and returns low/medium/high; `low` is auto-accepted and recorded, medium/high stops
     and asks. Set it to `off` to always require a human (equivalent to running with -ask). -->
- TEST_JUDGE: on
- TEST_JUDGE_MODEL: opus     <!-- the judge replaces a human gate — do not cheap out here -->
- TEST_PROTECTED_PATHS:      <!-- space-separated globs that ALWAYS escalate, judge or not, and that
                                  -bypass cannot relax. e.g. **/test_auth*.py **/test_permissions*.py
                                  **/test_billing*.py **/migrations/**. Blank = only the built-in
                                  deny-list rules (deleted test file, new skip/xfail, tests outside
                                  the modules the issue touches). -->

## Reviews (set each to `none` to skip that step)
<!-- Both run as isolated sub-agents, dispatched in parallel, each returning only its reconciled
     findings — the raw review output never reaches the main flow's context. -->
- CODE_REVIEW_SKILL: code-review:code-review   <!-- or `none` → reviewer does a plain self-review of the diff -->
- SECURITY_REVIEW: /security-review            <!-- or `none`; a finding here is a hard stop before merge -->

## Deploy verification (set DEPLOY_VERIFY to `none` unless there is a deploy to poll)
- DEPLOY_VERIFY: none                  <!-- e.g. `dokploy` -->
- DEPLOY_VERIFY_SKILL: /issue-flow:verify-deploy   <!-- the plugin's Dokploy verifier; or a project-local /verify-deploy for another platform -->
# Targets for /issue-flow:verify-deploy (only when DEPLOY_VERIFY != none; these are per-project, NOT in the marketplace):
- DOKPLOY_PROJECT_NAME:                <!-- used only to re-resolve app ids if they drift -->
- DEPLOY_DEV_APP_ID:                   <!-- Dokploy applicationId for the DEV_BRANCH env -->
- DEPLOY_DEV_HEALTH_URL:               <!-- e.g. https://app-dev.example.com/api/health/ (keep trailing slash) -->
- DEPLOY_DEV_BASE_URL:                 <!-- e.g. https://app-dev.example.com -->
- DEPLOY_PROD_APP_ID:
- DEPLOY_PROD_HEALTH_URL:
- DEPLOY_PROD_BASE_URL:
- HEALTH_SHA_FIELD: git_sha            <!-- JSON field in the health response carrying the running SHA -->
- LOGS_ENDPOINT_PATH: none             <!-- e.g. /api/logs/ for a redacted first-party logs endpoint, else `none` -->
- LOGS_TOKEN_ENV:                      <!-- NAME of the env var holding the logs token (referenced by name, never expanded) -->

## Playwright feature-check (optional; used by /issue-flow:verify-deploy step 3.5)
- PLAYWRIGHT_ENABLED: false            <!-- true to drive the live UI for UI-visible ACs -->
- PLAYWRIGHT_DIR: frontend
- PLAYWRIGHT_CONFIG: playwright.remote.config.ts
- PLAYWRIGHT_SPEC: e2e/remote/feature-check.spec.ts
- PLAYWRIGHT_AUTH_HELPER: frontend/e2e/remote/auth.ts
- PLAYWRIGHT_BASE_URL_ENV: PLAYWRIGHT_BASE_URL
- PLAYWRIGHT_ROLES:                    <!-- e.g. superadmin, manager, hr, employee -->
- PLAYWRIGHT_CREDS_ENV:                <!-- NAME/prefix of the test-account credential env vars (referenced by name only) -->

## Mutation testing (optional, manual-only; used by /issue-flow:mutation-test)
- MUTATION_BACKEND_TOOL: none          <!-- e.g. mutmut -->
- MUTATION_BACKEND_PATHS:              <!-- e.g. backend/apps/<app>/services.py backend/apps/<app>/serializers.py -->
- MUTATION_BACKEND_RUNNER:             <!-- optional narrower runner override -->
- MUTATION_FRONTEND_TOOL: none         <!-- e.g. stryker -->
- MUTATION_FRONTEND_GLOB:              <!-- e.g. src/lib/validation/**/*.ts -->

## Chat
- USER_LANGUAGE: English     <!-- language for user-facing chat messages the commands emit -->
