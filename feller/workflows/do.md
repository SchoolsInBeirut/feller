<purpose>
Take a freeform request and route it through a collapsible pipeline:

  triage → interview → research → plan → merge → execute → test → finalize

The shape is always the same. Triage sets dials; every later stage reads those dials and self-skips when it has nothing to do. This is what lets one flow handle trivial, small, medium, and large tasks without branching into separate commands.

This file is the single source of truth. The `/feller:do` command is a thin wrapper that loads this workflow.
</purpose>

<required_reading>
Read all files referenced by the invoking prompt's execution_context before starting.
</required_reading>

<handoff_protocol>
**Agents communicate via disk, not context.**

Every feller agent writes its detailed output to a file on disk. The value returned to this orchestrator is a short **status signal** — never content. The orchestrator routes based on the signal and passes **file paths** to downstream agents, never file contents.

### Standard return signal

Every feller agent ends its response with this block (target: under 80 words total):

```
STATUS: <canonical state for this agent type>
WROTE: <path> — <one-line description>
SIGNALS:
- [HIGH|MEDIUM|LOW]: <routing hint>
READ_NEXT: <path the next stage should read>
```

### Canonical states by agent type

| Agent | Success | Concern | Blocked |
|-------|---------|---------|---------|
| Planner | `PLAN_READY` | — | `BLOCKED`, `NEEDS_CONTEXT` |
| Researcher | `RESEARCH_COMPLETE` | `INSUFFICIENT` | `BLOCKED` |
| Merger | `MERGED` | `CONFLICTS_UNRESOLVED` | `BLOCKED` |
| Executor | `READY_FOR_REVIEW` | `DONE_WITH_CONCERNS`, `VERIFICATION_FAILED` | `BLOCKED`, `NEEDS_CONTEXT` |
| Verifier | `GREEN`, `YELLOW` | `PARTIAL` | `RED` |
| Reviewer | `PASS`, `PASS_WITH_NITS` | `NEEDS_HUMAN` | `FAIL` |

### Orchestrator routing rules

1. Parse only `STATUS` and `SIGNALS` from the return value
2. On success states → dispatch next stage with file paths from `WROTE` / `READ_NEXT`
3. On concern states → read the `SIGNALS` section; if any `[HIGH]`, pause and ask user
4. On blocked states → handle (retry, escalate, ask user)
5. **Never read the files agents wrote** — pass those paths to the next agent

**Exception:** if `SIGNALS` contains `[HIGH]`, the orchestrator MAY read the referenced file to decide whether to pause.

### Agent-to-agent handoff

Agents hand off through disk, not through the orchestrator's context:
- Planner writes `plans/backend.md` → Merger reads `plans/*.md` (orchestrator passes the directory path)
- Merger writes `merged-plan.md` → Executor reads `merged-plan.md` (orchestrator passes the file path)
- Executor writes `scratchpad.md` + `files-changed.txt` → Verifier runs commands (orchestrator passes task path)
- Verifier writes `verification-report.md` → on RED, executor reads it via `SendMessage` (orchestrator passes file path)
- Reviewer writes `review-report.md` → on FAIL, executor reads it via `SendMessage` (orchestrator passes file path)
</handoff_protocol>

<process>

<step name="parse_flags">
**Parse flags from $ARGUMENTS before any inference.**

| Flag | Effect |
|------|--------|
| `--hard` / `--h` | Force size=large, all dials max |
| `--medium` / `--m` | Force size=medium |
| `--light` / `--l` | Force size=small, minimal dials |
| `--test=<tier>` | Override testing tier only (`off`, `fast`, `medium`, `hard`) |
| `--no-test` | Skip testing stage — log as risky |
| `--domains=<csv>` | Override detected domains, e.g. `--domains=db,backend` |
| `--dry-run` | Run through merge, stop before execution |
| `--auto` | Auto-select recommended option for every question |
| `--text` | Plain-text numbered questions (no `AskUserQuestion` tool) |

