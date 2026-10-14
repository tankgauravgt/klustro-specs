# Feature 15 — Mesh user rules and approvals

| | |
|---|---|
| Increment | 15 |
| Depends on | 01–14 |
| Unblocks | 16 |
| Spec refs | §6.9, §7.2, §10.4, §10.7, §10.8, §13.3 |

## Goal

Let customers open exactly the network paths they need between the things Klustro knows about — cluster roles, specific clusters, label groups and network members — without ever hand-editing the network policy.

The default remains closed. Every path a customer opens is explicit, reviewed, and reversible.

## In scope

- The rule model and its selector vocabulary.
- Rule validation.
- The rule lifecycle and four-eyes approval.
- Rendering of active rules and their tests.
- The plan diff surface for rule changes.

## Out of scope

- Multi-term (AND) selectors (later).
- Rule templates or a rule catalogue.
- Deny rules — the model remains allow-only with a default-deny baseline.
- Approval of anything other than user rules; system changes are never held for approval.

## Requirements

**Rule model**

R1. A rule has a source selector, a destination selector, a protocol, and a port or port range.
R2. Valid selectors are: a label, a cluster, a cluster with a role, and tailnet members. The first three may be used as either source or destination; tailnet members may be a source only.
R3. In this phase a selector allows exactly one term. A multi-term selector is rejected when the rule is saved, with a message explaining that only single-term selectors are supported.
R4. A rule must name a protocol (TCP, UDP or ICMP) and specific ports or port ranges. "All ports" is not permitted and is rejected.
R5. A rule records its author, its approver, its status and its id.
R6. Rules are project-scoped.

**Validation**

R7. A rule is rejected if its selector references a label, cluster or role that does not exist in the project.
R8. A rule is rejected if it is structurally invalid, before it reaches the policy pipeline.
R9. Validation messages name the offending selector term.
R10. Selecting a cluster by role is validated against the cluster's current membership and remains valid as membership changes.

**Lifecycle**

R11. Rule states are: draft, pending approval, active, retired.
R12. Without the organisation policy requiring approval, saving a rule makes it active immediately.
R13. With that policy enabled, saving a rule places it in pending approval.
R14. Approval requires a principal holding the approve action, and the approver must be a different principal from the author.
R15. A principal must not be able to approve its own rule by any route, including through a group it shares with itself.
R16. A pending rule is inert: the pipeline renders only active rules, so a pending rule never delays or blocks a system-driven run.
R17. A rule can be retired by a principal holding the write action; retiring it makes it inactive and triggers a run.
R18. Activation, approval, rejection and retirement are audited, including who acted and when.

**Rendering and testing**

R19. Active rules are rendered after system rules, in a stable order, so that identical rule sets always produce identical output.
R20. Each active rule contributes an acceptance test asserting that its source can reach its destination on its ports.
R21. A rendered rule must not weaken default isolation: traffic not covered by a system rule or an active user rule remains denied, and this is asserted by generated denial tests.
R22. A rule change that would fail rendering or testing applies nothing and pages on-call, exactly as a system-rule failure would.

**Visibility**

R23. A run's plan shows the added and removed grants together with the rule id and the author and approver names.
R24. The plan and the rendered policy are readable in the UI and through the CLI.
R25. The CLI can list, create, approve and retire rules.
R26. The rule list shows status, selectors, ports, author, approver and the revision that last included the rule.
R27. The UI shows pending rules prominently for principals able to approve them, and never shows a pending rule as if it were in force.

**Plan gating**

R28. Four-eyes approval is a plan-gated feature; without it, rules take effect immediately and the approval actions are unavailable.

## Acceptance criteria

- A rule referencing a label selector opens exactly that path and nothing else, verified by an allowed-port and a blocked-port test.
- With the approval policy enabled, an authored rule is pending and has no network effect until a different principal approves it.
- A principal cannot approve their own rule.
- Retiring a rule closes the path on the next successful run.
- A user rule is always visible in a change plan with its provenance.
- A malformed or multi-term selector is rejected at save time with a specific message.
