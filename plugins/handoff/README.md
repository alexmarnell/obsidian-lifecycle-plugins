# handoff

Write disposable handoff documents to your OS temp directory so a fresh Claude Code agent — with no prior context — can pick up where the current session left off. Project-scoped so multiple handoffs from the same workspace group together; never written inside the workspace, so they don't rot as outdated docs.

## Skills

- **`/handoff`** — Capture the current session into a timestamped handoff document under `${TMPDIR}/claude-handoffs/<project-slug>/`. Pass an optional focus argument to scope the document to a specific objective.

## Usage

```
/handoff
/handoff debug the failing migration
/handoff continue the auth refactor, ignore the unrelated linting work
```

The skill will:

1. Compute a per-project directory under `${TMPDIR:-/tmp}/claude-handoffs/`.
2. List any prior handoffs in that directory and read the most recent one for context.
3. Write a new timestamped handoff with sections for Goal, Current Progress, What Worked, What Didn't Work, Next Steps, Suggested Skills, and References & Pointers.
4. Print the absolute path so you can start a fresh session and `Read` it.

## Constraints

- **Disposable only.** Handoffs are written to the OS temp directory, never into the workspace.
- **No duplication.** Content captured elsewhere (GitHub issues, PRs, repo files, vault notes) is referenced, not copied.
- **Redaction.** API keys, tokens, passwords, and PII are stripped before writing.
- **Tailored.** An optional focus argument is treated as a hard scope — every section bends toward that objective.
