# Autonomous Infrastructure Workflow Architecture

A reference architecture for AI-agent-driven infrastructure operations. Combines workflow orchestration, human approval gates, regulation-based governance, and shared team memory into a coherent eight-layer model.

---

## The Problem

Infrastructure teams face a structural contradiction: the work that AI agents can now do autonomously is exactly the work that carries the most operational risk. Giving agents unrestricted access to production systems is dangerous. Building approval chains that block every action recreates the coordination overhead the agents were supposed to eliminate.

The answer is not more process — it is **architecture**: a clear set of layers, each with a defined responsibility, that together allow agents to operate autonomously within boundaries that teams have already agreed on.

---

## Eight-Layer Model

```
┌─────────────────────────────────────────────────────────────┐
│  1. TRIGGER LAYER      Work enters the system               │
│                        Mission contracts · Tickets · Cron   │
├─────────────────────────────────────────────────────────────┤
│  2. ORCHESTRATION      Workflow engine coordinates agents   │
│                        INTAKE → DEBATE → [GATE] → EXECUTE   │
├─────────────────────────────────────────────────────────────┤
│  3. GOVERNANCE         Regulations enforced at runtime      │
│                        Capacity · Reboot · Role · Process   │
├─────────────────────────────────────────────────────────────┤
│  4. AGENT LAYER        Specialised agents execute work      │
│                        architect · sre · devops · security… │
├─────────────────────────────────────────────────────────────┤
│  5. EXECUTION LAYER    Human gates around production        │
│                        Decision Gate ⏸ → Execute → Validate │
├─────────────────────────────────────────────────────────────┤
│  6. MEMORY LAYER       Shared team knowledge                │
│                        Semantic search · Auto-publish       │
├─────────────────────────────────────────────────────────────┤
│  7. OBSERVATION        Closed-loop feedback                  │
│                        Outcome signals · Health · Iterate   │
├─────────────────────────────────────────────────────────────┤
│  8. KNOWLEDGE BASE     Static and dynamic context           │
│                        Regulations · Runbooks · Schedules   │
└─────────────────────────────────────────────────────────────┘
```

Data flows top-to-bottom for execution and bottom-to-top for feedback. The Governance Layer (3) is the only layer that touches all others — it runs checks before every step in layers 2, 4, and 5.

---

## Layer Descriptions

### Layer 1 — Trigger Layer

Work enters the system through three channels:

- **Mission contracts** — outcome-defined units of work, agreed by a stakeholder and executed by one IC. The lifecycle (Intake → Locked → Validated / Falsified / Inconclusive) provides a built-in validation framework.
- **Issue tickets** — classified on intake into task categories; matched to the appropriate workflow in the Orchestration Layer.
- **Scheduled triggers** — cron-based operational automations that run without a human initiating each run (reports, audits, capacity checks).

### Layer 2 — Orchestration Layer

The workflow engine sequences agent work and enforces human gates. The canonical pattern is an **epic-driven workflow**:

```
INTAKE → DEBATE → [GATE 1: approve approach] → DECOMPOSE → [GATE 2: approve plan] → EXECUTE → [GATE 3: validate result] → WRAP-UP
```

Each gate is a hard stop. The orchestrator presents an artifact (architecture decision record, sub-task list, execution result) and waits for explicit human confirmation. No auto-proceeding.

For teams of 10 specialised agents, composition into **agent teams** handles complex tasks that require multiple domains: an Architecture Review Board (architect + security + devops) debates approaches; an Incident Response team (SRE + networking + devops) investigates and mitigates.

### Layer 3 — Governance Layer

The layer that keeps autonomous execution within organisational bounds. Before any step that touches a production system, the Governance Layer runs a set of regulation-based checks. Checks are synchronous — a failing check blocks the step until the issue is resolved or a human explicitly accepts the risk.

Six check categories:

| Check | Trigger |
|---|---|
| G1 Capacity | Any resource operation (provision, scale, deallocate) |
| G2 Reboot Policy | Any server restart |
| G3 Role Authority | Any action requiring a specific decision-maker |
| G4 Cross-Team | Any workflow spanning more than one team |
| G5 Automation Eligibility | Any manual task being repeated for the second time |
| G6 Knowledge Artifact | Any design work or new implementation |

See [`governance-layer.md`](governance-layer.md) for the full rule set with thresholds.

### Layer 4 — Agent Layer

Specialised agents, each with a defined domain and a defined set of tools. Specialisation is key: no single agent handles everything. The recommended set:

