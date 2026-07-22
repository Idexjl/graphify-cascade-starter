---
trigger: always_on
---

# Project memory (second brain)

`docs/memory/` holds accumulated decisions, context, and progress as atomic
markdown notes. This is durable project knowledge — treat it as authoritative.

## At the start of a task
Read the relevant notes in `docs/memory/` before asking me to re-explain
architecture, past decisions, or why something is the way it is. Prefer these
over reconstructing context from scratch.

## When we decide something
When a meaningful decision is made or changed (architecture, a tradeoff, a
convention, a gotcha), write or update a note in `docs/memory/`:
- one idea per file, kebab-case filename
- frontmatter: `title`, `date`, `tags`, `related` (wikilinks to other notes)
- link it to related notes with `[[wikilinks]]`
Keep notes short and high-signal. Prune stale ones instead of letting them rot.