Strip recognized flags from $ARGUMENTS before passing the remainder to later stages. Store parsed flags as `$FLAGS`.
</step>

<step name="init_task">
**Create or resume a task folder.**

```bash
TASK_ROOT=".feller/tasks"
TODAY=$(date +%Y-%m-%d)
```

Derive a slug from the first 4-6 meaningful words of the request (lowercase, hyphens, no punctuation). Task folder: `$TASK_ROOT/$TODAY-$SLUG/`.

If the folder already exists:
- Read `state.json`
- If `lock.pid` is alive and `status == running` → refuse, tell the user a task is already active
- If stale lock or `status == paused` → resume from the recorded `stage`

If new:
- Create the folder structure:
  ```
  .feller/tasks/<task>/
    intent.md
    glossary.md
    research/
    plans/
    state.json
    scratchpad.md
  ```
- Write `state.json` with `{stage: "triage", lock: {pid, started_at}, status: "running"}`
- Write `.feller/active` with the task folder name
- Write the original request (with flags stripped) to `intent.md`
</step>

<step name="triage">
**Classify size and detect domains.**

If a size flag (`--hard`, `--medium`, `--light`) is in `$FLAGS`, skip inference — use the forced size.

Otherwise infer from the request text:

| Size | Signals |
|------|---------|
| Size | Hard rubric (measurable, not vibes) |
|------|-------------------------------------|
| trivial | ≤1 file, ≤5 lines changed, no new concepts |
| small | ≤3 files, 1 layer, scope fully clear from the sentence |
| medium | ≤10 files, ≤2 layers, localized refactor or minor new feature |
| large | >10 files, OR multi-layer, OR new capability, OR migration, OR architectural change |

Apply the **first matching** rule. When the request straddles tiers, round UP to the larger tier — false positives on size are cheaper than false negatives. Source: GSD `fast.md` (measurable gate beats qualitative description).

Detect domains by keyword + project-structure scan:

| Keyword / path signal | Domain |
|-----------------------|--------|
| "schema", "table", "migration", "RLS", "pgvector", `migrations/` | `db` |
| "endpoint", "service", "controller", "nestjs", `maestro-nestjs/` | `backend` |
| "component", "page", "react", "vite", `maestro-terminal/`, `maestro-ui/` | `frontend-web` |
| "expo", "mobile", "native", `maestro-corretor/` | `frontend-mobile` |
| "whisper", "embedding", "transcript", "insight" | `ai-pipeline` |

Set dials from size:

| Dial | trivial | small | medium | large |
|------|---------|-------|--------|-------|
| `interview_depth` | 0 | 0-1 | 2-3 | full |
| `research` | off | light | light | hard |
| `swarm_rounds` | 0 | 0 | 2 | 3 |
| `testing_tier` | off | fast | medium | hard |
| `verification` | build | build | full | full |

Apply flag overrides AFTER size-based defaults:
- `--test=<tier>` → override `testing_tier` only
- `--no-test` → `testing_tier=off`, set `warn_no_test=true`
- `--domains=<csv>` → replace detected domains

Store the full dial state as `$DIALS`. Persist to `state.json`.
</step>

<step name="present_triage">
**Show the triage decision. Require confirmation.**

Banner:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 FELLER ► TRIAGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Request:  {first 100 chars of cleaned $ARGUMENTS}
 Size:     {$DIALS.size}
 Domains:  {$DIALS.domains}
 Plan:
   - Interview:    {interview_depth}
   - Research:     {research}
   - Swarm rounds: {swarm_rounds}
   - Testing:      {testing_tier}
   - Verification: {verification}
