# graphify-cascade-starter

Wire a prebuilt **Graphify** codebase knowledge graph into **Cascade** (the Windsurf / Devin agent) so it navigates a graph instead of grepping the whole repo on every session. The Graphify + Obsidian + Claude Code flow is everywhere right now; if you choose to use **Devin and Cascade**, this repo reproduces the same flow, end to end. Primary path is an **MCP server**; there's a **rules-file + workflow** fallback for anyone who can't run custom MCP servers.

The whole trick: Graphify is a standalone CLI that produces a queryable `graph.json` and reads it back with `query` / `path` / `explain`. Claude Code's `/graphify` is just a thin skill over that CLI. This repo reproduces the same thing for Cascade — nothing here is Claude-Code-specific.

> **Validated against Graphify `vX`.** _(TODO: set the exact version this was dogfooded against.)_ Where behavior differs across versions, the caveat is called out inline.

> **License: not chosen yet.** See [`LICENSE`](LICENSE) — a placeholder until one is selected. _(TODO)_

---

## Quickstart

Five steps to a working setup (details in the sections below):

1. **Install** Graphify into a project venv: `python -m venv .venv` → activate → `pip install graphifyy`.
2. **Build the graph:** `graphify extract .` (code-only, no API key) → produces `graphify-out/graph.json`.
3. **Register the server:** paste [`examples/mcp_config.snippet.json`](examples/mcp_config.snippet.json) into your Cascade MCP config and set `REPO_ROOT` (see [Part 2b](#part-2b--point-cascade-at-it-each-developer-local)).
4. **Restart Cascade fully** (quitting the window isn't enough — MCP reloads only on a full restart).
5. **Add the rules file** — [`.windsurf/rules/graphify.md`](.windsurf/rules/graphify.md) is already committed and Always On, so Cascade prefers the graph over raw search.

Clone URL (replace `<your-org>`):

```bash
git clone https://github.com/<your-org>/graphify-cascade-starter.git
```

---

## Paths by OS

The commands below are written POSIX-first. On native Windows (PowerShell / cmd, Cascade or Devin Desktop), substitute the right-hand column. Only the **interpreter and config paths** differ — Graphify's own subcommands and MCP tool names are identical on every OS.

| POSIX (Mac / Linux / WSL) | Native Windows (PowerShell / cmd) |
| --- | --- |
| `.venv/bin/python` | `.venv\Scripts\python.exe` |
| `.venv/bin/graphify` | `.venv\Scripts\graphify.exe` |
| `. .venv/bin/activate` | `.venv\Scripts\Activate.ps1` |
| `~/.codeium/windsurf/mcp_config.json` | `%USERPROFILE%\.codeium\windsurf\mcp_config.json` |
| `export REPO_ROOT=/path/to/repo` | `setx REPO_ROOT C:\path\to\repo` |

In the MCP snippet, `${env:REPO_ROOT}` works as written on both — you only change *how* you set the variable (`export` vs. `setx`).

---

## The four `.windsurf/` files

This repo ships exactly four Cascade config files. Two are always-on **rules**; two are slash-command **workflows**. The two workflows have similar names — keep them straight:

| File | Type | Slash command | What it does |
| --- | --- | --- | --- |
| [`.windsurf/rules/graphify.md`](.windsurf/rules/graphify.md) | rule | — (always on) | Graph-first: tells Cascade to prefer the native `graphify.serve` tools over broad file search. |
| [`.windsurf/rules/memory.md`](.windsurf/rules/memory.md) | rule | — (always on) | Second brain: read / append decision notes in `docs/memory/`. |
| [`.windsurf/workflows/graph.md`](.windsurf/workflows/graph.md) | workflow | `/graph` | **Query** the graph without MCP — shells out to `graphify query/path/explain`. The fallback if the MCP server is blocked or flaky. |
| [`.windsurf/workflows/graphify.md`](.windsurf/workflows/graphify.md) | workflow | `/graphify` | **Build / refresh** — regenerate the graph and Obsidian vault (`extract`/`update` + `export obsidian`). |

The key distinction: **`/graph` reads** (fallback query path), **`/graphify` builds** (regenerate graph + vault). They are not interchangeable.

---

## What each piece does

The full stack is three layers, and it's worth being clear about which problem each one solves:

- **Structural map (Graphify graph):** `graphify extract .` turns the repo into `graphify-out/graph.json` — how the code actually connects. Indexing *code* is local AST parsing and needs no API key; source never leaves the building. Only docs/PDFs/images need an LLM backend. Built once, committed, pulled by everyone. Cascade reads it through the MCP server below.
- **Consume side (the integration):** Cascade reaches the graph through an MCP server — and current Graphify **ships one natively** (`python -m graphify.serve`), so there's nothing to build in the common case. A rules file tells Cascade to reach for those tools before broad file search.
- **Declarative memory (Obsidian):** the "second brain" — decisions, context, progress, and architectural intent that live *outside* the graph and persist across sessions. This is the half that stops every new session from re-explaining the project. It's covered in [Part 3](#part-3--the-obsidian-memory-layer) and can be adopted independently of the MCP piece.

A useful way to hold it: the graph answers *"how is the code wired?"*, the memory answers *"what did we decide and why?"* The MCP tools handle the first cheaply; only the memory layer handles the second.

---

## Prerequisites (each developer, one-time)

- Python 3.10+
- The Graphify CLI, installed into a project venv: `python -m venv .venv && . .venv/bin/activate && pip install graphifyy` (this repo standardizes on a venv at `.venv/`). uv/pipx work too, but nothing here requires them.
- Cascade with MCP enabled (Settings → Cascade → MCP Servers). Enterprise installs ship with MCP **off** by default and an admin has to turn it on; some locked-down installs block custom servers entirely. If that's the environment, jump to the [Fallback](#fallback-rules-file--workflow-no-mcp).

---

## Part 1 — Build the graph (shared, committed to the repo)

Run once from the repo root:

```bash
cd your-repo
graphify extract .            # builds graphify-out/graph.json — code-only = no API key
graphify hook install         # optional but recommended: auto-rebuilds on commit,
                              # and installs a git merge driver so graph.json never
                              # leaves conflict markers when two people commit in parallel
```

Sanity-check it works before wiring anything up:

```bash
graphify query "what connects auth to the database?"
graphify explain "UserService"
graphify path "UserService" "DatabasePool"
```

Then commit it so the team shares one graph:

```bash
git add graphify-out/graph.json .windsurf docs/memory
git commit -m "Add Graphify graph + Cascade integration"
```

Notes:
- Default output is `graphify-out/graph.json`, but the directory name has drifted across versions (`.graphify/` on some). Check where `extract` actually wrote the file and adjust paths below to match.
- `graphify-out/cache/` is optional to commit — commit it to save teammates a rebuild, skip it to keep the repo lean.
- When code changes a lot, refresh with `graphify update .` (incremental; the git hook does this for you if you installed it).

---

## Part 2 — The MCP way (primary)

### Part 2a — Use Graphify's built-in MCP server (recommended)

Recent Graphify ships a native MCP server, so there's nothing to build. It reads `graph.json` and exposes graph tools directly over stdio:

```bash
# POSIX
.venv/bin/python -m graphify.serve graphify-out/graph.json
# Native Windows
.venv\Scripts\python.exe -m graphify.serve graphify-out\graph.json
```

Tools it provides: `query_graph`, `get_node`, `get_neighbors`, `get_community`, `god_nodes`, `graph_stats`, `shortest_path`. Point Cascade at it ([2b](#part-2b--point-cascade-at-it-each-developer-local)) and you're done. Confirm your Graphify version has it with `.venv/bin/python -m graphify.serve --help`; if the module is missing, upgrade Graphify (`. .venv/bin/activate && pip install --upgrade graphifyy`).

### Part 2b — Point Cascade at it (each developer, local)

Cascade reads MCP config from `~/.codeium/windsurf/mcp_config.json` (Windows: `%USERPROFILE%\.codeium\windsurf\mcp_config.json`) — note the `windsurf/` segment. This is the **only** scope Cascade supports: there is no project- or workspace-level MCP config, so you can't commit a `.windsurf/mcp_config.json` and have teammates pick it up. Registering the server is a per-developer step; only the rules, workflows, and graph are shareable.

To make that step copy-paste instead of edit-your-own-paths, use config interpolation. Cascade supports `${env:VAR}` (environment variable) and `${file:/path}` (trimmed file contents) inside `command`, `args`, `env`, `serverUrl`, `url`, and `headers`. The canonical snippet lives at [`examples/mcp_config.snippet.json`](examples/mcp_config.snippet.json) — paste it in and set `REPO_ROOT` once in your shell profile:

```json
{
  "mcpServers": {
    "graphify": {
      "command": "${env:REPO_ROOT}/.venv/bin/python",
      "args": ["-m", "graphify.serve", "${env:REPO_ROOT}/graphify-out/graph.json"]
    }
  }
}
```

Set `REPO_ROOT` per OS:

```bash
# POSIX — add to ~/.zshrc / ~/.bashrc
export REPO_ROOT=/absolute/path/to/your/repo
```
```powershell
# Native Windows — sets it persistently for future shells
setx REPO_ROOT C:\absolute\path\to\your\repo
```

An unset variable resolves to an empty string rather than erroring, so if the server won't start, check `echo $REPO_ROOT` (POSIX) / `echo %REPO_ROOT%` (Windows) first.

Prefer hardcoded paths? The same block with literal values:

```json
{
  "mcpServers": {
    "graphify": {
      "command": "/absolute/path/to/your/repo/.venv/bin/python",
      "args": ["-m", "graphify.serve", "/absolute/path/to/your/repo/graphify-out/graph.json"]
    }
  }
}
```

On native Windows, the literal `command` is `C:\path\to\repo\.venv\Scripts\python.exe`.

Either way, `command` must be the **venv interpreter** (`.venv/bin/python`, or `.venv\Scripts\python.exe` on Windows) — an MCP stdio server doesn't launch with the repo as its working directory and does **not** inherit your activated venv, so bare `python` would be system Python without Graphify and you'd get `ModuleNotFoundError: No module named 'graphify'`. Confirm yours with `.venv/bin/python -c "import graphify, sys; print(sys.executable)"`.

Then **fully quit and reopen Cascade / Windsurf** — closing the window alone doesn't reload MCP config. You can also add it through the UI: MCPs icon in the Cascade panel, or Settings → Cascade → MCP Servers.

### Part 2c — Tell Cascade to actually use it

The server exposes tools, but Cascade still needs to be told to prefer them. That's the job of [`.windsurf/rules/graphify.md`](.windsurf/rules/graphify.md) (committed, Always On). It instructs Cascade to reach for the graph tools (`query_graph`, `shortest_path`, `get_node` / `get_neighbors`, `god_nodes` / `graph_stats`) before broad grep/glob/file-read sweeps, and to fall back to raw search only when the graph doesn't cover something.

If the frontmatter mode isn't picked up, set the rule to **Always On** in the Rules panel — that's the reliable lever.

---

## Fallback: rules file + workflow (no MCP)

For anyone whose install blocks custom MCP servers. This path needs the agent to be allowed to run **terminal commands**. Keep the same [`.windsurf/rules/graphify.md`](.windsurf/rules/graphify.md) from 2c (drop the "MCP server" wording), and add a workflow.

[`.windsurf/workflows/graph.md`](.windsurf/workflows/graph.md) is the committed workflow — invoked in Cascade as `/graph`. Give it a question or two component names and it shells out to `graphify query`, `graphify path`, or `graphify explain` against `graphify-out/graph.json`, reads the traced nodes and their file:line locations, and uses that as its map before opening any files.

Weakest but always-available option: run `graphify query "..."` in a terminal yourself and paste the trace into the chat.

---

## Part 3 — The Obsidian memory layer

This is the "second brain" half of the original stack, and it's easy to under-rate. It splits into two very different jobs — be deliberate about which you actually want.

### The honest split

- **Declarative memory (the real add):** decisions, context, progress, and architectural intent — the stuff that *isn't* in the code graph and would otherwise get re-explained every session. The MCP graph tools do nothing for this; it's the genuine gap. **Recommended.**
- **Graph-as-vault (mostly for humans):** rendering `graph.json` into one note per node plus a `.canvas` map. Nice for browsing, but for the *agent* it's redundant — `query_graph` against `graph.json` is cheaper and sharper than having Cascade read a pile of exported notes. Only export this if people will actually open Obsidian to navigate it.

So: adopt the memory layer for the agent, and treat the graph export as an optional human-facing map.

### Part 3a — Declarative memory as plain markdown (no Obsidian app needed)

The key realization: the vault is just markdown files. Cascade can read and append to them directly — **nobody needs Obsidian installed** for the agent to use this. That matters on a locked-down machine.

Keep a `docs/memory/` folder in the repo with atomic, interlinked notes (Zettelkasten-style: one decision per file, standardized frontmatter). The behavior is driven by [`.windsurf/rules/memory.md`](.windsurf/rules/memory.md) (committed, Always On): read the relevant notes at the start of a task instead of asking for re-explanation, and write or update a note whenever a meaningful decision changes. A note template is committed at [`docs/memory/_template.md`](docs/memory/_template.md) — one idea per file, kebab-case filename, frontmatter (`title`, `date`, `tags`, `related`), and `[[wikilinks]]` between notes.

Because these are real files in the repo, they version, review, and merge like any other doc — and if anyone *does* run Obsidian, the folder opens as a vault with working wikilinks and graph view for free.

### Part 3b — Rendering the graph into an Obsidian vault (optional, human-facing)

Current Graphify has a real `graphify export obsidian` subcommand, so this is a one-liner. Run it after each `extract`/`update`:

```bash
graphify export obsidian --dir docs/graph-vault
# omit --dir to default to graphify-out/obsidian
```

That writes one note per node (with frontmatter and Dataview queries) plus a `.canvas` map. The `--obsidian` flag you see in tutorials is just the `/graphify` skill running this same subcommand as a step — so on Cascade you call the subcommand directly.

Heads up on version: on **older** Graphify (roughly pre-v8, e.g. `graphifyy` 0.4.x) `export obsidian` didn't exist and `graphify . --obsidian` errored as "unknown command" — a real source of confusion. If you're on one of those, either upgrade (`. .venv/bin/activate && pip install --upgrade graphifyy`) or call `graphify.export.to_obsidian` / `to_canvas` from a script; check the signatures first with `.venv/bin/python -c "import inspect, graphify.export as e; print(inspect.signature(e.to_obsidian))"` since that internal API moved between releases.

The vault regenerates from scratch each run, so it always reflects the current graph. Keep it in a different folder from `docs/memory/` — the graph export is disposable; the memory notes are not.

### Part 3c — Obsidian MCP server (advanced, usually skip)

If someone genuinely lives in Obsidian and wants Cascade to *search* the vault as a tool (full-text / metadata queries, surgical section edits), there are MCP servers for that — the Local REST API plugin now ships a built-in MCP endpoint, and community servers like `MarkusPfundstein/mcp-obsidian` and `cyanheads/obsidian-mcp-server` expose search/read/patch/append tools.

The catch: all of them require Obsidian to be **running** with the Local REST API plugin enabled, plus trusting a self-signed cert on `127.0.0.1`. On a locked-down machine that's often a non-starter, and for most setups it's overkill — direct file reads ([3a](#part-3a--declarative-memory-as-plain-markdown-no-obsidian-app-needed)) already let Cascade use the notes. Reach for this only if vault search-as-a-tool is worth the setup.

---

## Part 4 — Reproducing `graphify . --obsidian` (hook / workflow / watch)

There's no magic to `graphify . --obsidian` — it's `graphify extract .` (build the graph) followed by `graphify export obsidian` (render the vault). "Implementing it" just means wrapping that pair behind a trigger. Match the trigger to *what* you're refreshing:

- **`graph.json`** is what Cascade reads through MCP — it must stay live, so automate it (a hook).
- **The Obsidian vault** is human-facing and disposable — refresh it on demand (a workflow), or fold it into the hook if you want it every commit.

### Part 4a — Git hook — keep the graph live automatically

The low-effort option handles the graph for you:

```bash
. .venv/bin/activate    # run the install from inside the venv so it embeds
graphify hook install    #   the venv's interpreter path into the hook
                         # post-commit + post-checkout AST rebuild of graph.json
```

Running it from the activated venv is what makes it robust: `hook install` bakes the venv interpreter path into the hook script, so it fires correctly from GUI git clients and CI where nothing is activated. It keeps `graph.json` current (code changes only; docs still need a manual `graphify update .`) and sets a merge driver so parallel commits don't conflict. If Cascade staying current is all you need, stop here.

To also refresh the vault on every commit **and** share the hook with the team, use the committed [`.githooks/post-commit`](.githooks/post-commit) instead of the per-machine `.git/hooks`. Git hooks run from the repo root without your venv activated, so a bare `graphify` won't resolve — the hook calls the venv binary by path. It stays bash (Git for Windows ships bash), and reads an optional `GRAPHIFY_PY` env var so native-Windows users can point it at `.venv\Scripts\graphify.exe` without editing the file:

```bash
# .githooks/post-commit  (committed)
GRAPHIFY="${GRAPHIFY_PY:-.venv/bin/graphify}"   # Windows: export GRAPHIFY_PY=".venv/Scripts/graphify.exe"
"$GRAPHIFY" update . >/dev/null 2>&1 || "$GRAPHIFY" extract . >/dev/null 2>&1
"$GRAPHIFY" export obsidian --dir docs/graph-vault >/dev/null 2>&1
echo "graphify: graph + vault refreshed"
```

Each developer enables it once: `git config core.hooksPath .githooks && chmod +x .githooks/post-commit`. One caveat — `core.hooksPath` *replaces* `.git/hooks`, so it bypasses `graphify hook install`; the committed hook does its own rebuild (the lines above). This assumes everyone's venv is at `.venv/`; if yours lives elsewhere, adjust the path or export the `GRAPHIFY_PY` var the hook reads.

### Part 4b — Cascade workflow — the on-demand `/graphify`

This is the closest reproduction of typing `/graphify --obsidian`. The committed [`.windsurf/workflows/graphify.md`](.windsurf/workflows/graphify.md) is invoked in Cascade as `/graphify`: it rebuilds from current code (`graphify update .`, or `graphify extract .` if there's no graph yet), refreshes the human-facing vault (`graphify export obsidian --dir docs/graph-vault`), reports node and note counts, then stops — deliberately without reading the vault back into context.

Needs terminal execution enabled. This is also the answer to the "skill" idea: Cascade has no Claude-Code-style installable skills, so a workflow *is* the skill equivalent here. (Graphify ships `graphify claude install` to write an always-on `CLAUDE.md` section, but there's no Windsurf/Cascade counterpart — which is exactly why this leans on rules + this workflow.)

### Part 4c — Watch mode — rebuild as you type

For a live loop without commits or slash commands, run a background watcher:

```bash
.venv/bin/python -m graphify.watch . --debounce 3
```

It re-runs AST extraction and rebuilds `graph.json` whenever code changes (no LLM, no API cost); doc/image changes just set a flag telling you to run `graphify update .`. It doesn't touch the Obsidian vault, so pair it with the workflow ([4b](#part-4b--cascade-workflow--the-on-demand-graphify)) when you want the vault refreshed. Good for a focused session; the git hook ([4a](#part-4a--git-hook--keep-the-graph-live-automatically)) is better for steady-state team use.

---

## Gotchas worth knowing

- **Enterprise MCP is off by default.** An admin enables it in settings, and some hardened installs forbid custom servers — that's the deciding factor for MCP vs. fallback. (If you've already added other MCP servers, you're clear.)
- **Admin whitelists are all-or-nothing.** By default everyone can configure their own servers, but the moment an admin whitelists *one* server, every non-whitelisted server is blocked team-wide. If `graphify` silently fails to load while your other servers work, that's the cause — ask your admin to add it. Matching is strict: the Server ID must equal the key in your config (case-sensitive), and `command` and `args` are regex-matched with the argument count required to match exactly. The `env` block isn't matched, so values there stay yours. Teams can also point Cascade at a custom internal MCP registry instead of the default marketplace.
- **No project-level MCP config.** `~/.codeium/windsurf/mcp_config.json` is the only scope — server registration can't be committed to the repo. Use the `${env:REPO_ROOT}` snippet in [2b](#part-2b--point-cascade-at-it-each-developer-local) so at least the block is identical for everyone.
- **Restart fully after config changes.** Quitting the window isn't enough; MCP servers only reload on a full restart.
- **Tool caps.** Cascade caps active MCP tools at 100 and ~20 tool-calls per prompt. The native server adds 7 tools — a non-issue unless you're already running large MCP servers.
- **Graph path drift.** Default is `graphify-out/graph.json`, but some versions use `.graphify/`. Confirm where `extract` wrote it and match the path in your `mcp_config.json` args and the workflow files.
- **Code needs no key; docs do.** Source indexing is local AST parsing (nothing leaves the building). Docs/PDFs/images require an LLM backend (`--backend openai` with `OPENAI_BASE_URL`, or Anthropic, or a local Ollama shim) — check that against your data-handling rules before indexing non-code.
- **The "70x fewer tokens" figure is a promoter claim,** not an independent benchmark. Savings are real and meaningful on large repos — which is the point — but treat the headline multiplier as marketing.
- **`graphify . --obsidian` is skill-only; the subcommand is `graphify export obsidian`.** The bare `--obsidian` flag is sugar the `/graphify` skill expands — on current Graphify the shell equivalent is `graphify export obsidian` ([Part 3b](#part-3b--rendering-the-graph-into-an-obsidian-vault-optional-human-facing)). On older builds (pre-v8) neither existed and `graphify . --obsidian` errored as "unknown command" — upgrade, or call the export functions from a script.
- **Two vaults, don't conflate them.** The graph export (`docs/graph-vault/`) is disposable and regenerated each run; the memory notes (`docs/memory/`) are durable and hand/agent-authored. Keep them in separate folders so a re-export never clobbers your decisions.

---

## Verify it's working

1. In Cascade's MCP panel, confirm `graphify` shows up with its tools (`query_graph`, `shortest_path`, `get_node`, and others).
2. Ask a structural question ("how does auth reach the database?") and watch that it calls `query_graph` instead of launching a repo-wide grep.
3. If it still greps, re-check that the rule is **Always On** and the graph path resolves.

---

## What's shared vs. local

| Shared (commit to repo) | Local (each developer) |
| --- | --- |
| `graphify-out/graph.json` (the graph) | Paste snippet into `~/.codeium/windsurf/mcp_config.json` + set `REPO_ROOT` |
| `.windsurf/rules/graphify.md` | `pip install graphifyy` into `.venv` |
| `.windsurf/rules/memory.md` + `docs/memory/` (second brain) | Enable MCP + full Cascade restart |
| `.windsurf/workflows/graphify.md` (build/export) + `graph.md` (query fallback) | `graphify hook install` (or `git config core.hooksPath .githooks`) |
| `.githooks/post-commit` (if auto-refreshing the vault) | Obsidian app — only if you want 3c |

---

## Command cheat sheet

```bash
graphify extract .                       # build the graph (code = no API key)
graphify update .                        # incremental refresh after changes
graphify hook install                    # auto-rebuild on commit + graph.json merge driver
graphify query "how does X reach Y?"     # structural question -> traced nodes + file:line
graphify path "NodeA" "NodeB"            # shortest path between two components
graphify explain "NodeName"              # what a node is, what calls it, what it calls
graphify --help                          # full command + flag surface

.venv/bin/python -m graphify.serve graphify-out/graph.json   # native MCP server for Cascade
graphify export obsidian --dir docs/graph-vault      # render graph.json -> Obsidian vault
.venv/bin/python -m graphify.watch . --debounce 3            # background auto-rebuild on code changes
```

_(Native Windows: swap `.venv/bin/python` → `.venv\Scripts\python.exe`. Graphify subcommands are identical.)_
