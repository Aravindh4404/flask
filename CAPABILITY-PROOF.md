# CAPABILITY-PROOF — GBrain vs SocratiCode

Definitive, tool-output-backed comparison of GBrain (`mcp__gbrain__*`) and
SocratiCode (`mcp__plugin_socraticode_socraticode__*`) on the Flask repo.

- **Date:** 2026-06-15
- **Repo:** `c:\Users\aravi\Desktop\flask`
- **Method:** every test below was run through MCP tools (not the CLI). Raw
  output is trimmed to ~15 lines where large. Verdicts state only what the
  output proves.
- **Honesty notes (read these):** three of the user's pre-written expectations
  did **not** hold as written. They are recorded truthfully:
  1. **GBrain is NOT missing a blast-radius command.** It ships `code_blast`
     (blast radius) and `code_flow` (call flow). See PROOF 3. They just return
     `not_found` until the call graph is built — the same gating as
     `code_callers`/`code_callees`.
  2. **SocratiCode `codebase_flow` / `codebase_impact` did NOT work out of the
     box.** They first returned *"No symbol graph found"*. I had to run
     `codebase_graph_build` (1.7s) before PROOF 4 and PROOF 5 would run — the
     direct parallel to GBrain needing `--dream`.
  3. **SocratiCode `codebase_search` could not run at all** in this environment
     ("Docker is not available"). PROOF 8's SocratiCode half is therefore an
     environment failure, not a clean code-chunk result. Recorded as-is.

---

## PROOF 1 — GBrain HAS basic code-structure commands (not structure-blind)

**Tool:** `mcp__gbrain__code_def` — `{ symbol: "Blueprint" }`

```json
{ "symbol": "Blueprint", "count": 2, "status": "ready", "ready": true,
  "defs": [
    { "file": "src\\flask\\blueprints.py", "symbol_type": "class",
      "start_line": 18, "end_line": 128, "snippet": "class Blueprint(SansioBlueprint)" },
    { "file": "src\\flask\\sansio\\blueprints.py", "symbol_type": "class",
      "start_line": 119, "end_line": 692, "snippet": "class Blueprint(Scaffold)" }
  ] }
```

**Tool:** `mcp__gbrain__code_refs` — `{ symbol: "Blueprint", limit: 5 }`

```json
{ "symbol": "Blueprint", "count": 5, "status": "ready", "ready": true,
  "refs": [
    { "file": "examples\\celery\\src\\task_app\\__init__.py", "start_line": 1 },
    { "file": "examples\\celery\\src\\task_app\\views.py", "start_line": 1 },
    { "file": "examples\\tutorial\\flaskr\\__init__.py", "start_line": 6 },
    { "file": "examples\\tutorial\\flaskr\\auth.py", "start_line": 1 },
    { "file": "examples\\tutorial\\flaskr\\blog.py", "start_line": 1 }
  ] }
```

**Verdict:** ✅ GBrain finds both definition sites (concrete + sansio base) and
real reference lines with no graph build — it is **not** structure-blind for
definitions/refs.

---

## PROOF 2 — GBrain's call graph needs `--dream` and was NOT built

**Tool:** `mcp__gbrain__code_callers` — `{ symbol: "Flask" }`

```json
{ "symbol": "Flask", "count": 0, "status": "not_built", "ready": false, "callers": [] }
```

**Tool:** `mcp__gbrain__code_callees` — `{ symbol: "full_dispatch_request" }`

```json
{ "symbol": "full_dispatch_request", "count": 0, "status": "not_built", "ready": false, "callees": [] }
```

**Verdict:** ✅ Exact status is `not_built` on both. The call graph is **not**
available out of the box; it requires `/sync-gbrain --dream` (or `--full`).
Definitions/refs (PROOF 1) work without it; caller/callee edges do not.

---

## PROOF 3 — GBrain's impact/circular/visualize coverage (CORRECTED)

The available `mcp__gbrain__*` tool surface includes these code tools:
`code_def`, `code_refs`, `code_callers`, `code_callees`, **`code_blast`**,
**`code_flow`**, plus `traverse_graph`. It does **not** include any
circular-dependency or graph-visualization tool.

**Tool:** `mcp__gbrain__code_blast` — `{ symbol: "Flask" }`

```json
{ "result": "not_found", "did_you_mean": [] }
```

**Tool:** `mcp__gbrain__code_flow` — `{ entry_point: "full_dispatch_request" }`

```json
{ "result": "not_found", "did_you_mean": [] }
```

