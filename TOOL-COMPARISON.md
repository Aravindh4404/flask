# GBrain vs SocratiCode — Side-by-Side on the Flask Codebase

Same queries, both tools, raw output. Run via the **MCP tools** (`mcp__gbrain__*`
and `mcp__plugin_socraticode_socraticode__*`), not the CLI. Generated 2026-06-14.

- GBrain source here is keyword-mode (no embedding column populated for the
  pinned `gstack-code-flask` source on this machine) — full-text/BM25-style ranking.
- SocratiCode is semantic (Ollama embeddings) + keyword fused via RRF.
- SocratiCode `projectPath` = `c:\Users\aravi\Desktop\flask`.

---

## TEST 1 — Semantic search (the OVERLAP)

**Same natural-language query through both.**

### GBrain
Tool: `mcp__gbrain__search` — `query: "how does Flask dispatch a request to a view function"`, `limit: 8`

Top hits (slug — score):
1. `concepts/blueprint-internals` — 1.300
2. `concepts/request-lifecycle` — 1.299
3. `context/docs/reference-core-api` — 1.000
4. `context/docs/knowledge-graph` — 1.000
5. `references/core-api` — 1.000
6. `context/docs/internals` — 0.9997  ← the actual step-by-step dispatch chain
7. `context/docs/architecture` — 0.993
8. `transcripts/claude-code/...2026-06-09` — 0.983

Every hit is a **compiled wiki/doc page** (`chunk_source: compiled_truth`). GBrain
returns large prose/markdown chunks that *explain* dispatch — the request-lifecycle
and internals docs literally walk `wsgi_app → full_dispatch_request → dispatch_request
→ view_functions[endpoint](**view_args)`. It returns knowledge *about* the code, not
the code.

### SocratiCode
Tool: `mcp__plugin_socraticode_socraticode__codebase_search` — same query, `limit: 8`

Top hits (file:lines — score):
1. `tests\test_views.py:79-99` — 0.625
2. `src\flask\views.py:16-135` (class `View`) — 0.625
3. `docs\views.rst:91-190` — 0.417
4. `tests\test_views.py:16-26` — 0.396
5. `context\docs\knowledge-graph.md:541-640` — 0.333
6. `context\docs\reference-core-api.md:541-629` — 0.303
7. `docs\views.rst:1-100` — 0.271
8. `src\flask\views.py:136-191` (class `MethodView`) — 0.253

Every hit is a **raw source/test/rst chunk**. SocratiCode returns the actual code
(`View`, `MethodView`, `dispatch_request`, the test that exercises it).

### How they differ
- **Overlap:** both are answering the same NL query and both surface dispatch-related
  material — and both even pick up the same `knowledge-graph.md` Request Lifecycle Call
  Chain. So there IS real overlap on "find me material about request dispatch."
- **Corpus, not just ranking:** GBrain's index here is the compiled wiki + docs +
  transcripts, so it ranks *explanations* first. SocratiCode indexes the raw tree, so
  it ranks *source files and tests* first. They answer the same question from different
  bodies of text.
- **Ranking quality:** SocratiCode's semantic embeddings pulled `views.py`/`dispatch_request`
  to the top — genuinely on-topic source. GBrain's keyword ranking floated the
  *blueprint*-internals page just above the request-lifecycle page (0.300 vs 0.299),
  arguably an inversion — the request-lifecycle page is the more direct answer. Keyword
  scoring rewarded term frequency ("dispatch"/"blueprint" density) rather than intent.
- **Embeddings visible?** Yes, indirectly. SocratiCode's scores are spread (0.62 → 0.25),
  a reranked semantic gradient. GBrain's are clustered hard at 1.000 for several doc
  pages — the hallmark of keyword/full-text scoring saturating, not a learned relevance
  curve. Neither is "wrong"; they're tuned for different corpora.

---

## TEST 2 — Symbol definition (the OVERLAP)

**Find the same symbol both ways.**

### GBrain
Tool: `mcp__gbrain__code_def` — `symbol: "Flask"`

