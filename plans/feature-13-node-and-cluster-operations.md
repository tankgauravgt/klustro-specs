# Feature 13 — Node and cluster operations

| | |
|---|---|
| Increment | 13 |
| Depends on | 01–12 |
| Unblocks | 14, 27, 28 |
| Spec refs | §5.3, §11.6, §11.7, §12.3, §12.7, §17 |

## Goal

Make a running cluster operable for its whole life: add capacity, take a node out for maintenance, retire hardware, and tear a cluster or a project down without leaving anything behind or destroying the customer's data by accident.

## In scope

- Drain.
- Node removal for both roles, with the safety ordering each requires.
- Node reassignment and unassignment.
- Cluster deletion.
- Project purge teardown.
- Node labels and taints.
- Operational error and retry paths.

## Out of scope

- Upgrades (Feature 27).
- Snapshot restore (later); snapshot creation is invoked here but defined in Feature 28.
- Automatic node replacement or autoscaling.

## Requirements

**Drain**

R1. Draining requires the drain action and cordons the node before evicting.
R2. Eviction respects pod disruption budgets and reports progress while it runs.
R3. Drain can be forced, and forcing is explicitly acknowledged and audited.
R4. Drain must be usable as a standalone maintenance step, not only as part of removal.

**Removing a worker**

R5. Removing a worker performs, in order: if not already drained, drain (unless forced); stop the cluster software on the node; delete the node from the cluster; optionally uninstall the cluster software from the machine, at the customer's choice; then either return the node to unassigned or remove it entirely, at the customer's choice.
R6. The device is removed from the project's network only when the node itself is removed, never when it merely returns to unassigned.

**Removing a server**

R7. Removing a server performs, in order: a quorum check; withdraw the server's API advertisement; drain; take an automatic datastore snapshot; stop the cluster software; delete the node from the cluster; verify the datastore member is actually gone; optionally uninstall the cluster software.
R8. A server removal that would break quorum is refused with the arithmetic shown to the user.
R9. Removal must not complete while the datastore member is still present. A failure at that step is surfaced with a retry path rather than a success.
R10. Removing the last remaining server out of an HA cluster is refused unless the cluster is being deleted.

**Unassigned nodes**

R11. A node removed from a cluster but kept becomes unassigned, keeps its enrolment and mesh identity, and receives no cluster traffic.
R12. An unassigned node can be assigned to a cluster, including a different one, through the standard assignment ordering.
R13. An unassigned node can be removed from the project entirely.

**Node removal and uninstall**

R14. Removing a node from a project performs: drain unless forced; remove from cluster; agent uninstalls itself; the device is removed from the provider's network.
R15. Agent uninstall removes the agent service, its dedicated mesh client instance and its state.
R16. The agent never removes the customer's own software.

**Cluster deletion**

R17. Cluster deletion requires typed-name confirmation.
R18. On deletion, nodes return to unassigned.
R19. The customer chooses whether the cluster software is uninstalled from the nodes, with uninstall as the default.
R20. The cluster's API entry point is removed by the policy pipeline, not by the cluster component.
R21. Cluster deletion is immediate — it has no restore window.
R22. Deletion is refused while an upgrade is in progress.

**Project purge**

R23. Project deletion is soft with a restore window; during the window nodes keep running and Klustro changes nothing.
R24. On purge, in order: agents are instructed to uninstall themselves and their dedicated mesh client; the cluster software is uninstalled only if the customer chose that at deletion time; the devices Klustro created are removed from the provider's network; the policy backed up at takeover is restored unless the customer opted out; the stored provider credentials are deleted.
R25. Purge must not delete anything in the customer's provider account that Klustro did not create.

**Labels and taints**

R26. Labels are `key=value` pairs registered per project, each with its own id, so that user-defined labels map to safe provider tags.
R27. Label keys and values follow Kubernetes label syntax and are synchronised to cluster node labels.
R28. Changing a node's labels triggers a policy run; the device is retagged only after the revision containing the new tag ownership is applied.
R29. Taints are set per node and synchronised to the cluster.
R30. Removing a label from every node in a project retires it through the normal policy run path.

**Failure and retry**

R31. Every long-running operation reports progress and can be observed to completion.
R32. A failed operation leaves the system in a described state rather than an unknown one, and can be retried without duplicating an already-completed step.
R33. If the Klustro control plane is unreachable, nodes keep running and management operations wait; nothing about removal or drain is executed by the node on its own initiative.

## Acceptance criteria

- A worker can be drained, removed and left uninstalled, and its device disappears from the provider network.
- A server removal takes a snapshot, verifies the member is gone, and refuses when it would break quorum.
- A node moved from one cluster to another ends up with the right tags, labels and cluster membership.
- Deleting a cluster returns its nodes to unassigned and removes the API entry point.
- Purging a project removes only Klustro-created devices and restores the takeover backup when the customer did not opt out.
- A removed label stops appearing on new nodes and is retired from the policy.
