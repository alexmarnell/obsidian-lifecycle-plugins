---
name: lc-recall
description: "This skill should be used when the user types \"/lc-recall\", asks to \"recall from vault\", \"search vault notes\", \"find in lifecycle vault\", \"bring notes into context\", \"check the vault for\", \"look up in vault\", \"pull notes about\", \"what do we have on\", or wants to retrieve knowledge from their Obsidian Lifecycle System vault to inform the current conversation."
argument-hint: "[--quick] [--limit N] [--all] <hint describing what to search for>"
allowed-tools: ["Agent", "Bash", "Read"]
---

# Lifecycle Vault Recall

Search an Obsidian Lifecycle System vault for relevant notes and surface them into the current conversation. Validate configuration, spawn the recall agent to search, then examine results and suggest files to load.

## Configuration Validation

Before any search, verify the `LIFECYCLE_VAULT_PATH` environment variable:

1. Run `echo $LIFECYCLE_VAULT_PATH` via Bash to retrieve the value
2. If empty or unset, display this message and stop:

> **`LIFECYCLE_VAULT_PATH` is not configured.**
>
> Add it to your Claude settings to point to your Obsidian vault:
>
> **User-level** (all projects) — `~/.claude/settings.json`:
> ```json
> { "env": { "LIFECYCLE_VAULT_PATH": "/path/to/your/vault" } }
> ```
>
> **Project-level** (override for one project) — `.claude/settings.json`:
> ```json
> { "env": { "LIFECYCLE_VAULT_PATH": "/path/to/work/vault" } }
> ```
>
> See the [obsidian-lifecycle-bridge README](https://github.com/alexmarnell/obsidian-lifecycle-bridge) for complete setup.

3. If set, verify the path exists and contains a `CLAUDE.md` file at its root. If the path exists but has no CLAUDE.md, display:

> **The vault at `<path>` does not appear to be a Lifecycle System vault** (no CLAUDE.md found). Verify `LIFECYCLE_VAULT_PATH` points to the vault root directory.

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
Vault path: <resolved LIFECYCLE_VAULT_PATH>

Search for: <user's search hint>
Mode: <"quick" if --quick flag was set, otherwise "full">
Limit: <integer from --limit, "none" if --all, or "10" as default>
```

In **full mode** (default), the recall agent uses a Catalog-first search strategy: it searches Catalog MOCs first, follows their curated links, then broadens to keyword search.

In **quick mode**, the recall agent skips MOC reading entirely — it only matches filenames across the vault using Glob, then returns results ranked by title relevance. No files are read for summaries; titles and folder locations are returned as-is. This is significantly faster.

## Processing Results

When the recall agent completes, parse the returned JSON. The expected result shape:

```json
{
  "query": "original search hint",
  "total_matches": 12,
  "showing": 10,
  "results": [
    {
      "path": "/absolute/path/to/note.md",
      "title": "Note Title",
      "folder": "Library",
      "summary": "One-line content summary",
      "relevance": "Why this matches the search hint",
      "via_moc": "Topic MOC Name (if found through a Catalog MOC)"
    }
  ],
  "suggestions": ["alternative search term 1", "alternative term 2"]
}
```

Present findings to the user:

1. **Display a readable summary** of matching notes — title, folder, and why each matches
2. **Highlight MOC-sourced results** — note which results came through Catalog MOCs (these are higher confidence)
3. **Recommend specific files to load** based on relevance to the current conversation or task
4. **Offer to load them** — use the Read tool to bring recommended files into context if the user agrees
5. **If the agent reported many matches**, relay the total count and suggest the user refine their search

**Presentation format:**

> Found **N** relevant notes in your vault:
>
> 1. **Note Title** (`Library/`) — one-line summary *(via Web Development MOC)*
> 2. **Note Title** (`Library/`) — one-line summary
> ...
>
> Recommended to load: #1 and #3 are most relevant to your current work. Load them?

If no results were found, relay the agent's alternative search term suggestions.
