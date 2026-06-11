<purpose>
The router is the entry point for all feller tasks. It:
1. Parses flags and initializes the task folder
2. Runs triage to detect domains (cheap, auto)
3. Runs the intent-only interview (3 questions max, never asks about file locations)
4. Dispatches the scout for code reconnaissance
5. Decides the path based on scout verdict
6. Dispatches the chosen path workflow

This file replaces the old monolithic do.md as the entry point. Path-specific logic lives in `paths/*.md` or (for feature path) in the existing `do.md` during the migration period.
</purpose>

<required_reading>
Read all files referenced by the invoking prompt's execution_context before starting.
</required_reading>

<handoff_protocol>
See `@$HOME/.claude/feller/workflows/do.md` for the full handoff protocol (disk-based agent handoff, status signals, canonical states). The router follows the same rules:
- Agents return status signals, not content
- Agent-to-agent data flows through files on disk
- Router reads `verdict.json` as the ONE documented exception (it needs the verdict to make the routing decision)
</handoff_protocol>

<process>

<step name="parse_flags">
Parse flags from `$ARGUMENTS` before any inference:

| Flag | Effect |
|------|--------|
| `--fast` / `--bug` / `--review` / `--test` / `--refactor` / `--feature` | Force a path, skip scout |
| `--auto` | Auto-select recommended options throughout |
| `--text` | Plain-text numbered questions (no `AskUserQuestion`) |
| `--dry-run` | Stop after route decision, show what would run |

Strip recognized flags from `$ARGUMENTS`. Store parsed flags as `$FLAGS`.
</step>

<step name="init_task">
**Create or resume a task folder.**

```bash
TASK_ROOT=".feller/tasks"
TODAY=$(date +%Y-%m-%d)
```

Derive a slug from the first 4-6 meaningful words of the request (lowercase, hyphens, no punctuation). Task folder: `$TASK_ROOT/$TODAY-$SLUG/`.

If the folder exists with an alive lock → refuse. If stale lock or paused → resume from recorded stage.

If new:
- Create folder structure:
  ```
  .feller/tasks/<task>/
    intent.md
    state.json
    scout-scratchpad.md (scout writes this later)
  ```
- Write `state.json` with `{stage: "triage", lock: {pid, started_at}, status: "running"}`
- Write `.feller/active` with the task folder name
- Write the original request (with flags stripped) to `intent.md`
</step>

<step name="triage">
**Auto-detect domains from the request text. Do NOT set size dials here — the scout does that.**

Detect domains by keyword + project-structure scan:

| Keyword / path signal | Domain |
|-----------------------|--------|
| "schema", "table", "migration", "RLS", "pgvector", `migrations/` | `db` |
| "endpoint", "service", "controller", "nestjs", `maestro-nestjs/` | `backend` |
| "component", "page", "react", "vite", `maestro-terminal/`, `maestro-ui/` | `frontend` |
| "whisper", "embedding", "transcript", "insight" | `ai-pipeline` |

This is a soft signal for the interview and scout — not authoritative. The scout confirms or overrides.

Persist detected domains to `state.json`.
</step>

<step name="interview">
**Intent-only interview. 4 categories, 3 questions max.**

Skip entirely if the request is fully clear (e.g., "fix typo in README line 12" — zero ambiguity).

If a path-force flag is in `$FLAGS`, skip the interview entirely and proceed to scout.

### Coverage scan (4 categories, Clear / Partial / Missing)

1. **Goal** — what outcome the user wants
2. **Motivation** — bug / feature / refactor / review / test — classification signal
3. **Constraints** — deadlines, must-not-break, compatibility
4. **Evidence of need** — failing test, user complaint, own observation, external request

Ask only Missing categories. Write answers to `intent.md` as they come in.

### Question rules

- **ONE question per `AskUserQuestion` call.** Never batch.
- **HARD CAP: 3 questions total.** Do not ask a 4th.
- **Early termination verbs:** if the user replies with `"proceed"`, `"done"`, `"stop"`, `"enough"`, `"just do it"` → end the loop immediately.
- **Each question:** header + one-sentence question + 3-5 options with short descriptions. Mark the recommended option in bold.

### EXPLICITLY BANNED questions

These would bias the scout — never ask:
- "Where in the code do you think this lives?"
- "What files do you expect to change?"
- "Have you seen this pattern elsewhere in the codebase?"
- "What approach would you take?"

The scout's entire purpose is independent verification. Location/scope questions defeat that.

### `--auto` behavior

For each Missing category, auto-select the bolded recommended option. Log inline: `[auto] <category>: auto-selected "<option>"`.
</step>

<step name="scout">
**Dispatch `feller-scout` with the FIXED TEMPLATE below. Do not expand it, do not add task-specific instructions, do not list sub-questions.**

The scout's system prompt (`~/.claude/agents/feller-scout.md`) already specifies: what to read, what to write, what to cap, what's banned. If you add "answer these 6 sub-questions with sections and tables" to the dispatch, you override the agent's caps and turn the scout into an ad-hoc planner. That defeats the whole flow.

### The fixed dispatch prompt

Use this verbatim — substitute only `<TASK_DIR>`:

