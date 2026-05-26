---
name: handoff
description: "This skill should be used when the user types \"/handoff\", asks to \"hand off this conversation\", \"write a handoff doc\", \"summarize for a fresh agent\", \"prepare context for next session\", \"create a continuity note\", or wants to capture the current session state so a new conversation can pick up where this one left off."
argument-hint: "[optional focus for the next session, e.g. \"debug the failing migration\"]"
allowed-tools: ["Bash", "Read", "Write", "Glob"]
---

# Handoff

Write a disposable handoff document summarizing the current conversation so a fresh agent — with no prior context — can continue the work. Save it to the OS temporary directory, grouped per-project, so multiple handoffs from the same workspace coexist without colliding and without rotting in the project tree.

## Argument Parsing

Treat any text passed after `/handoff` as a **hard description of what the next session will focus on**. Tailor every section of the document toward that objective: lead with the focus, drop unrelated threads, and bias `Next Steps` and `Suggested Skills` to serve it.

If no argument is provided, write a general-purpose handoff covering the active threads in the current conversation.

**Examples:**

- `/handoff debug the failing migration` — handoff scoped to the migration debugging
- `/handoff continue the auth refactor, ignore the unrelated linting work` — handoff scoped to the refactor
- `/handoff` — general handoff of the whole session

## Locating the Handoff Directory

Compute the destination via Bash before writing:

```bash
TEMP_ROOT="${TMPDIR:-/tmp}"
PROJECT_SLUG="$(pwd | sed 's|/|-|g')"
HANDOFF_DIR="${TEMP_ROOT%/}/claude-handoffs/${PROJECT_SLUG}"
mkdir -p "$HANDOFF_DIR"
```

The slug mirrors Claude Code's own `~/.claude/projects/<slug>/` convention (cwd with `/` replaced by `-`), so handoffs from the same project group together and different projects never collide.

**Never** write a handoff inside the workspace. The destination must always be under `${TMPDIR:-/tmp}/claude-handoffs/<project-slug>/`.

## Listing Prior Handoffs

Glob the directory for existing `handoff-*.md`. If any exist:

1. Show the user the list (filename + first-line title from each).
2. Read the most recent one before writing the new doc, so the new file can **reference** prior context rather than duplicate it.
3. If the user asks to update an existing handoff instead of creating a new one, edit that file in place. Otherwise, create a new timestamped file.

## Writing the Document

Default filename: `handoff-YYYYMMDD-HHMMSS.md`, with the timestamp generated in UTC:

```bash
TIMESTAMP="$(date -u +%Y%m%d-%H%M%S)"
HANDOFF_FILE="${HANDOFF_DIR}/handoff-${TIMESTAMP}.md"
```

Use this template:

```markdown
# Handoff — <short title derived from focus arg or active task>

**Project:** <pwd>
**Created:** <ISO 8601 UTC timestamp>
**Focus for next session:** <verbatim user argument, or "general continuation">

## Goal
What we're trying to accomplish.

## Current Progress
What's been done so far in this session.

## What Worked
Approaches that succeeded — keep doing these.

## What Didn't Work
Approaches that failed — don't repeat these.

## Next Steps
Concrete action items for the next agent, ordered by priority.

## Suggested Skills
Specific skills the next agent should invoke to immediately configure the
flavor of the new session — e.g. `lc-recall`, `verify`, `code-review`,
`systematic-debugging`, `test-driven-development`. Pick skills that match the
focus and the next steps, not a generic list.

## References & Pointers
Links and paths to existing artifacts — GitHub issues/PRs, files in the repo,
prior handoff files in this directory, vault notes, design docs. Reference
them; do **not** paste their contents.
```

## Hard Constraints

These are non-negotiable. Apply them every time:

1. **Disposable storage only.** Always save under `${TMPDIR:-/tmp}/claude-handoffs/<project-slug>/`. Never write a handoff inside the workspace — these files are designed to be entirely disposable and must not rot in the project codebase as outdated documentation.
2. **No duplication.** If content already lives in a GitHub issue, PR, repo file, prior handoff, vault note, or any other artifact, link or path-reference it. Do not paste raw text. Use pointers, not copies.
3. **Redaction.** Strip API keys, tokens, passwords, secrets, and PII before writing. Replace each with `<REDACTED>` and a brief note describing what kind of secret was redacted (e.g., `<REDACTED: AWS access key>`).
4. **Tailored context.** When the user passes a focus argument, treat it as a hard scope. Every section bends toward that objective; irrelevant threads are dropped, not preserved "just in case."
5. **Suggested Skills are specific.** The `Suggested Skills` section must name actual skills relevant to the focus and next steps. Do not list every skill available; pick the 2–5 that best flavor the new session.

## Closing Output

After writing the file:

1. Print the absolute path to the user.
2. Suggest how to start the next session — e.g.:

   > Start a fresh Claude Code conversation in this project and run:
   > `Read <absolute-path>`
   > then ask it to continue from there.
