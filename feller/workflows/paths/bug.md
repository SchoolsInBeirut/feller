<purpose>
The bug path runs a fused diagnose+fix agent on a single-domain bug. Invoked when scout verdict includes evidence of a failure mode (error, failing test, broken behavior) and scope is bounded.

Differs from fast path:
- Adds a REPRODUCE step before editing (confirms the bug is real and observable)
- Agent's intent note includes a "Bug hypothesis" line
- Verification explicitly confirms the reproducer now passes

Still a fused agent — no separate planner, no swarm.
</purpose>

<required_reading>
Read files referenced by the invoking prompt's execution_context.
</required_reading>

<handoff_protocol>
See `@$HOME/.claude/feller/workflows/do.md` for the handoff protocol.
</handoff_protocol>

<process>

<step name="read_verdict">
Read `.feller/tasks/<task>/verdict.json` to extract `domain` and confirm evidence includes a failure mode.

Verify `scope.txt` exists and is non-empty. If missing, log error and return to router.
</step>

<step name="dispatch_fused_bug_agent">
**Dispatch ONE fused bug agent based on verdict domain:**

| Domain | Agent |
|--------|-------|
| `frontend` | `feller-bug-frontend` |
| `backend` | `feller-bug-backend` |
| `db` | `feller-bug-db` |

Prompt the agent with:
- Path to `.feller/tasks/<task>/verdict.json`
- Path to `.feller/tasks/<task>/intent.md`
- Path to `.feller/tasks/<task>/scope.txt`
- Path to `.feller/tasks/<task>/` (task directory for writing intent-note.md, reproducer.md, files-changed.txt)

The agent does: read → hypothesize → reproduce → write intent note → fix → verify reproducer + suite → return status signal.
</step>

<step name="route_on_agent_status">
Parse the fused agent's return signal:

- `READY_FOR_REVIEW` → proceed to verify
- `CANNOT_REPRODUCE` → bug may be stale, already fixed, or environment-specific. Present `reproducer.md` to user; ask: close as not-a-bug / expand scope (upgrade to feature path) / cancel.
- `VERIFICATION_FAILED` → SendMessage agent once: "Reproducer still fails. Read verification-report.md and attempt one fix." If still fails, escalate.
- `BLOCKED` with `blocker_type: SCOPE_MISROUTED` → upgrade to feature path. Update `state.json.route.chosen_path = "feature"` and dispatch feature workflow.
- `BLOCKED` with `blocker_type: ROOT_CAUSE_UNCLEAR` → present the agent's hypotheses to user, ask which to pursue or escalate to feature path (needs proper planning).
- `BLOCKED` with other blocker types → present `what_you_need` to user.
</step>

<step name="verify">
Dispatch `feller-verifier` with task directory path. The verifier runs build + typecheck + lint + test suite + structural metrics.

Bug path has an EXTRA check: the agent's reproducer must now pass. The verifier reads `reproducer.md` to locate the specific test / command that reproduced the bug and asserts it now returns green. If it doesn't, the verdict is RED regardless of the rest of the suite.

- `GREEN` → proceed to review
- `YELLOW` without `[STRUCTURAL]` → proceed to review
- `YELLOW` with `[STRUCTURAL]` → same gate logic as fast path (check `.feller/config.json` for `structural_gate` preference; ask if unset)
- `RED` → SendMessage fused agent once: "Verification failed. Read verification-report.md." Loop once. If still RED, escalate.
- `PARTIAL` → log warning, dispatch reviewer with caveat
</step>

<step name="review">
Dispatch `feller-reviewer` with cold-eyes contract:
- Path to `.feller/tasks/<task>/intent-note.md` (the 5-line contract including bug hypothesis)
- Path to `.feller/tasks/<task>/reproducer.md` (what failure looked like and what proves it's fixed)
- `base_sha`, `head_sha`, `git_diff_text`

Reviewer checks all 6 dimensions plus a bug-specific check: does the diff address the stated hypothesis, or did the agent do something else?

- `PASS` / `PASS_WITH_NITS` → finalize
- `FAIL` → SendMessage fused agent: "Review failed. Read review-report.md." Loop once. If still FAIL, escalate.
- `NEEDS_HUMAN` → pause, ask user to verify manually.
</step>

<step name="finalize">
1. Update `state.json`: `status=complete`, clear `lock`
2. Clear `.feller/active`
3. Write `SUMMARY.md`:
   - **Bug** — from intent-note.md "Goal" + "Bug hypothesis"
   - **Fix** — from intent-note.md "Approach"
   - **Reproducer** — from reproducer.md (was failing → now passing)
   - **Verification result** — from verification-report.md
   - **Review result** — from review-report.md
   - **Path taken** — "Bug path (fused diagnose+fix agent)"
4. Present summary. Stop.
</step>

</process>

<success_criteria>
- [ ] `scope.txt` verified before dispatching fused agent
- [ ] Exactly ONE fused bug agent dispatched (not all three)
- [ ] Reproducer captured BEFORE the fix; verifier confirms reproducer now passes
- [ ] On SCOPE_MISROUTED or ROOT_CAUSE_UNCLEAR, path upgrades to feature without data loss
- [ ] Reviewer receives both intent-note.md and reproducer.md as context
- [ ] SUMMARY.md and state.json survive context compaction
</success_criteria>
