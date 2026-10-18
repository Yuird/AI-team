Upstreams the reviewed fork PR **Yuird/AI-team#16** (fork burn-down issue Yuird/AI-team#5). This PR brings the A65 P3 remediation: budget alerts are pushed through the existing terminal-outcome Web Push seam. The fork's adversarial reviewer passed it in round 2, on the code at `f264668`. This branch has the same two commits rebased onto this repo's `main` @ `d68d1ac`.

## Upstream submission: rebase and re-verification (2026-10-18)

**Provenance.** The commits are `7306407` (= fork `aaa8276`) and `3fb45c9` (= fork `f264668`, the round-1 review fix). `src/services/notification_service.py` and `tests/test_cost_alert_push.py` are byte-identical to the reviewed head. No code changed during the rebase.

**Conflicts resolved (docs only):**
| File | Cause | Resolution |
|---|---|---|
| `.ai/dispatch/DISPATCH_LOG.md` | The fork's A68 row sits next to A65, and the fork's burn-down triage had changed it. | Kept this repo's A68 row unchanged and took only this PR's A65 row. The fork's triage edits are **not** carried. |
| `.ai/dispatch/_DISPATCH_STATE.md` | Generated file | Re-rendered with `dispatch_state.py --render`. Besides A65 → `done`, the render refreshes STALE day counters. It also surfaces a pre-existing `main` gap: A67's evidence still points at the pre-`docs/backend/` path (`CLAIMED_DONE_NO_PROOF`). That gap is not caused by this PR, and this PR does not fix it. |

**Seam re-check against upstream `main`.** The 149 commits since the fork's base touched `notification_service.py` three times: `8c5596c` A104 Gate 3, `846b816` A101 quota-resume push, and `3c5dfa9` A98 restart notify. None of them changes `notify_task_outcome`'s contract or `PushService.fanout`. `notify_task_outcome` is still the single terminal-outcome entry point (5 orchestrator call sites, including the A84 managed-completion path under its `wait_for`). `main` added one `fanout` caller of its own, the A98 operator restart notify (`_maybe_push_restart`). So `grep "fanout(" src` now lists four `NotificationService` callers: `_maybe_push_outcome`, `_maybe_push_cost_alerts` (this PR), `_maybe_push_case_resume` and `_maybe_push_restart`. There is still one budget-alert pathway.

**Tests on this branch.** Run offline as targeted modules only, under the TEST COST GUARD:
```
$ AI_TEAM_BLOCK_LIVE_CLI=1 .venv/bin/python -m pytest -p no:warnings tests/test_cost_alert_push.py tests/test_cost_alerts.py tests/test_push_notifications.py
55 passed in 4.65s
```
That is 55, not 53, because `main` added push tests since the fork's run. **The tests still fail without the change:** with `notification_service.py` reverted to `main`, `tests/test_cost_alert_push.py` gives `9 failed, 7 passed`, the same split as the original review. The 7 that pass both ways are the "nothing happens" guards.

**Migrations:** none.
**Operator step after merge (not performed):** `deploy:gateway-restart`, then the live device check described in §5 below.

---

## Original reviewed PR body (Yuird/AI-team#16)

Budget alerts now reach subscribed devices through the existing Web Push seam. Before this change they were pull-only (the Cost tab polled `GET /api/cost/alerts`). This closes the A65 P3 final-review finding F1.

### 1. Invariant achieved
> When a configured billable-USD budget (`COST_ALERT_DAILY_BUDGET_USD`, `COST_ALERT_SESSION_BURN_USD`, `COST_ALERT_CASE_TOTAL_USD`) becomes crossed, subscribed devices receive one bounded, best-effort Web Push for that alert per rule/scope/UTC day. The push is delivered at the existing terminal-outcome notification seam; never blocks task completion; never adds a kill or enforcement path. With no knob set, or with push unavailable, behaviour is unchanged.

