Upstreams the reviewed fork PR **Yuird/AI-team#18** (fork burn-down issue Yuird/AI-team#4), dispatch jobs **A97**, **A78 §3**, and the **A63** refusal-ingest test residual. After this PR, a failed turn's `error_class` comes only from error-bearing or structured signals, never from the agent's own prose. A genuine SDK-path usage limit still reaches `record_refusal_snapshot` with the provider's `resetsAt`. The fork's adversarial reviewer passed it in round 3 of 3, on head `cf633cc`. This branch has the same five commits rebased onto this repo's `main` @ `d68d1ac`, plus **one test-only adaptation commit** for A102.

## Upstream submission: rebase and re-verification (2026-10-18)

**Provenance.**
| Here | Fork (reviewed) | |
|---|---|---|
| `bd69bf0` | `b73222d` | main fix |
| `9019acd` | `570b4dc` | dispatch reconcile |
| `02c9684` | `d5810c3` | round-1 fixes |
| `6815d3a` | `3747921` | round-2 fixes |
| `345d741` | `cf633cc` | dispatch, reviewed head |
| `68f3234` | — | **new:** test harness adapted to A102 (see below) |

Every source and doc hunk is **textually identical** to the reviewed PR: `claude_driver.py`, `failure_markers.py`, `orchestrator.py`, `result_text.py`, the two spec docs, and the test module before `68f3234`. I compared the added and removed lines of each file between the original PR diff and this branch's diff. `orchestrator.py` and `claude_driver.py` merged without textual conflicts over `main`'s changes.

**The one semantic seam: A102 S1-Claude (`e190ed4`).** `main` collapsed the Claude driver into one turn pipeline: `run_oneoff(..., *, turn: TurnControl)` now runs a throwaway Session through `_run_turn`, and `ClaudePrintResumeDriver` is deleted.
- **Production code is unaffected.** This PR's driver hunks are `_note_rate_limit_event` in `_SDKSession._reader_loop` and `rate_limit_rejected` in `_outcome_from_result`. Both sit in that single shared pipeline, so after A102 they cover every Claude turn type.
- **The test harness needed one line.** `_through_driver_and_orchestrator` called the pre-A102 `run_oneoff("/repo", "do it")`, so 5 SDK-path tests raised `TypeError` before any assertion. `68f3234` passes `turn=turn_control("t-sdk")`, the factory `main`'s own turn-queue tests use. It makes no assertion or behaviour change.
- **Builder invariant re-checked on the rebased tree.** `src/orchestrator.py` still has exactly 27 `TaskResult(` constructions, the same count as at the fork's base, so `main` added none. The `raw_stdout=` builders are the same 8 the round-3 reviewer enumerated:
  - Marked with `mark_reply_mirror`: the 4 that copy `output`, which are `_reattach_remote_task`, `_dispatch_to_node` completed and failed, and `_managed_turn_effect_inputs`.
  - Not marked: the other 4, which carry a real stream or `""`.
- Not affected: the worker-side `_parse_print_resume` path used by `test_mesh_print_resume_stream_as_output_keeps_the_refusal` still exists in `src/backends/claude_driver.py`, and that test passes.

**Conflicts resolved (docs only):**
| File | Resolution |
|---|---|
| `.ai/dispatch/DISPATCH_LOG.md` | `main` added rows A98–A108 directly above A97. I kept all of them and replaced only the A97 and A78 rows with this PR's rows. The fork's burn-down triage row edits (A75/A95/…) are **not** carried. |
| `.ai/dispatch/_DISPATCH_STATE.md` | Re-rendered with `dispatch_state.py --render`. A97 → `done`, A78 → `active`, and the STALE counters refresh. The render also surfaces a pre-existing `main` gap: A67's evidence still points at the pre-`docs/backend/` path. That gap is not caused by this PR. |

