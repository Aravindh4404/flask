# Context Management for AI Coding Assistants
## Presentation Notes — Full Story
**A practical system for giving Claude Code deep knowledge of any codebase**

Presenter: Aravindh
Test codebase: Flask web framework
Environment: Windows 11, Claude Code, Git Bash

---

# PART 1 — THE PROBLEM (Why this work matters)

## Open with the problem

When you ask an AI coding assistant a question about a large codebase, it has
two bad options:

1. **Guess from training data** — often outdated or wrong for project-specific
   details
2. **Read files one by one** — slow, expensive in tokens, and it cannot read
   thousands of files every session

For a codebase like Flask (113 Python files) or Hermes (13,000+ files), neither
works well. The assistant either hallucinates or burns enormous resources.

## The core question I set out to answer

> Can I build a system that gives an AI assistant accurate, instant knowledge
> of any codebase — without it having to read everything every time?

## The answer — three tools working together

```
GStack     → generates documentation from source code
GBrain     → indexes everything into a searchable database
LLM Wiki   → synthesises raw docs into structured articles
```

Plus a routing layer (CLAUDE.md) that tells the assistant which tool to use
for which question.

---

# PART 2 — THE THREE TOOLS (What each one is)

## Slide: Tool 1 — GStack

**What it is:** A collection of slash commands that give Claude Code
specialised job roles. Type `/document-generate` and Claude becomes a
technical writer. Type `/learn` and it becomes a knowledge manager.

**Why I used it:** To automatically generate documentation from source code
instead of writing it by hand.

**Key commands I used:**
- `/document-generate` — reads codebase, writes structured docs
- `/sync-gbrain` — indexes code into the GBrain database
- `/setup-gbrain` — sets up GBrain for the machine
- `/learn` — saves key facts permanently

## Slide: Tool 2 — GBrain

**What it is:** A local search database that stores code and docs as
searchable chunks. Runs entirely on my machine using PGLite (a PostgreSQL
database stored as local files).

**Why I used it:** So Claude can find the right code by meaning, not by
reading every file. Ask "how does Flask handle errors" and get back the exact
function at the exact line.

**Key insight:** It stores nothing new — it indexes what already exists and
makes it findable.

## Slide: Tool 3 — LLM Wiki

**What it is:** A Claude Code plugin that reads raw docs and compiles them
into structured wiki articles, synthesising multiple sources into one
coherent explanation.

**Why I used it:** Because raw generated docs are scattered. LLM Wiki combines
information spread across many files into single, cross-referenced articles.

## Slide: How they connect (the pipeline)

```
Source code (.py files)
        │
        ▼ GStack /document-generate
Generated docs (docs/*.md)
        │
        ├──────────────────────────┐
        ▼                          ▼
GBrain indexes code        LLM Wiki ingests docs
(searchable)               (compiles articles)
        │                          │
        │                  Wiki articles created
        │                          │
        └──────────────────────────┘
                    │
                    ▼ import wiki into GBrain
            Everything searchable
                    │
                    ▼
        Claude answers accurately
```

---

# PART 3 — THE STORY OF WHAT I DID (Chronological)

## Chapter 1 — Choosing a test codebase

**The challenge:** I needed a codebase that was:
- Real and complex enough to be meaningful
- Had no pre-existing internal documentation (so I could test fairly)
- In a language I could verify

**What I considered and rejected:**
- httpx, requests — fully documented, Claude already knows them
- nanoGPT — too small (5 files), not meaningful
- llama.cpp — Claude knows it too well from training

**What I chose:** Flask
- 113 Python files — substantial but completable in one session
- The `docs/` folder has user-facing docs (how to USE Flask) but the
  `src/flask/` source has minimal internal documentation (how Flask is BUILT)
- I could verify the output because I understand Python

**Key decision:** I documented Flask INTERNALS (how the code works) not user
docs (how to use it). This is what made it a valid test — the existing docs
do not cover internal architecture.

**Command used:**
```bash
git clone https://github.com/pallets/flask.git
cd flask
```

I cloned rather than forked because I was using Flask as a test bed, not
contributing back. Later I forked it to my own GitHub for the presentation.

---

## Chapter 2 — Installing GStack

**Why first:** GStack provides the `/document-generate` command that starts
the whole pipeline.

**How GStack installs — different from a normal plugin:**
GStack is cloned directly into Claude Code's skills directory. Claude Code
automatically discovers any folder placed in `~/.claude/skills/`.

```bash
git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack
./setup
```

**What setup did:**
- Generated SKILL.md documentation for all 54 skills
- Installed Node.js dependencies
- Installed Playwright for browser skills
- Created symlinks for each skill in `~/.claude/skills/`

**How I verified it worked:**
```bash
ls ~/.claude/skills/ | grep document-generate
# Output: document-generate@
```
The `@` symbol means it is a symlink pointing to the gstack folder.