```json
{
  "symbol": "Flask", "count": 1, "status": "ready",
  "defs": [{
    "slug": "src-flask-app-py",
    "file": "src\\flask\\app.py",
    "language": "python", "symbol_type": "class",
    "start_line": 109, "end_line": 1625,
    "snippet": "[Python] src\\flask\\app.py:109-1625 class Flask\n\nclass Flask(App):\n// Members: __init_subclass__, __init__, get_send_file_max_age, send_static_file, open_resource, open_instance_resource, create_jinja_environment, create_url_adapter, raise_routing_exception, update_template_context, make_shell_context, run, test_client, test_cli_runner, handle_http_exception, handle_user_exception, handle_exception, log_exception, dispatch_request, full_dispatch_request"
  }]
}
```

### SocratiCode
Tool: `mcp__plugin_socraticode_socraticode__codebase_symbol` — `name: "Flask"`

```
Symbol: Flask (class)
Defined: src\flask\app.py:109–1625  [python]
Callers (81):
  ← examples\celery\src\task_app\__init__.py:8
  ← examples\tutorial\flaskr\__init__.py:8
  ← tests\conftest.py:46
  ← tests\test_basic.py:1236 ... (+ many) ... and 51 more
Callees (3):
  → ImmutableDict [unresolved]
  → timedelta [unresolved]
  → SecureCookieSessionInterface [unique, 1 candidate(s)]
---
Symbol: Flask (class)
Defined: tests\test_config.py:202–203  [python]   ← a second local `Flask` (test fixture)
  Callers (0) / Callees (0)
```

### How they differ
- **Same anchor:** both nail the real definition — `src/flask/app.py:109–1625`, class.
  Full overlap on "where is it."
- **GBrain gives you the shape of the class:** it appends a **member list** (20 methods:
  `__init__`, `dispatch_request`, `full_dispatch_request`, `run`, `test_client`, …) plus
  a snippet showing `class Flask(App):`. Great for "what does this class expose" before
  reading it. It does **not** tell you who uses it.
- **SocratiCode gives you the graph neighborhood:** definition **+ 81 callers + 3 callees**
  inline, with confidence tags (`unresolved` / `unique`). It also surfaced a **second
  `Flask`** symbol defined in `tests/test_config.py` that GBrain's `code_def` collapsed
  to `count: 1` (GBrain matched the canonical source class; SocratiCode enumerated every
  symbol of that name).
- **Which gives more?** Different axes. GBrain = "what's *inside* this symbol" (members).
  SocratiCode = "what's *around* this symbol" (callers/callees + dup detection). For
  reading the API, GBrain's member list wins; for sizing a change, SocratiCode's caller
  graph wins.

---

## TEST 3 — Find references / callers ("who uses this")

### GBrain — `mcp__gbrain__code_refs` (`symbol: "Flask"`, `limit: 15`)
Returns `count: 15`, every literal mention with file + line + snippet, across
languages — including non-call references keyword/graph tools miss:

| file | line(s) | what it caught |
|---|---|---|
| `docs\conf.py` | 1-97 | doc-build config mentioning Flask |
| `examples\celery\pyproject.toml` | 1-17 | **TOML** dependency `"flask"` (not code!) |
| `examples\celery\src\task_app\__init__.py` | 1-26 | `from flask import Flask` + `app = Flask(__name__)` |
| `examples\celery\...\__init__.py` | 29-39 | `def celery_init_app(app: Flask)` — **type annotation** |
| `examples\javascript\pyproject.toml` | 1-33 | TOML description "...requests to Flask" |
| `examples\tutorial\flaskr\__init__.py` | 6-48 | `Flask(__name__, instance_relative_config=True)` |
| … | | (auth.py, blog.py imports, etc. — 15 total) |

`code_refs` is a **text-reference** finder: it catches imports, `pyproject.toml`
strings, type hints, comments — anything literally mentioning `Flask`. Returns line
numbers, not graph edges.

