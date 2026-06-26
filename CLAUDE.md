# Claude Code — Context Routing Guide

## Quick Reference (most common operations)

| Task | Tool(s) |
|---|---|
| Where is symbol X defined? | `mcp__gbrain__code_def X` |
| Where is X referenced? | `mcp__gbrain__code_refs X` |
| What calls X? | `mcp__gbrain__code_callers X` |
| What does X call? | `mcp__gbrain__code_callees X` |
| Full call tree from X | `mcp__socraticode__codebase_flow` |
| Blast radius of changing X | `mcp__socraticode__codebase_impact X` |
| Find code by concept | `mcp__socraticode__codebase_search` |
| 360° view of a symbol | `mcp__socraticode__codebase_symbol` |
| How does X work? / architecture? | `mcp__gbrain__search X` |
| Past decisions / what we learned | `mcp__gbrain__search X` |
| Circular dependencies | `mcp__socraticode__codebase_graph_circular` |
| Dependency graph | `mcp__socraticode__codebase_graph_visualize` |
| Is codebase indexed? | `mcp__socraticode__codebase_status` |

---

## What each layer does

| Layer | Purpose |
|---|---|
| **GBrain** | Persistent memory, semantic search, wiki articles, past decisions |
| **SocratiCode** | Structural analysis — call trees, blast radius, circular deps |
| **Grep** | Exact string / regex matching when you already know the string |
| **CLAUDE.md** | This file — routes you to the right tool for each situation |

**GBrain vs Grep decision rule:**

| Situation | Use |
|---|---|
| Searching by concept, don't know exact string yet | GBrain / SocratiCode search |
| Know the exact identifier or string | Grep |
| Past decisions, architecture docs, wiki | GBrain search |
| Multiline patterns, regex | Grep |

---

## Session start — always do this first

```
mcp__socraticode__codebase_status
```

- If **not indexed**: run `mcp__socraticode__codebase_index`, wait for 100%
- If **new codebase**: also run these in parallel after indexing:
  ```
  mcp__socraticode__codebase_graph_stats    # structure overview
  mcp__gbrain__search "<project name>"      # existing knowledge
  ```
- NEVER assume the codebase is the same as a previous session

---

## Parallel tool calls — always do this

Run independent tools **at the same time**, not one after another.

**Safe to call in parallel — these pairs have no dependency on each other:**

| Parallel pair | Note |
|---|---|
| `gbrain think` + `codebase_search` | Both are read-only lookups with different indices |
| `codebase_status` + `gbrain search` | No shared state |
| `codebase_symbol` + `codebase_search` | Different inputs, different indices |
| `codebase_impact` + `codebase_graph_circular` | Both post-fix verification checks |
| `gbrain search` + `codebase_search` | GBrain and SocratiCode are independent |

Never run a tool sequentially when it does not depend on the previous tool's output.

---

## Workflow A — Bug Fixing

Follow **IN ORDER**. Do not skip steps. Do not guess.

### A1 + A2 — Run in parallel

```
mcp__gbrain__think "<bug description>"          # synthesize what is known
mcp__socraticode__codebase_search "<key terms>" # find involved files
```

**If `gbrain think` fails** (e.g. no API key configured): skip it, do not retry.
Proceed with SocratiCode results only and use your own synthesis.

### A3 — Understand the symbol

```
mcp__socraticode__codebase_symbol "<function or class you plan to change>"
```

### A4 — Check blast radius BEFORE touching anything

```
mcp__socraticode__codebase_impact "<symbol you plan to change>"
```

Do not write a single line of code until this is done.

### A5 — Make the fix

- Smallest possible change that fixes the bug
- No refactoring, no cleanup, no unrelated changes

### A6 — Output as git diff (required format)

```diff
diff --git a/path/to/file.py b/path/to/file.py
--- a/path/to/file.py
+++ b/path/to/file.py
@@ -N,M +N,M @@
-old line
+new line
```

Never describe the fix in words only. Always show the actual diff.

### A7 — Post-fix verification (run in parallel)

```
mcp__socraticode__codebase_impact "<changed symbol>"   # nothing unexpected affected
mcp__socraticode__codebase_graph_circular              # no circular deps introduced
```

Then state clearly:
- File changed and line number
- Why that line fixes the bug
- What might break

---

## Workflow B — Feature Development

### B1 — Understand the area (run in parallel)

```
mcp__gbrain__search "<feature area>"
mcp__socraticode__codebase_search "<related concept>"
```

### B2 — Find integration points (run in parallel)

