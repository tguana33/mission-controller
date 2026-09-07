# Coordinator Workflows

Detailed procedures for each phase of the operating loop.

## Contents

1. Intake & Clarification
2. Decomposition
3. Dependency Mapping
4. Scheduling
5. Launch
6. Monitoring
7. Replanning (slips, blocks, stops, failed reviews)
8. Closure

---

## 1. Intake & Clarification

Extract from the request: goal/deliverable, deadline, constraints, quality bar, review preferences, and human-in-the-loop checkpoints.

Ask the user only when a gap changes the mission structure (e.g., unclear deliverable, missing deadline that drives sequencing). For everything else, state assumptions explicitly in the plan and proceed — the user can correct at plan confirmation.

If the user names a human-in-the-loop checkpoint, encode it as a mission whose normal flow includes `awaiting review`, and never auto-pass it.

## 2. Decomposition

Sizing rules:

- A mission should be one assignable unit: one owner, one clear completion criterion, estimable in duration. If it needs "and then also..." in the description, split it.
- Decompose until leaf missions are executable directly by a single agent/team/person without further planning.
- Typical depth: 2–3 levels (project → phase/deliverable → task). Deeper only for genuinely large efforts.
- Each mission's completion criterion must be verifiable — define what "done" looks like before creation, because it drives the review gate later.
- Give every leaf a distinct deliverable; overlap between siblings signals a bad split.

## 3. Dependency Mapping

For each mission ask: "What must exist before this can start?" — that answer is its prerequisites, nothing more.

- Wire hierarchy first (parents/children), then prerequisites between leaves or subtrees.
- Check for cycles after wiring: walk each mission's prerequisite chain to a root.
- Identify the critical path (longest prerequisite chain) and flag it in the plan; it determines the earliest possible project end.
- Prefer prerequisite links over artificial date padding — let the system's auto-start do the sequencing work.

## 4. Scheduling

- Schedule backward from a hard deadline if one exists; otherwise forward from today.
- Children fit inside parent dates; prerequisites end before dependents start (leave buffer on the critical path).
- Respect assignee availability: an agent already owning an in-flight mission at the same time is a conflict — stagger, parallelize across agents, or reassign.
- Record schedule assumptions (effort estimates) in the plan so replanning can revisit them.

## 5. Launch

1. Present the complete plan once: dependency tree, per-mission specs (mission-spec template), assignments, dates, gates, critical path, top risks.
2. Stage the structure without triggering runs — create missions in `backlog`, or use `--suppress-run` when assigning — so nothing fires half-wired.
3. Wire all links (parents, prerequisites) and verify with `nino issue children <parent-id>` and `nino issue get <id> --output json`.
4. Launch: move root missions and anything with no unmet prerequisites to `todo` — the CLI enqueues runs immediately. Leave blocked missions parked/waiting; flip them to `todo` as their prerequisites complete (or trust the system's auto-start if the runtime handles dependency triggers — confirm this behavior on the first project).
5. Final verification pass: every mission's links, dates, assignees, and statuses landed as planned.

## 6. Monitoring

The coordinator has no push channel by default — monitoring is pull-based: poll `nino agent tasks <agent-id> --output json` for assigned agents and `nino issue get` / `nino issue children` for mission state. If the platform's `autopilot` (scheduled/triggered agent runs) is available, it is the right mechanism for a recurring coordination check — confirm via `nino autopilot --help`.

On each poll or check-in:

- Verify the transition is legal (see status lifecycle in mission-model.md).
- When a mission completes: check whether its completion unblocks dependents (confirm auto-start fired), and whether it was the last incomplete child of a parent (parent → done).
- When a mission hits `awaiting review` at a human gate: summarize the deliverable for the user, ask for confirm/reject, and record the decision.
- Watch for stalls: an `in progress` mission with no progress past expectation, or a `waiting` mission whose prerequisite is done but never started.

## 7. Replanning

Trigger → procedure:

- **Slip** (mission will miss its date): propagate along dependents, shift dates, re-check critical path, report the new project end date if it moved.
- **Block** (cannot proceed but may resume): keep `in progress` with the blocker recorded; look for a workaround path or resequence so other work proceeds.
- **Stop**: record the reason; then for each dependent — re-parent, re-sequence, descope, or escalate to the user if the stop invalidates the project goal.
- **Failed review**: return to `in progress` with feedback attached; decide whether the same assignee continues or the mission needs re-specification/reassignment.
- **Scope change from user**: treat as a mini intake (Section 1) against the existing graph; add/restructure missions coherently rather than bolting on.

Every replan ends with: updated graph, updated dates, and a short change summary to the user (what moved, why, new end date).

## 8. Closure

- Confirm all missions `done` (or explicitly `stopped` with accepted reasons) and parents rolled up correctly.
- Deliver a final summary: what was produced, actual vs. planned dates, gates passed, stops/blocks encountered.
- Record durable lessons (estimates that were off, assignment mismatches, recurring blockers) so the next project plans better.
