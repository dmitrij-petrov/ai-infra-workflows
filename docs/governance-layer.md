# Governance Layer

The component that keeps autonomous workflow execution within your team's agreed operational boundaries. Every regulation your team has written down — capacity thresholds, access requirements, maintenance windows, response SLAs, change gates — becomes a runtime check that runs before a workflow step executes.

---

## Design Principle

**Governance checks are synchronous gates, not async logs.**

```
[Workflow step requested]
        ↓
[Governance Layer: classify step type]
        ↓
[Run applicable regulation checks]
        ↓
    ┌───┴───┐
  PASS    FAIL / WARN
    ↓         ↓
[Execute]  [Block or escalate to human gate]
```

A step is blocked until all applicable checks pass. A check that returns WARN surfaces a human gate at the Decision Gate (Layer 5) — the agent presents the warning with the artifact and waits for explicit approval before proceeding.

---

## Regulation Examples

The examples below are drawn from the taxonomy described in the [AI Governance Patterns](https://github.com/dddeeemmm/ai-governance-patterns) framework. They illustrate what well-formed operational regulations look like when retrieved at runtime. Replace or extend them with regulations from your organisation's knowledge base — the number and content are determined by what your team has encoded in the KB, not by this document.

---

### G1 — Capacity Headroom Policy

**Applies to:** any operation that provisions, scales, or deallocates infrastructure resources.

| Check | Gate behaviour |
|---|---|
| Post-operation headroom ≥ 20% | PASS |
| Post-operation headroom < 20% | WARN → human gate |
| DTE ≤ TTP + 30 days | WARN → human gate + procurement suggestion |
| Action threshold sustained (≥80% utilisation for 15 min, or DTE ≤ 30 days) | BLOCK; require scale/optimise first |
| Critical threshold sustained (≥90% for 10 min, or DTE ≤ 7 days) | HARD BLOCK; mitigate first; same-day plan required |
| Autoscaling response after scaling action | Verify response < 2 minutes |

**Key formulas:**

- **Days to Exhaust (DTE):** `(Capacity_at_Action_Threshold − Current_Utilisation) ÷ Daily_Growth_Slope`
- **Procurement trigger:** `DTE ≤ Time_to_Procure + 30 days`

**Cold Spare rule:** Minimum N+1 per critical service; at least 1 unit per site or 2% of fleet. If a cold spare is consumed, flag for replenishment within 30 days.

**Operating targets:** Sustained utilisation 70–80%. Headroom ≥ 20% at all times per service and per site.

---

### G2 — Server Reboot Policy

**Applies to:** any operation that reboots a physical or virtual server.

| Check | Gate behaviour |
|---|---|
| Maintenance window approved by relevant stakeholders | Required; BLOCK if absent |
| Notification sent to Infra + Team Leads + Security ≥ 3 business days prior | Required; BLOCK if not confirmed |
| Alert suppression configured in monitoring system | Required; BLOCK if not confirmed |
| Pre-reboot checklist completed | Required; BLOCK if any item missing |
| Reboot order defined by service owners | Required |

**Pre-reboot checklist items:**
- Hostname / IP / FQDN documented
- Services, processes, and open ports recorded
- Network configuration reviewed (routing, interfaces, firewall rules)
- Configuration files backed up if relevant
- Service owners consulted on graceful shutdown requirements

**Post-reboot:** Governance Layer must verify that blackbox + whitebox checks pass before the step is marked complete and the server is returned to production status.

**Cadence enforcement:** Track last-reboot date per server. Raise an alert when a server exceeds 6 months without a reboot.

**Exception path:** If a reboot cannot be performed due to technical or business constraints, escalate to the Head of Infrastructure with documented reason, risk assessment, and mitigation plan. Exception must be reviewed at the next maintenance cycle.

---

### G3 — Production Access Policy

**Applies to:** any direct access to a production system (SSH, console, admin panel, database, or equivalent).

| Check | Gate behaviour |
|---|---|
| 2FA verified for the executing principal | Required; BLOCK if not confirmed |
| Access request references an active issue ticket | Required; BLOCK if no ticket linked |
| Documented purpose stated before session opens | Required |
| Credential or session not older than 8h | Required; WARN if near expiry; BLOCK if expired |

**Post-access:** Access duration, actions taken, and outcome are recorded in the access log before the session is closed. The access record references the originating ticket.

**Exception path:** Emergency break-glass access (active incident, no time to open a ticket) requires notification to the Head of Infrastructure and Security within 1h, with documented reason and actions taken.

---

### G4 — Planned Maintenance Notification

**Applies to:** any planned maintenance that affects services used by other teams.

| Check | Gate behaviour |
|---|---|
| Coordinator identified and named | Required; BLOCK if missing |
| Affected teams notified ≥48h in advance | Required; BLOCK if not confirmed |
| Maintenance window approved by relevant stakeholders | Required |
| Rollback plan documented and reviewed | Required |

**Work model:** Notification must be sent through the team's coordination channel, not via direct message. All key agreements are documented in the issue tracker before technical work begins.

**Exception path:** Emergency maintenance (active outage mitigation) is excepted from the 48h requirement; notify affected teams within 2h of the maintenance start with reason and estimated duration.

---

### G5 — Incident Response SLA

**Applies to:** any workflow triggered by a customer-impacting or service-degrading incident.

| Priority | Condition | Required response | Gate behaviour |
|---|---|---|---|
| P1 | Service down, customer-impacting | First response ≤ 15 min; mitigation plan ≤ 30 min | HARD BLOCK on unrelated work; switch to incident mode |
| P2 | Degraded service, partial impact | First response ≤ 1h | WARN: SLA timer active |
| P3 | Non-impacting, investigation needed | First response ≤ next business day | PASS; log |

**SLA timer:** Starts from first report (ticket creation or alert trigger), not from when the team picks it up.

**Post-incident:** P1 and P2 incidents require a post-incident review documented in the knowledge base within 5 business days.

---

### G6 — Change Gate Regulation

**Applies to:** any change to production infrastructure configuration, service topology, or data schema.

| Check | Gate behaviour |
|---|---|
| Change documented in the issue tracker with scope and expected impact | Required; BLOCK if absent |
| Rollback plan defined with estimated rollback duration | Required; BLOCK if absent |
| Impact assessment includes capacity and dependency review | Required |
| Rollback duration ≤ 30 min | PASS; proceed |
| Rollback duration > 30 min | WARN → Irreversibility Gate (see ai-approval-gates) |

**Staged delivery:** Changes that affect multiple systems must be delivered in stages, with validation between each stage. Each stage is a separate governed step.

**Exception path:** Emergency changes made during active incident mitigation are excepted; create a change record in the issue tracker within 2h of the change with reason, actions taken, and outcome.

---

## Configuring for Your Organisation

The regulation examples above illustrate six categories from the [AI Governance Patterns taxonomy](https://github.com/dddeeemmm/ai-governance-patterns): Resource & Capacity (G1), Infrastructure Lifecycle (G2, G6), Security & Access (G3), Cross-team Collaboration (G4), and Operational SLA (G5). Your organisation's regulation KB may contain more, fewer, or different regulations in each category.

To adapt this layer:

1. **Identify your regulations** by mapping them to the [seven taxonomy categories](https://github.com/dddeeemmm/ai-governance-patterns/docs/framework.md): Operational SLA, Resource & Capacity, Infrastructure Lifecycle, Security & Access, Cross-team Collaboration, Team Organization, Templates & SOPs
2. **Replace thresholds** in the examples above with your organisation's agreed values
3. **Add or remove regulation examples** to match what is actually encoded in your KB
4. **Store the regulation set** in your Knowledge Base so agents can retrieve applicable regulations at execution time rather than relying on a hardcoded list

The Governance Layer is a skill or rule file, not a separate service. It loads at the start of each workflow session and is called by the orchestrator before each step.
