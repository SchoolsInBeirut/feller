# feller:do

The `feller:do` Claude Code skill — routes a freeform request through
**triage → intent interview → scout (code reconnaissance) → route decision →
chosen path**, collapsing to the smallest pipeline the task actually needs.

The router classifies intent, then a scout does independent code reconnaissance
to determine real scope — the scout's verdict overrides user framing (catches
both "looks like a feature, actually a 2-liner" and "looks like a fix, actually
15 files").

**Paths:** `fast` (trivial) · `bug` · `review` · `test` · `refactor` ·
`feature` (full pipeline).

## Contents

This repo mirrors the `~/.claude/` layout, so installing is a straight copy.

```
commands/feller/do.md              # the /feller:do slash-command entry point
feller/workflows/do.md             # main workflow
feller/workflows/route.md          # router (triage → interview → scout → dispatch)
feller/workflows/paths/
  ├── fast.md                      # trivial changes
  ├── bug.md                       # bug fixes
  ├── review.md                    # code review
  ├── test.md                      # test authoring
  ├── refactor.md                  # behavior-preserving refactors
  └── feature.md                   # full plan→merge→execute→test pipeline
agents/                            # 20 feller-* subagents the paths dispatch to
  ├── feller-scout.md  feller-researcher.md  feller-merger.md
  ├── feller-reviewer.md  feller-verifier.md
  ├── feller-planner-{backend,db,frontend-web}.md
  ├── feller-fast-{backend,db,frontend}.md
  ├── feller-bug-{backend,db,frontend}.md
  ├── feller-refactor-{backend,db,frontend}.md
  └── feller-test-{backend,db,frontend}.md
```

## Install

Copy the trees into your Claude Code config directory:

```bash
cp -r commands/feller   ~/.claude/commands/
cp -r feller            ~/.claude/
cp    agents/*.md       ~/.claude/agents/
```

Then run it from any project:

```
/feller:do <what you want to do> [--hard|--medium|--light] [--test=<tier>] [--no-test] [--domains=<list>] [--dry-run] [--auto]
```

## Notes

- **Scope:** this is the `feller:do` skill plus exactly the 20 agents its paths
  reference. Other feller skills (`feller:do-legacy`, `feller:crazy-review`) and
  their extra agents (auditors, validators, `ai-pipeline`/`frontend-mobile`
  planners, `executor`) are **not** included.
- The `feature` path falls back to `/feller:do-legacy` if you need to force the
  legacy full pipeline without the router — that command is not bundled here.
