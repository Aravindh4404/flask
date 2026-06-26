# Flask Context System — Routing Table

You have four context layers for answering questions about this Flask codebase.
Use the table below to pick the right tool for each question. ALWAYS use the
specified tool — do not fall back to grep or manual file reading when a tool
is listed.

## Question routing — exact mapping

| Question shape | Tool to use |
|---|---|
| "Where is symbol X defined?" | `mcp__gbrain__code_def` |
| "Where is X mentioned / referenced?" | `mcp__gbrain__code_refs` |
| "How does X work conceptually?" | `mcp__gbrain__search` (returns wiki articles) |
| "Architecture / lifecycle / mechanism" | `mcp__gbrain__search` |
| "What calls X?" / "What does X call?" (one hop) | `mcp__gbrain__code_callers` / `code_callees` |
| **Full call tree from X** | `mcp__plugin_socraticode_socraticode__codebase_flow` |
| **Blast radius / impact of changing X** | `mcp__plugin_socraticode_socraticode__codebase_impact` (symbol-mode) |
| **Circular dependencies** | `mcp__plugin_socraticode_socraticode__codebase_graph_circular` |
| **Dependency graph visualization** | `mcp__plugin_socraticode_socraticode__codebase_graph_visualize` |
| **Graph statistics** | `mcp__plugin_socraticode_socraticode__codebase_graph_stats` |
| "What did we discover before?" | `mcp__gbrain__search` (includes transcripts) |

## Hard rules

1. For symbol definitions, ALWAYS call `code_def` first. Do not grep.
2. For blast radius / impact, ALWAYS use SocratiCode. Manual tracing is not acceptable.
3. For circular dependencies, ALWAYS use `codebase_graph_circular`. Do not trace imports manually.
4. For full call trees, ALWAYS use `codebase_flow`. Do not trace by reading files.
5. For "how does X work" questions, search GBrain first — the wiki articles are indexed there.
6. SocratiCode `codebase_impact` MUST use symbol-mode (pass `Flask`, not `app.py`).

## Sources and pins

- GBrain source: `default` (pinned via `.gbrain-source`)
- Wiki articles: indexed inside GBrain (search returns them as `concepts/*`, `topics/*`, `references/*`)
- SocratiCode project path: `c:\Users\aravi\Desktop\flask`

## Verified anchor facts

- `Flask` class: `src/flask/app.py:109`
- `Blueprint` class: `src/flask/blueprints.py:18`
- Request flow: `wsgi_app` (`app.py:1566`) → `full_dispatch_request` (`app.py:992`) → `dispatch_request` (`app.py:966`)
- `has_request_context`: `src/flask/ctx.py:209`
- Context globals defined in: `src/flask/globals.py`
- Context objects managed in: `src/flask/ctx.py`

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
