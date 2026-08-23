## What & why
Closes #

## Test changes
<!-- One of:
     - Only new tests added; existing tests not touched.
     - Existing tests modified/deleted — one line each:
       `path/to/test::name` — deleted/rewritten/skipped. Judge: low|medium|high. Accepted: automatically / confirmed by user / `-bypass` / deny-list override. Reason: … -->

## Checklist
- [ ] TDD: red → green → refactor completed
- [ ] `pytest` green (including `tests/e2e/`)
- [ ] `ruff check .` clean
- [ ] `mypy --strict` clean
- [ ] E2E test added/updated OR noted "E2E not applicable: <reason>"
- [ ] Existing-test changes judged and recorded above OR existing tests not touched
- [ ] docs/README updated if needed
