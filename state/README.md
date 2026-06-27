# Project State Schema Reference

## Purpose
`project_state.json` is the single source of truth for all project progress.
Main reads this file at the start of every turn to determine routing decisions.
Main writes to this file after every significant state change.

## Full Schema

```jsonc
{
  "meta": {
    "version": "1.0.0",           // schema version
    "last_updated": "ISO-8601",   // last modification timestamp
    "updated_by": "main"
  },
  "projects": {
    "<project_slug>": {
      "project_id": "<slug>_YYYY",
      "name": "Human-readable project name",
      "created_at": "ISO-8601",
      "current_phase": "PLANNING | DESIGN | IMPLEMENTATION | VERIFICATION | INTEGRATION | COMPLETE",
      "workspace_root": "projects/<slug>/",

      "architecture": {
        "node_count": 4,
        "node_order": ["node_a", "node_b", "node_c", "node_d"],
        "integration_plan": "PLAN/01_<task>_spec.md"
      },

      "sub_tasks": {
        "<TASK_ID>": {
          "type": "design | implement | verify | asset | research | format",
          "status": "PENDING | IN_PROGRESS | VERIFYING | PASSED | FAILED | LOCKED",
          "owner": "architect | forge | verify | illustrator | director | scribe",
          "description": "What this task does",
          "retry_count": 0,
          "max_retries": 3,

          "depends_on": ["TASK_100"],
          "blocked_by": [],

          "input_artifacts": [
            "PLAN/02_<node>_spec.md"
          ],
          "output_artifacts": [],
          "evaluation_spec": "PLAN/03_<task>_test_spec.md",

          "history": [
            {
              "timestamp": "ISO-8601",
              "event": "ASSIGNED | STARTED | COMPLETED | FAILED | RETRIED | LOCKED",
              "agent": "forge",
              "summary": "What happened",
              "artifact_paths": ["CODE/..."]
            }
          ]
        }
      },

      "circuit_breakers": {
        "<TASK_ID>": {
          "locked_at": "ISO-8601 or null",
          "locked_reason": "Max retries (3) exceeded: [specific reason] or null",
          "unlocked_by": null,
          "unlocked_at": null
        }
      }
    }
  }
}
```

## State Transitions

```
PENDING → IN_PROGRESS → VERIFYING → PASSED
                          ↓
                        FAILED → (retry_count < 3) → IN_PROGRESS
                          ↓
                        FAILED → (retry_count ≥ 3) → LOCKED → (user clears) → IN_PROGRESS
```

## Phase ↔ Owner Mapping

| Phase | Allowed Owners |
|-------|---------------|
| PLANNING | (Main only — no sub-agent) |
| DESIGN | architect, director, illustrator |
| IMPLEMENTATION | forge |
| VERIFICATION | verify |
| INTEGRATION | forge, verify |
| COMPLETE | (no active tasks) |

## Main's State Machine Logic

```
1. Read project_state.json
2. Find project with current_phase != COMPLETE
3. Find sub_task with:
   - status == PENDING (or FAILED with retry_count < 3)
   - depends_on all PASSED
   - owner matches current_phase
4. If no task found, advance current_phase
5. Spawn sub-agent with task context
6. On JSON Envelope response:
   - Update sub_task status, retry_count, history
   - Update artifact paths
   - If all_passed: status = PASSED
   - If FAILED: retry_count++, status = PENDING (or LOCKED if retry_count ≥ 3)
7. Write project_state.json
```

## Example: Active Project

```json
{
  "meta": {
    "version": "1.0.0",
    "last_updated": "2026-06-27T07:00:00Z",
    "updated_by": "main"
  },
  "projects": {
    "sample_api": {
      "project_id": "sample_api_2026",
      "name": "Sample REST API",
      "created_at": "2026-06-27T06:00:00Z",
      "current_phase": "IMPLEMENTATION",
      "workspace_root": "projects/sample_api/",
      "architecture": {
        "node_count": 3,
        "node_order": ["auth", "crud", "search"],
        "integration_plan": "PLAN/01_api_spec.md"
      },
      "sub_tasks": {
        "TASK_101": {
          "type": "design",
          "status": "PASSED",
          "owner": "architect",
          "description": "API architecture and spec design",
          "retry_count": 0,
          "depends_on": [],
          "input_artifacts": [],
          "output_artifacts": [
            "PLAN/01_api_spec.md",
            "PLAN/02_api_auth_spec.md",
            "PLAN/03_api_test_spec.md"
          ],
          "evaluation_spec": null,
          "history": [
            {"timestamp": "2026-06-27T06:30:00Z", "event": "COMPLETED", "agent": "architect", "summary": "API design complete"}
          ]
        },
        "TASK_102": {
          "type": "implement",
          "status": "VERIFYING",
          "owner": "forge",
          "description": "JWT auth module implementation",
          "retry_count": 1,
          "depends_on": ["TASK_101"],
          "input_artifacts": ["PLAN/02_api_auth_spec.md"],
          "output_artifacts": ["CODE/api/auth_handler.py"],
          "evaluation_spec": "PLAN/03_api_test_spec.md",
          "history": [
            {"timestamp": "2026-06-27T06:45:00Z", "event": "COMPLETED", "agent": "forge", "summary": "Auth module v1"},
            {"timestamp": "2026-06-27T06:50:00Z", "event": "FAILED", "agent": "verify", "summary": "Lint errors: 2, missing exception handling"},
            {"timestamp": "2026-06-27T06:55:00Z", "event": "COMPLETED", "agent": "forge", "summary": "Auth module v2 — lint fixed, exceptions added"}
          ]
        }
      },
      "circuit_breakers": {}
    }
  }
}
```
