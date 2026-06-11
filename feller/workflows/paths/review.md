<purpose>
The review path runs ONLY the cold-eyes reviewer against an existing diff. No execution, no verification. Use cases:
- User wants a second opinion on uncommitted work
- User wants a review of a specific SHA range
- Scout flagged path as `review` (request was "review X", no change intended)

Zero fused agent. Zero planner. Zero executor. The review itself IS the deliverable.
</purpose>

<required_reading>
Read files referenced by the invoking prompt's execution_context.
</required_reading>

<handoff_protocol>
See `@$HOME/.claude/feller/workflows/do.md` for the handoff protocol.
</handoff_protocol>

<process>

<step name="resolve_diff_range">
**Determine what the reviewer should review.**

Read `intent.md` for hints on the review target. In priority order:

1. **Explicit SHA range** — if intent mentions `base_sha..head_sha`, `<commit>`, a branch name, or a PR number, use that.
2. **Staged + unstaged** — if intent says "my current work" / "uncommitted" / "what I just did", use `git diff HEAD` (covers staged + unstaged).
3. **Last N commits** — if intent says "last commit" / "last N commits", resolve to `HEAD~N..HEAD`.
4. **Default** — `git diff HEAD` (current working tree vs HEAD).

Write the resolved range to `.feller/tasks/<task>/review-range.txt`:
```
base_sha=<sha>
head_sha=<sha or WORKING_TREE>
description=<human-readable, e.g., "last 3 commits" or "uncommitted changes">
```

If no diff exists (empty range), abort with a clear message: `No changes to review. Nothing to do.` Mark state `complete` and exit.
</step>

<step name="resolve_review_contract">
**Pick what the reviewer compares against.**

The reviewer needs a "contract" — something describing intent — to check drift against. Options in priority:

1. **`.feller/tasks/<task>/intent.md`** — always exists (router wrote it)
2. If a prior feller task exists for this diff range (match by `base_sha`), use its `merged-plan.md` or `intent-note.md` as the contract.

Write the chosen contract path to `.feller/tasks/<task>/review-contract.txt`.
</step>

<step name="capture_diff">
**Capture the diff to disk so the reviewer reads it once.**

```bash
git diff <base_sha> <head_sha> > .feller/tasks/<task>/review-diff.patch
git diff --name-only <base_sha> <head_sha> > .feller/tasks/<task>/files-changed.txt
```

For `head_sha=WORKING_TREE`, use `git diff HEAD`.

If the diff exceeds 50K lines, warn and ask the user: "Diff is very large (N lines). Reviewer may miss detail. Continue, split by file, or cancel?"
</step>

<step name="dispatch_reviewer">
**Dispatch `feller-reviewer` with cold-eyes contract.**

Pass file paths only:
- Path to `review-contract.txt` (points to intent.md or plan)
- Path to `review-diff.patch`
- Path to `files-changed.txt`
- `base_sha` and `head_sha` (values, not files)

Reviewer checks all 6 dimensions (drift vs contract, wiring, correctness, plan-caused bugs, security, module boundary health).

Parse return signal:
- `PASS` — log, proceed to finalize
- `PASS_WITH_NITS` — log nit count, proceed to finalize
- `FAIL` — log failure reasons, proceed to finalize (this path's job is to REPORT, not to fix)
- `NEEDS_HUMAN` — proceed to finalize with a visible escalation note
</step>

<step name="finalize">
**Close out and present the review.**

1. Update `state.json`: `status=complete`, clear `lock`
2. Clear `.feller/active`
3. Write `SUMMARY.md`:
   - **Verdict** — PASS / PASS_WITH_NITS / FAIL / NEEDS_HUMAN
   - **Range reviewed** — from `review-range.txt`
   - **Files** — count from `files-changed.txt`
   - **Contract** — from `review-contract.txt`
   - **Top findings** — first 5 issues from `review-report.md` (READ this file — reviewer output is the deliverable here, which makes this a documented exception to the no-read rule for this path only)
   - **Full report** — pointer to `review-report.md`
   - **Path taken** — "Review path (reviewer only, no execution)"
4. Present summary + top findings to the user. Stop.
</step>

</process>

<success_criteria>
- [ ] Diff range resolved from intent.md and captured to disk once
- [ ] Review contract selected (intent.md or prior plan) and referenced, not guessed at by reviewer
- [ ] Reviewer dispatched with cold-eyes inputs (contract path, diff path, files list, SHAs)
- [ ] Review path NEVER edits files or runs verification
- [ ] SUMMARY.md surfaces top findings inline + points to full report
- [ ] state.json survives context compaction
</success_criteria>