**First lesson learned (Windows):** The setup script is a bash script. It
will not run in PowerShell or Command Prompt. I had to use Git Bash. This is
a recurring theme — most of these tools assume a Unix environment.

---

## Chapter 3 — Generating documentation with /document-generate

**What I ran:** In Claude Code chat, I typed:
```
/document-generate
```

**The dialog:** It asked where to place documentation. I chose
"Standalone docs/ only" so it created a clean `docs/` directory without
touching the existing Flask source.

**What it produced — 12 files following the Diataxis framework:**

The Diataxis framework organises all documentation into four types:
- **Tutorials** — learning-oriented ("let's build together")
- **How-to guides** — task-oriented ("how to do X")
- **References** — information-oriented ("what is X exactly")
- **Explanations** — understanding-oriented ("why X works this way")

Files generated:
```
docs/architecture.md
docs/internals.md
docs/explanation-contexts-and-proxies.md
docs/explanation-app-factory-pattern.md
docs/howto-blueprints.md
docs/howto-error-handling.md
docs/howto-testing.md
docs/reference-configuration.md
docs/reference-core-api.md
docs/tutorial-getting-started.md
docs/knowledge-graph.md
docs/index.md
```

**Key point for presentation:** I evaluated these docs critically. They were
good for high-level orientation but had gaps in deep cross-module detail.
This is why the next tools matter — GStack alone is about 70% of the picture.

---

## Chapter 4 — Installing GBrain

**Why GBrain:** The generated docs need to be searchable. GBrain is the
search layer.

**Step 1 — Install Bun (the runtime GBrain needs):**
GBrain is written in TypeScript and uses PGLite, a WebAssembly PostgreSQL
database. Bun is the runtime that executes it. Node.js cannot substitute
because of how PGLite uses WebAssembly.

```bash
curl -fsSL https://bun.sh/install | bash
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
```

**Step 2 — Install GBrain:**
```bash
bun install -g gbrain
gbrain --version
# Output: gbrain 0.18.2
```

**Step 3 — Initialize the database:**
```bash
gbrain init --pglite --json
```

**Step 4 — Register GBrain as an MCP server:**
This lets Claude Code use GBrain as a tool automatically.
```bash
claude mcp add gbrain -s user -- \
  "C:/Program Files/Git/bin/bash.exe" \
  -c "export HOME='/c/Users/aravi'; export BUN_INSTALL=\"\$HOME/.bun\"; \
  export PATH=\"\$HOME/.bun/bin:\$PATH\"; exec gbrain serve"
```

---

## Chapter 5 — The GBrain database isolation problem (Key research finding)

**This is one of my most important findings — present it as a real
research moment.**

**What I tried first:** Separate databases per project.
```
~/.gbrain/config.json          → ~/.gbrain/brain.pglite       (Hermes)
~/.gbrain-flask/config.json    → ~/.gbrain-flask/brain.pglite (Flask)
```
Using the `GBRAIN_CONFIG_DIR` environment variable to switch between them.

**Why it failed:** GBrain on Windows ignores the `database_path` value in
config.json. It always uses `~/.gbrain/brain.pglite` no matter what the config
says. The environment variable is also not picked up reliably on Windows.

I spent significant time troubleshooting this — copying config files,
re-initializing, verifying paths — before recognising it was a platform
limitation, not my mistake.

**What actually works — named sources:**
All projects share one database, but each project gets a named isolated source.
```bash
gbrain sources add flask-internals \
  --path "C:/Users/aravi/Desktop/flask" \
  --name "Flask Web Framework"

gbrain search "request context" --source flask-internals
```

**The lesson:** Use `--source <name>` on every search and import to keep
projects separate. Separate database files are not supported on Windows for
this version.

**Presentation framing:** This is a genuine research finding. Anyone
replicating this work on Windows needs to know it. Failures and workarounds
are legitimate research output, not setbacks to hide.

---

## Chapter 6 — Indexing Flask code with /sync-gbrain

**What I ran:** In Claude Code chat:
```
/sync-gbrain --full
```

**The version upgrade discovery:** GStack's `/sync-gbrain` requires GBrain
v0.20.0 or higher, but the install gave me v0.18.2. The skill detected this
and upgraded GBrain automatically from source:
```bash
cd ~/gbrain
git checkout master
git pull origin master   # jumped from v0.18.2 to v0.42.37.0
```

This triggered 90 database schema migrations (v24 → v115).

**What the new version added (major finding):**
- **v0.18.2** — indexed markdown files only, keyword search only
- **v0.42.37.0** — indexes Python/JS/TS/C++ source code directly, plus new
  code navigation commands

**What got indexed for Flask:**
```
113 Python files
1070 chunks
34 links between files
```

