<purpose>
The refactor path restructures existing code without changing its observable behavior. Invoked when scout verdict's suggested_path is `refactor`, or user explicitly requests "refactor / rename / extract / move X".

Fused scan+plan+execute agent. Core discipline: tests must be green BEFORE and AFTER — no new features, no behavior changes. If tests don't exist for the refactor surface, this path REFUSES and escalates to feller:test first.
</purpose>

<required_reading>
Read files referenced by the invoking prompt's execution_context.
</required_reading>

<handoff_protocol>
See `@$HOME/.claude/feller/workflows/do.md` for the handoff protocol.
</handoff_protocol>

<process>

<step name="read_verdict">
Read `.feller/tasks/<task>/verdict.json` to extract `domain`. Verify `scope.txt` exists.

Refactor scope can legitimately be broader than fast path — 10+ files is common for a rename or extract-interface refactor. That's fine. The guardrail is behavior invariance, not line count.
</step>

<step name="pre_verify">
**Before dispatching the agent, capture a baseline.**

Run the verifier with task directory path. It records current build / typecheck / test / lint status to `.feller/tasks/<task>/baseline-verification.md`.

- If baseline is RED (build broken, tests failing) → do NOT refactor. Present the failures and ask the user: "Baseline is red. Refactor is unsafe without green baseline. Fix first via feller:bug, or cancel?"
- If baseline is GREEN or YELLOW → proceed. The agent must match or beat this baseline at the end.

Check test coverage for the refactor surface. If the scope files have <30% coverage (heuristic: less than N tests importing them), warn: "Refactor surface has thin test coverage. Behavior preservation cannot be verified. Add tests via feller:test first, or proceed at your own risk?"
</step>

<step name="dispatch_fused_refactor_agent">
**Dispatch ONE fused refactor agent based on domain:**

| Domain | Agent |
|--------|-------|
| `frontend` | `feller-refactor-frontend` |
| `backend` | `feller-refactor-backend` |
| `db` | `feller-refactor-db` |

Prompt the agent with:
- Path to `.feller/tasks/<task>/verdict.json`
- Path to `.feller/tasks/<task>/intent.md`
- Path to `.feller/tasks/<task>/scope.txt`
- Path to `.feller/tasks/<task>/baseline-verification.md` (the bar to match)
- Path to task dir

Agent does: scan → write refactor plan (5-line intent note + move list) → execute → run suite → iterate until green matches baseline.
</step>

<step name="route_on_agent_status">
- `READY_FOR_REVIEW` → proceed to verify
- `BASELINE_REGRESSED` → refactor broke something. SendMessage agent once: "Tests regressed vs baseline. Read baseline-verification.md vs current output and revert or fix." If still regressed, escalate.
- `BLOCKED` with `blocker_type: SCOPE_MISROUTED` → upgrade to feature path (refactor turned out to require a behavior change).
- `BLOCKED` with `blocker_type: INSUFFICIENT_TESTS` → upgrade to feller:test (refactor needs more test coverage to be safe).
- Other blockers → present to user.
</step>

<step name="verify">
Dispatch `feller-verifier`. It must match or beat the baseline captured in `baseline-verification.md` (same or fewer test failures, same or fewer lint warnings, same or better build status).

- `GREEN` + matches/beats baseline → proceed to review
- `YELLOW` + matches/beats baseline → proceed
- Regression vs baseline → RED. SendMessage agent. Loop once. If still regressed, escalate.
- `YELLOW` with `[STRUCTURAL]` → GOOD sign for refactor path (structural metrics should IMPROVE). Check `.feller/config.json` gate preference and proceed if advisory, treat as RED if hard gate.
</step>

<step name="review">
Dispatch `feller-reviewer` with:
- Path to `.feller/tasks/<task>/intent-note.md` (refactor scope + rationale)
- Path to `.feller/tasks/<task>/baseline-verification.md` (what "same behavior" means)
- `base_sha`, `head_sha`, `git_diff_text`

Reviewer checks all 6 dimensions PLUS refactor-specific checks:
- Did any PUBLIC API surface change? (Breaking change should be explicit in intent-note, not incidental.)
- Did module boundary health improve (or at minimum not regress)?
- Are there now tests for behavior that was previously implicit?

- `PASS` / `PASS_WITH_NITS` → finalize
- `FAIL` → SendMessage fused agent, loop once.
- `NEEDS_HUMAN` → pause.
</step>

<step name="finalize">
1. Update `state.json`: `status=complete`
2. Clear `.feller/active`
3. Write `SUMMARY.md`:
   - **Refactor** — from intent-note.md
   - **Baseline** — green/yellow → green/yellow (or structural-improvement-noted)
   - **Files touched** — from files-changed.txt
   - **Behavior change** — "none" (or explicit list if any API was intentionally changed with user approval)
   - **Verification result** — from verification-report.md
   - **Review result** — from review-report.md
   - **Path taken** — "Refactor path (fused scan+plan+execute agent)"
4. Present summary. Stop.
</step>

</process>

<success_criteria>
- [ ] Baseline captured BEFORE the refactor — no-op if baseline is red
- [ ] Exactly ONE fused refactor agent dispatched
- [ ] Thin-coverage warning surfaced when refactor surface has few tests
- [ ] Verifier result matches or beats baseline (no regression allowed)
- [ ] Reviewer flags any incidental public-API change
- [ ] On INSUFFICIENT_TESTS, path upgrades cleanly to feller:test
- [ ] SUMMARY.md and state.json survive context compaction
</success_criteria>