**Tests on this branch.** Run offline as targeted modules only, under the TEST COST GUARD:
```
# the PR's own module + the original review's related set (16 modules)
$ AI_TEAM_BLOCK_LIVE_CLI=1 pytest tests/test_failure_text_scope.py tests/test_claude_driver.py tests/test_retry_transient.py \
    tests/test_case_quota_resume.py tests/test_output_truncation.py tests/test_quota_window_coordinator.py \
    tests/test_claude_session_backend.py tests/test_case_transient_resume.py tests/test_usage_propagation.py \
    tests/test_affinity_fallback.py tests/test_backend_activity.py tests/test_claude_telemetry_adapter.py \
    tests/test_result_text_ndjson.py tests/test_sdk_driver_proactive.py tests/test_turn_queue_4b_carrier.py \
    tests/test_turn_queue_sdk_ownership.py
337 passed, 2 failed
```
The 2 failures are `test_case_quota_resume::test_worded_reset_*` with `ZoneInfoNotFoundError: 'Europe/Kiev'`. They fail **identically on pristine `main`** in this container, which lacks that tzdata alias. With the `tzdata` package installed in the venv (no repo change), `tests/test_case_quota_resume.py` gives `57 passed`.
```
# every test module that references a seam this PR touches
# (_classify_error / _failure_text / failure_text / _short_failure_reason / _dispatch_to_node /
#  _reattach_remote_task / _managed_turn_effect_inputs / classify_error_text / _extract_rate_limit_info)
10 modules: 230 passed
# A84 managed-effects + A104 inbox modules (pending_for, managed_turn_effect, …)
17 modules: 286 passed
```
**The tests still fail without the change on the new base:**
- With `orchestrator.py`, `result_text.py` and `claude_driver.py` reverted to `main`, keeping only the new constants module, `tests/test_failure_text_scope.py` gives **25 failed, 10 passed**.
- With all of them in place, all **35 pass**.
- With only `claude_driver.py` reverted, the SDK-path tests (`test_sdk_429_with_rate_limit_event_records_refusal_snapshot`, `test_sdk_rejected_event_without_429_is_usage_limit`, `test_telemetry_store_failure_never_changes_the_class`, `test_driver_5xx_still_wins_over_a_rejected_event`) fail. So the adapted harness is not vacuous.

**Migrations:** none.
**Operator steps after merge (not performed):** `deploy:worker-restart` (the driver runs in the worker) and `deploy:gateway-restart` (orchestrator classification).

---

## Original reviewed PR body (Yuird/AI-team#18)

Covers dispatch jobs **A97** (`AGENT_97_FAILURE_TEXT_ERROR_CLASS_FIX.md`, primary), **A78 §3 SDK path + §1 decision record**, and the **A63** refusal-ingest test residual.

### 1. Invariant achieved

> 1. A failed turn's `error_class` is derived only from **error-bearing or structured** signals. Text in which the agent merely *wrote about* timeouts, "503", "usage limit" or "rate limit" never changes the class.

**Holds:** `TaskOrchestrator._failure_text` and `result_text.failure_text` now read only `errors`, `raw_stderr`, the error events in `raw_stdout` (an `is_error` result, a **rejected** `rate_limit_event`, an `error` event, a CLI-synthesised API-error message), and the error fields of a terminal-error `parsed_output` (never its `assistant_text`). Only **recognised** event types count; a bare `{"error": ...}` line does not. When `raw_stdout` is only a copy of the reply, nothing is read from it, neither by the text pass nor by the `rate_limit_event`/terminal-result parsers. Since round 2, "copy of the reply" is decided where the copy is made: the gateway builders that put `output` into `raw_stdout` call `mark_reply_mirror` (`_dispatch_to_node` when the worker shipped no transcript, `_reattach_remote_task`, `_managed_turn_effect_inputs`). Equality of the two fields is not evidence, because print_resume ships `output == raw_stdout` when its stream holds no extractable text. Also since round 2: when `raw_stderr` falls back to the worker's `error_detail`, its `stdout_tail` (the agent's stream) is dropped (`error_detail_without_stdout`). The reply (`output`, assistant events, success result text) is read **only** when no error-bearing text exists. The one shared selector is `src/core/failure_markers.py` (`error_bearing_values`, `error_bearing_stdout_lines`). Bare `503`/`504` now match only as standalone tokens (`UPSTREAM_STATUS_RE`).

