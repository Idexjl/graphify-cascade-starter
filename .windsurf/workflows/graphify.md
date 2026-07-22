---
description: Rebuild the Graphify graph and refresh the Obsidian vault
---

When I invoke `/graphify`:
1. Rebuild from current code: run `graphify update .`
   (if there's no graph yet, run `graphify extract .` instead).
2. Refresh the human-facing vault: run `graphify export obsidian --dir docs/graph-vault`.
3. Report the node and note counts, then stop. Do NOT read the vault back into
   context — query the graph through the MCP tools, not by reading the notes.
