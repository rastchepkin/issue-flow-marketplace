# issue-flow — a portable Claude Code plugin

The **issue-driven Claude Code workflow** from another repo, packaged as a **plugin marketplace** so
you can install it in any project and keep every project in sync with one command. The flow turns a
chat discussion into a GitHub issue, drives it to a merged PR via a TDD loop, reports the outcome on
the issue, and cuts releases — autonomously, with the **GitHub issue/PR as the single source of truth**.

## Why a plugin (and how sync works)

The commands are written **stack-agnostic**: every project-specific value (branch names, the
green-before-PR gate commands, e2e path, CI check name, deploy hooks) lives in a per-project file,
`.claude/flow.config.md`, which the commands **read at runtime**. The command bodies are therefore
identical across all projects and can be shared verbatim.

That makes synchronization trivial:

```
edit a command  →  git push to the marketplace repo  →  in each project: /plugin marketplace update issue-flow-marketplace
```

Because `plugin.json` carries **no `version` field**, every commit is its own version — projects pick
up your latest commit on the next `/plugin marketplace update`. Your per-project `.claude/flow.config.md`
is never touched by updates, so there are no conflicts.

## The flow in one picture

```
discussion ──/issue-flow:plan-issue──▶ GitHub Issue ──/issue-flow:work-on-issue N──▶ branch → TDD → PR (Closes #N)
                                                                              │
                                                                  green CI → merge to DEV_BRANCH
                                                                              │
                                                          /issue-flow:report (auto) ──▶ one issue comment
                                                                              │
                                      many issues accumulate on DEV_BRANCH ───┘
                                                                              │
                                          /issue-flow:push-to-prod ──▶ DEV_BRANCH → PROD_BRANCH PR
                                                                              │
                                                  merge-commit → project card moves to "Production"
```

## What's in the box

```
issue-flow-marketplace/                      ← repo root (the marketplace)
├── .claude-plugin/
│   └── marketplace.json                     ← marketplace manifest (lists the issue-flow plugin)
├── plugins/issue-flow/
│   ├── .claude-plugin/plugin.json           ← plugin manifest (no version → SHA-based auto-versioning)
│   └── commands/                            ← the slash commands, shared verbatim across projects
│       ├── plan-issue.md        ★ discussion → issue
│       ├── work-on-issue.md     ★ issue → branch → TDD → PR → merge → report
│       ├── report.md            ★ one final comment on an issue
│       ├── push-to-prod.md      ★ cut a DEV→PROD release
│       ├── batch-work.md        ○ run work-on-issue over many issues
│       ├── ux-explore.md        ○ UX wireframing partner → hands off to plan-issue
│       ├── verify-deploy.md     ○ Dokploy: is the merge live? (targets from flow.config)
│       └── mutation-test.md     ○ manual mutmut/Stryker over critical logic (targets from flow.config)
├── templates/                               ← copied INTO each target project (NOT part of the plugin)
│   ├── flow.config.example.md               → becomes the project's .claude/flow.config.md (the only per-project coupling)
│   ├── CLAUDE.snippet.md                    → merge into the project's CLAUDE.md
│   ├── github/                              → copy into the project's .github/ (issue templates, PR template, project-status.yml)
│   ├── github.mcp.json                      → merge the github server into the project's .mcp.json
│   └── settings.example.json                → merge into the project's .claude/settings.json
├── README.md                                ← you are here
└── APPLY.md                                 ← runbook: install the plugin + config into a target repo
```

★ core · ○ optional helper

## How to use it

1. **Publish the marketplace once.** Put the contents of this folder in a new git repo (e.g.
   `github.com/<you>/issue-flow-marketplace`) and push.
2. **In each project**, tell the agent:
   > Install the issue-flow plugin from `<marketplace-repo-url>` and set it up here — follow the project's
   > `agent-flow-kit/APPLY.md` (or fetch APPLY.md from the marketplace repo).

   The agent adds the marketplace, installs the plugin, writes `.claude/flow.config.md` for this
   project's stack, installs the GitHub infra, and verifies the wiring.
3. **To change the flow later**: edit a command in the marketplace repo, push, then run
   `/plugin marketplace update issue-flow-marketplace` in each project.

See `APPLY.md` for the exact step-by-step.

## Cost: per-command model tiering

Each command pins a `model:` in its frontmatter, so the flow runs cost-effectively regardless of your
session model — the strong (expensive) model is reserved for the work that actually benefits from it:

| Model | Commands | Why |
|---|---|---|
| **opus** | `work-on-issue` | writes/debugs code, drives the TDD loop, makes design calls — quality here = fewer failed iterations |
| **sonnet** | `plan-issue`, `ux-explore`, `push-to-prod`, `report`, `mutation-test` | structured writing, UX reasoning, procedural orchestration with guardrails |
| **haiku** | `batch-work`, `verify-deploy` | thin orchestrator / mechanical poll-and-compare — the heavy lifting lives in the sub-agents they spawn |

Sub-agents get their model from the spawn, not the command frontmatter, so those are set explicitly too:
`batch-work`'s work sub-agents run on **opus** (they execute the full `/work-on-issue` flow) while its
read-only preflight runs on **haiku**; the `verify-deploy` sub-agents dispatched by `work-on-issue` and
`push-to-prod` run on **haiku**.

To change a command's tier, edit its `model:` line (`opus` / `sonnet` / `haiku`), push, and
`/plugin marketplace update` in each project. To let a command follow your session model instead, delete
its `model:` line.