### GBrain — `mcp__gbrain__code_callers` (`symbol: "Flask"`)
```json
{ "symbol": "Flask", "count": 0, "status": "not_built", "ready": false, "callers": [] }
```
**The call graph is not built on this source.** `code_callers` needs the tree-sitter
call-graph pass (`/sync-gbrain --dream` / `--full`) which hasn't run here. So GBrain's
*structural* "who calls this" returns nothing until you build it.

### SocratiCode — `mcp__plugin_socraticode_socraticode__codebase_symbol` (`name: "Flask"`)
Shows the **81 callers inline** as part of the symbol view (see TEST 2):
```
Callers (81):
  ← examples\celery\src\task_app\__init__.py:8
  ← tests\conftest.py:46
  ← tests\test_basic.py:1236 ... and 51 more
```

### How they differ
- **Three different notions of "who uses this":**
  - GBrain `code_refs` = **every textual mention** (imports, TOML, type hints, docs) —
    the widest net, language-agnostic, ideal for a rename/deprecation sweep.
  - GBrain `code_callers` = **structural call edges**, but **unavailable here** (graph
    not built — returns `status: not_built`).
  - SocratiCode `codebase_symbol` = **structural call edges, ready out of the box** (81
    callers), bundled into the symbol view.
- **Readiness is the headline:** for the structural "who calls this," SocratiCode answers
  immediately; GBrain requires a `--dream` build step first. For the textual "who mentions
  this," GBrain's `code_refs` is richer than anything SocratiCode returns here (it caught
  `pyproject.toml` dependency strings and type annotations).
