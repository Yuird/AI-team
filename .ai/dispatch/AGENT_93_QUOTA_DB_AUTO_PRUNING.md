```yaml
job_id: AGENT_93_QUOTA_DB_AUTO_PRUNING
created_at: "2026-10-01T22:25:00+00:00"
status: done
owner: ""
depends_on: []
results_ref: DISPATCH_LOG.md#A93
evidence: tests/test_quota_window_coordinator.py
updated_at: "2026-10-22T12:00:32.482977+00:00"
```

# DISPATCH — A93 · Bounded retention for `quota_windows.db` (worker + controller)

**Level:** 2 (retention of an existing store; no schema change expected) · **Type:** fix
**Authored:** 2026-10-02 (from A88 inventory) · **Status:** ready
**Branch:** `feat/quota-db-auto-pruning` + PR. No worker restart without the operator (the worker
copy only starts pruning after the operator restarts the worker on merged code).

## Problem (measured read-only 2026-10-02)

A88 classified the native worker's `state/quota_windows.db` as a **worker-local observation cache**
(the prewarmer decides from this host's Claude status line; see `docs/backend/DATABASE_AUTHORITY.md` §2/§4 R2).
A cache must be bounded. It is not:

| File | Size | `coordinator_events` | `snapshots` |
|---|---|---|---|
| worker `~/dev/AI-team/state/quota_windows.db` | 120 MB, `auto_vacuum=0` | 225,418 rows since 2026-07-31 (≈10k / 7 days; top: `quota.adapter_unavailable` 106k, `quota.duplicate_snapshot` 87k) | 31,439 rows, **oldest 2026-07-31** (> 14-day retention) |
| controller `~/ai-team-data/controller/state/quota_windows.db` | 31 MB | 36,788 rows in 7 days (≈5k/day) | 25,465 rows (oldest 2026-09-24 = still within retention) |

Root causes (verify in the tree before changing):
1. **Worker never prunes anything.** The worker prewarmer drives the coordinator via
   `coordinator.observe_once()` (`src/services/quota_window_prewarmer.py:516`); snapshot pruning
   (`_prune_once_daily` → `store.prune_snapshots`, 14-day `_SNAPSHOT_RETENTION_DAYS`) runs only inside
   the coordinator's own `_observe_loop` (`src/services/quota_window_coordinator.py` ~1407-1438), which the
   worker never starts.
2. **`coordinator_events` has no retention anywhere** (only `snapshots` has a prune), so it grows without
   bound on both sides.
3. Deleted pages are never returned to the OS (`auto_vacuum=0`, no `VACUUM`).

## Task

1. Make retention run on **every** process that owns a quota store (controller coordinator loop AND the
   worker's prewarmer-driven coordinator), once per UTC day, best-effort, off the event loop — reuse the
   existing `_prune_once_daily` seam rather than adding a second scheduler.
2. Add a bounded retention for `coordinator_events` (age-based; pick the window from what the readers
   actually need — check every reader of the table, e.g. quota API/digest/diagnostics — and record it).
   Consider whether high-volume, low-value events (`quota.duplicate_snapshot`, repeated
   `quota.adapter_unavailable`) should be shortened or deduplicated at write time.
3. Batched deletes (bounded rows per statement/transaction) so a first prune of 225k rows does not hold
   the SQLite write lock for long (the controller store is shared by gateway + task-server).
4. Space reclamation: decide between `PRAGMA incremental_vacuum` (needs `auto_vacuum=INCREMENTAL`, which
   requires a one-time `VACUUM` to switch) or a periodic bounded `VACUUM` on the worker-local cache only.
   The controller file is shared by two processes — do not run a blocking `VACUUM` there without a plan.
5. Tests: retention runs from the worker prewarmer path; events older than the window are removed while
   fresh ones and anything a reader depends on survive; batching bound holds; prune failure never kills
   the observe/prewarm loop.

## Acceptance

- After one daily cycle on merged code, the worker store holds no snapshot older than retention and no
  event older than the chosen event window; same for the controller store.
- Steady-state size is bounded and documented (rows/day × window) in `docs/backend/DATABASE_AUTHORITY.md` §4 R2.
- No behaviour change to quota decisions (prewarm, windows API) — prove with the existing coordinator/
  prewarmer tests plus the new ones.

## Non-goals

- Moving the worker cache into the controller (A88 R2 decided it stays worker-local).
- Changing quota observation semantics or the `QUOTA_COORDINATOR_ENABLED` env-vs-registry split
  (recorded separately in `docs/backend/DATABASE_AUTHORITY.md` §2).

## Milestone (burndown)

- [x] Readers of `coordinator_events` inventoried; retention window chosen and recorded
- [x] Worker-path + controller-path daily retention (snapshots + events), batched
- [x] Space reclamation decision implemented for the worker cache
- [x] Tests green (PR: issue #6, branch `feat/quota-db-auto-pruning`)
- [ ] Operator step: restart the worker on merged code (and redeploy the gateway); then check the
  worker log for `quota_snapshots_pruned` / `quota_events_pruned` / `quota_store_space_reclaimed`

## Closure (fill on completion)
**2026-10-22 — done on test evidence (cloud-burn issue #6).** The live row-count check after the
operator restarts the worker is a noted operator step, not the done-gate (per the issue).
- **One seam.** `QuotaWindowCoordinator._prune_once_daily` is renamed to `prune_once_daily` and now
  prunes `snapshots` and `coordinator_events`. The controller still drives it from `_observe_loop`.
  The worker's `QuotaWindowPrewarmer.tick_once` now calls it after its observe, through the same
  per-UTC-day guard. There is no second scheduler.
- **Readers of `coordinator_events`.** None in `src/`, `web/` or `scripts/`. The only readers are
  three tests that read back events written in the same test. The Telegram digest consumes events in
  memory via `event_handlers`. Window: `QUOTA_EVENT_RETENTION_DAYS` = 14, matching snapshots.
  Write-time dedup of `quota.duplicate_snapshot` / `quota.adapter_unavailable` was considered and
  not done, because age retention already bounds them and changing what is emitted is out of scope.
- **Batched.** `QuotaWindowStore._delete_batched` runs `DELETE … LIMIT 5000` with one write
  transaction per batch. The A78 ROW_NUMBER anchor exclusion is reused unchanged inside each batch.
- **Space.** Only the worker passes `reclaim_space=True` (`src/worker/agent.py`). Its first pass sets
  `auto_vacuum=INCREMENTAL` and runs one `VACUUM`, and later passes run `incremental_vacuum`. Both
  then do a WAL TRUNCATE checkpoint. The shared controller file is never vacuumed.
- **Proof:** `tests/test_quota_window_coordinator.py` (A93 section, 9 tests; all fail without the
  change) and the unchanged coordinator and prewarmer suites, 80 passed.
- **Bound** documented in `docs/backend/DATABASE_AUTHORITY.md` §4 R2.