**Where it holds:** `NotificationService.notify_task_outcome` calls the new `_maybe_push_cost_alerts()` right after the existing `_maybe_push_outcome(...)` (`src/services/notification_service.py`). The new method:
- returns at once if no knob is > 0 (`read_budget_knobs`, which keeps the existing `_env_float` behaviour for 0, negative and garbage values), or if `PushService.available()` is false;
- if a check ran in the last 60 s or is still running, arms **at most one trailing check** for the end of that window instead of dropping the call (added after review F1);
- otherwise schedules a detached task (`loop.create_task`, never awaited). The task waits 5 s so the finishing turn's telemetry has been projected, runs `check_cost_alerts` through `asyncio.to_thread` and calls the existing bounded `PushService.fanout` (same concurrency and per-send timeout) once for each alert whose `(utc_day, rule, scope)` key is new;
- sends a payload of `💸 Budget alert` with a body such as `Daily spend: $12.50 of $10.00 budget (125.0%) · today (UTC)` and the deep link `/cost`. It has the same shape as the quota-resume push, so the service worker needs no change.

**Cross-layer trace:** knob env → gateway process → `notify_task_outcome` → `PushService` → device.

| Hop | Status |
|---|---|
| Knob env read in the gateway process (`os.environ`, the same source `/api/cost/alerts` already uses) | verified offline (test `test_unset_or_invalid_knob_means_no_check_no_push`) |
| `notify_task_outcome` → `_maybe_push_cost_alerts` → real `check_cost_alerts` over a seeded MeshDB | verified offline (`test_real_read_model_crossing_pushes_once`) |
| → `PushService.fanout` payload | verified offline. `fanout` is stubbed; its transport bounds are covered by the existing `tests/test_push_notifications.py` |
| `fanout` → pywebpush → push relay → a real device | **not verified.** Needs the operator: a gateway restart plus a live device |
| Which process runs it | only `main.py` (gateway) builds `TaskOrchestrator`/`NotificationService`. `server_main.py` (standalone task server) does not, so pushes are not doubled. The gateway already sends outcome pushes, so it has the VAPID/`PushService` configuration. The stop condition does not apply. |

Both completion paths reach the seam: the legacy `notify_task_outcome` call sites and the A84 managed-completion consumer, which awaits it under a 4 s `wait_for`. Because the alert work is detached, that `wait_for` never waits on it.

### 2. Proof
New module `tests/test_cost_alert_push.py` (16 tests):
- `test_crossed_budget_pushes_once_per_rule_scope_day`: 5 outcomes produce 5 checks and **1** push.
- `test_distinct_scopes_each_push_once`: two cases produce two pushes, once each.
- `test_utc_day_rollover_resets_dedupe`: the alert pushes again on the next UTC day.
- `test_unset_or_invalid_knob_means_no_check_no_push[None, "", 0, -5, garbage]`: no check and no push.
- `test_push_unavailable_means_no_check`.
- `test_failing_check_or_fanout_never_raises_into_outcome` and `test_available_raising_never_raises_into_outcome`.
- `test_check_runs_off_the_event_loop`: spies on `asyncio.to_thread` and checks that the check runs on another thread. With a 0.5 s read model, `notify_task_outcome` returns in < 0.2 s inside a 4 s `wait_for`.
- `test_throttle_bounds_read_model_calls`: a burst of 10 outcomes produces **1** read-model call.
- `test_inflight_check_is_not_doubled`: no overlapping check while a slow one is running; the burst folds into one trailing check after it.
- `test_crossing_on_a_throttled_completion_is_still_pushed` (review F1): with the production 60 s window, turn A is under budget and turn B, inside the window, crosses it. Exactly one trailing check is armed (delay of about 60 s) and it pushes once.
- `test_real_read_model_crossing_pushes_once`: offline cross-layer test against the real read model.

