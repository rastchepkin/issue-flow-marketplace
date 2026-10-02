## What & why
Closes #

## Test changes
<!-- One of:
     - Only new tests added; existing tests not touched.
     - Existing tests modified/deleted — one line each:
       `path/to/test::name` — deleted/rewritten/skipped. Judge: low|medium|high. Accepted: automatically / confirmed by user / `-bypass` / deny-list override. Reason: … -->

## Prod steps
<!-- Only if this change needs a manual step outside the code to go live (env var, secret, infra
     setting, one-off script). Track the steps in the issue's `## Prod checklist` and label it
     `prod:action-required` — /push-to-prod will not deploy until the "Before deploy" items are done.
     Delete this section otherwise. Never paste secret values. -->

## Checklist
- [ ] TDD: red → green → refactor completed
- [ ] `pytest` green (including `tests/e2e/`)
- [ ] `ruff check .` clean
- [ ] `mypy --strict` clean
- [ ] E2E test added/updated OR noted "E2E not applicable: <reason>"
- [ ] Existing-test changes judged and recorded above OR existing tests not touched
- [ ] docs/README updated if needed
