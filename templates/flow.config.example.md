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

## Deploy verification (set to `none` unless there is a deploy to poll)
- DEPLOY_VERIFY: none        <!-- e.g. `dokploy`; when not `none`, point DEPLOY_VERIFY_SKILL at a project-local command -->
- DEPLOY_VERIFY_SKILL:       <!-- e.g. /verify-deploy (a project-local .claude/commands skill, NOT shipped by this plugin) -->

## Chat
- USER_LANGUAGE: English     <!-- language for user-facing chat messages the commands emit -->