**Verification — the moment it worked:**
```bash
gbrain search "request context" --source gstack-code-flask-7d20071a
# Returned: src/flask/ctx.py:209 has_request_context

gbrain code-def Blueprint --source gstack-code-flask-7d20071a
# Returned: src/flask/blueprints.py:18 (class)
```

**Key demo for presentation:** I asked in plain English "how does Flask handle
errors" and GBrain returned the exact function:
```
src/flask/app.py:992 full_dispatch_request
```
No grep. No knowing the function name. No reading files. Plain English →
exact location.

---

## Chapter 7 — How GBrain actually stores code

**For the audience — explain the mechanism:**

GBrain does not store whole files. It breaks code into chunks using AST
(Abstract Syntax Tree) parsing:
- Each function becomes one chunk
- Each class becomes one chunk
- Each method becomes one chunk

So `app.py` (1500 lines) becomes ~30 searchable chunks. Searching "error
handling" returns the exact 27-line function, not the whole file.

Each chunk stores:
```
file:     src/flask/app.py
lines:    992-1019
type:     function
name:     full_dispatch_request
class:    Flask
content:  [the actual code]
```

This is why results are precise — you get the function, not the file.

---

## Chapter 8 — Installing and using LLM Wiki

**Why LLM Wiki:** The generated docs are scattered across 12 files. To answer
"how does the request lifecycle work" Claude had to read three separate files
and combine them. LLM Wiki compiles them into one coherent article.

**How LLM Wiki installs — different from GStack:**
GStack is cloned to a folder. LLM Wiki is a proper plugin installed through
the marketplace system.
```bash
claude plugin marketplace add nvk/llm-wiki
claude plugin install wiki@llm-wiki
```

**Critical lesson:** `/wiki` commands only work in the Claude Code chat panel,
NOT the terminal. I wasted time typing `/wiki` in PowerShell getting
"No such file or directory" before realising this.

**The workflow (3 steps):**

Step 1 — Copy docs to inbox (PowerShell):
```powershell
Copy-Item "C:\Users\aravi\Desktop\flask\docs\*.md" `
  "C:\Users\aravi\wiki\topics\flask-internals\inbox\"
```

Step 2 — Ingest (Claude Code chat):
```
/wiki:ingest --inbox --wiki flask-internals
```
This took my 12 docs and stored them as raw sources.

Step 3 — Compile (Claude Code chat):
```
/wiki:compile --wiki flask-internals
```

**What LLM Wiki produced — 12 sources compiled into 10 articles:**

Concepts (how things work):
- request-lifecycle.md
- context-and-proxies.md
- blueprint-internals.md

Topics (how to do things):
- app-factory-pattern.md
- error-handling.md
- testing-flask-apps.md
- organizing-with-blueprints.md

References (precise lookup):
- configuration.md
- core-api.md
- module-architecture.md

**The quality difference:** Each article synthesises multiple sources. For
example, request-lifecycle.md draws from architecture.md + internals.md +
knowledge-graph.md — combining what was spread across three files into one
explanation. Plus 40+ bidirectional cross-references between articles, and
every article has confidence and volatility ratings.

**Testing the wiki:**
```
/wiki:query "what is flask" --wiki flask-internals
```
It returned a detailed technical answer drawn ENTIRELY from the wiki — citing
which sources it used and honestly noting knowledge gaps. No training data.

---

## Chapter 9 — Importing wiki articles back into GBrain

So both tools can find the same content:
```bash
gbrain import "C:/Users/aravi/wiki/topics/flask-internals/wiki" \
  --source gstack-code-flask-7d20071a --no-embed
# 14 pages imported, 83 chunks created
```

**Final database state:**
```
Source code:     113 files, ~970 chunks
Wiki articles:    14 files, 83 chunks
Session history:  45 transcripts (indexed automatically)
Total:           257 pages, 2691 chunks
```

**Lesson learned:** Do not import both raw docs AND wiki articles — that
creates duplicate content. Import only the compiled wiki articles since they
are higher quality.

---

## Chapter 10 — LSP (Language Server Protocol) for deep code navigation

**What LSP is:** A language server that understands code at the symbol level —
it knows exactly what every function calls, what calls it, and where every
symbol is defined.

**Why I used it for Hermes but not Flask:**
- Hermes is C++ with complex templates and macros. I used clangd LSP to build
  accurate function-level call graphs because GBrain could not parse C++ well.
- Flask is Python. GBrain v0.42.37.0 indexes Python directly with
  `code-def`, `code-refs`, `code-callers`, `code-callees`. So for Python,
  GBrain's built-in code commands replaced the need for a separate LSP.

**The new GBrain code commands (which replace LSP for Python):**
```bash
gbrain code-def <symbol>      # find where defined
gbrain code-refs <symbol>     # find references
gbrain code-callers <symbol>  # find what calls it
gbrain code-callees <symbol>  # find what it calls
```

**Building the call graph:**
```
/sync-gbrain --dream
```
This builds the symbol relationship graph required for callers/callees.

**Key finding:** GBrain v0.42.37.0 eliminates the need for a separate Python
LSP server. For C++ codebases like Hermes, LSP (clangd) is still valuable.

---

## Chapter 11 — The routing layer (CLAUDE.md)

**The final piece:** A CLAUDE.md file at the repo root that Claude Code reads
automatically at the start of every session. It tells Claude which tool to use
for which question.

**Why it is the highest-leverage file:** Without it, Claude either loads
everything or guesses wrong. With it, Claude knows:
- Architecture question → read the wiki concept article
- Where is X defined → use gbrain code-def
- How-to question → read the wiki topics folder
- Exact line numbers → gbrain search with --source

**Example routing entry:**
```
### Architecture questions
Read: wiki/concepts/request-lifecycle.md
Or: gbrain search "request lifecycle"

