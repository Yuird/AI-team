# Backlog burn-down triage — 2026-10-06 (temporary coordination record)

**Scope:** every dispatch job that was `active`, `ready` or `blocked` on 2026-10-06 (27 jobs), triaged
against `main` @ `01446d7`. Each classification below is backed by repository evidence. The full
evidence is in the linked GitHub issue or in the job file.

**Control plane:** worker rules are in `.claude/rules/cloud-burn.md`. Workstreams are GitHub issues on
`Yuird/AI-team`, with status labels `burn:ready|blocked|defer|decision|obsolete`. Issue numbers are
those of this repo. Older `#NNN` PR references in CONTEXT and packets belong to upstream
`nydiokar/AI-team`.

**Authority:** the dispatch protocol (`.ai/dispatch/CLAUDE.md`) is unchanged. Job yaml state plus
DISPATCH_LOG remain the job ledger. Issues are workstream specifications, not a second ledger.
Delete this file and the rule file when the burn-down ends.

## Totals
`27 jobs = 4 REQUIRED_NOW + 4 REQUIRED_AFTER_DEPENDENCY + 1 COMBINE_WITH + 5 ALREADY_SATISFIED +
5 SUPERSEDED + 0 INVALIDATED + 4 DEFER + 4 OWNER_DECISION`

