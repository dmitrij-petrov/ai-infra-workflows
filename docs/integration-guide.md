# Integration Guide

How to adopt the Autonomous Infrastructure Workflow Architecture in an existing engineering team. This guide covers three integration paths — from a single-component addition to a full eight-layer deployment — and provides a concrete end-to-end worked example.

---

## What You Are Adding

The architecture proposes three things. You can adopt them independently:

| Component | What it adds | Where it fits |
|---|---|---|
| **Governance Layer** | Regulation-based checks before each production step | Layer 3 |
| **Approval Gate Pattern** | Consistent human gates around irreversible actions | Layers 2, 5 |
| **Eight-layer model** | Shared vocabulary and structure for the full system | All layers |

Start with the Governance Layer. It is the highest-leverage addition and requires no existing tooling to change.

---

## Integration Path A — Add Governance to an Existing Workflow Engine

**Precondition:** You have a workflow orchestration system (any: n8n, custom scripts, Claude Code skills, Jira automation) that executes infrastructure tasks.

### Step 1: Classify each workflow step

Before adding checks, map each step in your workflow to one or more of the applicable governance regulations:

| Step type | Applicable checks |
|---|---|
| Provision / scale / deallocate resources | G1 (Capacity) |
| Reboot any server | G2 (Reboot Policy) |
| Direct access to a production system (SSH, console, DB) | G3 (Access Policy) |
| Planned maintenance affecting other teams | G4 (Maintenance Notification) |
| Incident-triggered workflow | G5 (Incident SLA) |
| Any change to production config, topology, or schema | G6 (Change Gate) |

### Step 2: Add a governance pre-flight to your entrypoint

Add a session-start rule to your workflow agent:

```markdown
## Governance Pre-Flight

Before executing any step that touches a production system, run the applicable
Governance Layer checks from governance-layer.md.

PASS: proceed.
WARN: surface the warning at the next human gate with the full context.
BLOCK: stop and report the blocking condition; do not proceed without explicit approval.
```

### Step 3: Map checks to existing gates

If your workflow already has approval checkpoints, governance checks slot into them without adding new stops:

| Existing gate | Add governance checks |
|---|---|
| Approach approval | G1 (capacity impact of decision), G6 (change documented with scope) |
| Plan approval | G1 (final capacity), G2 (if reboots in plan), G4 (if multi-team impact) |
| Result confirmation | G1 (post-change headroom), G2 (post-reboot validation) |

### Step 4: Check G5 and G6 at task intake

G5 (Incident SLA) fires as soon as an incident-triggered workflow starts: classify the incident priority and start the SLA timer immediately. G6 (Change Gate) fires before the first production-touching step: confirm the change is documented in the issue tracker before execution begins.

- G5: Is this an incident? If yes, what priority? The SLA timer starts from first report.
- G6: Is this change documented with scope, expected impact, and rollback plan?

These are intake-level checks, not step-level checks.

---

## Integration Path B — Add the Execution Model to Manual Operations

**Precondition:** Your team currently handles operational tasks ad-hoc (Slack messages, direct SSH, manual Jira updates).

