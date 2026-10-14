# Feature 14 — Labels and taints

| | |
|---|---|
| Increment | 14 |
| Depends on | 01–13 |
| Unblocks | 15 |
| Spec refs | §5.2 (Label), §9.4, §11.7, §16.5 |

## Goal

Give customers a way to describe their own machines — region, site, hardware class, environment — in a form that is safe to turn into network identity, so those descriptions can later drive access rules.

This increment exists separately from Feature 13 because labels are an input to user-authored mesh rules (Feature 15), and their tag-identity semantics deserve their own review.

## In scope

- The label registry per project.
- The mapping from a label to a provider tag.
- Label synchronisation to cluster node labels.
- Taint synchronisation.
- The ordering guarantee between a label change and a policy revision.

## Out of scope

- Rules that reference labels (Feature 15).
- Label-based scheduling or placement.
- Bulk import of labels from an existing cluster.
- Automatic label discovery from machine facts.

## Requirements

**Registry**

R1. A label is a distinct `key=value` pair registered per project and has its own UUIDv7.
R2. The same `key=value` pair in two projects is two distinct labels with two distinct ids.
R3. Label keys and values follow Kubernetes label syntax.
R4. A label record is created the first time the pair is used in the project.
R5. A label that is no longer carried by any node in the project is retired, and its tag stops being owned in the rendered policy.
R6. Labels are immutable in the sense that changing a value creates a new label; renaming must never silently retag existing machines.
R7. Registry membership is visible to principals holding the nodes read action.

**Tags**

R8. Each label maps to a provider tag derived from the label's own id, never from its human-readable key or value.
R9. This indirection is required so that arbitrary user text never becomes provider syntax, and so a label's identity survives renaming.
R10. A node carrying a label carries that label's tag in addition to its role tags.
R11. An unassigned node carries the unassigned tag plus its label tags and receives no traffic grants.
R12. Label tags must satisfy the provider's character and length limits for the resulting name, and this must be validated when the label is registered rather than when the policy is rendered.

**Ordering**

R13. Adding, changing or removing a node's labels triggers a policy run.
R14. The device is retagged only after a revision whose ownership block contains the new tags has been applied.
R15. A retag failure leaves the node's Klustro-side label state and its device tags in a described, reconcilable state, and the divergence is visible.
R16. If credentials are invalid, label changes queue rather than fail permanently.

**Cluster synchronisation**

R17. Labels synchronised to the cluster appear as node labels on the corresponding node.
R18. Label synchronisation to the cluster is idempotent and re-converges after an out-of-band edit.
R19. Klustro-managed labels are namespaced so they cannot collide with customer labels.
R20. Klustro also maintains its own identifying labels on nodes, tied to the node and cluster ids, separately from customer labels.

**Taints**

R21. Taints are set per node and synchronised to the cluster.
R22. Setting or removing a taint does not itself change network policy.
R23. Taint synchronisation is idempotent and re-converges.
R24. Taints are visible on the node and cluster views.

**Audit and visibility**

R25. Label registration, node label changes, taint changes and label retirement are audited.
R26. The node view shows the labels and taints Klustro holds, and distinguishes Klustro-internal labels from customer labels.
R27. The project view lists all registered labels, with how many nodes currently carry each.

## Acceptance criteria

- Adding a label to a node triggers exactly one policy run, after which the device carries the label's tag and the cluster node carries the label.
- A label whose value contains characters invalid for provider tag syntax still works, because tags are derived from the label id.
- Renaming a label's value retags affected nodes without breaking in-flight rules referencing the old label.
- Removing a label from the last node carrying it retires the label and removes its tag ownership from the rendered policy.
- Divergence introduced by an out-of-band label or taint edit is detected and re-converged.
