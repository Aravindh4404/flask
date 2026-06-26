# Context Management System — Universal Routing

You have a four-layer context system. Use it for ANY codebase,
not just Flask. The system works for both answering questions
AND fixing bugs.

## What this system is

- GBrain = persistent memory + knowledge + semantic search
- SocratiCode = structural understanding (what calls what, blast radius)
- CLAUDE.md = this file, routes you to the right tool
- GStack/LLM Wiki = compiled knowledge articles

## STEP 1 — When you start any session

Always do these three things first:

1. Check what codebase you are in:
   `codebase_status` — is it indexed?

2. If not indexed yet:
   `codebase_index` — index it first, wait for completion

3. Check GBrain has knowledge:
   `gbrain search "<project name>"` — see what is known

## STEP 2 — Question routing

| Question shape | Tool to use |
|---|---|
| "Where is symbol X defined?" | `mcp__gbrain__code_def` |
| "Where is X mentioned / referenced?" | `mcp__gbrain__code_refs` |
| "How does X work conceptually?" | `mcp__gbrain__search` (returns wiki articles) |
| "Architecture / lifecycle / mechanism" | `mcp__gbrain__search` |
| "What calls X?" / "What does X call?" (one hop) | `mcp__gbrain__code_callers` / `code_callees` |
| Full call tree from X | `mcp__plugin_socraticode_socraticode__codebase_flow` |
| Blast radius / impact of changing X | `mcp__plugin_socraticode_socraticode__codebase_impact` (symbol-mode) |
| Circular dependencies | `mcp__plugin_socraticode_socraticode__codebase_graph_circular` |
| Dependency graph visualization | `mcp__plugin_socraticode_socraticode__codebase_graph_visualize` |
| Graph statistics | `mcp__plugin_socraticode_socraticode__codebase_graph_stats` |
| Find relevant code by concept | `mcp__plugin_socraticode_socraticode__codebase_search` |
| "What did we discover before?" | `mcp__gbrain__search` (includes transcripts) |

## STEP 3 — Bug fixing workflow

When given a bug report, follow these steps IN ORDER.
Do not skip steps. Do not guess.

### 3a — Understand the bug
`gbrain think "<bug description>"`
Get GBrain to synthesize what it knows about this area.

### 3b — Find the relevant code
`codebase_search "<key terms from bug report>"`
Find which files are involved.

### 3c — Understand the structure
`codebase_symbol "<relevant function or class name>"`
See definition, callers, callees in one call.

### 3d — Check blast radius BEFORE changing anything
`codebase_impact "<symbol you plan to change>"`
Know what else might break.

### 3e — Make the fix
Make the smallest possible change that fixes the bug.
Do not refactor. Do not change unrelated things.

### 3f — Output as git diff patch
ALWAYS output the fix in this exact format:

```
diff --git a/path/to/file.py b/path/to/file.py
--- a/path/to/file.py
+++ b/path/to/file.py
@@ line numbers @@
- old line
+ new line
```

Never just describe the fix in words. Always show the actual diff.

## STEP 4 — After any fix, verify it

1. `codebase_impact "<changed symbol>"`
   — confirm nothing unexpected is affected

2. `codebase_graph_circular`
   — did the fix introduce any circular dependencies?

3. State clearly:
   - What file you changed
   - What line you changed
   - Why that fixes the bug
   - What might break

## Hard rules — never break these

1. NEVER grep for symbol definitions. Always use `code_def`.
2. NEVER trace blast radius manually. Always use `codebase_impact`.
3. NEVER trace call trees by reading files. Always use `codebase_flow`.
4. NEVER guess at a fix without first running `codebase_search`.
5. ALWAYS output bug fixes as a git diff patch.
6. ALWAYS check `codebase_status` before searching a new codebase.
7. NEVER assume the codebase is the same as a previous session.
8. For circular dependencies, ALWAYS use `codebase_graph_circular`. Do not trace imports manually.
9. For "how does X work" questions, search GBrain first — the wiki articles are indexed there.

## Codebase setup — for any new project

When pointed at a new codebase:

1. `codebase_index` — index it
2. `gbrain sync` — sync code into GBrain
3. Confirm both are done before answering anything

## Gap analysis rule

After every answer or fix, check:
- Is there anything GBrain flagged as unknown or stale?
- Did `codebase_impact` show files you did not check?
- Are there circular dependencies affecting this area?

If yes — investigate before declaring done.

---

## Current codebase context

SocratiCode project path: auto-detect from current directory

GBrain source: auto-detect from `.gbrain-source` file if present

Anchor facts: NONE hardcoded.
Use `codebase_status` and `codebase_search` to discover
facts fresh for each new codebase.

## When starting on a NEW codebase

Run this sequence before anything else:

1. `codebase_status` — confirm it is indexed
2. `codebase_flow` — discover entry points automatically
3. `codebase_graph_stats` — understand the structure
4. `gbrain search "<project name>"` — see existing knowledge

Do NOT assume anything from previous codebases.
Do NOT use hardcoded file paths or line numbers.

---

# Flask-Specific Context

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