## Classification (one row per job)
| Job | Was | Class | Where it goes | Key evidence |
|---|---|---|---|---|
| A97 failure-text error class | ready | REQUIRED_NOW | **#4** (wave 1) | `_failure_text` appends `output` (`orchestrator.py:1066`); bare `"503"`; `_run_backend_local` drops `error_class` |
| A78 quota telemetry truthfulness | ready | COMBINE_WITH A97 | **#4** | §2/§4-controller/§5 fixed (`5c92ce3`, `637c487`, `dea142b`); §3 dead on the SDK path (no `RateLimitEvent` handling), and it shares `_classify_error → _usage_limit_class` with A97; §6 operator-only |
| A65 cost monitoring | active (stale 63d) | REQUIRED_NOW | **#5** (wave 1) | `check_cost_alerts` is pull-only (`routes/cost.py`); no budget push via `PushService` (final-review F1) |
| A93 quota DB retention | ready | REQUIRED_NOW | **#6** (queued) | Prewarmer calls only `observe_once`; `coordinator_events` is never deleted |
| A62 non-boolean flags | ready | REQUIRED_NOW | **#7** (queued) | `RuntimeFlagBody.value: bool`; all 35 registry entries are boolean |
| A71 per-node credentials | ready | REQUIRED_AFTER_DEPENDENCY (#1 + L3 approval) | **#8** | Shared `!=` token on all task-server routes; self-reported node identity; textual and migration collision with the held Stage 8a |
| A94 System-One core | ready | REQUIRED_AFTER_DEPENDENCY (#1 migration no. + L3 approval) | **#9** | No S1 code; needs a mesh migration while 43 is reserved by the held Stage 8a |
| A95 S1 pre-flight | ready | REQUIRED_AFTER_DEPENDENCY (A94) | **#10** | Reuses the A94 client, batteries and table |
| A84 completion outbox (slice 2) | active | REQUIRED_AFTER_DEPENDENCY (A82 8a) | **#11** | Slice 1 merged (`06f3f69`); 0 sessions enrolled until 8a; migration freeze |
| A82 session turn queue | active | OWNER_DECISION | **#1** | R0 on main; Stage 8a (upstream nydiokar/AI-team#185) not in this repo; R1 restarts are operator-only |
| A88 DB authority | active | OWNER_DECISION | **#1** | Merged (`684b506`); remaining work is the worker restart plus doc §6 steps. Evidence list fixed here |
| A91 OpenCode parity | active | OWNER_DECISION | **#1**, **#3** | Merged (`0c88c3a` ancestor of main); live check needs the worker restart |
| A75 dashboard token | ready | OWNER_DECISION | **#2** | Upstream PRs 156–158 re-introduced trusted-peer injection; `WORKER_TOKEN` fallback residual |
| A60 warm-worker idle reaper | active (stale 45d) | ALREADY_SATISFIED | reconciled → `done` | `_reap_idle_warm_workers_once`, TTL setting, 8 tests on main; default-ON awaits ratification (#3) |
| A68 peer messaging study | ready | ALREADY_SATISFIED | reconciled → `done` | `docs/PEER_MESSAGING_INVESTIGATION.md`; go/no-go is an owner call (#3) |
| A72 input caps | ready | ALREADY_SATISFIED | reconciled → `done` | `_MAX_INSTRUCTION_CHARS` applied in 5 bodies; `tests/test_control_api_write.py` |
| A73 node `.env` guard | ready | ALREADY_SATISFIED | reconciled → `done` | `_assert_env_file_private`; `tests/test_safe_worker_deploy_env_guard.py` |
| A74 proactive-turn ownership | ready | ALREADY_SATISFIED | reconciled → `done` | 403 check at `task_server.py:~2148`; `tests/test_proactive_turn_delivery.py` |
| A24 decomposer generator | blocked | SUPERSEDED (A56) | reconciled → `dead` | `docs/harness/generators/decomposer.md` explicitly resumes A24 |
| A58 coordinator activation | blocked | SUPERSEDED | owner ruling **#3** | Coordinator observing and acting (prewarmer, Case quota-resume); production flag state is an operator fact |
| A63 audit of A61 | ready | SUPERSEDED | owner ruling **#3**; residuals → #4 | Audit targets deleted (nydiokar/AI-team#95 `5d0b52e`) or moved |
| A87 runtime coordinator | active | SUPERSEDED (de facto) | owner ruling **#3** | Reviews have run outside A87 since 2026-09-25; ledger stale |
| A90 backend locality | ready | SUPERSEDED (A82 managed path / 8b) | owner ruling **#3** | Its seams sit behind A82 managed branches; 8b deletes them |
| A66 collision detection | ready | DEFER | owner ruling **#3** | Not on current focus; build gated on A62 plus its own R1 fork |
| A85 container acceptance | blocked | DEFER | — | Baseline merged; live docker gate needs a docker host and the substrate decision |
| A86 runtime update automation | blocked | DEFER | — | Activates only if workers are containerized (operator 2026-09-27) |
| A96 S1 bounce | blocked | DEFER | — | Needs A94 replay precision ≥ 0.90 on ≥ 100 verdicts plus 2 weeks of live data |

Also reconciled (not open, but flagged by `--audit`): **A67** evidence repointed from the gitignored
`.security/…` to `docs/backend/MESH_SECURITY.md` plus `tests/test_task_server_upload_safety.py`
(this clears `CLAIMED_DONE_NO_PROOF`). **A88** evidence converted from a comma-joined string, which
the audit reads as one non-existent path, to a yaml list.

## Causal graph
```
#4 (A97+A78)  ─┐  wave 1, independent
#5 (A65)      ─┘
#6 (A93), #7 (A62)          queued; independent of the wave and of each other
#1 decision (A82 8a / R1 / migration freeze) ──► #8 (A71) ──► #2 option (b) only
                                             ──► #9 (A94) ──► #10 (A95) ──► A96 (deferred)
                                             ──► #11 (A84 slice 2)
#2 decision (A75)   #3 decision (state rulings: A58 A63 A87 A90 A91 A60 A78 A85 A86 A95 A96 A66 A68)
```
Operator-only remainders share **one** worker restart (#1, item 3): A82 R1, A88, A89, A91, PR #172
prewarm fix, and any merged worker-side burn PR (#4, #6, #7).

## Roadmap and wave policy
The live roadmap is the tracker issue **#13**. It has:
- wave milestones W0–W4 and native "blocked by" links;
- the label taxonomy: `burn:` status, `gate:` blocker, `area:`, `track:` cohort, `type:`, `level:`
  and `deploy:` after-merge step.

Dispatch runs in waves of at most 3 workers. The next wave starts only after the current wave has
results and the graph has been recomputed (`BURN_GOVERNOR.md`).
- **Wave 1 (2026-10-06):** #4, #5 and #14. #14 is the Stage 8a integration that resolved #1.
- **Candidates after wave 1:**
  - #6, after #4;
  - #7;
  - #11 and #8, after #14 merges;
  - #9, after `/approve`;
  - #10, after #9.
- **Decisions:** #1 and #2 were resolved on 2026-10-06. #3 was applied on owner ruling.
