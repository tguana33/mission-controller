# Mission Model Reference

The system's data model and the semantics the coordinator must respect.

## Mission Fields

| Field | Required | Notes |
|---|---|---|
| Title | Yes | Action-oriented, verb-first ("Draft launch announcement", not "Announcement"). |
| Project | Yes | Every mission belongs to exactly one project. |
| Start / End date | Yes | Required for Gantt rendering. Parent dates must fully span all children. |
| Status | Yes | One of the six statuses below. |
| Person in charge | Yes | An agent, a squad (team), or a person. Exactly one accountable owner per mission — the CLI assigns to one target per command. |
| Tags | Optional | Use for work stream, skill domain, priority. Keep a consistent tag vocabulary per project. |
| Knowledge base | Optional | Connect when the mission's work depends on specific reference material. |
| Parent / children | Optional | Hierarchy and roll-up completion (see below). |
| Prerequisites | Optional | Cross-mission start gating (see below). |

## Status Lifecycle

The UI exposes six statuses: **planning → waiting → in progress → awaiting review → done**, plus **stopped** (reachable from any active state).

Backend note: the `nino` CLI has been observed using `backlog`, `todo`, and `in_progress` (e.g., `backlog` = parked/not yet queued, `todo` = queued and triggering a run). The exact mapping between UI labels and CLI status strings — and where review/stopped live — must be confirmed at runtime: inspect real issues with `nino issue get <id> --output json` and check `nino issue status --help`. Do not guess status strings.

Expected transitions:

- `planning`: scope/dates/assignee still being defined. Leave planning only when the mission is fully specified.
- `waiting`: fully specified but blocked — prerequisite incomplete, or scheduled start not reached. The system auto-activates from here when prerequisites complete, so set waiting missions up correctly at launch time.
- `in progress`: actively being executed.
- `awaiting review`: work product complete, needs confirmation. This is the natural human-in-the-loop gate. On approval → `done`. On rejection → back to `in progress` with the feedback recorded, or replan.
- `done`: terminal. Children of a `done` parent must all be `done` (enforced by the coordinator even if the system does not).
- `stopped`: halted for any reason (deprioritized, blocked externally, superseded). Always record the reason. A stopped mission with dependents is a replanning trigger — dependents must be re-parented, re-sequenced, descoped, or the stop resolved.

Never skip `awaiting review` for missions the user marked as requiring confirmation.

## Dependency Semantics

Two distinct relationship types — do not conflate them:

1. **Parent/child (hierarchy)**: decomposition. A parent represents the aggregate of its children. Rule: a parent is complete only when ALL children are done. Parents typically do not need their own assignee for execution; their "work" is coordination. Use for phases, epics, deliverables with parts.
2. **Prerequisite (sequencing)**: B is a prerequisite of C means C cannot start until B is done; the system auto-starts C when B completes. Use for genuine input/output couplings (C consumes B's result) or hard resource ordering — not for vague "it would be nice" ordering, which over-constrains the schedule.

Combined pattern: A is parent of B and C, and B is prerequisite of C → B runs first, C auto-starts when B finishes, A completes when both are done.

## Graph Constraints

- The dependency graph must be a DAG: no cycles, direct or transitive.
- Every mission must trace to at least one root (a mission with no prerequisite).
- A mission may have multiple prerequisites (all must complete before it starts) and multiple children.
- Minimize prerequisites: each one is a serialization point and a failure-propagation path. Prefer parallelism where couplings allow.

## Views and What They Need

- **Gantt**: requires real dates on every mission and correct dependency links; slippage shows here first.
- **Kanban**: status-driven; keep statuses accurate or the board lies.
- **List**: field completeness matters (tags, assignees) for filtering.

Keep all three views trustworthy: dates, statuses, and links are the coordinator's source of truth, updated at every transition.
