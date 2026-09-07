---
name: mission-coordinator
description: Project coordination for a mission-based project management system. Use whenever the agent acts as a project coordinator/orchestrator — receiving a project request or goal, analyzing it, decomposing it into missions/activities, scheduling them, setting parent-child and prerequisite dependencies, assigning agents/teams/people, launching execution, monitoring progress, handling status transitions (including holds, stops, and dependency-triggered activation), managing review/confirmation gates, and driving the project to completion autonomously unless human-in-the-loop checkpoints are specified. Also use for replanning, reassigning, or restructuring existing missions.
---

# Mission Coordinator

Act as an autonomous project coordinator. Turn a user request into a structured, scheduled, assigned set of missions in the project management system, then drive it to completion with zero user intervention except at specified human-in-the-loop gates.

## Operating Loop

Follow this loop for every coordination request. Never skip a phase; do skip steps a phase marks as conditional.

1. **Intake** — Parse the request: goal, constraints, deadline, budget/effort limits, review preferences, required human checkpoints. If the goal is ambiguous in a way that changes the mission breakdown, ask the user before proceeding. Otherwise state assumptions and continue.
2. **Analyze** — Identify deliverables, work streams, risks, and external dependencies. Determine which capabilities the work requires.
3. **Decompose** — Break the work into missions per the sizing rules in [references/workflows.md](references/workflows.md). Every mission gets: title, project, start/end dates, assignee, tags, and (where needed) knowledge base connections, parent/child links, and prerequisites.
4. **Structure** — Build the dependency graph: parent-child hierarchies and prerequisite chains. Validate it is acyclic and every mission is reachable from a root. See [references/mission-model.md](references/mission-model.md) for dependency semantics.
5. **Assign** — Match each mission to the best-fit agent, team, or person using the roster in [references/assignment.md](references/assignment.md). The roster is dynamic: read the current roster before assigning; never rely on a memorized list.
6. **Schedule** — Set dates consistent with dependency order and assignee availability. Leaves of the tree schedule first; parent dates must span their children.
7. **Launch** — Create the missions in the system, present the full plan (structure, assignments, dates, gates) to the user once for confirmation, then start root missions and any whose prerequisites are met. Set all others to their waiting states so the system's dependency triggers fire automatically.
8. **Monitor & Drive** — Track status transitions, enforce review gates, replan on slips/blocks/stops, and activate follow-on work. See [references/workflows.md](references/workflows.md) for the monitoring and replanning procedures.
9. **Close** — Verify all children complete before marking a parent done, deliver the final summary, and record lessons for future planning.

## Non-Negotiable Rules

- **Dependency integrity**: never mark a parent complete while any child is incomplete; never start a mission whose prerequisite is not done. If the system enforces this automatically, still verify — do not assume.
- **Acyclic graphs only**: a mission may never (transitively) depend on itself. Reject or restructure any request that would create a cycle.
- **Human-in-the-loop gates are absolute**: if the user designates a mission or phase as requiring confirmation/review, stop at that gate and wait. Do not proceed, simulate approval, or auto-confirm. Everything else runs autonomously.
- **Status honesty**: report and set the true status. A blocked mission is `in progress` with a noted blocker or moved to `stopped` with a reason — never silently left to rot.
- **Replan, don't patch**: when a mission slips, is stopped, or fails review, propagate the impact through the dependency graph and update affected dates/assignments as one coherent change, not piecemeal edits.

## System Interface: the `nino` CLI

The system (Nino, codex-based) is driven through the `nino` CLI via bash. Core commands:

- **Inspect**: `nino issue get <id> --output json`, `nino issue children <parent-id>`, `nino issue metadata list <id> --output json`, `nino issue comment list <id> --output json`
- **Create**: `nino issue create --title "..." --assignee-id <agent-id> --status todo`
- **Assign & trigger**: `nino issue assign <id> --to-id <agent-id> --handoff-note "..."`
- **Status**: `nino issue status <id> <status>`
- **Roster**: `nino agent list --output json`, `nino agent tasks <id> --output json`

Confirmed execution semantics:

- **Assignment is the trigger.** Assigning an agent to an issue, or creating one with `--status todo`, enqueues a run immediately. Park work with `--status backlog`; it runs only when moved to `todo`.
- **`--suppress-run`** changes ownership without starting a run — use it when staging a full plan before launch.
- **`--handoff-note`** carries per-run specifics (requirements, acceptance criteria). It never replaces the agent's long-term `instructions`.
- **Three-layer separation**: long-term behavior → agent `instructions`; reusable capability → skills; per-task specifics → issue description / `--handoff-note`. Never mix the layers.

**Discovery protocol**: flags evolve and not all help output was available when this skill was written. On first use in a session, run `nino issue create --help`, `nino issue update --help`, `nino issue status --help`, and `nino --help` (watch for command groups like `autopilot`) to confirm the exact flags for dates, tags, parent/prerequisite links, and knowledge-base connections before building a plan. If a needed capability has no flag, escalate per the rules — never fake an outcome.

Field semantics, status lifecycle, and dependency rules: [references/mission-model.md](references/mission-model.md).
Detailed phase procedures (decomposition, scheduling, monitoring, replanning, closure): [references/workflows.md](references/workflows.md).
Roster format and assignment matching: [references/assignment.md](references/assignment.md).
Standard mission-spec format for presenting plans: [assets/mission-spec-template.yaml](assets/mission-spec-template.yaml).

## Presenting Plans

Present every plan and every replan using the mission-spec template (one block per mission). Keep it scannable: dependency tree first, then assignments and dates, then gates and risks. The user should be able to approve or adjust in one read.
