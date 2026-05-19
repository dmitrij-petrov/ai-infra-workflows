# Governance Layer

The component that keeps autonomous workflow execution within your team's agreed operational boundaries. Every regulation your team has written down — capacity thresholds, reboot procedures, decision authority, cross-team coordination rules — becomes a runtime check that runs before a workflow step executes.

---

## Design Principle

**Governance checks are synchronous gates, not async logs.**

```
[Workflow step requested]
        ↓
[Governance Layer: classify step type]
        ↓
[Run applicable rule checks]
        ↓
    ┌───┴───┐
  PASS    FAIL / WARN
    ↓         ↓
[Execute]  [Block or escalate to human gate]
```

A step is blocked until all applicable checks pass. A check that returns WARN surfaces a human gate at the Decision Gate (Layer 5) — the agent presents the warning with the artifact and waits for explicit approval before proceeding.

---

## Rule Set

The six rules below are based on operational regulations commonly found in infrastructure engineering teams. Thresholds are drawn from a real-world regulation set; adapt them to your organisation's definitions.

---

### G1 — Capacity Check

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

### G3 — Role & Decision Authority

**Applies to:** any action that requires a specific decision-maker's approval before proceeding.

| Action domain | Decision authority | Gate |
|---|---|---|
| Code & Delivery (local scope) | Individual engineer — Consent: "Is it safe to try?" | PASS on Consent |
| Local architecture change | Tech Lead — competence-based decision | Require Tech Lead confirmation |
| Cross-team architecture | Engineering Manager + Tech Leads — Bubble-Up | BLOCK until EM+TL sign-off |
| Process & rituals | Scrum Master / Flow Guardian | Require sign-off |
| Vision, strategy, budget | Director / Engineering Director | BLOCK until Director approval |

**Decision model:** Consent over Consensus. The question is not "does everyone agree?" but "is there a paramount objection backed by evidence?" If no such objection exists, proceed.

**Arbitrator Protocol:** If two decision-makers are blocked on each other, escalate to the person with deepest domain competence (not seniority). Their decision stands; both parties commit.

**Bubble-Up escalation path:** Cell → Tech Lead → Engineering Manager → Director. A decision escalates only as far as it needs to go to find a resolution.

---

### G4 — Cross-Team Coordination

**Applies to:** any workflow that spans more than one team or department.

| Check | Gate behaviour |
|---|---|
| Coordinator identified and named | Required; BLOCK if missing |
| Responsible point of contact per team identified | Required; BLOCK if any team unrepresented |
| Key agreements documented in the team's issue tracker | Required before technical work begins |
| MVP scope defined; iterative delivery model agreed | Required |

**Work model:** Deliver the minimum viable product first, obtain stakeholder approval, then deliver in short iterations with approval after each. All key agreements are documented — not communicated verbally.

**Escalation path:** If coordinator or team representative is missing → escalate through the team's meta-coordination layer before technical work begins.

---

### G5 — Automation Eligibility

**Applies to:** any task being performed manually that has been performed manually before.

| Condition | Gate behaviour |
|---|---|
| Task performed manually for the **first time** | PASS; log the run |
| Task performed manually for the **second time** | WARN: "This task has been run manually before. Build an automation or agent skill before the third run." |
| Task sending a priority notification via direct message (not P0/P1) | BLOCK; route through the team's coordination channel |

**The second-run rule:** First time, doing it manually is fine. Second time, build the automation. Third time should be zero-touch. This rule enforces the shift from reactive manual operations to a self-improving automated system.

---

### G6 — Knowledge Artifact Gate

**Applies to:** any design work or new implementation.

| Check | Gate behaviour |
|---|---|
| Solution document exists describing approach, trade-offs, and expected outcome | Required before moving from Design to Build; WARN if missing |
| Key decisions documented in the team's knowledge base | Required for all decisions with cross-team or long-term impact |

**Rationale:** Design is cheap; rebuilding after wrong assumptions is expensive even with AI assistance. A solution document is not bureaucracy — it is the thinking that prevents rework. The gate enforces: think first, build fast.

---

## Configuring for Your Organisation

The six rules above define the **structure** of the Governance Layer. The **thresholds** (capacity percentages, notification lead times, role boundaries) should match your organisation's own regulations.

To adapt this layer:

1. **Map your regulations** to the six rule categories (G1–G6)
2. **Replace thresholds** with your organisation's agreed values
3. **Add rules** for any regulation category not covered by G1–G6 (security compliance, data classification, change freeze windows, etc.)
4. **Store the rule set** in your Knowledge Base so agents can read it at session start

The Governance Layer is a skill or rule file, not a separate service. It loads at the start of each workflow session and is called by the orchestrator before each step.

---

## SLA Integration

If your organisation has defined SLA parameters (response time, resolution time by priority level), add them as a seventh rule:

**G7 — SLA Compliance**

| Check | Gate behaviour |
|---|---|
| Estimated execution time within SLA response window | PASS |
| Execution would breach SLA | WARN → escalate or pre-notify affected parties |
| Active SLA breach (customer-impacting) | HARD BLOCK on non-urgent work; switch to incident mode |

Until SLA parameters are formally defined, use the Capacity Management Critical threshold (≥90% for 10 min) as a proxy for SEV-1/SEV-2 classification, and require post-incident review within 5 business days.