```

If `--auto` in `$FLAGS`: log `[auto] Triage confirmed.` and continue.

Otherwise, use `AskUserQuestion` (or plain-text fallback if `--text` is set):

- **header:** "Triage"
- **question:** "Does this read correctly?"
- **options:**
  - **Proceed** — Run with these settings
  - **Change size** — Pick a different size
  - **Change domains** — Add or remove domains
  - **Force hard mode** — Run everything at max
  - **Cancel** — Stop here

Handle each response:
- `Proceed` → continue to `interview`
- `Change size` → ask which size (trivial/small/medium/large), recalculate dials, return to `present_triage`
- `Change domains` → ask which domains (multi-select), recalculate, return to `present_triage`
- `Force hard mode` → set `$DIALS` to large defaults, continue
- `Cancel` → mark state `cancelled`, exit
- Empty answer → retry once, then fall back to text-mode numbered list. Never proceed on empty.
</step>

<step name="interview">
**Coverage-driven interview. One question at a time. Hard cap 5.**

Skip entirely if `interview_depth == 0`.

This step is INLINE, not a subagent. Verdict from prior-art research: `AskUserQuestion` doesn't work cleanly across subagent boundaries, and the interview needs to mutate `intent.md` as it runs — both break under subagent dispatch. (Sources: spec-kit `/speckit.clarify`, Superpowers `brainstorming`, GSD `discuss-phase` — all three are inline.)

### Phase A — Prior-decisions load (cheap, high-leverage)

Before asking anything:
1. Read the last 3 completed `.feller/tasks/*/intent.md` files (if any exist)
2. Read project-level context: `CLAUDE.md`, `.planning/PROJECT.md`, `.planning/roadmap.md` (if any)
3. Extract decisions that already answer plausible questions for this request
4. Annotate your internal coverage scan: "already decided in prior task X" → marked Clear

Source: GSD `discuss-phase.md` — "These are pre-answered — don't re-ask unless this phase has conflicting needs."

### Phase B — Coverage scan (9 categories, Clear / Partial / Missing)

Scan the request against these 9 categories. Mark each as **Clear**, **Partial**, or **Missing**. Source: github/spec-kit `/speckit.clarify`.

1. **Functional scope** — what behavior changes, what stays
2. **Domain & data model** — entities, fields, relationships touched
3. **Interaction / UX** — user-visible changes, if any
4. **Non-functional** — performance, cost, latency, scale constraints
5. **Integration** — external services, APIs, tools touched
6. **Edge cases** — failure modes, empty states, concurrency
7. **Constraints** — hard requirements from project CLAUDE.md, compliance, deadlines
8. **Terminology** — naming conventions, preferred vocabulary (feeds `glossary.md` later)
9. **Completion signals** — how you and the user will know it's done

### Phase C — Question loop

Only ask about **Missing** categories. Rules (strict):

- **ONE question per `AskUserQuestion` call.** Never batch.
- **Never reveal future queued questions.**
- **Hard cap: 5 questions total.** Do not ask a 6th, regardless of `interview_depth`. Source: spec-kit research — 5 is the absolute ceiling; feller's earlier "up to 6 for large" was over the line.
- **Soft cap by dial:** `small=1`, `medium=3`, `large=5`. Stop early when zero Missing remain, even under the soft cap.
- **Self-check before every question:** "do I have enough to plan?" If yes, end the loop.
- **Early termination verbs** — if the user replies with any of: `"proceed"`, `"done"`, `"stop"`, `"no more"`, `"enough"`, `"just do it"` → end the loop immediately and continue to planning. Treat as explicit authorization.
- **Each question** must be: header + one-sentence question + 3-5 options with short descriptions. Mark the recommended option in bold. The AskUserQuestion tool provides free-text fallback automatically — no need to add "Other."

### Phase D — Scope guardrail (block scope creep)

If the user's answer would expand scope beyond the triaged domains or beyond the roadmap, use this response verbatim:

> *"[Requested expansion] would be a new capability beyond this task's scope. Want me to note it for the backlog so a future task picks it up, or do you want to force this task larger?"*

Source: GSD `discuss-phase.md` — "Discussion clarifies HOW to implement what's scoped, never WHETHER to add new capabilities."

### Phase E — Write back

Append every Q&A to `intent.md` as it happens (not at the end). Each entry:

```
## Q<n>: <question>
- Category: <one of the 9>
- Options shown: <list>
- Chose: <user's answer>
- Rationale: <if user provided>
```

### Optional — cold-eyes sanity check

After the interview ends, if `size == large` or `--hard` was forced: dispatch `feller-reviewer` in a one-shot "interview-sanity-check" mode with only `intent.md` as input. It flags internal contradictions between answers. The review is advisory — if it finds conflicts, bounce back to Phase C with one targeted question per conflict (these count against the 5-question cap).

### `--auto` behavior

For each Missing category, select the bolded recommended option without prompting. Log inline: `[auto] Q<n> <category>: auto-selected "<option>"`.
</step>

<step name="research">
**Run research per the `research` dial.**

Skip if `research == off`.

**`light`:** Quick single-pass search for prior art and existing libraries that solve ≥80% of the problem. Write findings to `research/pre-plan.md` before dispatching planners.

**`hard`:** Two-phase research.
1. Pre-plan pass (like light) → `research/pre-plan.md`
2. After planners produce drafts, dispatch `feller-researcher` with all draft plans. It identifies unknowns across plans, researches each, writes `research/post-plan.md`. Planners re-read before merge.

**Prior-art forcing rule** (applies to both modes, enforced in planner prompts):

> Before proposing any net-new implementation, document: (1) the 2-3 existing solutions you found, (2) which one comes closest, (3) the specific reason it doesn't fit. If you cannot name 2-3 candidates, your research was insufficient — escalate.

**STATUS: STUB.** First version: `light` mode only, inline `WebSearch`. Hard mode comes after the basic loop works end-to-end.
</step>

<step name="glossary">
**Write a shared vocabulary before planners diverge.**

Skip if only one planner will run.

Write `glossary.md` with up to 5 entries covering:
- Entity names (e.g., `Lead` vs `Prospect` — pick one)
- API surface names (endpoint paths, function names that cross domains)
- Key types/interfaces that multiple layers consume

Every planner prompt must include: "Reference names in glossary.md verbatim. Do not invent alternatives."
</step>

<step name="plan">
**Dispatch one planner per active domain.**

Skip if `size == trivial` — trivial tasks go straight to `execute`.

Swarm behavior based on `swarm_rounds`:

- **0** → Sequential handoff. Order by dependency: `db` → `backend` → `frontend-*` → `ai-pipeline`. Each planner sees the previous plans.
- **2** → Parallel dispatch. Round 1: all planners write drafts in parallel. Round 2: coordinator passes concatenated drafts back via `SendMessage`, each refines.
- **3** → Parallel dispatch, up to 3 rounds. Stop early if ≥80% of a plan is unchanged between rounds.

Each planner writes to `plans/<domain>.md`.

**Dispatch:** Pass file paths, not content. Each planner prompt includes:
- Path to `intent.md`
- Path to `glossary.md` (if exists)
- Path to `research/pre-plan.md` (if exists)
- Paths to earlier plans (for `swarm_rounds=0` sequential mode)
- Target output path: `plans/<domain>.md`

**Routing:** Parse the planner's return signal per `<handoff_protocol>`:
- `PLAN_READY` → proceed to next planner or to merge
- `BLOCKED` / `NEEDS_CONTEXT` → handle (supply missing info, ask user, or abort)
- Any `[HIGH]` signal → pause, read the referenced file, decide whether to continue

**STATUS: STUB.** First working version forces `swarm_rounds=0` and ships one planner agent: `feller-planner-backend`. Add `feller-planner-db`, `feller-planner-frontend-web`, etc. after the sequential path works end-to-end.
</step>

<step name="merge">
**Reconcile plans into one coherent plan.**

Skip if only one planner ran — promote the single plan directly to `merged-plan.md`.

**Dispatch:** Pass file paths only — the merger reads from disk. Merger prompt includes:
- Path to `plans/` directory (it reads all plans)
- Path to `glossary.md` (if exists)
- Path to `intent.md`
- Path to `research/` directory (if exists)

The merger detects conflicts, reconciles, orders by dependency, and writes `merged-plan.md` with: Overview, Full plan, Decisions I Made, Risks, and TL;DR (at the END per user preference).

**Routing:** Parse the merger's return signal per `<handoff_protocol>`:
- `MERGED` → present `merged-plan.md` to the user (read it for presentation, this is the exception). Require explicit proceed before execution.
- `CONFLICTS_UNRESOLVED` → read the referenced file, present unresolved items to the user
- `BLOCKED` → handle (revise plans, ask user, or abort)

`--auto` proceeds automatically but still shows the plan. If `--dry-run`, stop here with a clean exit message.

**STATUS: STUB.**
</step>

<step name="execute">
**Run executors per domain in the dependency order from `merged-plan.md`.**

For each domain:

1. **Dispatch executor** — pass file paths, not content:
   - Path to `merged-plan.md` (or the domain's plan slice)
   - Path to `glossary.md` (if exists)
   - Path to domain role file (`feller/roles/<domain>.md`) if it exists
   - Path to `scope.txt` (orchestrator writes this with allowed file paths)
   - Task directory path for `scratchpad.md` and `files-changed.txt`

2. **Route on executor return signal** per `<handoff_protocol>`:
   - `READY_FOR_REVIEW` → dispatch verifier
   - `DONE_WITH_CONCERNS` → dispatch verifier (concerns noted for reviewer)
   - `BLOCKED` / `NEEDS_CONTEXT` → handle (supply info via `SendMessage` with file paths, or escalate to merger/user)
   - `VERIFICATION_FAILED` → dispatch verifier to confirm, then `SendMessage` executor with path to `verification-report.md`

3. **Dispatch verifier** — pass task directory path. Verifier discovers commands, runs gates, writes `verification-report.md`.

4. **Route on verifier return signal:**
   - `GREEN` → dispatch reviewer
   - `YELLOW` → check SIGNALS for `[STRUCTURAL]`. If present and no `structural_gate` preference in `.feller/config.json`, ask user: "Structural metrics found violations. Hard gate (block on violations) or advisory (flag and continue)?" Save their answer to `.feller/config.json`. If hard gate and structural violations → treat as RED. Otherwise → dispatch reviewer.
   - `RED` → `SendMessage` executor: "Verification failed. Read `.feller/tasks/<task>/verification-report.md` for details." Loop until green or budget exhausted.
   - `PARTIAL` → log warning, dispatch reviewer with caveat

5. **Dispatch reviewer** — pass ONLY: path to `merged-plan.md`, `base_sha`, `head_sha`, `git diff` output. Cold eyes — no scratchpad, no executor history.

6. **Route on reviewer return signal:**
   - `PASS` / `PASS_WITH_NITS` → mark domain complete in `state.json`, next domain
   - `FAIL` → `SendMessage` executor: "Review failed. Read `.feller/tasks/<task>/review-report.md` for details." Loop.
   - `NEEDS_HUMAN` → pause, ask user to verify manually

**STATUS: STUB.** First working version: one executor, no reviewer (add cold-eyes reviewer in the second iteration). Ship the loop without the review gate first, then add review.
</step>

<step name="test">
**Run the testing stage per `testing_tier`.**

Skip if `testing_tier == off`. If `warn_no_test` is set, log a visible red warning: `⚠ Testing stage skipped via --no-test. Verify manually before shipping.`

**`fast`:** `superpowers:verification-before-completion` pass only. Build + existing tests. Paste output. Done.

**`medium`:**
1. `gsd-verifier`-style goal-backward check against `merged-plan.md`
2. `gsd-nyquist-auditor`-style test-gap fill for changed files
3. Build + full test suite with verbatim output

**`hard`:** everything in `medium`, plus:
4. `gsd:add-tests`-style classification pass (TDD / E2E / Skip per changed file)
5. `everything-claude-code:e2e-runner` for any UI paths touched
6. Integration smoke: spin stack, hit critical paths end-to-end
7. Regression sweep: full existing suite
8. Cross-domain check: each side actually sees the other's changes (types match, DB migration applied, frontend reads new field)
9. **Cold-eyes goal-backward audit:** fresh agent with ONLY the original `intent.md` and the final diff. Answers: "does this actually do what was asked?"

Always produce a **test-gap report** at `tests-gap.md`:
- Scenarios that could not be automated cheaply
- Explicit "please manually verify" items
- Honest admission > fake green

**STATUS: STUB.**
</step>

<step name="finalize">
**Close out the task.**

1. Update `state.json`: `status=complete`, clear `lock`
2. Clear `.feller/active`
3. Write `SUMMARY.md`:
   - **What shipped** — 1-3 bullets
   - **Test results** — pass/fail counts, gaps
   - **Deviations** — anything that drifted from the plan
   - **Manual verification needed** — from test-gap report
4. Present the summary to the user. Stop.
</step>

</process>

<success_criteria>
- [ ] Flags parsed and applied before triage inference
- [ ] Task folder created with durable state before any stage begins
- [ ] Triage presented and confirmed (or auto-confirmed with log)
- [ ] Interview questions interleaved, one at a time, capped by `interview_depth`
- [ ] Research ran in the mode set by the dial; prior-art forcing rule enforced in planner prompts
- [ ] Glossary written before planners diverged (when >1 planner)
- [ ] Plans reconciled by the merger with `Decisions I Made` and `Risks` sections
- [ ] Merged plan presented full-then-TL;DR; explicit proceed required before execute
- [ ] Executor → verifier → reviewer ran per domain with cold-eyes separation
- [ ] Testing tier ran (or was explicitly disabled with a visible warning)
- [ ] Test-gap report produced
- [ ] SUMMARY.md, state.json, and scratchpad survive context compaction
</success_criteria>

<state_shape>
```
.feller/
  active                            # one-line: current task folder name
  tasks/
    YYYY-MM-DD-<slug>/
      intent.md                     # original request + interview Q&A
      glossary.md                   # shared vocabulary
      research/
        pre-plan.md                 # light research output
        post-plan.md                # hard-mode research (after drafts)
      plans/
        db.md
        backend.md
        frontend-web.md
        frontend-mobile.md
        ai-pipeline.md
      merged-plan.md                # merger output
      scratchpad.md                 # executor working notes (live)
      tests-gap.md                  # honest manual-verification list
      SUMMARY.md                    # final record
      state.json                    # {stage, dials, domains, lock, status}
```

`state.json` minimum fields:
- `stage` — current stage name
- `dials` — the triage dials object
- `domains` — active domains array
- `flags` — parsed flags
- `lock` — `{pid, started_at}` for collision detection
- `status` — `running` | `paused` | `complete` | `cancelled` | `failed`
</state_shape>

<notes_for_future_iterations>
**What's stubbed in the first build:**
- `interview` — one hardcoded question, no real interleaving logic yet
- `research` — light mode only, inline `WebSearch`, no `feller-researcher` agent yet
- `plan` — forces `swarm_rounds=0`, ships only `feller-planner-backend`
- `merge` — stubbed; start with single-plan passthrough
- `execute` — ships without the cold-eyes reviewer; add it in iteration 2
- `test` — stubbed; start with `fast` tier only, grow to medium, then hard

**Build order:** parse_flags → init_task → triage → present_triage → (stubbed interview) → (stubbed research) → (stubbed plan) → execute → (stubbed test) → finalize. Run ONE real task through this end-to-end before building any other stage out.

**What to add later (in order):**
1. Cold-eyes `feller-reviewer` agent
2. `feller-planner-db` agent + glossary step wired up
3. `feller-merger` agent
4. `medium` testing tier composition
5. `feller-planner-frontend-web` agent
6. Swarm rounds > 0 in `plan`
7. `hard` testing tier with cross-domain checks
8. `feller-researcher` agent (hard-mode research)
9. The `using-feller` skill for natural auto-activation
</notes_for_future_iterations>
