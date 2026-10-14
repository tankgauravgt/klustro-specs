# Feature 29 — Platform admin console

| | |
|---|---|
| Increment | 29 |
| Depends on | 01–28 |
| Unblocks | — |
| Spec refs | §21, §13.5, §13.9, §14.1, §10.13, §19.1 |

## Goal

Give Klustro's own staff the tools they need to support customers without reaching for database access. Every staff capability is explicit, permissioned and audited, so that operating the platform does not require breaking its own rules.

Until this exists, several capabilities specified elsewhere are stated but unimplementable: refunds, goodwill credit, postpaid approval, price list publication, lifecycle overrides and rollout control.

## In scope

- Staff access and permissioning.
- Customer lookup.
- Read-only impersonation.
- Credit grants and refunds.
- Postpaid approval.
- Price list management.
- Lifecycle overrides.
- Rollout control for renderer and agent.
- Abuse suspension.

## Out of scope

- Any capability that lets staff read customer secrets, cluster tokens, join token secrets or workload data. None exists.
- Any capability that lets staff reach a customer's cluster API. Klustro has no path to it.
- Break-glass into a customer's organisation as a member.
- Bulk data export for staff convenience.

## Requirements

**Staff access**

R1. Staff access is a separate identity and permission set from customer access and is never granted through a customer organisation role.
R2. Staff capabilities are individually permissioned; a support engineer does not automatically hold the ability to change prices or override lifecycle state.
R3. Every staff action is audited with the individual staff member's identity, never a shared or generic identity.
R4. Staff authentication is held to at least the same standard as customer authentication, including mandatory multi-factor authentication.
R5. Staff access to customer data is limited to what the specific capability requires.

**Customer lookup and inspection**

R6. Staff can locate a customer by organisation, user, billing account or network identifier.
R7. Lookup shows the state needed to support a customer: plan, lifecycle state, usage, balance or outstanding amount, projects, clusters and nodes, mesh connection status, last policy run and open drift events.
R8. Lookup never displays secret material, credentials or token values.
R9. Staff views of customer data are themselves audited.

**Impersonation**

R10. Read-only impersonation lets staff view the console as a specific customer user to diagnose a problem.
R11. Impersonation cannot perform write actions of any kind — this is enforced by the platform, not by the console UI.
R12. Impersonation is time-bounded and must be explicitly ended.
R13. Impersonation is prominently indicated in the console and is audited as impersonation, distinguishable from the customer's own actions, and appears in the customer's audit log.

**Money operations**

R14. Staff can grant credit to a wallet with a specified source, amount and, where relevant, expiry.
R15. Staff can refund purchased credit as a reversing ledger entry with a recorded reason.
R16. Staff can approve a postpaid arrangement with agreed payment terms and an agreed credit limit.
R17. Every money operation requires a reason and produces ledger entries that are indistinguishable in structure from ordinary entries, aside from their source.
R18. Staff cannot edit or delete a ledger entry. Corrections are always reversing entries.

**Price list management**

R19. Staff can publish a new price list version with an effective-from date.
R20. Publishing a version never alters already-rated history.
R21. A price list version's contents are immutable once published; a correction is a new version.
R22. Publishing requires a distinct capability from money operations.

**Lifecycle overrides**

R23. Staff can extend grace, pause enforcement, or force a lifecycle state transition, each with a mandatory reason and a bounded duration for the exception.
R24. An override expires automatically; it does not need to be remembered and undone.
R25. Overrides are visible to the customer organisation's administrators, and the reason is shown to Klustro staff.
R26. An override can never enable an action that violates the platform's lifecycle invariants — nothing stopped, no device removed, no policy rewritten.

**Rollout control**

R27. Staff can observe, pause, advance and roll back the staged rollout of the policy renderer and the agent version, including per-project pinning.
R28. A rollout pause caused by automatic validation failures is visible with the affected projects.
R29. Rollout actions are audited.

**Abuse suspension**

R30. Staff can suspend an account for terms-of-service violation.
R31. Abuse suspension is a distinct mechanism from billing lifecycle and is presented as such, so the two cannot be confused in reporting or in the customer's notifications.
R32. Abuse suspension requires a reason and is audited, and its effect is stated to the customer.

## Acceptance criteria

- A support engineer can diagnose a customer's mesh or billing issue entirely from the console, with no database access.
- Attempting a write action while impersonating is refused by the platform, not merely hidden by the console.
- A goodwill grant and a refund each produce correct reversing or originating ledger entries with a reason, and both appear in the customer's audit log.
- Publishing a new price list version changes future charges only.
- A lifecycle override expires by itself without staff follow-up.
- No staff capability anywhere can display a secret or reach a customer cluster's API.