### Specific function definitions
Use: gbrain code-def <FunctionName>
```

---

# PART 4 — RESULTS AND FINDINGS

## What the system achieves

| Without the system | With the system |
|--------------------|-----------------|
| Claude guesses architecture | GBrain returns exact file:line |
| Reads 50 files per question | Finds the right 2 chunks |
| Scattered docs | One synthesised article |
| Forgets between sessions | /learn saves facts permanently |
| No tool guidance | CLAUDE.md routing table |

## Key research findings (the genuine contributions)

1. **GBrain version matters enormously.** v0.18.2 indexes markdown only;
   v0.42.37.0 indexes source code directly and adds code navigation.

2. **Windows has real limitations.** Separate databases do not work; named
   sources are the workaround. Bash scripts in the tools do not run; direct
   CLI commands are needed.

3. **GBrain v0.42.37.0 replaces LSP for Python** but LSP remains valuable
   for C++.

4. **The routing table is the highest-leverage component.** It costs 30
   minutes and improves every future session.

5. **Tool layering matters.** Source code search + wiki concept articles +
   routing table together give far better answers than any one alone.

## Windows-specific issues documented (for reproducibility)

1. GStack setup requires Git Bash, not PowerShell
2. GBrain separate databases unsupported — use named sources
3. GBrain CLI not on PATH in Claude Code — needs .gbrain-source file
4. Embeddings require an API key — without it, keyword search only
5. GBrain version upgrade needed for /sync-gbrain (v0.20+)
6. PGLite single-writer lock — close Claude Code before CLI import
7. /wiki commands only work in chat, not terminal
8. Sources lost after version upgrade — must re-import
9. Bracket paste mode corrupts pasted multi-line commands

---

# PART 5 — THE COMPLETE WORKFLOW (Summary slide)

```
ONE TIME PER MACHINE:
1. git clone gstack → ~/.claude/skills/gstack && ./setup
2. curl bun.sh/install | bash
3. bun install -g gbrain && gbrain init --pglite
4. claude mcp add gbrain ...
5. claude plugin install wiki@llm-wiki

PER REPO:
1. Clone repo, open Claude Code
2. /document-generate          → generates docs/
3. /sync-gbrain --full         → indexes source code
4. Copy docs to wiki inbox
5. /wiki:ingest --inbox        → ingest sources
6. /wiki:compile               → compile articles
7. gbrain import wiki/         → index articles
8. Create CLAUDE.md routing table
9. /learn                      → save key facts

EACH SESSION:
- Claude reads CLAUDE.md automatically
- Searches GBrain for code/docs
- Queries LLM Wiki for concepts
- Answers accurately
```

---

# PART 6 — HOW TO CLOSE THE PRESENTATION

## The narrative arc to emphasise

1. **Started with a real problem** — AI assistants cannot handle large
   codebases well
2. **Built a layered solution** — three tools each solving one part
3. **Tested rigorously** — on Flask, a codebase I could verify, with no
   pre-existing internal docs
4. **Hit real obstacles** — Windows limitations, version mismatches — and
   documented them as findings
5. **Produced a reproducible system** — anyone can follow the guide on any
   codebase

## The one-sentence takeaway

> By layering documentation generation, semantic code search, and synthesised
> wiki articles — connected by a routing table — an AI assistant can answer
> questions about any codebase accurately and instantly, without reading
> everything every time.

## Honest reflection (good for academic presentations)

The most valuable parts of this work were not when things worked smoothly —
they were the failures. Discovering that GBrain ignores database paths on
Windows, that the version needed upgrading, that bash scripts do not run —
these are the findings someone replicating this work actually needs. Research
documentation that only shows the happy path is not useful. Documenting what
breaks and why is the real contribution.
