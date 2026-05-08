# obsidian-lifecycle-bridge

Bridge your [Obsidian Lifecycle System](https://github.com/alexmarnell/obsidian-lifecycle) vault into any Claude Code project.

Capture knowledge from work sessions and recall vault notes into your current context — without switching projects.

## Features

- **`/lc-collect`** — Capture ideas, meeting notes, or conversation context into your vault's Collect folder
- **`/lc-recall`** — Search your vault and surface relevant notes in your current conversation

## Prerequisites

- An Obsidian vault using the [Lifecycle System](https://github.com/alexmarnell/obsidian-lifecycle) starter
- [obsidian-cli](https://github.com/Yakitrak/obsidian-cli) (optional, improves search — falls back to ripgrep)

## Installation

```bash
claude plugin install obsidian-lifecycle-bridge
```

## Configuration

Set the `LIFECYCLE_VAULT_PATH` environment variable to point to your Obsidian vault.

### User-level (default for all projects)

Add to `~/.claude/settings.json`:

```json
{
  "env": {
    "LIFECYCLE_VAULT_PATH": "/path/to/your/obsidian/vault"
  }
}
```

### Project-level (override for a specific project)

Add to `your-project/.claude/settings.json`:

```json
{
  "env": {
    "LIFECYCLE_VAULT_PATH": "/path/to/your/work/vault"
  }
}
```

Project-level settings override user-level, so you can set a personal vault globally and a work vault per-project.

## Usage

### Capture to vault

```
/lc-collect meeting notes about the auth service refactor decisions
```

Capture conversation context with the `--from-chat` flag:

```
/lc-collect --from-chat key decisions from this debugging session
```

### Recall from vault

```
/lc-recall authentication patterns we've documented
```

Returns a list of matching notes with paths and summaries. The AI will suggest which files to load into your current context.

## How It Works

- **`/lc-collect`** spawns an agent that reads your vault's CLAUDE.md and Style Guide, then writes a properly-formatted note to the vault's `Collect/` folder following your vault's conventions.
- **`/lc-recall`** spawns an agent that searches your vault (using obsidian-cli or ripgrep), returns structured results, and the main conversation decides which notes are worth loading.

Both commands respect your vault's customizations — the agents read your vault's configuration files to understand your specific conventions.

## License

MIT