| Agent | Domain |
|---|---|
| architect | System design, migrations, Architecture Decision Records |
| sre | SLOs, incidents, monitoring, capacity |
| devops-engineer | CI/CD, infrastructure-as-code, containers, automation |
| security-engineer | Threat surface, compliance review |
| programmer | Scripts, exporters, monitoring configurations |
| technical-writer | Runbooks, SOPs, postmortems |
| scrum-master | Epic coordination, clarifying questions |
| networking-engineer | Connectivity, routing, DNS |
| project-manager | Deadline tracking, overdue escalation |
| investigator | SSH-level investigation (read-only by default; approval gate before any mutation) |

Agents are declared in team composition files and invoked by the orchestrator, not by each other.

### Layer 5 — Execution Layer

How an individual workflow step is executed once the orchestrator hands it off:

```
Decision Gate ⏸ (human approves) → AI-Assisted Execution → Validation Gate ⏸ (human confirms outcome)
```

The gate at each end ensures that production changes are never made without a human in the loop. The execution step itself is fully AI-driven — the agent assembles context, applies the change, and reports results. The human reviews artifacts, not individual commands.

For automations with a full lifecycle, the **automation-builder** model extends this:

```
Design → Build → Test → [Irreversibility Gate ⏸] → Deploy → Observe → Iterate
```

Stages before the gate are reversible and can be chained automatically. The gate fires before Deploy (irreversible) regardless of how routine the change appears.

### Layer 6 — Memory Layer

A shared, searchable store of team learnings. Every workflow run that discovers something worth remembering — a root cause, a workaround, a failure mode, a verification technique — publishes it to the team's shared memory.

Key properties:

- **Semantic search:** query by description of the problem, not by keyword
- **Auto-publish:** a Stop hook fires at the end of each agent session; learnings from `work/<task>/learnings.jsonl` are published automatically
- **Deduplication:** SHA256 hash prevents duplicate entries
- **Secret redaction:** content is filtered before publishing
- **Protocol:** standard MCP server (JSON-RPC 2.0 over HTTP); works with any MCP client

The Memory Layer transforms institutional knowledge from a person-dependent resource into a team-wide asset that survives vacations, departures, and context switches.

### Layer 7 — Observation Layer

Post-execution outcome verification. After the Validation Gate (Layer 5) confirms that a step is complete, the Observation Layer checks whether the change achieved its intended effect over time.

The observation is specific to each workflow run: it measures the signal the change was intended to move — error rate, utilisation delta, latency, health check result — against the expected outcome. A healthy signal closes the loop. Drift or regression triggers a follow-up entry in the Trigger Layer (1) and is recorded in the Knowledge Base (8).

This is the "Observe → Iterate" step in the automation lifecycle described in Layer 5. See the worked example in [`integration-guide.md`](integration-guide.md) for how this step appears at the end of each workflow.

Observation outputs feed back into Layer 1 (follow-up triggers) and Layer 8 (Knowledge Base).

### Layer 8 — Knowledge Base

The static and dynamic context that all other layers query:

- **Regulations** — the team's agreed-on operational rules, stored in a knowledge base (e.g. a Slite collection). The Governance Layer reads from here at runtime.
- **Runbooks** — step-by-step procedures for known operations. Agents reference these rather than reconstructing procedure from first principles each time.
- **Scheduled operational outputs** — recurring analyses (weekly status reports, Jira reporting, OKR signals, knowledge-transfer plans) that synthesise the state of the system for leadership and for agents.

---

## Key Design Principles

**1. Governance is synchronous.** Regulation checks are gates, not logs. A step cannot proceed past a failing check. An audit trail that you read after something goes wrong is not governance.

**2. The Approval Gate Pattern is universal.** Three implementations (epic workflow gates, task-level Decision/Validation gates, automation lifecycle irreversibility gate) express the same principle at different scopes. They should look and behave consistently. See [`approval-gate-pattern.md`](approval-gate-pattern.md).

**3. Memory is shared.** Learnings from every workflow run are published to the team's shared memory — not trapped in individual sessions or in one person's head.

**4. Agents specialise; teams compose.** No single agent handles everything. Composition is explicit and declared. This prevents "do everything" agents that are hard to test, audit, or replace.

**5. Knowledge lives in artifacts.** Every design decision is documented before execution begins. If it is in someone's head, it dies when they are on vacation. The Knowledge Base, Memory Layer, and G6 check collectively enforce this.

**6. Speed and compliance are not opposites.** A well-designed governance layer makes teams faster, not slower — by eliminating the ambiguity that creates ad-hoc escalations and after-the-fact audits.
