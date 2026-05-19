# The Approval Gate Pattern

Three independent components in this architecture each implement the same fundamental principle: **stop before an irreversible action and require explicit human confirmation**. This document unifies them into a named, reusable pattern.

---

## The Pattern

```
[Preparatory work] → [Gate: present artifact + wait] → [Execution] → [Validation Gate: confirm outcome]
```

A gate is not a notification. It is a **hard stop**: the system does not proceed based on assumed approval, elapsed time, or inference from prior context. Explicit confirmation is required every time, at the moment of the gate.

---

## Three Implementations

### Implementation 1 — Epic-Driven Workflow

**Scope:** Multi-week work (epic → sub-tasks → execution)

```
INTAKE → DEBATE → [GATE 1: approve approach] → DECOMPOSE → [GATE 2: approve plan] → EXECUTE → [GATE 3: validate result] → WRAP-UP
```

Gate format:
```
=== CHECKPOINT [N]: <gate name> ===

[Present the artifact: architecture decision / sub-task list / execution result]

Approve? Type one of:
  "approve" — proceed to next phase
  "changes: <feedback>" — revise and return
  "stop" — halt, save state

Waiting for explicit confirmation. I will NOT proceed without it.
```

Three gates per epic, each at a decision point that is hard to reverse: committing to an architectural approach (Gate 1), creating work items in the tracker (Gate 2), and accepting the execution outcome (Gate 3).

---

### Implementation 2 — Task Execution Model

**Scope:** Individual operational task (minutes to hours)

```
Task Intake → Context Assembly → [Decision Gate ⏸] → AI-Assisted Execution → [Validation Gate ⏸] → Done
```

Decision Gate: the agent assembles full context (goal, affected systems, rollback path, risk level, Governance Layer results) and presents it to the human. The human approves or rejects the plan before any production action is taken.

Validation Gate: after execution, the agent presents the outcome. The human confirms one of: Validated / Falsified / Inconclusive. These states map directly to the mission-contract lifecycle terminal states.

**Reference:** [ai-opex](https://github.com/dddeeemmm/ai-opex)

---

### Implementation 3 — Automation Lifecycle

**Scope:** Full automation lifetime (design to production)

```
Design → Build → Test → [Irreversibility Gate ⏸] → Deploy → Observe → Iterate
```

Stages are classified as reversible or irreversible:

- **Reversible:** Design, Build, Test, Iterate (before live re-deploy)
- **Irreversible:** Deploy, Iterate (when modifying a live automation)

The gate fires before every irreversible stage regardless of how routine the change appears. Reversible stages chain automatically — the agent proceeds without waiting.

Done-criteria per stage are explicit and verified before progression. A stage cannot be declared complete without evidence that its criteria are met.

---

## Unified Properties

All three implementations share:

| Property | Behaviour |
|---|---|
| **Gate trigger** | Before an irreversible or high-blast-radius action |
| **Artifact requirement** | The gate presents a concrete artifact — not a summary |
| **Confirmation** | Explicit keyword from the human; no inference or timeout |
| **No auto-proceed** | The gate stays open until explicitly resolved |
| **Audit trail** | Gate resolution is recorded in the workflow artifact |
| **Reversibility classification** | Reversible stages run without gates; irreversible stages always gate |

---

## When to Apply

Apply the Approval Gate Pattern when:

1. The next step is **irreversible** — deploy, delete, migrate, publish, send
2. The **blast radius** spans multiple systems or teams
3. A **Governance Layer check** returned WARN, requiring human judgement
4. The workflow is crossing a **trust boundary** — production environment, external system, another team's scope

Do not apply the pattern for read-only operations, local reversible changes, or dry-run steps — these can proceed without gates.

---

## Anti-Patterns

These behaviours violate the pattern and must be explicitly prohibited in any implementation:

| Anti-pattern | Description |
|---|---|
| **Timeout-as-approval** | Proceeding after N minutes of silence |
| **Prior-turn approval** | Treating "approve the plan" as approving each subsequent irreversible step |
| **Context inference** | Inferring consent because the human did not object to a similar previous step |
| **Summary gate** | Presenting a description of what will happen instead of the actual artifact |
| **Batch approval** | Asking the human to approve multiple irreversible steps in a single confirmation |
