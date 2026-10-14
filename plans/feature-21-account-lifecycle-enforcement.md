# Feature 21 — Account lifecycle and enforcement

| | |
|---|---|
| Increment | 21 |
| Depends on | 01–20 |
| Unblocks | 22 |
| Spec refs | §14 (all), §5.3, §7.5 step 6, §13.5, §13.6, §17 |

## Goal

Apply the product's central promise: running out of money degrades the **management plane** gradually and predictably, and **never** takes down customer workloads, removes their network devices, or rewrites their network to destroy access.

This is the highest-risk increment in the platform. It is deliberately late so that the warnings, the ledger and the enforcement each exist and are tested before anything can be restricted.

## In scope

- The lifecycle state machine and its transitions.
- The exact effect of each state.
- Timers, thresholds and pause conditions.
- Recovery on payment.
- The interaction with authorization and token issuance.
- Overrides and audit.

## Out of scope

- Postpaid eligibility, credit limits and collections, which feed the state machine but are defined in Feature 22.
- Notifications, which are defined in Feature 20 and consumed here.
- Terms-of-service abuse suspension, which is a staff action defined in Feature 29 and is a separate mechanism from billing lifecycle.

## Requirements

**States and transitions**

R1. Five states exist: Active, Grace, Frozen, Suspended, Deleted.
R2. Exactly one component decides and writes lifecycle state. Every other component only reads it.
R3. Active is entered when a prepaid balance is above zero, or a postpaid account is within its credit limit with no overdue invoice.
R4. Grace is entered when a prepaid balance reaches zero or below, or a postpaid account exceeds its credit limit, or a postpaid invoice remains unpaid past its due date. Grace lasts a configurable period (default 7 days).
R5. Frozen is entered at the end of Grace. Frozen lasts until day 30 from the start of Grace.
R6. Suspended is entered at day 30 and lasts until day 60.
R7. Deleted is entered at day 60, after final notices at day 45 and day 55.
R8. Timers and thresholds are configurable platform values, not constants.
R9. Transitions are idempotent and driven by a schedule as well as by events, so a missed event does not strand an account in the wrong state.

**Effects**

R10. Grace: everything continues to work. The customer sees a persistent banner and receives daily reminders.
R11. Frozen: the management plane becomes read-only. New projects, clusters, nodes and join tokens are refused; upgrades and mesh rule changes are refused; kubeconfig tokens are still issued.
R12. Frozen must not degrade isolation: policy pipeline system runs and drift reversion continue running so the network's isolation is preserved.
R13. Suspended: everything Frozen does, plus Klustro stops issuing kubeconfig tokens, so access through Klustro ends within the token lifetime, and agents stop reconciling, so Klustro makes no further cluster changes.
R14. In Suspended the cluster, the network and customer workloads keep running untouched.
R15. Deleted: Klustro instructs agents to uninstall themselves, leaving cluster software and workloads running; deletes stored mesh credentials; and deletes organisation data except billing and tax records retained for legal obligations.
R16. Deletion must not modify the network policy or remove devices.

**Invariants**

R17. Klustro must never, for billing reasons, stop customer workloads, remove network devices, or rewrite the network policy to destroy access. This invariant must be enforced by design — no code path may express these actions — and verified by test.
R18. Existing authentication tokens remain valid until their own expiry; enforcement does not revoke them out of band.
R19. Enforcement applies to the whole billing account, and therefore to every organisation linked to it.

**Recovery**

R20. A top-up or a successful payment at any point before Deleted returns the account to Active immediately, after settling the outstanding amount.
R21. Recovery unblocks all management-plane operations without requiring any per-resource intervention.
R22. Outstanding usage accrued during degraded states is settled from the payment, never discarded.

**Exceptions**

R23. Transitions into Frozen are paused while the payment provider is unhealthy, so a customer is never restricted because Klustro could not take their money.
R24. Enforcement can be paused or Grace extended by Klustro staff with a recorded reason and an expiry for the exception.
R25. Every transition, override and recovery is audited with the cause that triggered it.

**Interaction with authorization**

R26. The authorization check consults lifecycle state and denies write actions with a distinct code while Frozen or Suspended.
R27. Denial codes distinguish "frozen" from "suspended" from "feature not in plan" from plain missing permission.
R28. Reads remain available in Frozen, so a customer can always see their account and pay.

## Acceptance criteria

- A clock-driven simulation moves an unpaid account through Active → Grace → Frozen → Suspended → Deleted on the configured timers, with the correct effect at each stage.
- Throughout the entire sequence, customer workloads keep running and network devices are never removed by Klustro.
- Frozen blocks new resources but still issues kubeconfig tokens; Suspended stops issuing them within one token lifetime.
- Frozen and Suspended still run system policy and revert drift.
- A top-up at any stage before Deleted restores full function immediately and settles arrears.
- A payment provider outage prevents entry into Frozen.
- No code path anywhere can stop a workload, remove a device or rewrite policy for a billing reason.
