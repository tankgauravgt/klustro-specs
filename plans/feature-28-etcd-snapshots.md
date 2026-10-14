# Feature 28 — Datastore snapshots

| | |
|---|---|
| Increment | 28 |
| Depends on | 01–13 |
| Unblocks | 27 |
| Spec refs | §12.2, §12.7, §12.9, §3.2, §17 |

## Goal

Ensure a customer can always recover the cluster's control-plane state, and that Klustro itself always has a recovery point before it does anything risky. This ships before upgrades (Feature 27) because every upgrade depends on it.

## In scope

- Scheduled local snapshots.
- Automatic snapshots before risky operations.
- On-demand snapshots.
- Listing, retention and visibility.
- The documented recovery path for this phase.

## Out of scope

- Snapshots to customer-provided object storage (later).
- Guided restore and quorum-loss recovery workflows (later).
- Backup of customer workloads or persistent volumes, which Klustro never owns.
- Klustro-hosted snapshot storage (later, paid add-on).

## Requirements

**Scheduled snapshots**

R1. Each cluster takes datastore snapshots on a schedule, defaulting to every 12 hours.
R2. The schedule and retention count are per-cluster creation options and can be changed afterwards.
R3. The default retention keeps a fixed number of snapshots (default 5), discarding the oldest beyond that.
R4. Scheduled snapshots run on a server node and require no Klustro connectivity to succeed.
R5. A missed schedule is not silently skipped: the next run records that the previous window was missed.

**Automatic snapshots before risky operations**

R6. An automatic snapshot is taken before every cluster upgrade.
R7. An automatic snapshot is taken before every server removal.
R8. A risky operation must not proceed if its automatic snapshot fails; the failure is surfaced with a retry path.
R9. Automatic snapshots are distinguishable from scheduled and on-demand ones, so an operator can see why each exists.

**On-demand snapshots**

R10. A principal holding the snapshot-create action can take an on-demand snapshot, optionally with a note explaining why.
R11. On-demand snapshots are not subject to the automatic retention count until they are no longer needed, and the customer can delete them.
R12. Creating a snapshot reports when it is complete, not merely when it started.

**Listing and visibility**

R13. A principal holding the snapshot-read action can list a cluster's snapshots, including when each was taken, on which node, its size, and whether it was scheduled, automatic or on-demand.
R14. The list indicates which node holds each snapshot, since this matters for recovery.
R15. Snapshot creation failures are visible with their reason, not merely absent from the list.
R16. The cluster page summarises the most recent successful snapshot and its age, and warns when the cluster has no recent successful snapshot.

**Recovery in this phase**

R17. This phase does not offer a guided restore workflow.
R18. A documented, tested runbook exists describing how to restore a cluster's datastore from a snapshot using standard cluster tooling, including the quorum-loss case.
R19. The runbook is reachable from the console where snapshot state is shown, so a customer in trouble finds it without support.
R20. Klustro must not claim restore capability it does not have; the console states clearly that restore is manual in this phase.

**Boundaries**

R21. Snapshots are never used by Klustro to take any billing or enforcement action.
R22. Snapshots are never accessed by Klustro to read customer data; they are files on the customer's own machines.
R23. Snapshot metadata is organisation-scoped, and snapshot contents never leave the customer's machines in this phase.
R24. Snapshot creation, failure, deletion and the automatic-before-operation decisions are audited.

## Acceptance criteria

- A cluster takes scheduled snapshots and retains the configured number, discarding the oldest.
- An upgrade and a server removal each refuse to proceed without a successful pre-operation snapshot.
- An on-demand snapshot can be created with a note and deleted afterwards.
- The list shows time, node, size and kind for each snapshot, and indicates failures with reasons.
- The documented runbook is reachable from the console and has been exercised against a real cluster, including a quorum-loss scenario.
- The console does not imply a guided restore exists in this phase.
