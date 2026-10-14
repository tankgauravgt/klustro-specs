# Feature 23 — Audit log and export

| | |
|---|---|
| Increment | 23 |
| Depends on | 01, 04, 05–22 |
| Unblocks | — |
| Spec refs | §19.1, §13.3, §7.2, §21 |

## Goal

Make every action in Klustro answerable after the fact, for customers answering their own security questions and for Klustro answering questions about its own staff. Success criterion 2 — every action is controlled and audited — is only met once this is readable by the customer.

The audit spine itself was built in Feature 01; this increment gives it a customer-facing surface, retention and export.

## In scope

- The query, filter and export surface for the audit log.
- Retention by plan.
- The catalogue of what is audited, and the guarantee that it is complete.
- Staff action visibility rules.

## Out of scope

- The audit event schema and emission mechanics (Feature 01).
- Log, metric and trace observability (Feature 30).
- SIEM-style streaming export (later).
- Tamper-evidence mechanisms beyond append-only storage, unless required later.

## Requirements

**Coverage**

R1. The audit log receives an event for every API action, every authorization denial, every JIT activation and approval, every policy run and drift event, every lifecycle transition, and every staff action performed in the admin console.
R2. Each event carries: id, time, organisation, actor principal and actor type, the channel the action arrived through (console, API, CLI, SCIM or system), the action name, the scope, the target, the result (allowed, denied or failed), a reason where applicable, the source address, user agent, request id and metadata.
R3. System-initiated actions appear with a system actor rather than being omitted.
R4. Metadata never contains secrets, credentials or token values; this is structurally enforced by the emission path, not by convention.
R5. A completion guarantee exists for the catalogue in R1: each of those event classes is enumerated and covered by a test asserting that performing the action produces an event.

**Immutability**

R6. The audit log is append-only across every interface. No API, console, staff tool or internal job can modify or delete an entry.
R7. Retention expiry is the only mechanism that removes entries, and it is driven by plan and by the legally required retention rules, not by any user action.
R8. Organisations cannot delete their own audit history.

**Read and filter**

R9. Reading the audit log requires a dedicated action.
R10. The log can be filtered by time range, actor, action, scope, target, result and channel.
R11. A customer can answer concrete questions such as who deleted a specific resource, who was denied, and which policy runs failed.
R12. The log is scoped to the reader's organisation and never exposes another tenant's events.
R13. A principal's access to the log is limited by their scope: a project-scoped reader sees only that project's events.
R14. Entries are readable individually with full detail, and the detail view never reveals secret material.

**Export**

R15. The audit log can be exported in a machine-readable flat format for a chosen range.
R16. Large exports are produced asynchronously and delivered as a durable, retrievable artifact rather than in a single response.
R17. Export generation and download are themselves audited.
R18. Exports are scoped to exactly the same events the requester could read interactively.

**Retention**

R19. Retention follows the plan: a shorter period on lower plans, longer on higher ones, and contractual on Enterprise.
R20. The retention period applicable to an organisation is visible to its administrators.
R21. Backfilled or late-arriving events respect the same retention rules as any other event.
R22. Audit records required for billing or tax legal retention survive organisation deletion.

**Staff actions**

R23. Every staff action is recorded with the individual staff member's identity, never a shared or generic service identity.
R24. Staff actions taken in the admin console appear in the affected customer's audit log where they concern that customer's data, so a customer can see when Klustro acted on their account.
R25. Read-only impersonation is recorded as such and is distinguishable from the customer's own actions.

## Acceptance criteria

- Performing any action in the R1 catalogue produces exactly one audit event with the actor, channel, scope and result.
- No interface exists that can alter or delete an audit entry.
- A customer can answer "who deleted this cluster, when, and through which channel?" from the console and the CLI.
- A project-scoped reader cannot see another project's events, and no reader can see another organisation's events.
- An export contains exactly the events the requester is entitled to and is itself audited.
- Retention expiry removes old entries according to plan and never removes billing or tax records still under legal retention.
- A customer can see that Klustro staff acted on their account.