```
You are the feller-scout. Your role, budget, and output caps are defined in
your agent file — follow them exactly. Ignore any temptation to produce
inventory tables, ranked remediation steps, or "notes for the planner".

Inputs (file paths only — read them yourself):
- Intent:   <TASK_DIR>/intent.md
- Task dir: <TASK_DIR>

Deliverables:
- <TASK_DIR>/verdict.json   (machine-readable, router reads this)
- <TASK_DIR>/scout-report.md (≤ 60 lines, human-readable, for disagreement presentation only)

Target: verdict in 60–90 seconds of tool calls. HIGH confidence = terminate
immediately. If the task is genuinely too broad to scope in that budget,
return `confidence: low` with a one-sentence note — NOT a long survey.

Return a status signal only — no content in the reply.
```

### What the router must NOT add

- Do NOT describe the user's request beyond what's in `intent.md` (the scout reads it itself).
- Do NOT list sub-questions ("A. inventory, B. gaps, C. staleness, D. domain, E. path, F. scope") — these mushroom the scope.
- Do NOT ask for tier tables, coverage breakdowns, or remediation plans.
- Do NOT provide file-path hints or pre-bucketed locations — that biases the scout.

If a task genuinely needs deeper up-front mapping, that's a planner job, not a scout job. The scout exits with `confidence: low` and the planner takes it from there.

### After dispatch

The scout writes `verdict.json` and `scout-report.md`, returns a status signal. Do NOT read `verdict.json` here — the routing step handles that.

If STATUS is `BUDGET_EXHAUSTED` or `LOW_CONFIDENCE`, proceed to route — the router will surface the low confidence and let the user decide to override or kick off a planner run.
</step>

<step name="route">
**Decide the path. This step reads `verdict.json` (documented exception to the no-read rule — the router needs the verdict to route).**

If a path-force flag was set in `$FLAGS`, use that path and skip the decision logic below.

### Auto-route (no user prompt)

Proceed without asking when ALL of these are true:
- `confidence == "high"`
- `disagreement_note == null`
- `estimated_scope.cross_domain == false`
- `domain != "multiple"`

Log one line: `Route: <path> (scope: <N> files, ~<M> lines, domain: <domain>). Dispatching.`

### Present-then-confirm

Otherwise, show the user:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 FELLER ► ROUTE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Your framing: <short summary from intent.md>
 Scout found:  <estimated_scope + domain from verdict.json>

 Evidence:
   - <file:line — reason>
   - <file:line — reason if present>

 Suggested path: <suggested_path>
 Why:            <disagreement_note or low-confidence explanation>
```

Read `scout-report.md` for the human-readable evidence. Include quoted code from `verdict.json` evidence when present.

Offer via `AskUserQuestion`:
- **Accept scout** — proceed with suggested_path
- **Override** — force a specific path (user picks: fast / bug / refactor / test / review / feature)
- **Cancel** — stop here

On `Accept scout` → use `suggested_path` from verdict.
On `Override` → ask user which path, use that.
On `Cancel` → mark state `cancelled`, exit.

Write the final path decision to `state.json` as `route.chosen_path`.

If `--dry-run` is set, print "Would dispatch: <path>" and exit cleanly.
</step>

<step name="write_scope">
**Write `.feller/tasks/<task>/scope.txt` before dispatching the path.**

From `verdict.json` evidence, extract the absolute file paths the executor is allowed to touch. Write one path per line.

This is load-bearing — fused agents refuse to edit files outside `scope.txt`.
</step>

<step name="dispatch_path">
Invoke the path workflow:

| Path | Workflow file |
|------|---------------|
| `fast` | `@$HOME/.claude/feller/workflows/paths/fast.md` |
| `bug` | `@$HOME/.claude/feller/workflows/paths/bug.md` |
| `review` | `@$HOME/.claude/feller/workflows/paths/review.md` |
| `test` | `@$HOME/.claude/feller/workflows/paths/test.md` |
| `refactor` | `@$HOME/.claude/feller/workflows/paths/refactor.md` |
| `feature` | `@$HOME/.claude/feller/workflows/paths/feature.md` (wraps legacy do.md preserving router context) |

Dispatch the chosen workflow file by invoking its contents. The path workflow handles its own execute → verify → review → finalize loop and updates `state.json` as it progresses.

If the path workflow file is missing on disk (e.g., user deleted it), log error and fall back to feature path with a warning: `⚠ <path>.md missing — falling back to feature path. To run the legacy workflow directly, use /feller:do-legacy.`
</step>

</process>

<success_criteria>
- [ ] Flags parsed and applied before any inference
- [ ] Task folder created with durable state before any stage begins
- [ ] Triage detects domains from text only (no scope inference)
- [ ] Interview asks intent-only questions, capped at 3, never asks location questions
- [ ] Scout dispatched with only intent path + task path
- [ ] Route decision auto-routes only on HIGH confidence + single domain + no disagreement
- [ ] Present-then-confirm shows scout evidence before asking user
- [ ] `scope.txt` written from scout evidence before dispatching fused agents
- [ ] Path workflow dispatched with the chosen path's workflow file
- [ ] State.json survives context compaction
</success_criteria>
