# Feature 17 — Metering

| | |
|---|---|
| Increment | 17 |
| Depends on | 01, 10 |
| Unblocks | 18, 21 |
| Spec refs | §13.1, §13.2, §16.3, §17 |

## Goal

Produce a single, trustworthy, idempotent record of what each customer used. Every billing outcome — prepaid drawdown, postpaid invoice, overage, entitlement checks — reads from this one meter, so prepaid and postpaid cannot disagree.

## In scope

- The two meters and what they mean.
- Usage events, idempotency and append-only storage.
- Late and duplicate event handling.
- The connected-hour definition and role precedence.

## Out of scope

- Rating, pricing and included allowances (Feature 18).
- Wallet drawdown and invoices (Features 18, 19).
- Lifecycle state changes driven by usage (Feature 21).
- Any metering of network bandwidth, which is explicitly excluded on every plan.

## Requirements

**Meters**

R1. Exactly two meters exist: server node-hours and agent node-hours.
R2. A node is connected in a given hour if Klustro received at least one heartbeat during that hour.
R3. A node offline for an entire hour is not billable for that hour.
R4. If a node held the server role at any point in the hour, the whole hour is billed as a server hour. Role precedence is resolved before the event is written, so the event carries a single unambiguous meter.
R5. Network throughput, mesh relay usage and bandwidth are never metered, billed or limited on any plan, and this must be enforced structurally — there is no meter that could express it.

**Usage events**

R6. A usage event contains: id, idempotency key, billing account, organisation, project, optional cluster, node, meter, hour start, and recorded time.
R7. The idempotency key is the node id combined with the hour start. A duplicate event must never produce a second charge.
R8. Events are append-only. Nothing updates or deletes a usage event; corrections are compensating events.
R9. Events are produced at the end of each hour for every node connected during it.
R10. Event production must be resilient to a missed or delayed run: a late run must still produce the correct events for the hours it missed.
R11. Events carry the project and cluster context at the time of the hour, not at the time of production, so a node that moves between clusters does not corrupt history.

**Timeliness**

R12. Late events are accepted for up to 72 hours after the hour they describe.
R13. An event arriving after a postpaid invoice has been finalised is billed on the next invoice rather than retroactively changing a closed one.
R14. Duplicate and out-of-order heartbeats must not produce duplicate hours or shift a node between meters within an hour.

**Boundaries**

R15. Metering produces facts only. It never computes money, never reads prices and never decides whether a customer may continue.
R16. Metering starts when a node enrols and continues for as long as the node exists in a project, regardless of its cluster membership.
R17. Usage for a node continues to accrue through grace and frozen states; only the account's lifecycle state, not meter behaviour, changes.

**Visibility**

R18. A customer can see their org's usage by hour, by project, by cluster and by meter, for the current and past billing periods.
R19. Usage visibility respects the billing read action.
R20. Metering events are observable by Klustro staff for support purposes without exposing them through the customer API beyond the customer's own data.

## Acceptance criteria

- A node connected for a full hour produces exactly one event for that hour, no matter how many times the producing job runs.
- A node offline for a full hour produces no event for that hour.
- A node that changes role mid-hour produces a single server-hour event for that hour.
- Re-delivering a day of heartbeats produces no duplicate events and no changed history.
- A node moved from one cluster to another has historical events correctly attributed to the cluster it was in at the time.
- No meter or event field can express bandwidth.