```
mcp__socraticode__codebase_symbol "<entry point or symbol to extend>"
mcp__socraticode__codebase_flow "<entry point>"
```

### B3 — Check blast radius of the extension point

```
mcp__socraticode__codebase_impact "<symbol you will extend>"
```

### B4 — Implement

- Add only what the feature requires
- No speculative abstractions, no "future-proofing"

### B5 — Verify (run in parallel)

```
mcp__socraticode__codebase_graph_circular   # no new circular deps
mcp__socraticode__codebase_impact "<new symbol>"  # understand what depends on new code
```

---

## Workflow C — Code Explanation / Understanding

### C1 — Semantic search first (run in parallel)

```
mcp__gbrain__search "<concept to explain>"
mcp__socraticode__codebase_search "<concept>"
```

### C2 — Symbol deep-dive (run in parallel)

```
mcp__socraticode__codebase_symbol "<symbol>"
mcp__socraticode__codebase_flow "<symbol>"
```

### C3 — Read only the files the tools identified

Do not speculatively read files. Let the tools find them first.

---

## Workflow D — Refactoring

### D1 — Baseline blast radius (mandatory before any move)

```
mcp__socraticode__codebase_impact "<symbol to refactor>"
```

### D2 — Full call tree (understand all touch points)

```
mcp__socraticode__codebase_flow "<symbol>"
```

### D3 — Refactor

- Change the symbol. Update all callers identified in D1/D2.
- Do not change behavior — structure only.

### D4 — Verify (run in parallel)

```
mcp__socraticode__codebase_impact "<refactored symbol>"
mcp__socraticode__codebase_graph_circular
```

---

## Hard rules — never break these

1. NEVER grep for symbol definitions → always use `code_def`
2. NEVER trace blast radius manually → always use `codebase_impact`
3. NEVER trace call trees by reading files → always use `codebase_flow`
4. NEVER guess a fix without `codebase_search` first
5. ALWAYS output bug fixes as a git diff patch
6. ALWAYS check `codebase_status` at the start of every session
7. NEVER assume codebase state carries over from a previous session
8. NEVER check circular deps manually → always use `codebase_graph_circular`
9. ALWAYS run `codebase_symbol` before changing any symbol
10. ALWAYS run impact + circular check after every fix
11. ALWAYS run independent tools in parallel — sequential calls on independent inputs waste time
12. When `gbrain think` has no API key: skip it, proceed with SocratiCode; do not block on it

---

## Gap analysis — after every answer or fix

Check all three:
- Did `codebase_impact` reveal files you did not examine?
- Did GBrain flag anything as unknown or stale?
- Are there circular dependencies in this area?

If yes to any — investigate before declaring done.

---

## New project setup

When pointed at a codebase that is not yet indexed:

1. `mcp__socraticode__codebase_index` — index it, wait for completion
2. `/sync-gbrain` — sync code into GBrain
3. Confirm both are done before answering anything

## If both tools return empty on a new codebase

Do this in order:
1. `mcp__socraticode__codebase_index` — wait for 100%
2. `/sync-gbrain` — wait for completion
3. Only then start any workflow
4. Do NOT attempt to answer questions while indexing is still running

---

# Flask-Specific Context

## GBrain source pins

- GBrain source: `default` (pinned via `.gbrain-source`)
- SocratiCode project path: `c:\Users\aravi\Desktop\flask`
- Wiki articles indexed as `concepts/*`, `topics/*`, `references/*`

## Anchor facts

> **Always verify these with `codebase_symbol` before using in a diff — line numbers drift.**

| Symbol | Approximate location | Status |
|---|---|---|
| `Flask` class | `src/flask/app.py` ~line 109 | Verify |
| `Blueprint` class | `src/flask/blueprints.py` ~line 18 | Verify |
| `has_request_context` | `src/flask/ctx.py` ~line 209 | Verify |
| Context globals | `src/flask/globals.py` | Stable (file, not line) |
| Context objects | `src/flask/ctx.py` | Stable (file, not line) |

Request flow (verify line numbers via `codebase_symbol` before citing):
`wsgi_app` → `full_dispatch_request` → `dispatch_request`

## GBrain call-graph notes

- `code-callers` / `code-callees` need the graph built first
- If they return `count: 0` → run `/sync-gbrain --dream` (or `--full`)
- Do NOT run `/sync-gbrain` while `gbrain autopilot` is active (races the daemon, #1734)
- Register repos with `gbrain sources add --path <dir>`, not `--url`
- Two indexed corpora: this worktree's code (auto-pinned) + `~/.gstack/` curated memory