> 2. A **genuine** provider usage limit on the default `driver_type="sdk"` path is still classified `usage_limit`, and its provider `resetsAt` reaches `_usage_limit_class` → `record_refusal_snapshot`.

**Holds.** Code path, change → observation:
1. CLI wire `rate_limit_event` (status `rejected`) → SDK `parse_message` → `RateLimitEvent`. I checked that the pinned SDK 0.2.110 yields this through `receive_messages()`, so the stop condition does not apply.
2. `_SDKSession._reader_loop` → new `_note_rate_limit_event`. This mirrors `{"type":"rate_limit_event","rate_limit_info":{...}}` in the CLI's wire keys into the turn NDJSON and flags `acc.rate_limit_rejected`.
3. `classify_error_text(..., rate_limit_rejected=True)` returns `usage_limit`. A 429 also returns `usage_limit`. A 5xx still wins, which matches the orchestrator's precedence.
4. `ExecutionResult(raw_stdout=raw_ndjson, error_class=...)`. Every gateway `TaskResult` builder on a backend result path now copies the backend's `error_class`; before this change they dropped it. That covers `_run_backend_local`, the main in-process path, `_dispatch_to_node`'s failed and completed branches (the mesh **session** path, added in review round 1) and `_reattach_remote_task`.
5. `_classify_error` → `_extract_result_terminal_signal`/`_extract_rate_limit_info` (unchanged parser, reused) → `_usage_limit_class` → `store.record_refusal_snapshot(provider, reset_at)`.

> 3. There is exactly one failure-label marker set, with no drifted copy.

**Holds.** `USAGE_LIMIT_MARKERS`, `RATE_LIMIT_MARKERS` and `QUOTA_FAILURE_LABEL_MARKERS` in `src/core/failure_markers.py` are imported by the driver, `_classify_error`, `_short_failure_reason` and `result_text.short_failure_reason`. The driver's `_USAGE_LIMIT_MARKERS`/`_RATE_LIMIT_MARKERS` are deleted. `result_text` used to lack "session limit", "usage limit" and "hit your session limit"; it now uses the shared set.
```
$ grep -rnE '"hit your limit"|"session limit"|"rate-limit"|overagestatus"' src --include=*.py
src/core/failure_markers.py:26:    "session limit",
src/core/failure_markers.py:27:    "hit your limit",
src/core/failure_markers.py:30:    "overagestatus",
src/core/failure_markers.py:38:    "rate-limit",
```

**Structured-class preference (A97 item 4, in scope):** `_classify_error` takes a backend class as-is when it is in `PREFERRED_BACKEND_ERROR_CLASSES` (`max_turns`, `upstream_error`, `sdk_stream_closed`, `context_overflow`, `rate_limit`), or routes `usage_limit` through `_usage_limit_class`. These are only classes the classifier already returns, so no class reaches the retry policy without an existing entry. Other backend classes (`session_lost`, `cache_unhealthy`, `permission_block`, `transient`, `managed_conflict`, `recovery_required`, the OpenCode classes) are **still re-derived from text**. Making them authoritative would route them to the retry policy's *default* branch (2 retries), which is a retry-policy decision and out of scope. That is the recorded A97 item 4 finding.