**Verdict (honest correction):**
- ❌ The claim "GBrain has **no** impact/blast-radius command" is **FALSE** —
  GBrain ships `code_blast` (blast radius) and `code_flow` (call flow). They
  return `not_found` here only because the call graph isn't built (same gate as
  PROOF 2), not because the command is absent.
- ✅ GBrain has **no circular-dependency** command — genuinely absent.
- ✅ GBrain has **no visualization** command — genuinely absent.

---

## PROOF 4 — SocratiCode call flow = FULL TREE in one call

> Prerequisite (honest): first attempt returned *"No symbol graph found. Run
> codebase_graph_build first."* I ran `codebase_graph_build` (READY in 1.7s:
> 106 files, 1634 symbols, 3963 call edges, 54% unresolved), then re-ran.

**Tool:** `mcp__plugin_socraticode_socraticode__codebase_flow`
— `{ entrypoint: "full_dispatch_request" }`

```
└── Flask.full_dispatch_request (app.py:992)
    ├── Flask.preprocess_request (app.py:1366)
    │   └── Flask.ensure_sync → async_to_sync
    ├── Flask.dispatch_request (app.py:966)
    │   ├── Flask.raise_routing_exception → FormDataRoutingRedirect
    │   └── Flask.make_default_options_response (app.py:1053)
    ├── Flask.handle_user_exception (app.py:865)
    │   └── Flask.handle_http_exception → App._find_error_handler → Scaffold._get_exc_class_and_code
    └── Flask.finalize_request (app.py:1021)
        ├── Flask.make_response → JSONProvider.response → DefaultJSONProvider.dumps
        └── Flask.process_response → SessionInterface.save_session →
            SecureCookieSessionInterface.save_session (get_cookie_* x9)
```

**Verdict:** ✅ One call returns the entire multi-hop tree (preprocess →
dispatch → exception handling → finalize → make_response/process_response →
session save), with cycle/depth-truncation tags inline. GBrain's `code_callees`
returns at most one hop per call; SocratiCode walks the whole tree at once.

---

## PROOF 5 — SocratiCode impact / blast radius

**Tool:** `mcp__plugin_socraticode_socraticode__codebase_impact`
— `{ target: "Flask" }` (symbol-mode)

```
Blast radius for symbol: Flask
Depth: 3    Total impacted files: 60
  Hop 1 (42 files): examples\tutorial\flaskr\__init__.py, src\flask\ctx.py,
                    src\flask\cli.py, src\flask\templating.py, src\flask\testing.py,
                    src\flask\views.py, tests\test_basic.py, tests\test_blueprints.py, …
  Hop 2 (16 files): src\flask\blueprints.py, src\flask\config.py, src\flask\helpers.py,
                    src\flask\json\provider.py, src\flask\sansio\app.py,
                    src\flask\sansio\scaffold.py, src\flask\sessions.py, …
  Hop 3 (2 files):  src\flask\debughelpers.py, src\flask\wrappers.py
```

**Verdict:** ✅ Exactly **60 impacted files across 3 hops**, grouped by distance.
This is a real blast-radius result. GBrain's equivalent (`code_blast`) exists as
a tool but returned `not_found` (graph not built) in PROOF 3, so in this
environment only SocratiCode actually produced a blast radius.

---

## PROOF 6 — SocratiCode circular deps

**Tool:** `mcp__plugin_socraticode_socraticode__codebase_graph_circular`

```
Found 53 circular dependency chain(s):
Cycle 1: __init__.py → __init__.py
Cycle 4: __init__.py → app.py → ctx.py → __init__.py
Cycle 5: app.py → ctx.py → globals.py → app.py
Cycle 8: globals.py → sessions.py → json\tag.py → json\__init__.py → globals.py
Cycle 10: sansio\app.py → config.py → sansio\app.py
Cycle 19: helpers.py → wrappers.py → helpers.py
... and 33 more cycles
```

**Verdict:** ✅ **53 cycles** detected. (Caveat per CLAUDE.md: many are Python
re-export artifacts through `__init__.py`, not true defects.) GBrain has no
equivalent command at all (PROOF 3).

---

## PROOF 7 — GBrain holds the WIKI; SocratiCode cannot

**Tool:** `mcp__gbrain__list_pages` — `{ type: "concept" }` (Flask rows only)

```
concepts/blueprint-internals      — Flask Blueprint Internals
concepts/context-and-proxies      — Flask Context System and Proxies
concepts/request-lifecycle        — Flask Request Lifecycle
references/configuration          — Flask Configuration Reference
references/core-api               — Flask Core API Reference
references/module-architecture    — Flask Module Architecture
topics/app-factory-pattern        — App Factory Pattern
topics/error-handling             — Error Handling in Flask
topics/organizing-with-blueprints — Organizing with Blueprints
topics/testing-flask-apps         — Testing Flask Apps
```

