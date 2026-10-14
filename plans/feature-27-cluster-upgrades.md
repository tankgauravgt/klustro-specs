# Feature 27 — Cluster upgrades

| | |
|---|---|
| Increment | 27 |
| Depends on | 01–13, 28 |
| Unblocks | — |
| Spec refs | §12.4, §12.8, §3.1, §17 |

## Goal

Keep customer clusters on a supported Kubernetes version without the customer hand-holding each node, and without an upgrade ever being the reason a cluster loses quorum or a workload disappears.

Upgrades are the most dangerous routine operation in the platform, so this increment depends on snapshots (Feature 28) being in place first.

## In scope

- The supported version set and upgrade selection.
- Upgrade orchestration across servers and workers.
- Safety checks and ordering.
- Pause, resume, retry, skip and abort.
- Progress, notification and audit.

## Out of scope

- Downgrades, which are not offered.
- Automatic unattended upgrades on a schedule, which is not in this phase.
- Node operating system patching.
- Cluster software version changes that also change the datastore format in a way requiring migration.

## Requirements

**Versions**

R1. Klustro maintains a supported version set: the latest three minor versions.
R2. A cluster's version can only be changed to a version in the supported set.
R3. A cluster running a version leaving the supported set is flagged as needing an upgrade before it becomes unsupported.
R4. Upgrades require a dedicated permission.
R5. A cluster cannot be upgraded while it is degraded in a way that makes the datastore unsafe, and cannot be upgraded while another operation on the cluster is in progress.

**Preparation**

R6. An automatic datastore snapshot is taken before an upgrade starts. The upgrade must not start if the snapshot fails.
R7. The upgrade checks that all datastore members are healthy and that a leader is stable before it begins, and re-checks before each server.

**Server upgrades**

R8. Servers are upgraded strictly one at a time.
R9. For each server: verify datastore health and leader stability; withdraw that server's API advertisement; upgrade the software; wait for the server's API to be ready and its datastore membership healthy; re-advertise the advertisement.
R10. The cluster must remain available through the cluster's API entry point for the whole upgrade; a customer running `kubectl` must not be interrupted beyond a brief reconnect.
R11. The next server is not started until the previous one is fully healthy.
R12. Quorum protection from Feature 11 applies: an upgrade is refused or paused if it would leave fewer healthy members than quorum requires.

**Worker upgrades**

R13. Workers are upgraded in batches, with the batch size defaulting to the larger of one node or a configured percentage of the worker count.
R14. Each worker is drained before upgrade and uncordoned afterwards.
R15. Draining respects pod disruption budgets. If a worker cannot be drained because budgets block eviction, the upgrade pauses on that node with a clear explanation rather than force-draining.
R16. A worker failure mid-batch does not stop the batch's already-upgraded nodes from functioning.

**Failure handling**

R17. On failure the upgrade pauses, the failing node is identified, and the customer can see the agent's logs for it.
R18. From a paused upgrade the customer can retry the failing node, skip it (workers only — a server cannot be skipped), or abort the upgrade.
R19. Already-upgraded nodes are never automatically downgraded.
R20. Aborting leaves the cluster in a described mixed-version state, with which nodes are on which version clearly shown, and the cluster remains fully functional.
R21. Resuming after the underlying cause is fixed continues from the failed node.

**Progress and visibility**

R22. Upgrade progress is visible per node: pending, draining, upgrading, verifying, done, failed.
R23. The cluster page shows current version, per-node version, and whether a mixed-version state exists.
R24. Customers are notified when an upgrade starts, when it pauses, and when it completes or aborts.
R25. Upgrade start, each node transition, pause, retry, skip, abort and completion are audited.

## Acceptance criteria

- A three-server cluster with workers upgrades to a new version with no loss of datastore quorum and no interruption to `kubectl` beyond a brief reconnect.
- An automatic snapshot exists from before the upgrade.
- A worker whose pods are protected by a disruption budget pauses the upgrade with a clear explanation instead of force-draining.
- Inducing a failure on one worker pauses the upgrade, allows retry and skip, and leaves the rest of the cluster healthy.
- Aborting mid-upgrade leaves a working cluster with an accurate per-node version report.
- A cluster with unhealthy datastore members refuses to start an upgrade.