```
$ .venv/bin/python -m pytest -q -p no:warnings tests/test_cost_alert_push.py tests/test_cost_alerts.py tests/test_push_notifications.py
.....................................................                    [100%]
53 passed
```
**Fails without the change:** with `src/services/notification_service.py` reverted to `main`, 9 of the 16 new tests fail. The 7 "nothing happens" tests pass both ways, which is expected. Against the first head (`aaa8276`), the F1 regression test and the updated in-flight test fail.

### 3. Regression and failure-path review
- **Flag-OFF byte-identity.** With no knob set, the only added work per outcome is three `os.environ` reads. No SQL, no task, no push.
- **Push unavailable.** Returns before any check. Behaviour is unchanged.
- **Error paths.** Every exception is swallowed at debug level, both in the scheduler and in the detached task. The in-flight guard is released in `finally`.
- **Slow read model during a completion burst.** Completion latency is unaffected because of the detached task and `to_thread`. The throttle (≤ 1 check per 60 s, plus one trailing check per window) and the in-flight guard bound read-model calls and threads.
- **Telemetry lag.** The task server projects telemetry about once a second, so each check waits 5 s first. The trailing check then covers a turn whose cost lands later than that.
- **Concurrency.** The dedupe check-and-add runs on the event loop with no `await` in between, so two concurrent checks cannot double-push.
- **UTC rollover.** The same `now` drives both the check's day window and the dedupe day key, and yesterday's keys are pruned.
- **Budget 0, negative or garbage.** No alert (existing `_env_float`), as tested above.
- **Restart.** Dedupe state is in-process only, so a gateway restart can repeat an already-pushed alert **at most once per restart** that day. This is accepted per the issue. No table and no migration were added (the migration freeze is respected).
- **Session-burn scope** is the current top spender. If a different session becomes the top spender and crosses the budget, it gets its own push. That is one push per scope, as specified.

### 4. No duplicate pathway
- `grep -rn "check_cost_alerts" src` finds only `cost_alerts.py` (the definition), `routes/cost.py` (the existing pull endpoint) and `notification_service.py` (the new push).
- `grep -rn "fanout(" src` finds only the three `NotificationService` callers: outcome, budget alert and case-resume.
- No new channel, table, flag or kill path was added.

### 5. Dispatch state reconciled
- A65 was set through `dispatch_state.py --set`: `status done`, and `evidence` now also lists `tests/test_cost_alert_push.py`. `--audit` reports no gaps, no unproven-done and no stale-active.
- The `_DISPATCH_STATE.md` view was regenerated by the tool.
- The `DISPATCH_LOG.md` A65 row is updated, and a `## Closure (2026-10-06 …)` section is added to the job file.
- A65 is removed from CONTEXT → Current Focus and from the Active Work table.
- **Operator remainder (`deploy:gateway-restart`):** run `pm2 restart ai-team-gateway`. Then, as a live check, set a low `COST_ALERT_DAILY_BUDGET_USD` and confirm that exactly one "Budget alert" push arrives on a subscribed device after the next task completes. I did not perform this step.

### 6. Unresolved assumptions
- **Status already `done`.** A65 is set to `done` (as the issue instructs) before the live-device check. That check is an operator step after the redeploy, recorded in the closure.
- **60 s throttle.** The interval is a class constant, not a knob. Moving the knobs into `/api/flags` belongs to A62 and is out of scope.
- **Checks are triggered by completions.** A check happens only after a terminal outcome, either right away or at the end of the throttle window. Spend recorded with no completion at all, such as an in-flight turn's partial usage, is not pushed until a completion triggers a check. This matches the prescribed seam.
- **Dedupe before fan-out.** An alert is marked sent before fan-out, so a send that fails or is skipped is not retried that day. This is best-effort by design.
- **Unattributed spend.** The daily rule already counts the unattributed bucket, via the existing `check_cost_alerts` semantics, which are unchanged.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_012xYTv8z8nJRSo5fxC2887K
