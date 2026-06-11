<purpose>
The test path adds tests to existing code — no production code changes. Invoked when scout verdict's suggested_path is `test`, or user explicitly requests "add tests for X".

Fused agent pattern: one agent does gap analysis → write tests → run them → iterate until green. No separate planner.

Differs from other paths:
- Production code is READ-ONLY. The agent must NOT modify non-test files.
- Reviewer checks that tests fail for valid reasons if the code under test is changed (mutation testing is out of scope — just ensure tests exercise real behavior).
</purpose>

<required_reading>
Read files referenced by the invoking prompt's execution_context.
</required_reading>

<handoff_protocol>
See `@$HOME/.claude/feller/workflows/do.md` for the handoff protocol.
</handoff_protocol>

<process>

<step name="read_verdict">
Read `.feller/tasks/<task>/verdict.json` to extract `domain` (picks which test agent to dispatch).

Verify `scope.txt` exists. For the test path, scope.txt contains:
- Production files under test (READ-ONLY for the agent)
- Test file locations (WRITABLE — may be new files)

If scope.txt doesn't distinguish, that's the router's bug — return to router with error.
</step>

<step name="dispatch_fused_test_agent">
**Dispatch ONE fused test agent based on verdict domain:**

| Domain | Agent |
|--------|-------|
| `frontend` | `feller-test-frontend` |
| `backend` | `feller-test-backend` |
| `db` | `feller-test-db` |

Prompt the agent with:
- Path to `.feller/tasks/<task>/verdict.json`
- Path to `.feller/tasks/<task>/intent.md`
- Path to `.feller/tasks/<task>/scope.txt`
- Path to `.feller/tasks/<task>/` (task dir for writing gap-analysis.md, intent-note.md, files-changed.txt)

Agent does: read target → gap analysis → write intent note → write tests (RED) → run suite → iterate until all new tests pass → return status.
</step>

<step name="route_on_agent_status">
- `READY_FOR_REVIEW` → proceed to verify
- `NO_TESTABLE_GAPS` → present gap-analysis.md to user: "No meaningful gaps found. Existing coverage: N. Close as no-op or expand scope?"
- `VERIFICATION_FAILED` → SendMessage agent once: "New tests still failing. Read verification-report.md." If still fails, escalate.
- `BLOCKED` with `blocker_type: PROD_CODE_CHANGE_REQUIRED` → upgrade to feature path (the agent detected untestable code needing refactor). Update `state.json.route.chosen_path = "feature"` and dispatch feature workflow.
- `BLOCKED` with `blocker_type: SCOPE_MISROUTED` → upgrade to feature path.
- Other blockers → present `what_you_need` to user.
</step>

<step name="verify">
Dispatch `feller-verifier`. In addition to standard gates, verifier runs:
- New tests (from files-changed.txt) — must all pass
- Full suite — must still be green (adding tests must not break existing ones)

- `GREEN` → proceed to review
- `YELLOW` without `[STRUCTURAL]` → proceed
- `YELLOW` with `[STRUCTURAL]` → same gate logic as fast path
- `RED` → SendMessage fused agent once. Loop once. If still RED, escalate.
</step>

<step name="review">
Dispatch `feller-reviewer` with:
- Path to `.feller/tasks/<task>/intent-note.md`
- Path to `.feller/tasks/<task>/gap-analysis.md`
- `base_sha`, `head_sha`, `git_diff_text`

Reviewer checks all 6 dimensions PLUS a test-path-specific check: the diff must ONLY touch test files (or newly-created test fixtures). If prod code changed, that's a FAIL.

- `PASS` / `PASS_WITH_NITS` → finalize
- `FAIL` → SendMessage fused agent, loop once.
- `NEEDS_HUMAN` → pause.
</step>

<step name="finalize">
1. Update `state.json`: `status=complete`
2. Clear `.feller/active`
3. Write `SUMMARY.md`:
   - **Coverage added** — from intent-note.md "Goal" + test count
   - **Gaps filled** — from gap-analysis.md
   - **Test files touched** — from files-changed.txt
   - **Verification result** — from verification-report.md
   - **Review result** — from review-report.md
   - **Path taken** — "Test path (fused gap+write+run agent)"
4. Present summary. Stop.
</step>

</process>

<success_criteria>
- [ ] scope.txt distinguishes read-only prod files from writable test files
- [ ] Exactly ONE fused test agent dispatched
- [ ] Agent writes tests that FAIL before passing (TDD discipline — captured in gap-analysis.md)
- [ ] Verifier confirms new tests pass AND full suite stays green
- [ ] Reviewer FAILS if prod code was modified
- [ ] SUMMARY.md and state.json survive context compaction
</success_criteria>
