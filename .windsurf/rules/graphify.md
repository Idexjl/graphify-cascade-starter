---
trigger: always_on
---

# Codebase knowledge graph (Graphify)

This repo has a prebuilt knowledge graph at `graphify-out/graph.json`, served by
the `graphify` MCP server. It maps how components actually connect (calls,
imports, data flow) with file:line evidence.

## Prefer the graph over raw search
Before broad grep/glob/file-read sweeps to understand structure, call the graph tools:
- `query_graph` — "what connects X to Y?", "where is X used?", "how does a
  request reach Z?". First stop for any architectural / "how does this fit
  together" question.
- `shortest_path` — trace how two named components connect, hop by hop.
- `get_node` / `get_neighbors` — understand one component and what it touches
  before editing.
- `god_nodes` / `graph_stats` — get the high-level shape of the codebase.

Fall back to grep/read only when the graph doesn't cover it (a brand-new file
not yet indexed, or line-level implementation detail).

## Keep it fresh
The graph is committed. If a lot of code changed this session, remind me to run
`graphify update .` before trusting structural answers.
