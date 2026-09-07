# Assignment & Roster

How to match missions to agents, squads (teams), or people. The roster is **dynamic** — it grows and changes. Never rely on a memorized roster; always consult the live system before assigning.

## Roster Source

**Primary source — live query.** The system itself is the roster. At the start of every planning cycle, run:

```bash
nino agent list --output json | jq '[.[] | {name, id, runtime_id, permission_mode, skills: (.skills | map(.name))}]'
```

This yields current agents with their bound skills — the closest thing to declared capabilities. Use `nino agent tasks <id> --output json` for an agent's current load.

**Secondary source — performance notes.** The live query tells you what agents *can* do, not how well they did it. Keep a knowledge-base note (`agent-roster-notes`) with per-agent observations: strengths, failure patterns, preferred task granularity, constraints discovered in practice. Update it at every project closure (workflows.md §8). Never put secrets or one-off task state in it.

Interpretation guide:

- **skills** → capability match (an agent bound to a PPT skill gets deck missions, not data pipelines).
- **permission_mode / runtime_id** → where and how the agent runs; relevant if missions have environment requirements.
- **current tasks** → load check before assigning overlapping dates.

## Matching Procedure

For each leaf mission, in order:

1. **Capability match** — the assignee's capabilities must cover the mission's required skills. A partial match is acceptable only if the gap is minor and noted in the plan.
2. **Constraint check** — reject any candidate whose constraints conflict with the mission.
3. **Load check** — prefer the candidate whose existing in-flight/waiting missions don't overlap this mission's dates. One accountable owner per mission; don't stack simultaneous missions on one agent unless the system supports parallel execution for that agent.
4. **Type fit** — use a squad when the mission genuinely needs parallel multi-skill work; use a person for human-in-the-loop gates, approvals, and work requiring human judgment or credentials; agents for everything else.
5. **Tie-break** — prefer past strong performance (roster notes), then lower current load.

If no candidate is a viable match: flag it in the plan with a concrete recommendation (add an agent with capability X, upskill, or descope the mission) — do not force a bad assignment silently.

## Parents and Squads

- Parent (coordination) missions may be assigned to the coordinator itself or left to the roll-up rule; they don't need an executor.
- When a squad owns a mission, name the squad as owner; internal task-splitting within the squad is its own business unless the user asks otherwise.

## Reassignment

Reassignment is a normal replanning tool (slips, blocks, failed reviews, new roster members). When reassigning: update the mission's owner, note the reason in the change summary, and update roster notes after completion.
