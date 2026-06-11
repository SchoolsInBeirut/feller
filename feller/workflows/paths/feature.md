<purpose>
The feature path runs the full pipeline (research → plan → merge → execute → test) for non-trivial, potentially cross-domain work. It wraps `do.md` so the router's hard-earned context (intent.md, verdict.json, scope.txt) is preserved rather than re-derived.

Invoked by the router when scout verdict is any of:
- `suggested_path: do`
- `estimated_scope.cross_domain: true`
- `domain: multiple`
- `estimated_scope.lines_rough > ~50` or `.files > 10`
- Low confidence + user override to feature path
</purpose>

<required_reading>
Read files referenced by the invoking prompt's execution_context.
</required_reading>

<handoff_protocol>
See `@$HOME/.claude/feller/workflows/do.md` for the handoff protocol. Same rules apply.
</handoff_protocol>

<process>

<step name="seed_from_router">
**Preserve the router's work. Do NOT re-run triage or interview.**

The router has already:
- Created the task folder (`.feller/active` points at it)
- Written `intent.md` (request + intent-only interview answers)
- Written `scout-scratchpad.md`, `verdict.json`, `scout-report.md`
- Written `scope.txt` (allowed paths from scout evidence)
- Written `state.json` with `stage: "triage"` and `route.chosen_path: "feature"`

Read `verdict.json` to seed dials for the legacy pipeline:

| Verdict field | Dial |
|---------------|------|
| `estimated_scope.files ≤ 3` AND `lines ≤ 30` | `size: small` |
| `estimated_scope.files ≤ 10` AND `lines ≤ 150` | `size: medium` |
| Anything larger, OR `cross_domain: true`, OR `domain: multiple` | `size: large` |
| `domain: "multiple"` | set `domains` from scout's `evidence[].file` path prefixes |
| Otherwise | set `domains` to the single verdict domain |

Apply user flag overrides (`--hard`, `--medium`, `--light`, `--domains=...`) from `$FLAGS` AFTER these defaults. Router-level flags propagate through `state.json.flags`.

Update `state.json`:
- `dials` — computed from verdict + flag overrides
- `domains` — as above
- `stage: "present_triage"` (skip `triage` and `init_task` — already done)

Log one line: `Feature path seeded from scout: size=<N>, domains=<list>. Dispatching legacy pipeline.`
</step>

<step name="present_triage_confirmation">
**Show the user the dial derivation before running the expensive pipeline.**

Unless `--auto` is set, require explicit confirmation. Banner:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 FELLER ► FEATURE PATH (seeded from scout)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Scout scope:  <N> files, ~<M> lines, <domain>
 Derived size: <size>
 Domains:      <list>
 Plan:
   - Research:     <research dial>
   - Swarm rounds: <swarm_rounds>
   - Testing:      <testing_tier>
   - Verification: <verification>
```

Use `AskUserQuestion` (or plain text if `--text` is set):
- **Proceed** — run legacy pipeline with these dials
- **Change size** — pick different size (recalculate dials)
- **Cancel** — stop here

On `--auto`: skip the question, log `[auto] Feature path dials confirmed.`
</step>

<step name="skip_interview_if_covered">
**Router's intent-only interview already wrote to `intent.md`. The legacy interview (9-category coverage scan) runs against the same file.**

Re-evaluate coverage. Categories the router's interview already covered (Goal, Motivation, Constraints, Evidence of need) map onto legacy categories 1, 7, 9 (Functional scope, Constraints, Completion signals).

For large tasks with `size == large`, the legacy interview MAY still need to cover categories 2-6, 8:
- Domain & data model
- Interaction / UX
- Non-functional
- Integration
- Edge cases
- Terminology

Skip the legacy interview if `size ≤ small` — the router's intent interview is sufficient.

Otherwise, run the legacy `interview` step from `do.md` but with a tightened cap: `max_questions = min(soft_cap_by_dial, 5 - questions_already_asked_by_router)`. Track how many questions the router asked by counting `## Q<n>:` entries in `intent.md`.
</step>

<step name="delegate_to_do_md">
**Jump into `do.md` at the `research` stage.**

Invoke `@$HOME/.claude/feller/workflows/do.md` with an instruction to skip the first four stages (`parse_flags`, `init_task`, `triage`, `present_triage`) and start from `research`. All required state is already in `state.json` and on disk.

Concretely: read do.md's `<process>` block, locate the `<step name="research">` element, and execute that step and every subsequent step (`glossary` → `plan` → `merge` → `execute` → `test` → `finalize`) as written.

Do NOT re-invoke `init_task` — the folder exists. Do NOT re-run triage inference — dials are already seeded. Do NOT re-run the router's interview.

Pass `scope.txt` to every executor dispatched by the `execute` step (legacy already does this — just confirm).

Pass `verdict.json` as an additional input to every planner prompt — it gives planners the scout's evidence so they don't re-investigate:

```
Additional context for planning:
- Scout verdict: .feller/tasks/<task>/verdict.json
- Scout evidence anchors the scope. Use it as ground truth for "which files" questions. You may expand the set if the plan requires it, but note expansions explicitly.
```
</step>

</process>

<success_criteria>
- [ ] Router's work (intent.md, verdict.json, scope.txt) preserved — no re-triage, no re-scout
- [ ] Dials seeded from scout verdict, with user flag overrides applied after
- [ ] Interview question budget reduced by questions router already asked
- [ ] Legacy pipeline runs research → plan → merge → execute → test → finalize from do.md
- [ ] Every planner prompt receives verdict.json path for evidence grounding
- [ ] state.json survives context compaction; feature path is resumable
</success_criteria>
