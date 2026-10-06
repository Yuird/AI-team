# Cloud backlog burn-down — rules for stateless cloud workers (TEMPORARY)

This applies to any session asked to work a GitHub issue labelled `burn:*` (e.g.
"Work issue #NNN autonomously to proven completion."). It sits **on top of** the root
`CLAUDE.md` (project rules: test cost guard, minimal diff, cross-layer honesty). Remove
this file when the burn-down ends.

## Before touching code
1. Read the issue in full. It is the **workstream specification**: the invariant to
   establish, the dispatch jobs it covers, the scope and non-scope, the stop conditions.
2. Read `.ai/dispatch/CLAUDE.md` (the dispatch state protocol). Then read **every**
   dispatch job file the issue references, in full. The issue only summarizes them, and
   the files are authoritative for the details.
3. **Check the premise against current HEAD.** Read the cited code, `git log` the touched
   paths, and confirm that the defect or gap still exists. The issue was written on
   2026-10-06 and code may have moved since. Resolve symbols cheaply with
   `.venv/bin/python scripts/repo_index/symbol_lookup.py --defs-only <Symbol>`.
4. Check the issue's **prerequisites**. If a prerequisite issue is still open, or its
   result is not on `main`, stop and report. Do not build around it.

## While working
- **Pursue the invariant, not the checklist.** Listed edits are hints. Do what is
  strictly necessary to make the stated invariant true end-to-end, including necessary
  causal follow-through the issue did not list. Do nothing else: no drive-by cleanup,
  refactors or renames.
- **One pathway.** Extend the existing seam. Never add a parallel or duplicate
  implementation of something that already exists. If the right seam is unclear, stop
  and report.
- **Operator gates stay closed.** Never restart workers, deploy, flip production flags,
  run paid or live backends, or use secrets. New behavior ships **flag-gated OFF** where
  the job says so. Note any operator-gated remainder in the PR and the job file.
- **Tests:** follow the TEST COST GUARD in the root `CLAUDE.md`. Run plain `pytest` on
  the touched or related test modules only, and never the full or e2e suite. Run web
  checks with `cd web && pnpm typecheck && pnpm test` only when `web/` changed.
- **Stop, don't force.** If evidence invalidates the issue's premise (already fixed,
  superseded, wrong seam, needs an owner decision), stop. Comment on the issue with the
  evidence, open no implementation PR, and leave dispatch state unchanged.

## Standing constraints found in triage (2026-10-06)
- **Mesh migration freeze.** Add no new migration to `_get_migrations()` in
  `src/control/db.py` unless the issue names an assigned number. Migration 43 is reserved
  by the held A82 Stage 8a branch (`nydiokar/AI-team#185`, not in this repo). A gap or
  duplicate number silently skips a migration (see #1).
- **Level-3 work.** A `burn:ready` label is the operator's approval for that issue's
  stated scope only. If you need anything beyond that scope, stop and report.
- **Shallow clone, upstream refs.** Local history starts at 2026-09-18. To see older commits,
  use the GitHub API (`mcp__github__get_commit`). PR numbers in packets and CONTEXT before
  2026-10-06 refer to upstream `nydiokar/AI-team`, not to this repo's numbering.
- **Line numbers in dispatch packets are often stale** (for example, the control-API router
  split `00efbad`). Resolve symbols; never trust a cited line number.

## Completion protocol (all six, stated in the PR body)
1. **Invariant achieved.** Quote the issue's invariant and show where it now holds.
   Give the code path from change to observation, and say which seams you verified and
   which you did not.
2. **Proof.** Name the tests that prove it and paste their pass output. New behavior
   needs a test that fails without the change.
3. **Regression and failure-path review.** Cover error paths, flag-OFF byte-identity,
   restart or concurrency where relevant, and the issue's adversarial checks.
4. **No duplicate pathway.** Grep for the concept and confirm that only one
   implementation exists.
5. **Dispatch state reconciled.** Follow `.ai/dispatch/CLAUDE.md`: use
   `.venv/bin/python scripts/dispatch/dispatch_state.py --set <JOB> status|evidence ...`
   (never hand-edit the yaml block or `_DISPATCH_STATE.md`), and update the job's
   `DISPATCH_LOG.md` row and `## Closure`. Set `done` only when the merged-to-be state
   proves the job satisfied and `evidence:` points at an on-disk artifact. If an
   operator-gated remainder exists, keep the job `active` or `blocked` and say why.
6. **Unresolved assumptions** are listed explicitly in the PR.

Before you open the PR, re-read the **complete diff** adversarially and ask what would
make CI or a reviewer reject it.

## PR boundary (overrides the root CLAUDE.md merge rule for burn work)
- Open **one PR per issue/workstream**, from the branch your session was given (or
  `feat/<slug>`), into `main`. Put `Closes #NNN` in the body.
- **Never merge your own PR.** Never push to `main`. Never force-push. Merge belongs to
  the operator or governor during the burn-down. This deliberately overrides the root
  `CLAUDE.md` "merge it yourself" rule.
- Never carry another workstream's edits into your branch.

## Coordination labels
The roadmap, label legend and dispatch order live in the tracker issue **#13**.
Milestones W0–W4 are the waves. Labels are owner/governor-managed, so workers read them
and never change them.
- **`burn:` status.** Work only `burn:ready`. For any other status, stop and report the
  label. A `gate:owner-*` label means the owner must act first.
- **`deploy:` labels.** These name the operator step needed after merge. List that step in
  your PR, and never perform it.
- **Triage record.** The classification of every job, with evidence, is in
  `.ai/context/BURN_DOWN_TRIAGE_2026-10-06.md`.