**Scope item 2, a premise correction with evidence:** the issue says the driver's `error_text`→`result_text` fallback is "the agent's own text". The bundled CLI of the pinned SDK says otherwise. It builds the terminal result as `is_error: lastMsg.isApiErrorMessage, api_error_status: lastMsg.apiErrorStatus, result: <text of that last message>`. On an `is_error` success result, `result` is therefore the CLI's API-error message. The agent's narration travels separately as `salvaged_output` → `output`, and that is now excluded. So the driver's classification input was already prose-free; the leak was `output`, `parsed_output` and the assistant lines in `raw_stdout`, all fixed above. The human-readable message is unchanged. The structured signals (429, rejected event, subtype) now come first in the driver, so text over the error string is only the fallback.

**Seams verified** (by tests, in-process): CLI wire dict → SDK `parse_message` → driver `_reader_loop` → `_outcome_from_result` → driver `run_oneoff` error branch → `ExecutionResult` → orchestrator `_run_backend_local` → `_classify_error` → `_usage_limit_class` → fake store `record_refusal_snapshot`.
**Seams not verified:** a real CLI 429 against the live account (A78 §6, operator-only); the real `QuotaWindowStore.record_refusal_snapshot` *through* the orchestrator (the store's own behaviour is covered by existing `test_quota_window_coordinator.py` tests); the remote/mesh worker → gateway hop with a real worker.

### 2. Proof

New module `tests/test_failure_text_scope.py`, 35 tests. Round 2 added 3 that fail on `d5810c3`, both through the `_dispatch_to_node` shape with the real worker payload helpers: `test_mesh_print_resume_stream_as_output_keeps_the_refusal` (with and without a result line) and `test_mesh_error_detail_stdout_tail_is_not_error_text`. Round 1 brought it to 32. The 8 added in review round 1 (reply mirror, untyped error dict, `_dispatch_to_node` structured class ×4, legacy-worker mirror, print_resume stream-as-output) include **7 that fail on `570b4dc`**; the print_resume one guards against over-applying the mirror rule. Of the original 24, **17 fail on `main`** (I ran it with the behaviour changes stashed and only the new constants module present) and all pass on the branch:
```
FAILED test_reply_text_does_not_change_the_class[...timeout...-timeout]
FAILED test_reply_text_does_not_change_the_class[...503...-network]
FAILED test_reply_text_does_not_change_the_class[...usage limit...-usage_limit]
FAILED test_reply_text_does_not_change_the_class[...rate limit middleware.-rate_limit]
FAILED test_reply_mentioning_usage_limit_does_not_pause_a_case
FAILED test_error_events_in_the_stream_are_still_read
FAILED test_bare_status_codes_are_word_bounded[checksum mismatch in abc5031-fatal]
FAILED test_bare_status_codes_are_word_bounded[failed to open src/503/handler.py-fatal]
FAILED test_bare_status_codes_are_word_bounded[unsupported schema v1.504-fatal]
FAILED test_allowed_warning_rate_limit_event_is_not_a_refusal
FAILED test_structured_backend_class_is_preferred_over_text
FAILED test_run_backend_local_keeps_the_backend_error_class
FAILED test_sdk_429_with_rate_limit_event_records_refusal_snapshot
FAILED test_sdk_rejected_event_without_429_is_usage_limit
FAILED test_telemetry_store_failure_never_changes_the_class
FAILED test_driver_5xx_still_wins_over_a_rejected_event
FAILED test_one_shared_failure_label_marker_set
```
The 7 tests that pass on `main` are regression guards: the output-only fallback, unknown backend classes still re-derived, a 429 with empty text, an allowed-warning event not mirrored, and the positive status-code cases.

Targeted modules only (TEST COST GUARD; never the full or e2e suite):
```
$ pytest -q tests/test_failure_text_scope.py tests/test_claude_driver.py tests/test_retry_transient.py \
    tests/test_case_quota_resume.py tests/test_output_truncation.py tests/test_quota_window_coordinator.py
2 failed, 224 passed (head `3747921`)
FAILED tests/test_case_quota_resume.py::test_worded_reset_on_the_refusal_becomes_the_recorded_instant
FAILED tests/test_case_quota_resume.py::test_worded_reset_parser_handles_the_clock_forms
```
These 2 fail **identically on `main`**. The cause is environmental: this container's tzdata has `Europe/Kyiv` but not the `Europe/Kiev` alias (`ZoneInfoNotFoundError`). With a scratch tz path that adds the alias (`PYTHONTZPATH`, no repo change), the result is **226 passed** (head `3747921`). Related modules also pass: `test_claude_session_backend`, `test_case_transient_resume`, `test_usage_propagation`, `test_affinity_fallback`, `test_backend_activity`, `test_claude_telemetry_adapter`, `test_result_text_ndjson`, `test_sdk_driver_proactive`, `test_turn_queue_4b_carrier`, `test_turn_queue_sdk_ownership` (all green).

### 3. Regression and failure-path review
- **Error only in `output`** (no errors, no stderr, no error event): the reply is still read, so the class is the same as today (`test_error_only_in_output_keeps_todays_class`).
- **Real 429 with empty or unrelated text**: `usage_limit` in the driver and the orchestrator (`test_sdk_429_with_empty_text_and_no_event_is_usage_limit`, `..._records_refusal_snapshot`).
- **`allowed_warning`**: not mirrored, not flagged, not a refusal, store not called (`test_sdk_allowed_warning_event_is_not_mirrored_or_flagged`, `test_allowed_warning_rate_limit_event_is_not_a_refusal`).
- **Store failure inside `_usage_limit_class`**: the class stays `usage_limit` and nothing raises. The existing try/except isolation is untouched (`test_telemetry_store_failure_never_changes_the_class`).
- **`503` inside a word or path** (`abc5031`, `src/503/`, `v1.504`): no longer `network`.
- **Older SDK without `RateLimitEvent`**: imported defensively like the A82 message types; the behaviour is then exactly as before.
- **Flag-OFF byte-identity**: no flag is involved, and the job did not ask for one. A successful turn is unaffected: classification returns `none` first, and the mirrored line is only added for a *rejected* event.
- **Concurrency/restart**: the per-turn accumulator is reset after every `ResultMessage`, so a rejected event is attributed to the turn it arrived in.

#### Class changes for historically common cases (A97 RESERVED DECISIONS)
1. A failed turn with a real error in `errors`/`raw_stderr` whose **reply or stream prose** mentions limits, timeouts, 503 or rate limits is now classified by the real error, often `fatal`/`auth`. Before, it was `usage_limit`/`rate_limit`/`timeout`/`network`, which meant spurious paid retries or a spurious Case pause.
2. An `allowed`/`allowed_warning` `rate_limit_event` carrying `overageStatus` is no longer `usage_limit` (before, the `overagestatus` substring matched).
3. A bare `503`/`504` inside an identifier or path is no longer `network`.
4. **Mesh/remote path (including the session path through `_dispatch_to_node`, fixed in round 1):** the worker's structured driver class is now honoured. A remote 5xx (`upstream_error`) used to be re-derived from text, typically to `fatal`. It now gets the existing `upstream_error` retry plus transient-pause behaviour, the same as the local path already had. Remote `max_turns`/`context_overflow`/`rate_limit` are unchanged in effect.
5. A driver `sdk_stream_closed` result now classifies `sdk_stream_closed`, so its existing 0-retry policy applies. Before, it was re-derived from its text (`network` → up to 2 blind retries, or `fatal`).
6. A single-line stdout CLI-synthesised API-error message with `"error":"rate_limit"` now classifies `rate_limit`; before, the marker was lost and the class was `fatal`.
7. Failure label (`result_text.short_failure_reason`, used by notifications and the task server): "session limit"/"usage limit" wording is now labelled "Claude usage limit reached", the same as the orchestrator's label.
8. Driver: a rejected `RateLimitEvent` plus a non-429, non-5xx status (for example 400) is now `usage_limit`; it was `upstream_error`.
9. A JSON line in a mirrored reply (`{"error": ...}`, or a quoted `rate_limit_event`/result line) no longer feeds classification or the quota store. Before this PR, the quoted `rate_limit_event` case could pause a Case.
10. Mesh output-only failures: `_dispatch_to_node` always supplies `errors` (the worker synthesises "Backend task failed …"), so a legacy-mirrored reply that carries the only error text is no longer read. For example, "You've hit your limit · resets 3pm" in the reply alone was `usage_limit` on `main` and is `fatal` now. This is by design: a real refusal arrives as a structured event, a 429, or the worker's class.
11. Mesh print_resume with empty worker stderr: the agent text in `error_detail`'s `stdout_tail` no longer feeds classification. This predates the PR; it was fixed in round 2.
12. Pathological error string with both a context-overflow and a rate-limit marker: the driver's class (`context_overflow`, 0 retries) now wins over the orchestrator's text order (`rate_limit`, 2 retries).

### 4. No duplicate pathway
- One marker vocabulary and one error-bearing selector: `src/core/failure_markers.py`. The two pre-existing `failure_text` implementations (orchestrator and `result_text`, which use different text extractors) both delegate their field selection to it; neither carries its own copy.
- No second rate-limit parser. The driver emits the CLI's own wire shape, which the existing `_extract_rate_limit_info` reads. `grep -rn "def _extract_rate_limit_info\|def extract_rate_limit_info" src` shows the same two pre-existing definitions and nothing new.
- No new store path: the existing `_usage_limit_class` → `record_refusal_snapshot` is reused.

### 5. Dispatch state reconciled
Via `dispatch_state.py --set` (follow-up commit on this branch):
- **A97**: `done` on final head `cf633cc` (the code is `3747921`; it was briefly `active` during round 2), `evidence: tests/test_failure_text_scope.py`; Closure records rounds 1 and 2.
- **A78**: `active`. Remainders: §6 (needs a real 429, operator-only) and §4's worker/events part (Yuird/AI-team#6). §1 decision recorded: keep the plumbing; the get_usage adapter has no session sensor.
- **A63**: status left to the owner (Yuird/AI-team#3); the closed residual is noted in its file.
- DISPATCH_LOG A97 and A78 rows updated.

### 6. Unresolved assumptions
- `result` on an `is_error` result is the CLI's API-error text. I verified this in the bundled CLI of SDK 0.2.110; a future CLI could change it.
- The real CLI emits a **rejected** `rate_limit_event` before the refusal result, as the SDK's `RateLimitEvent` docstring describes. I have not observed this live; the 429 path covers usage limits without it.
- Codex/OpenCode stream events count as error-bearing only when their type is `error`. Other failure events from those backends rely on `errors`/stderr, as their classification mostly did before. Codex/OpenCode error mapping is non-scope.
- Mirror marking covers the four gateway builders that copy `output` into `raw_stdout` today. A future builder that mirrors without calling `mark_reply_mirror` would have its copy read as a stream. `grep -n "raw_stdout=.*output" src/` lists exactly those four sites.
- `error_detail_without_stdout` relies on the worker's `_backend_error_detail` block order (`stdout_tail:` last). A stderr tail that itself contains a `\n\nstdout_tail:` line would be cut early, which drops error text but never adds prose.
- The worker's `_bound_output` keeps the head of `raw_stdout`, so on a very long turn the trailing `rate_limit_event` can be cut before it reaches the gateway. This predates the PR; I'm noting it for Yuird/AI-team#6/A78.

### Operator-gated after merge
- `deploy:worker-restart`: the driver change runs in the worker. It is the same restart as Yuird/AI-team#1, and I did **not** perform it.
- `deploy:gateway-restart`: the orchestrator classification change. Not performed.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_012xYTv8z8nJRSo5fxC2887K