The execution model ([ai-operational-execution](https://github.com/dddeeemmm/ai-operational-execution)) adds structure without requiring a workflow engine:

```
1. Engineer describes the task to the AI agent
2. Agent assembles context: goal, affected systems, rollback, risk, Governance results
3. Decision Gate: agent presents Context Brief → engineer approves
4. Agent executes the approved plan step by step
5. Validation Gate: agent presents outcome → engineer confirms Validated / Falsified / Inconclusive
```

**What this replaces:** the pattern where an engineer SSH's into a server, makes a change, and hopes it worked. The agent assembles the context, the engineer approves it, the agent executes it, the engineer validates the result. Every production change has a before-and-after record.

**Governance integration:** The Context Brief at the Decision Gate includes the Governance Layer results. If G1 shows headroom will drop below 20%, the engineer sees it before approving.

---

## Integration Path C — Adopt the Memory Layer

**Precondition:** Your team repeatedly solves the same problems from scratch because knowledge is in people's heads.

The Memory Layer gives every workflow run access to the team's accumulated knowledge — root causes, workarounds, failure modes, verification techniques.

### Setting up the Shared Memory server

The Memory Layer is a standard MCP server (JSON-RPC 2.0 over HTTP). Any MCP-compatible agent can read from and write to it.

**Read (before starting work):**
```
search memory for "<description of the problem>"
```
Returns semantically similar past learnings, even if the terminology differs.

**Write (after completing work):**
Each workflow run produces a `learnings.jsonl` file. A Stop hook publishes entries automatically:
- SHA256 dedup (idempotent — running the hook twice does not duplicate entries)
- Secret redaction gate (tokens, passwords, keys are filtered before publishing)
- Structured fields: `insight`, `what_done`, `root_cause`, `rollback`, `blast_radius`, `refs`

**What gets published:** Not raw logs. A structured record of what was learned — the insight that would have saved time if it had been available at the start.

---

## Integration Path D — Full Eight-Layer Deployment

**Precondition:** Your team is ready to operate AI agents as first-class team members for infrastructure work.

Deploy the layers in this sequence (each layer is independently useful; later layers build on earlier ones):

| Order | Layer | What to set up |
|---|---|---|
| 1 | Knowledge Base (8) | Regulations in a searchable knowledge base; runbooks; scheduled report automations |
| 2 | Governance Layer (3) | Governance checks as a skill/rule file; session-start pre-flight |
| 3 | Execution Layer (5) | Task execution model with Decision and Validation gates |
| 4 | Agent Layer (4) | Specialised agents; team composition files; routing rules |
| 5 | Orchestration Layer (2) | Epic-driven workflow; three gates; agent team composition |
| 6 | Memory Layer (6) | Shared memory server; Stop hook for auto-publish; `.mcp.json` config |
| 7 | Observation Layer (7) | Post-execution outcome verification; health signals per workflow run; feedback into Trigger and Knowledge Base layers |
| 8 | Trigger Layer (1) | Work request intake; ticket classification; alert routing; scheduled triggers |

You do not need all eight layers on day one. A team that starts with Governance (Layer 3) and the Execution Model (Layer 5) has already eliminated the two most common failure modes: unchecked production changes and invisible decision authority.

---

## Worked Example: Capacity Assessment Workflow

A capacity assessment is a good first workflow to implement because it is high-value, well-bounded, and demonstrates all three components (Governance, Approval Gates, Memory).

**Workflow:**

```
1. Trigger: ticket created ("assess capacity for cluster X")
2. Governance G1: check current utilisation before starting
   → if Critical threshold active: BLOCK, notify team
3. Agent: collect metrics
   - Query monitoring system: P95 utilisation, 13-week trend
   - Cross-reference with asset management: expected capacity
   - Compute DTE = (Capacity_at_Action − Current) ÷ Daily_Growth_Slope
4. Decision Gate: present findings to engineer
   → DTE, current headroom, recommendation (procure / optimise / monitor)
   → Governance G1 result: PASS / WARN / BLOCK
5. Engineer approves recommendation
6. Agent: execute (open procurement ticket, or apply optimisation)
7. Validation Gate: confirm execution result
8. Memory: publish learning (what the DTE was, what action was taken, what the outcome was)
9. Observation: update capacity dashboard; schedule next assessment
```

**Governance integration points:**
- Step 2: G1 check before starting (don't start work if Critical is active)
- Step 4: G1 check on post-action headroom (does the proposed action restore headroom?)
- Step 6: G6 check (procurement action documented in issue tracker with scope and expected impact)

**Why this is the right first workflow:** It is entirely read-collect-compute-report until the human approves at step 5. The risk is low. The value is immediate. And it forces the team to define G1 thresholds concretely — which then applies to all future resource operations.

---

## Worked Example: Incident Response

**Workflow:**

```
1. Trigger: P0/P1 alert or engineer declaration
2. Incident Commander (IC): classify severity, identify blast radius
   → Governance G5: P1 SLA timer started (first response ≤ 15 min); P0 escalation to Director
3. Agent: multi-source investigation
   → Monitoring: metric anomalies in time window
   → Change history: recent deploys, config changes
   → Ground truth: host-level checks via SSH (read-only)
   → Rank hypotheses by probability; present to IC
4. Decision Gate: IC approves mitigation plan
   → Governance G1: capacity impact of mitigation
   → Governance G6: mitigation change documented with rollback plan
5. Agent: apply mitigation (restart, config rollback, traffic shift)
6. Validation Gate: IC confirms service restored
7. Memory: publish root cause, timeline, what failed, what fixed it
8. Observation: postmortem → knowledge base; follow-up tasks → tracker
```

**Hard safety rule:** The investigator agent is read-only by default. Any mutating action (restart, config change, deletion) requires an explicit approval gate — no matter how confident the agent is about the fix. This is the Approval Gate Pattern applied to incident response.

---

## Summary: Minimum Viable Adoption

If you want to start today with minimal setup:

1. **Add the governance pre-flight** to your existing workflow agent's `CLAUDE.md` (or equivalent session context) — 5 minutes
2. **Run one capacity assessment** using the worked example above — demonstrates G1, approval gates, and memory in a single, low-risk workflow
3. **Connect one team member** to the Shared Memory server — even one person publishing learnings is better than zero

The full eight-layer architecture is the destination. Any layer you adopt today is independently valuable.
