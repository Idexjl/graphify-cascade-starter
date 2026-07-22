---
description: Answer a codebase-structure question from the Graphify graph before searching files
---

After I invoke `/graph`, I'll give you a question or two component names.

1. For a question, run in the terminal:
   `graphify query "<my question>" --graph graphify-out/graph.json`
2. For two component names, run:
   `graphify path "<first>" "<second>" --graph graphify-out/graph.json`
3. To understand one component, run:
   `graphify explain "<name>" --graph graphify-out/graph.json`
4. Read the output — it returns the connected nodes and their file:line
   locations. Use that as your map.
5. Only then open the specific files it points to. Don't grep the whole repo for
   structure the graph already answered.