- **Net:** use GBrain `code_refs` for rename-sweeps (text), SocratiCode for call-graph
  "who actually invokes this" (until GBrain's graph is dreamed).

---

## TEST 4 — Call flow (SocratiCode UNIQUE)

**"What does this entry point call into?" on `Flask.full_dispatch_request`.**

### SocratiCode — `mcp__plugin_socraticode_socraticode__codebase_flow` (`entrypoint: "full_dispatch_request"`, `depth: 5`)
Returns the **entire multi-hop call tree in one call** (trimmed):
```
└── Flask.full_dispatch_request (src\flask\app.py:992)
    ├── Flask.preprocess_request (app.py:1366)
    │   └── Flask.ensure_sync (app.py:1065) → async_to_sync (app.py:1079)
    ├── Flask.dispatch_request (app.py:966)
    │   ├── Flask.raise_routing_exception (app.py:562) → FormDataRoutingRedirect (debughelpers.py:50)
    │   └── Flask.make_default_options_response (app.py:1053)
    ├── Flask.handle_user_exception (app.py:865)
    │   └── Flask.handle_http_exception (app.py:830)
    │       └── App._find_error_handler (sansio\app.py:868)
    │           └── Scaffold._get_exc_class_and_code (sansio\scaffold.py:657)
    └── Flask.finalize_request (app.py:1021)
        ├── Flask.make_response (app.py:1224)
        │   └── JSONProvider.response/dumps (json\provider.py:89/41/166)
        └── Flask.process_response (app.py:1394)
            ├── AppContext._get_session (ctx.py:381) → SecureCookieSessionInterface.open_session (sessions.py:323)
            └── SecureCookieSessionInterface.save_session (sessions.py:337)
                └── get_cookie_{name,domain,path,secure,samesite,httponly,...} (sessions.py)
```
It walks `preprocess → dispatch → handle_user_exception → finalize → make_response /
process_response → session save`, multiple levels deep, with file:line on every node
and explicit `[truncated: cycle]` / `[truncated: depth]` markers where it stops.

### GBrain — `mcp__gbrain__code_callees` (`symbol: "full_dispatch_request"`)
```json
{ "symbol": "full_dispatch_request", "count": 0, "status": "not_built", "ready": false, "callees": [] }
```
**Nothing.** And even with the graph built, `code_callees` returns only the **direct
(one-hop) callees** — you'd have to recurse manually, symbol by symbol, to reconstruct
the tree SocratiCode produced in a single call.

### How they differ — THE KEY DIFFERENCE
- **SocratiCode produces the whole call tree in one call.** This is its signature
  capability: a recursive DFS over resolved call edges, depth-limited, cycle-aware.
- **GBrain has no whole-tree primitive.** `code_callees` is one hop, and here it isn't
  even available (`status: not_built` → needs `/sync-gbrain --dream`). Reconstructing
  the flow would mean N manual `code_callees` calls plus your own cycle detection.
- This is exactly why the routing table sends **call-flow questions to SocratiCode**.
  (Caveat from CLAUDE.md: ~54% of Python call edges are unresolved via dynamic dispatch,
  so the tree is an orientation aid — verify critical edges against source.)

---

## TEST 5 — Impact / blast radius (SocratiCode UNIQUE)

**"What breaks if I change `Flask`?"**

### SocratiCode — `mcp__plugin_socraticode_socraticode__codebase_impact` (`target: "Flask"`, symbol-mode)
```
Blast radius for symbol: Flask
Depth: 3    Total impacted files: 60

Hop 1 (42 files): examples\...\__init__.py, src\flask\cli.py, src\flask\ctx.py,
  src\flask\templating.py, src\flask\testing.py, src\flask\views.py,
  tests\conftest.py, tests\test_basic.py, tests\test_blueprints.py, ... (42 total)
Hop 2 (16 files): src\flask\blueprints.py, src\flask\config.py, src\flask\helpers.py,
  src\flask\json\provider.py, src\flask\sansio\app.py, src\flask\sessions.py, ...
Hop 3 (2 files): src\flask\debughelpers.py, src\flask\wrappers.py
```
A ranked, hop-distance-bucketed blast radius: 60 files that could break, grouped by
how many edges away they sit. (CLAUDE.md guardrail confirmed: **symbol-mode** is what
works — file-mode returns 0 on re-export-heavy Flask.)

### GBrain — no equivalent
GBrain has **no blast-radius / impact primitive**. The closest is `code_callers`
(one hop only) — and as TEST 3 showed, that returns `status: not_built` here anyway.
Even fully built, `code_callers` gives you direct callers, not a multi-hop,
hop-bucketed impact set. To approximate SocratiCode's answer you'd recurse
`code_callers` by hand and de-duplicate to files yourself.

### How they differ
Impact analysis is a **SocratiCode-only capability**. "Before I refactor/rename/delete
X, what's the blast radius?" has a single-call answer in SocratiCode and no direct
answer in GBrain. This is the second reason the routing table reserves SocratiCode for
impact work.

---

## TEST 6 — Circular dependencies (SocratiCode UNIQUE)

### SocratiCode — `mcp__plugin_socraticode_socraticode__codebase_graph_circular`
```
Found 53 circular dependency chain(s):
Cycle 1:  src\flask\__init__.py → src\flask\__init__.py
Cycle 4:  __init__.py → app.py → ctx.py → __init__.py
Cycle 5:  app.py → ctx.py → globals.py → app.py
Cycle 8:  globals.py → sessions.py → json\tag.py → json\__init__.py → globals.py
Cycle 9:  __init__.py → app.py → ctx.py → globals.py → sessions.py → json\tag.py
          → json\__init__.py → json\provider.py → sansio\app.py → __init__.py
Cycle 10: sansio\app.py → config.py → sansio\app.py
Cycle 19: helpers.py → wrappers.py → helpers.py
... and 33 more cycles (53 total)
```
A full enumeration of import cycles in the module graph. (Per CLAUDE.md, many of these —
especially the `__init__.py`-routed ones, e.g. Cycles 1, 4, 9, 17 — are **Python
re-export artifacts**, not real defects. The genuinely interesting tight cycles are the
ones between leaf modules, e.g. `helpers.py ↔ wrappers.py`, `sansio\app.py ↔ config.py`.)

### GBrain — no equivalent
GBrain indexes symbols and (when dreamed) call edges, but exposes **no module-level
circular-dependency / import-graph primitive**. There is nothing in `mcp__gbrain__*` that
answers "what import cycles exist."

### How they differ
Circular-dependency detection is **SocratiCode-only**. GBrain has no comparable tool.

---

## SUMMARY TABLE

| Capability | GBrain result | SocratiCode result | Winner / Notes |
|---|---|---|---|
| **Semantic search** | Ranks compiled wiki/docs/transcripts (knowledge *about* code); keyword/BM25 ranking, scores saturate at 1.0; minor intent inversion (blueprint page over request-lifecycle) | Ranks raw source/tests/rst (the code itself); semantic gradient 0.62→0.25 via Ollama embeddings | **Tie / depends.** GBrain for "explain it" (richer corpus), SocratiCode for "show me the code." Both overlap on relevant material. |
| **Symbol definition** | Exact def + **member list** (20 methods) + snippet; `count:1` (canonical) | Exact def + **81 callers + 3 callees** inline; also found a 2nd `Flask` in tests | **Tie / different axes.** GBrain = inside the symbol; SocratiCode = around it. |
| **References / callers** | `code_refs` = widest textual net (imports, **TOML deps**, type hints, docs) w/ line numbers; `code_callers` = `not_built` here | `codebase_symbol` = 81 structural callers, **ready out of the box** | **Split.** GBrain `code_refs` for rename sweeps; SocratiCode for call edges now. |
| **Call flow (whole tree)** | `code_callees` = `not_built` (and only 1 hop even when built) | **Full multi-hop tree in one call**, cycle/depth-aware, file:line on every node | **SocratiCode.** Signature capability; GBrain has no whole-tree primitive. |
| **Impact / blast radius** | **None** (closest = `code_callers`, one hop, `not_built`) | 60 files, **bucketed by hop distance** (42/16/2), single call | **SocratiCode.** No GBrain equivalent. |
| **Circular dependencies** | **None** — no import-graph primitive | 53 cycles enumerated (many `__init__` re-export artifacts) | **SocratiCode.** No GBrain equivalent. |
| **Infrastructure needed** | **PGLite only** (local, embedded; mode local-stdio) | **Docker + Qdrant + Ollama** (vector DB + local embedding model) | **GBrain** far lighter to run. |
| **Embeddings** | Keyword-only here (no embedding column populated for this source) | Semantic via **Ollama** local embeddings | **SocratiCode** for true semantic recall; GBrain trades that for zero infra. |

---

## CONCLUSION

The two tools genuinely **overlap on retrieval** — semantic search, symbol definition,
and "who references this" can all be answered by either, and on the Flask tree they
surface much of the same material (both even pulled the same `knowledge-graph.md`
dispatch chain in TEST 1). The difference there is corpus and ranking style, not
capability: GBrain indexes the compiled wiki/docs with keyword ranking and returns
*explanations* plus a symbol's **member list**, while SocratiCode indexes the raw tree
with Ollama embeddings and returns the *actual code* plus a symbol's **caller/callee
graph** — so for plain search and symbol lookup it's a wash that tilts on whether you
want prose or source.

Where each **wins** is unambiguous: SocratiCode owns the **structural graph** — whole
call-flow trees, hop-bucketed blast radius, and circular-dependency enumeration are
single-call answers it gives out of the box, and GBrain has either no primitive at all
(impact, circular deps) or only a one-hop one that requires a `--dream` build and still
returned `not_built` in every graph test here. GBrain wins on **operational cost and
breadth of text**: it runs on PGLite alone (no Docker/Qdrant/Ollama), and its
`code_refs` casts the widest textual net, catching `pyproject.toml` dependency strings
and type annotations that a pure call-graph never sees.

That is exactly why the routing table keeps **search and symbol lookup on GBrain** —
they overlap, GBrain is already indexed and infra-free here, and its compiled-wiki
corpus answers "how/why" questions directly — and reserves SocratiCode **only for flow,
impact, circular-deps, and visualization**, the structural questions GBrain structurally
cannot answer on this machine (graph not built) or cannot answer at all. Each tool is
pointed at what it's uniquely good at, and neither is asked to do the other's job.

