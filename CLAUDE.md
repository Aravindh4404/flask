## GBrain Configuration (configured by /setup-gbrain)
- Mode: local-stdio
- Engine: pglite
- Config file: ~/.gbrain/config.json (mode 0600)
- Setup date: 2026-06-09
- MCP registered: yes (user scope)
- Artifacts sync: off (re-run /setup-gbrain after `gh auth login` to enable)
- Current repo policy: read-write

## GBrain Search Guidance (configured by /sync-gbrain)
<!-- gstack-gbrain-search-guidance:start -->

GBrain is set up and synced on this machine. The agent should prefer gbrain
over Grep when the question is semantic or when you don't know the exact
identifier yet.

**This worktree is pinned to a worktree-scoped code source** via the
`.gbrain-source` file in the repo root (kubectl-style context).
`gbrain code-def`, `code-refs`, `code-callers`, `code-callees`, `search`, and
`query` from anywhere under this worktree route to that source by default —
no `--source` flag needed (gbrain >= 0.41.38.0; on older gbrain the call-graph
commands need `--source "$(cat .gbrain-source)"`). Conductor sibling worktrees
of the same repo each have their own pin and their own indexed pages, so
semantic results match the code on disk here.

Call-graph queries (`code-callers`/`code-callees`) also need the graph to be
built first — run `/sync-gbrain --dream` (or `--full`) if they return
`count: 0`. This only works if this source's gbrain schema pack extracts code
symbols; on a non-code-aware pack `--dream` completes but the graph stays empty
and reports a WARN. `code-def`/`code-refs` need the same extraction.

Two indexed corpora available via the `gbrain` CLI:
- This worktree's code (auto-pinned via `.gbrain-source`).
- `~/.gstack/` curated memory (registered as `gstack-brain-<user>` source via
  the existing federation pipeline).

Prefer gbrain when:
- "Where is X handled?" / semantic intent, no exact string yet:
    `gbrain search "<terms>"` or `gbrain query "<question>"`
- "Where is symbol Y defined?" / symbol-based code questions:
    `gbrain code-def <symbol>` or `gbrain code-refs <symbol>`
- "What calls Y?" / "What does Y depend on?":
    `gbrain code-callers <symbol>` / `gbrain code-callees <symbol>`
- "What did we decide last time?" / past plans, retros, learnings:
    `gbrain search "<terms>" --source gstack-brain-<user>`

Grep is still right for known exact strings, regex, multiline patterns, and
file globs. Run `/sync-gbrain` after meaningful code changes; for ongoing
auto-sync across all worktrees, run `gbrain autopilot --install` once per
machine — gbrain's daemon handles incremental refresh on a schedule.

