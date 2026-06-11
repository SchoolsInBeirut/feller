<purpose>
The fast path executes trivial single-domain changes. Invoked by the router when scout confirms:
- scope ≤ 5 lines across ≤ 2 files
- single domain (frontend / backend / db)
- no new logic beyond the trivial change

Zero planning ceremony. One fused agent plans+executes. Reviewer and verifier still run because quality gates don't skip for small changes.
</purpose>

<required_reading>
Read files referenced by the invoking prompt's execution_context.
</required_reading>

<handoff_protocol>
See `@$HOME/.claude/feller/workflows/do.md` for the full handoff protocol. Same rules apply:
- Agents return status signals, not content
- Agent-to-agent data flows through files on disk
- Orchestrator passes file paths, not content
</handoff_protocol>

<process>

<step name="read_verdict">
**Read `.feller/tasks/<task>/verdict.json` to extract:**
- `domain` — picks which fused agent to dispatch
- `evidence` — confirms scope.txt was written correctly by the router

Verify `.feller/tasks/<task>/scope.txt` exists and is non-empty. If missing, log error and return to router.
</step>

<step name="dispatch_fused_agent">
**Dispatch ONE fused agent based on verdict domain:**

| Domain | Agent |
|--------|-------|
| `frontend` | `feller-fast-frontend` |
| `backend` | `feller-fast-backend` |
| `db` | `feller-fast-db` |

Prompt the agent with:
- Path to `.feller/tasks/<task>/verdict.json`
- Path to `.feller/tasks/<task>/intent.md`
- Path to `.feller/tasks/<task>/scope.txt`
- Path to `.feller/tasks/<task>/` (task directory for writing intent-note.md, files-changed.txt)

The agent does: read → intent note → edit → basic verification → return status signal.
</step>

<step name="route_on_agent_status">
Parse the fused agent's return signal:

- `READY_FOR_REVIEW` → proceed to verify step
- `VERIFICATION_FAILED` → SendMessage agent with one retry instruction: "Read verification output and attempt a single fix." If still fails, escalate to user.
- `BLOCKED` with `blocker_type: SCOPE_MISROUTED` → the scout misrouted. Log: `⚠ Fast path scope exceeded. Routing up to feller:do.` Update `state.json.route.chosen_path = "feature"` and dispatch `@$HOME/.claude/feller/workflows/do.md`.
- `BLOCKED` with other blocker types → present to user with `what_you_need` from SIGNALS, ask how to proceed.
</step>

<step name="verify">
**Dispatch `feller-verifier` with task directory path.**

The verifier runs all gates (build → typecheck → lint → tests → structural metrics), writes `verification-report.md`, returns status.

- `GREEN` → proceed to review
- `YELLOW` without `[STRUCTURAL]` → proceed to review (minor warnings acceptable)
- `YELLOW` with `[STRUCTURAL]` → check `.feller/config.json` for `structural_gate` preference. If unset, ask user: "Structural metrics found violations. Hard gate (block) or advisory (flag and continue)?" Save preference. If hard → treat as RED.
- `RED` → SendMessage fused agent: "Verification failed. Read `.feller/tasks/<task>/verification-report.md` for details." Loop once. If still RED, escalate to user.
- `PARTIAL` → log warning, dispatch reviewer with caveat
</step>

<step name="review">
**Dispatch `feller-reviewer` with cold-eyes contract:**
- Path to `.feller/tasks/<task>/intent-note.md` (the 5-line contract — reviewer compares diff against this)
- `base_sha` (git SHA before fused agent ran)
- `head_sha` (git SHA after)
- `git_diff_text` (inline `git diff base_sha head_sha` output)

Reviewer checks all 6 dimensions (drift, wiring, correctness, plan-caused bugs, security, module boundary health).

- `PASS` / `PASS_WITH_NITS` → proceed to finalize
- `FAIL` → SendMessage fused agent: "Review failed. Read `.feller/tasks/<task>/review-report.md` for details." Loop once. If still FAIL, escalate to user.
- `NEEDS_HUMAN` → pause, ask user to verify manually
</step>

<step name="finalize">
**Close out the task.**

1. Update `state.json`: `status=complete`, clear `lock`
2. Clear `.feller/active`
3. Write `SUMMARY.md`:
   - **What shipped** — from `intent-note.md` + `files-changed.txt`
   - **Verification result** — from verification-report.md status
   - **Review result** — from review-report.md status
   - **Path taken** — "Fast path (fused agent, no planning, no merge)"
4. Present summary to the user. Stop.
</step>

</process>

<success_criteria>
- [ ] `scope.txt` verified before dispatching fused agent
- [ ] Exactly ONE fused agent dispatched (not all three)
- [ ] On SCOPE_MISROUTED, router re-routes to feature path without data loss
- [ ] Verifier runs all gates including structural metrics
- [ ] Reviewer compares diff against intent-note.md (the contract)
- [ ] SUMMARY.md and state.json survive context compaction
</success_criteria>
