---
name: lc-recall
description: "This skill should be used when the user types \"/lc-recall\", asks to \"recall from vault\", \"search vault notes\", \"find in lifecycle vault\", \"bring notes into context\", \"check the vault for\", \"look up in vault\", \"pull notes about\", \"what do we have on\", or wants to retrieve knowledge from their Obsidian Lifecycle System vault to inform the current conversation."
argument-hint: "[--quick] [--limit N] [--all] <hint describing what to search for>"
allowed-tools: ["Agent", "Bash", "Read"]
---

# Lifecycle Vault Recall

Search an Obsidian Lifecycle System vault for relevant notes and surface them into the current conversation. Validate configuration, spawn the recall agent to search, then examine results and suggest files to load.

## Argument Parsing

Parse the input provided after `/lc-recall`:

**Flags (extract before the search hint):**

- **`--quick`** — Title-only search. Skip MOC reading and graph walking. Match filenames only (does not need to be exact). Fast lookup for when the user knows roughly what the file is called.
- **`--limit N`** — Set the maximum number of results to return (default: 10). Example: `--limit 5` returns at most 5 results.
- **`--all`** — Return all matching results with no limit. Overrides `--limit` if both are provided.

**Search hint:** Everything remaining after flags are stripped is the search hint — a freeform description of what to find. If no hint is provided, ask the user what to search for.

**Examples:**
- `/lc-recall authentication patterns` — full Catalog-first search, default 10 results
- `/lc-recall --quick telemetry-go v2 design` — fast title-only search, no MOC walking
- `/lc-recall --limit 3 deployment pipeline` — full search, return top 3 only
- `/lc-recall --all OTel` — full search, return everything that matches
- `/lc-recall what do we know about deployment pipelines` — natural language query
- `/lc-recall` — no arguments, ask the user what to look for

## Spawning the Recall Agent

Use the Agent tool to spawn the `recall` agent. Build the prompt with this structure:

```
Vault: !`echo "${LIFECYCLE_VAULT:?LIFECYCLE_VAULT is not set in your settings.json}"`

Search for: <user's search hint>
Mode: <"quick" if --quick flag was set, otherwise "full">
Limit: <integer from --limit, "none" if --all, or "10" as default>
```

In **full mode** (default), the recall agent uses a Catalog-first search strategy: it searches Catalog MOCs first, follows their curated links, then broadens to keyword search.

In **quick mode**, the recall agent skips MOC reading entirely — it only matches filenames across the vault using Glob, then returns results ranked by title relevance. No files are read for summaries; titles and folder locations are returned as-is. This is significantly faster.

## Processing Results

The agent uses graph-based ranking: it picks 2–5 seed notes from the user's query, walks each seed's outgoing and incoming wikilinks, then scores every neighborhood note by adjacency to seeds (with hub-seeds down-weighted, à la Adamic-Adar).

Results split into two buckets:
- **`notes`** — Library, Refine, Collect, Daily entries. Substantive content the user wants. Capped by `--limit`.
- **`maps`** — Catalog MOCs. Organizational scaffolding that links the notes together. Capped at 5.

`Archive/` and `Resources/` are excluded by default.

```json
{
  "query": "original search hint",
  "seeds": ["Seed Note Title", "Another Seed"],
  "notes": [
    {
      "path": "/absolute/path.md",
      "title": "Note Title",
      "folder": "Library",
      "summary": "One-line summary, or null",
      "relevance": "Why this matches",
      "score": 1.42,
      "match_sources": ["title", "content"],
      "connections": ["Seed Note Title", "Another Seed"]
    }
  ],
  "maps": [
    {
      "path": "/absolute/path.md",
      "title": "Topic (MOC)",
      "folder": "Catalog",
      "summary": "...",
      "score": 1.83,
      "match_sources": ["graph"],
      "connections": ["Seed Note Title"]
    }
  ],
  "totals": {
    "notes_total": 12, "notes_showing": 10,
    "maps_total": 4, "maps_showing": 4
  },
  "suggestions": []
}
```

**Present results:**

1. Mention the seeds the agent used. Example: *"Walked the graph from these seeds: **Seed A**, **Seed B**."*
2. **Lead with `notes`** — these are what the user is most likely to want. List as `**Title** (`Folder/`) — summary`. Append `*(connected to: Seed A, Seed B)*` for multi-connection results.
3. **Then list `maps`** under a separate "Related maps" or similar heading. Frame them as scaffolding: "These index notes link to the results above and may have more pointers."
4. Recommend specific notes to load, then offer to Read them. Don't recommend loading maps unless the user asks — they're linked-to from the notes anyway.
5. If `totals.notes_total > totals.notes_showing` or similar, mention the overflow and suggest narrowing.
6. If both `notes` and `maps` are empty, relay `suggestions`.