Safety: don't run `/sync-gbrain` while `gbrain autopilot` is active — the
orchestrator refuses destructive source ops when it detects a running autopilot
to avoid racing it (#1734). Prefer registering user repos with `gbrain sources
add --path <dir>` (no `--url`): URL-managed sources can auto-reclone, and the
sync code walk for them requires an explicit `--allow-reclone` opt-in.

<!-- gstack-gbrain-search-guidance:end -->

## Skill routing

When the user's request matches an available skill, invoke it via the Skill tool. When in doubt, invoke the skill.

Key routing rules:
- Product ideas/brainstorming → invoke /office-hours
- Strategy/scope → invoke /plan-ceo-review
- Architecture → invoke /plan-eng-review
- Design system/plan review → invoke /design-consultation or /plan-design-review
- Full review pipeline → invoke /autoplan
- Bugs/errors → invoke /investigate
- QA/testing site behavior → invoke /qa or /qa-only
- Code review/diff check → invoke /review
- Visual polish → invoke /design-review
- Ship/deploy/PR → invoke /ship or /land-and-deploy
- Save progress → invoke /context-save
- Resume context → invoke /context-restore
- Author a backlog-ready spec/issue → invoke /spec

# Flask Internals — Context Guide

## What this repo is
Flask is a Python WSGI web framework. This guide documents its **internal
architecture** (how Flask is built), not user-facing usage docs.

## Context system layers available
- **Generated docs** — 12 markdown files in `context/docs/` (Diataxis-structured;
  note: located under `context/docs/`, not a top-level `docs/`).
- **GBrain source** — `gstack-code-flask-7d20071a` (pinned via `.gbrain-source`).
  Indexes the Python source as AST chunks (function/class/method granularity)
  plus the imported wiki articles. Query with `--source gstack-code-flask-7d20071a`
  (or rely on the `.gbrain-source` pin on gbrain >= 0.41.38.0).
- **LLM Wiki** — 10 compiled articles at
  `C:\Users\aravi\wiki\topics\flask-internals\wiki\` (3 concepts, 4 topics,
  3 references).

## Routing table — which tool for which question

### Architecture / "how does X work overall"
Read: `wiki/concepts/request-lifecycle.md`
Or: `gbrain search "<terms>" --source gstack-code-flask-7d20071a`

### Request lifecycle / WSGI to response flow
Read: `wiki/concepts/request-lifecycle.md`

### Context system (request, g, session, current_app)
Read: `wiki/concepts/context-and-proxies.md`

### Blueprints — how they work internally
Read: `wiki/concepts/blueprint-internals.md`

### How-to questions (factory, errors, testing, blueprints)
Read: `wiki/topics/` — pick the matching article:
`app-factory-pattern.md`, `error-handling.md`, `testing-flask-apps.md`,
`organizing-with-blueprints.md`

### Config keys
Read: `wiki/references/configuration.md`

### API reference (classes, functions, decorators)
Read: `wiki/references/core-api.md`

### Module dependency graph
Read: `wiki/references/module-architecture.md`

### Where a symbol is defined
Use: `gbrain code-def <Symbol> --source gstack-code-flask-7d20071a`

### Exact code / specific implementation
Use: `gbrain search "<terms>" --source gstack-code-flask-7d20071a`

## What NOT to do
- Do not grep through `src/` for architecture questions — the wiki covers this.
- Do not read all docs — read only the specific one routed above.
- Do not load `context/docs/knowledge-graph.md` unless asked about module
  relationships.

## Key facts about Flask internals (verified against source)
- The `Flask` application class lives in `src/flask/app.py:109` and subclasses
  `App` (`src/flask/sansio/app.py:59`), which subclasses `Scaffold`
  (`src/flask/sansio/scaffold.py:52`).
- **sansio/ pattern**: framework-agnostic, I/O-free base logic
  (`sansio/app.py`, `sansio/blueprints.py`, `sansio/scaffold.py`) is separated
  from the WSGI-bound concrete classes in `app.py`/`blueprints.py`. The concrete
  `Flask`/`Blueprint` inherit the sansio base and add request/response handling.
- `Blueprint` (`src/flask/blueprints.py:18`) subclasses the sansio `Blueprint`
  (`src/flask/sansio/blueprints.py:119`); blueprints record deferred setup
  operations that run when registered on an app.
- **Request handling flow** (in `src/flask/app.py`): `wsgi_app` (line 1566) →
  `full_dispatch_request` (line 992) → `dispatch_request` (line 966). The
  full-dispatch phase fires before-request handlers, dispatches to the view,
  then finalizes/after-request handlers.
- The context globals `current_app`, `request`, `session`, and `g` are
  `LocalProxy` objects defined in `src/flask/globals.py`, backed by the
  request/app context stacks managed in `src/flask/ctx.py`.
- `has_request_context()` (`src/flask/ctx.py:209`) reports whether a request
  context is currently pushed.
- Core modules: `app.py` (the Flask class), `ctx.py` (context objects),
  `globals.py` (proxies), `blueprints.py`, `config.py`, `sessions.py`,
  `wrappers.py` (Request/Response), `helpers.py`, `views.py` (class-based views).
