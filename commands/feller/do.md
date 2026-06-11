---
name: feller:do
description: Route a freeform request through triage → plan → execute → test, collapsible by task size
argument-hint: "<what you want to do> [--hard|--medium|--light] [--test=<tier>] [--no-test] [--domains=<list>] [--dry-run] [--auto] [--text]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Agent
  - AskUserQuestion
  - Task
---

<objective>
Entry point for all feller tasks. Routes the request through:

  triage → intent-only interview → scout (code reconnaissance) → route decision → chosen path

The router classifies intent from the text + 3-question max interview, then the scout does independent code reconnaissance to determine actual scope. The scout's verdict overrides user framing — catches both "looks like a feature, actually a 2-liner" and "looks like a fix, actually 15 files."

Paths: `fast` (trivial), `bug`, `review`, `test`, `refactor`, `feature` (full pipeline).

Fall back to `/feller:do-legacy` if you need to force the legacy full pipeline without the router.
</objective>

<execution_context>
@$HOME/.claude/feller/workflows/route.md
</execution_context>

<context>
$ARGUMENTS
</context>

<process>
Execute the feller router at `@$HOME/.claude/feller/workflows/route.md` end-to-end.

The router handles: task init → triage → interview → scout → route decision → dispatch to the chosen path workflow.

Respect flags parsed from $ARGUMENTS. Path-force flags (`--fast`, `--bug`, `--review`, `--test`, `--refactor`, `--feature`) skip scout and go straight to the chosen path.
</process>
