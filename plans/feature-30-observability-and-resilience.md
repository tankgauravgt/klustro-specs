# Feature 30 — Observability and resilience

| | |
|---|---|
| Increment | 30 |
| Depends on | 01–29 |
| Unblocks | — |
| Spec refs | §17, §19.2, §19.3, §10.15, §14.1, §22 |

## Goal

Make Klustro's own health and the customer's health visible and honest, and prove the platform's degradation behaviour matches its promises. This is the increment that turns "the control plane can be down without taking customer infrastructure with it" from a design claim into a tested property.

This is last because it measures and hardens everything before it, and because several resilience guarantees can only be tested once all the components that must survive an outage exist.

## In scope

- Health surfaces for nodes, clusters and projects.
- Klustro's own operational health and alerting.
- Provider status visibility.
- The degradation behaviour catalogue.
- Resilience verification.

## Out of scope

- Customer-facing log, metric and alert forwarding from their clusters (later).
- Customer-facing webhooks and event streams (later).
- A public status page.
- Capacity planning and autoscaling of Klustro's own infrastructure.

## Requirements

**Node health**

R1. A node's health shows online or offline, its collected facts, and its mesh transport type and round-trip time to each peer.
R2. A node's health shows certificate validity and whether the agent is running the expected version.
R3. A server node going offline escalates to a cluster-level quorum warning.
R4. Health data is presented with its timestamp, so a stale value is never mistaken for a current one.

**Cluster health**

R5. A cluster's health shows cluster component health, datastore member health, quorum status, each server's API advertisement status, and per-node software version.
R6. A cluster with an unhealthy datastore, quorum at risk, or a server whose API is not ready is shown as degraded, with the specific reason.
R7. Health is reported per cluster and aggregated per project, so a project's overall condition is visible without opening each cluster.

**Project health**

R8. A project's health shows mesh connection status and last successful validation, the last policy run and its outcome, whether a revision is pending, open drift events, and the count of nodes in each state.
R9. A project whose credentials have gone invalid is shown as degraded with a call to action naming the fix.
R10. A project with a policy run that could not be applied shows that a policy update is pending, rather than appearing healthy.

**Klustro's own health and alerting**

R11. Klustro's components expose health and key operational metrics for its own operators: policy run outcomes, drift rates, provider API error rates, payment provider error rates, metering completeness and lifecycle enforcement state counts.
R12. Conditions that indicate a Klustro defect rather than a customer problem — a render or validation failure, a ledger invariant breach, an idempotency-key collision, an integrity failure on an agent task — page Klustro's on-call.
R13. Alerts distinguish customer-caused problems from Klustro-caused problems, so on-call is not paged for a customer's expired credentials.
R14. An audit-entry or usage-event emission failure is itself detectable, so a silent gap in the record cannot go unnoticed.
R15. Metering completeness is monitored: missing hours for a connected node are detectable rather than merely absent from a bill.

**Provider status**

R16. The state of every external dependency Klustro relies on is visible to Klustro operators and, where it explains a customer-visible symptom, to the affected customer.
R17. A provider outage is reported as a provider problem, not as the customer's configuration being wrong.

**Degradation behaviour**

R18. Each of the following conditions has a stated, tested behaviour:
  - Klustro's own control plane unreachable: networks, clusters and workloads keep running; existing tokens remain valid until expiry; break-glass local access always works; management actions wait rather than fail destructively.
  - Mesh provider API unreachable: policy runs, joins and retags queue with backoff; existing connectivity is unaffected.
  - Mesh provider control plane outage: existing peer connections continue per the provider's own behaviour; Klustro shows the provider's status.
  - Node offline: marked offline after the threshold, not billed for full offline hours, and a server's absence raises quorum warnings.
  - Duplicate or late heartbeats: idempotent, accepted within the late window.
  - Cluster software upgrade failure: paused with logs, retry, skip or abort.
  - Join token leaked: instant revocation with the enrolled-node set enumerable.
  - Provider credentials revoked: mesh management degrades, workloads unaffected.
  - Policy validation failure: nothing applied, on-call paged.
  - Payment provider outage: top-ups retried, transitions into frozen paused.
  - Identity provider outage: SSO users cannot sign in, existing tokens remain valid, break-glass Owner passkey login works.
  - Datastore quorum loss: cluster degraded with the documented recovery runbook.
  - Lost Owner access: support-assisted recovery with identity verification, audited.

R19. Every behaviour in R18 is exercised by a test that induces the condition, rather than being asserted by inspection.
R20. Degradation is always one-directional and safe: no failure of a Klustro component may escalate into stopping a workload, removing a network device, or rewriting the network policy destructively.

**Verification**

R21. An exercise runs the platform with its control plane unavailable for an extended period (a full day) and confirms that networks, clusters, workloads, existing tokens and break-glass access all continue to work, and that the platform recovers cleanly afterwards without operator intervention.
R22. Recovery after each induced condition is verified, not just the degradation.
R23. A post-incident capability exists: given a time window, an operator can reconstruct what Klustro did, what it failed to do, and what the customer was told.

## Acceptance criteria

- Node, cluster and project health each show the specific reason for a degraded state, with timestamps.
- A Klustro-caused failure pages on-call; a customer-caused one does not.
- A gap in metering or audit emission is detectable before it is noticed by a customer or an auditor.
- Each behaviour in R18 has an automated test that induces the condition and asserts the stated outcome, including clean recovery.
- Cutting Klustro's control plane for a day leaves customer networks, clusters, workloads, tokens and break-glass access working, and the platform recovers without manual repair.
- "What did Klustro do, and what did it tell the customer?" is answerable for any past window.
