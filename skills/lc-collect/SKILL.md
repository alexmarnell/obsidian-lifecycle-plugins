---
name: lc-collect
description: "This skill should be used when the user types \"/lc-collect\", asks to \"capture to vault\", \"collect a note\", \"save to lifecycle vault\", \"send to collect\", \"save this to Obsidian\", \"capture this idea\", \"jot this down in my vault\", \"save what we discussed to the vault\", \"capture this conversation to vault\", or wants to create a new note in their Obsidian Lifecycle System vault's Collect folder."
argument-hint: "[--from-chat] <what to capture — an idea, meeting summary, decision, etc.>"
allowed-tools: ["Agent", "Bash", "Read"]
---

# Lifecycle Vault Capture

Capture information into an Obsidian Lifecycle System vault's Collect folder. Validate configuration, parse arguments, and spawn the collector agent to write the note.

## Configuration Validation

Before any capture, verify the `LIFECYCLE_VAULT_PATH` environment variable:

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

Parse the input provided after `/lc-collect`:

**`--from-chat` flag detection:**
- If present, gather relevant context from the current conversation before spawning the agent
- Summarize key decisions, findings, code changes, or discussion points that relate to the capture description
- Strip the flag from the description text

**Description text:**
- Everything after the optional flag is the description of what to capture
- If no description is provided, ask the user what to capture

**Examples:**
- `/lc-collect meeting notes about the auth refactor` — freeform description, no conversation context
- `/lc-collect --from-chat key decisions from this debugging session` — include summarized conversation context
- `/lc-collect` — no arguments, ask the user what to capture

## Spawning the Collector Agent

Use the Agent tool to spawn the `collector` agent. Build the prompt with this structure:

```
Vault path: <resolved LIFECYCLE_VAULT_PATH>

Task: Create a new note in the vault's Collect/ folder.

What to capture:
<user's description text>
```

If `--from-chat` was used, append a clearly delimited section:

```
---
Conversation context:
<summarized conversation context relevant to the description>
```

The collector agent reads vault conventions (CLAUDE.md and Style Guide), chooses a title, writes the note, and searches the Catalog for relevant MOCs to suggest as wikilinks.

## After Capture

Display the note title and full file path created. Summarize what was captured in one sentence. Remind the user the note is in Collect/ and will be processed during weekly review.