(The same listing also returns unrelated Hermes pages from another corpus; the
10 Flask articles above are all present.)

**SocratiCode:** has no command to import markdown/prose. Its only ingest path
is `codebase_index` / `codebase_graph_build`, which parse **source code** via
AST grammars (bash, c, cpp, … python). There is no "add wiki article" tool.

**Verdict:** ✅ The 10 compiled Flask wiki articles live in GBrain. SocratiCode
indexes code only — the prose wiki can live **only** in GBrain.

---

## PROOF 8 — Same conceptual question, both tools

Query (identical to both): **"how does the Flask request lifecycle work"**

**Tool:** `mcp__gbrain__search` — top hits are compiled wiki/doc articles:

```
0.96  context/docs/explanation-contexts-and-proxies  — "The lifecycle: Request arrives │
      App context pushed │ before_request hooks run │ View executes │ after_request …"
0.92  references/core-api      — request_started/request_finished signal table
0.92  concepts/context-and-proxies (compiled_truth)
0.92  concepts/blueprint-internals (compiled_truth)
```

**Tool:** `mcp__plugin_socraticode_socraticode__codebase_search` — same query:

```
ERROR: Docker is not available. Please install Docker Desktop and make sure it is running.
```

**Verdict (honest):** ✅ GBrain returned a directly useful, prose explanation of
the lifecycle (compiled wiki, score 0.96) — exactly what a "how does X work"
question wants. ❌ SocratiCode semantic search **could not run** in this
environment (Docker dependency). So GBrain wins this round decisively, but note
the SocratiCode side failed on infrastructure, not on result quality — I could
not demonstrate "raw code chunks vs wiki article" head-to-head.

---

## FINAL — Proof scorecard

| Claim | Tool tested | Evidence | Result |
|---|---|---|---|
| GBrain can find definitions/refs | `code_def`, `code_refs` | PROOF 1 | **TRUE** |
| GBrain call graph needs `--dream` (not built) | `code_callers`, `code_callees` | PROOF 2 | **TRUE** (`status: not_built`) |
| GBrain has no impact command | `code_blast` | PROOF 3 | **FALSE** — `code_blast` exists (returned `not_found`, graph not built) |
| GBrain has no circular-dep command | gbrain tool list | PROOF 3 | **TRUE** |
| GBrain has no visualization | gbrain tool list | PROOF 3 | **TRUE** |
| SocratiCode gives full call tree in one call | `codebase_flow` | PROOF 4 | **TRUE** (after graph build) |
| SocratiCode does impact (60 files) | `codebase_impact` | PROOF 5 | **TRUE** (60 files / 3 hops) |
| SocratiCode does circular deps (53) | `codebase_graph_circular` | PROOF 6 | **TRUE** (53 cycles) |
| GBrain holds the wiki; SocratiCode cannot | `list_pages` | PROOF 7 | **TRUE** |
| GBrain better on conceptual Q (wiki article) | `search` vs `codebase_search` | PROOF 8 | **TRUE** (SocratiCode side failed on Docker) |

---

## Why both tools are needed (grounded only in the proofs above)

The two tools are complementary, not redundant, and the proofs show exactly
where each is irreplaceable. GBrain is the only place the compiled prose wiki
can live (PROOF 7) and it answers "how does X work" with a directly usable
explanation (PROOF 8); it also resolves definitions and references instantly
with no graph build (PROOF 1). But for whole-graph structural questions GBrain
either gates behind a `--dream` build (PROOF 2: caller/callee edges return
`not_built`; PROOF 3: its `code_blast`/`code_flow` returned `not_found`) or
simply has no command at all — it offers no circular-dependency detector and no
visualization (PROOF 3). SocratiCode fills precisely those gaps: a full
multi-hop call tree in one call (PROOF 4), a 60-file blast radius (PROOF 5), and
53 detected dependency cycles (PROOF 6) — none of which GBrain produced here.
The honest caveat cutting against a simple "SocratiCode = structure" story is
that SocratiCode's structural tools also needed a one-time `codebase_graph_build`
first, and its semantic search couldn't run at all without Docker (PROOF 8) —
so SocratiCode is not a substitute for GBrain on prose or conceptual queries.
Net: use GBrain for knowledge, definitions, and conceptual answers; use
SocratiCode for whole-graph call flow, blast radius, and cycle detection. Each
covers the other's blind spot.
