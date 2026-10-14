# Feature 10 — Node agent and enrolment

| | |
|---|---|
| Increment | 10 |
| Depends on | 01–09 |
| Unblocks | 11, 12, 13, 24, 27, 28 |
| Spec refs | §6.5, §11 (all), §8.4 (node driver), §18.3 |

## Goal

Turn an arbitrary customer machine into a Klustro-managed node that is reachable over the project's mesh, without Klustro ever opening an inbound port and without Klustro holding anything that can reach customer workloads.

At the end of this increment a machine can enrol and appear as an unassigned node. Cluster membership comes next.

## In scope

- The agent binary and its module responsibilities.
- Support matrix and preflight checks.
- Enrolment, node identity and certificates.
- The dedicated mesh client instance and the join.
- Heartbeats, node states and offline detection.
- Node facts and mesh peer records.
- Agent self-update.

## Out of scope

- k3s installation and cluster join (Feature 11).
- Node assignment, drain and removal (Feature 13).
- Cluster RBAC reconciliation (Feature 12).
- JWKS upload (Feature 24, which needs a cluster).

## Requirements

**Support matrix**

R1. Linux on amd64 and arm64 is supported: Ubuntu, Debian, RHEL-family and Raspberry Pi OS.
R2. systemd is required.
R3. An unsupported platform is rejected at preflight with a message naming the platform.

**Agent structure**

R4. The agent is a single process with clearly separated responsibilities: identity, control stream, mesh, cluster management, health, cluster RBAC reconciliation, key upload and self-update.
R5. Each responsibility must be independently testable and independently versioned in behaviour, so the cluster-management and reconciliation responsibilities can be added later without disturbing the mesh and identity paths.

**Preflight**

R6. Preflight runs before anything is installed or joined and reports all findings to the UI.
R7. Preflight checks at minimum: supported OS and architecture; presence of required kernel modules; presence of the tunnel device; availability of the ports the node will need; absence of a pre-existing conflicting Kubernetes installation; clock skew within tolerance; outbound reachability to Klustro and to the mesh provider.
R8. A failed preflight stops enrolment with an actionable message per finding.
R9. Insufficient compute resources are reported as a warning, not a failure.

**Enrolment and identity**

R10. The agent generates a key pair locally and never transmits the private key.
R11. Enrolment presents the join token and a certificate request to the Klustro agent endpoint.
R12. Klustro validates the token — valid, unexpired, unrevoked, uses remaining — and validates its own billing lifecycle state before creating the node record.
R13. On success Klustro issues a node identity certificate whose subject is the node's UUIDv7.
R14. Node certificates rotate automatically on a fixed cadence (default 30 days) and are revoked when the node is removed.
R15. The node identity is used only for agent-to-Klustro communication and is entirely separate from the node's mesh identity.
R16. Node identity is a principal type. Permissions for it are resolved by the same authorization rules as any other principal.

**Mesh join**

R17. Klustro requests a single-use, pre-authorised, non-ephemeral, tagged enrolment credential with a short expiry (default 10 minutes).
R18. The tags requested are derived from the join token: the unassigned tag, or the cluster and role tags, plus any label tags.
R19. Klustro sends the enrolment credential to the agent over the authenticated channel; it is never persisted in Klustro's own storage.
R20. The agent runs a dedicated mesh client instance with its own tunnel interface, state directory, socket and port.
R21. The dedicated instance must coexist with any mesh client software already installed by the customer, without disturbing it.
R22. The node's mesh hostname is derived from its UUIDv7 and is stable for the node's life.
R23. After joining, the agent reports the mesh address, interface and effective MTU, and the node's peer record is created.
R24. The node's peer record tracks transport type (direct or relayed), last handshake time and tags.
R25. Nodes are tagged, so provider key expiry does not apply to them. A server must never fall off the mesh because a credential expired.

**Heartbeats and state**

R26. The agent maintains a long-lived outbound control connection to Klustro over the public internet, not over the mesh.
R27. The agent sends a heartbeat on a fixed interval (default 30 seconds).
R28. A node is marked offline after three missed heartbeats (default 90 seconds) and returns to online on the next heartbeat.
R29. Node states are: enrolling, unassigned, installing, ready, offline, draining, removing, removed — plus a failed state reachable from installing with logs and a retry path.
R30. Node facts are collected and reported: OS, kernel, architecture, CPU and memory.
R31. Klustro can issue tasks to the agent over the control connection, and every task is authenticated and integrity-protected by the agent before execution.
R32. The agent opens no inbound network port.

**Self-update**

R33. The agent updates itself in a staged rollout that can be paused, and its version can be pinned per project.
R34. A self-update failure leaves the previous version running and reports the failure.

**Agent lifecycle**

R35. An agent can uninstall itself cleanly on instruction, removing its own service, its dedicated mesh client instance and its state, and optionally the cluster software.
R36. The agent never removes the customer's own software.

## Acceptance criteria

- On each supported distribution and architecture, the one-line command brings a machine to an unassigned, online node.
- A machine already running the customer's own mesh client joins without conflict and without disturbing the existing client.
- Preflight failures each produce a specific, actionable message.
- A node shows online and offline correctly against the heartbeat threshold.
- Removing the node revokes its certificate, and further agent connections are refused.
- The agent never opens an inbound port and never sends the join secret or any provider credential beyond what is required.
