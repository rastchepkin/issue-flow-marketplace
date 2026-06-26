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

## Reviews (set each to `none` to skip that step)
- CODE_REVIEW_SKILL: code-review:code-review   <!-- or `none` → reviewer does a plain self-review of the diff -->
- SECURITY_REVIEW: /security-review            <!-- or `none` -->

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
