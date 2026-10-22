# A62 — `scripts/ops_flag.sh` end-to-end transcript (2026-10-22)

Run against a real `build_control_api` served by uvicorn on 127.0.0.1:9411 with an
isolated, freshly-migrated mesh DB (no paid backend, not the live gateway). Server env:
`CLAUDE_SDK_MAX_BUDGET_USD=2.5`, `CASE_CONTINUATION_ENABLED=1`, `CLAUDE_SDK_MAX_TURNS` unset.
Covers packet ACCEPTANCE 2–5: typed `set`, `/health` reflection, 422 on wrong type (server
and client side), `on` against an int flag, the R2 `null` convention, `unset`, and `migrate`
round-tripping bool + int + float + null + str flags (idempotent second run).

```text
$ ops_flag.sh set CLAUDE_SDK_MAX_TURNS 40
{"flag_name":"CLAUDE_SDK_MAX_TURNS","value":40,"raw_value":"40","value_type":"int","source":"registry"}
$ curl /health | .governor
{"sdk_max_turns":40,"sdk_max_budget_usd":2.5}
$ ops_flag.sh set CLAUDE_SDK_MAX_TURNS forty
error: CLAUDE_SDK_MAX_TURNS expects an int value, got 'forty'
exit=1
$ curl PUT {"value":"40"} (server-side 422)
{"detail":{"ok":false,"reason":"invalid_flag_value","value_type":"int","message":"CLAUDE_SDK_MAX_TURNS expects an int value, got str"}}422
$ ops_flag.sh on CLAUDE_SDK_MAX_TURNS
error: CLAUDE_SDK_MAX_TURNS expects an int value, got bool
exit=1
$ ops_flag.sh set CLAUDE_SDK_MAX_TURNS null
{"value":null,"raw_value":"","source":"registry"}
{"sdk_max_turns":null,"sdk_max_budget_usd":2.5}
$ ops_flag.sh unset CLAUDE_SDK_MAX_TURNS
{"deleted":true,"flag":{"value":null,"source":"default"}}
$ ops_flag.sh migrate
not registry-writable, cannot migrate (bootstrap/worker-side, left on env/default): CLAUDE_SKIP_PERMISSIONS CONTROL_API_ENABLED GUARDED_WRITE MESH_EMBEDDED_SERVER MESH_ENABLED MESH_SHADOW_WRITE OPENCODE_SERVER_ENABLED TELEMETRY_DETAILED_EVENTS TELEMETRY_ENABLED WORKER_ACCEPT_UNPINNED WORKER_CANARY WORKER_REAP_STALE_SESSIONS 
migrating (value unchanged, source -> registry):
  OK    CACHE_HEARTBEAT_ACTIVE -> false (registry)
  OK    CACHE_HEARTBEAT_CHECKPOINT_ENABLED -> true (registry)
  OK    CACHE_HEARTBEAT_OBSERVE -> true (registry)
  OK    CASE_CONTINUATION_ENABLED -> true (registry)
  OK    CASE_QUOTA_RESUME_AUTO -> false (registry)
  OK    CASE_QUOTA_RESUME_ENABLED -> true (registry)
  OK    CASE_RESPAWN_REQUIRES_APPROVAL -> true (registry)
  OK    CLAUDE_SDK_MAX_BUDGET_USD -> 2.5 (registry)
  OK    CLAUDE_SDK_MAX_TURNS -> null (registry)
  OK    CONTROL_API_DOCS -> false (registry)
  OK    COST_ALERT_ENFORCE_ENABLED -> false (registry)
  OK    DURABLE_RELAY_ENABLED -> false (registry)
  OK    HARNESS_FLOW_DRIVE -> false (registry)
  OK    HARNESS_LEVEL3_GUARD -> false (registry)
  OK    MANAGER_ADVANCEMENT_GATE -> false (registry)
  OK    MANAGER_ROLE_ENABLED -> false (registry)
  OK    MANAGER_TOOLS_ENABLED -> false (registry)
  OK    QUOTA_COORDINATOR_ENABLED -> false (registry)
  OK    QUOTA_DIGEST_TELEGRAM_ENABLED -> false (registry)
  OK    QUOTA_PREWARM_ENABLED -> false (registry)
  OK    RESPAWN_ON_RESTART_ERROR_DISABLED -> false (registry)
  OK    RESTART_CONTEXT_RESTORE_DISABLED -> false (registry)
  OK    RESTART_LOST_SESSION_FORK_DISABLED -> false (registry)
  OK    RESTART_NOTIFY_DISABLED -> false (registry)
  OK    REVIEW_EMITTER_ENABLED -> false (registry)
  OK    SPEC_AUTHORING_ENABLED -> false (registry)
  OK    TOOL_PROFILES_DYNAMIC -> false (registry)
  OK    TOOL_PROFILES_OVERLAY -> "" (registry)
  OK    TRANSIENT_PROVIDER_RESUME_ENABLED -> false (registry)
  OK    TURN_QUEUE_ENROLLMENT_ENABLED -> false (registry)
  OK    WORKER_MEMORY_WATCHDOG_DISABLED -> false (registry)
migration complete. Verify with 'ops_flag.sh list'.
exit=0
$ post-migrate state of a sample
{"flag_name":"CASE_CONTINUATION_ENABLED","value":true,"raw_value":"1","source":"registry"}
{"flag_name":"CASE_RESPAWN_REQUIRES_APPROVAL","value":true,"raw_value":"1","source":"registry"}
{"flag_name":"CLAUDE_SDK_MAX_BUDGET_USD","value":2.5,"raw_value":"2.5","source":"registry"}
{"flag_name":"CLAUDE_SDK_MAX_TURNS","value":null,"raw_value":"","source":"registry"}
{"flag_name":"TOOL_PROFILES_DYNAMIC","value":false,"raw_value":"0","source":"registry"}
{"flag_name":"TOOL_PROFILES_OVERLAY","value":"","raw_value":"","source":"registry"}
{"sdk_max_turns":null,"sdk_max_budget_usd":2.5}
$ ops_flag.sh migrate (second run)
already in registry (skipping): CACHE_HEARTBEAT_ACTIVE CACHE_HEARTBEAT_CHECKPOINT_ENABLED CACHE_HEARTBEAT_OBSERVE CASE_CONTINUATION_ENABLED CASE_QUOTA_RESUME_AUTO CASE_QUOTA_RESUME_ENABLED CASE_RESPAWN_REQUIRES_APPROVAL CLAUDE_SDK_MAX_BUDGET_USD CLAUDE_SDK_MAX_TURNS CONTROL_API_DOCS COST_ALERT_ENFORCE_ENABLED DURABLE_RELAY_ENABLED HARNESS_FLOW_DRIVE HARNESS_LEVEL3_GUARD MANAGER_ADVANCEMENT_GATE MANAGER_ROLE_ENABLED MANAGER_TOOLS_ENABLED QUOTA_COORDINATOR_ENABLED QUOTA_DIGEST_TELEGRAM_ENABLED QUOTA_PREWARM_ENABLED RESPAWN_ON_RESTART_ERROR_DISABLED RESTART_CONTEXT_RESTORE_DISABLED RESTART_LOST_SESSION_FORK_DISABLED RESTART_NOTIFY_DISABLED REVIEW_EMITTER_ENABLED SPEC_AUTHORING_ENABLED TOOL_PROFILES_DYNAMIC TOOL_PROFILES_OVERLAY TRANSIENT_PROVIDER_RESUME_ENABLED TURN_QUEUE_ENROLLMENT_ENABLED WORKER_MEMORY_WATCHDOG_DISABLED 
not registry-writable, cannot migrate (bootstrap/worker-side, left on env/default): CLAUDE_SKIP_PERMISSIONS CONTROL_API_ENABLED GUARDED_WRITE MESH_EMBEDDED_SERVER MESH_ENABLED MESH_SHADOW_WRITE OPENCODE_SERVER_ENABLED TELEMETRY_DETAILED_EVENTS TELEMETRY_ENABLED WORKER_ACCEPT_UNPINNED WORKER_CANARY WORKER_REAP_STALE_SESSIONS 
nothing to migrate — all writable flags already in registry.
exit=0
```
