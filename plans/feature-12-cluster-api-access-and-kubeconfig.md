# Feature 12 — Cluster API access and kubeconfig

| | |
|---|---|
| Increment | 12 |
| Depends on | 01–11 |
| Unblocks | 13, 26, 27 |
| Spec refs | §6.6 (JWKS prerequisite), §7.4 R9, §9.6, §10.5, §12.5, §12.6, §16.2, §18.1 |

## Goal

Let an authorised human or automation reach the cluster API from their own machine, over the mesh, with short-lived credentials tied to their Klustro permissions — without Klustro ever proxying or terminating that traffic.

This completes success criterion 1: `kubectl get nodes` against a self-service cluster.

## In scope

- The per-cluster API entry point and its lifecycle.
- Health-aware advertising and withdrawal.
- Klustro's OIDC issuer for cluster tokens.
- Token issuance rules and claim mapping.
- Kubernetes RBAC reconciliation inside the cluster.
- Kubeconfig generation and the CLI surface.
- The two-layer access model.

## Out of scope

- Cluster-level workload authorization beyond the built-in Kubernetes roles.
- Namespace-scoped bindings for namespaces that do not exist yet (the creation rule is specified but its full lifecycle lands with Feature 13).
- JWKS upload and workload identity (Feature 24).
- Any form of proxying or web terminal (permanently out of scope).

## Requirements

**Entry point**

R1. Every cluster has exactly one stable API entry point, created with the cluster by the policy pipeline and deleted with it.
R2. The entry point lists every server of the cluster as a host and is automatically approved for the cluster's server tag.
R3. The entry point exposes only the API port, as TCP passthrough; TLS is terminated by the cluster, not by Klustro.
R4. Single-server clusters get the entry point too, so a kubeconfig never changes when servers are added.
R5. Joining nodes register through the entry point; after registration, cluster nodes balance across servers themselves.
R6. The entry point's address must be stable across server membership changes.

**Health-aware advertising**

R7. Each server's agent advertises the entry point toward its own local API.
R8. The agent probes its local API health on a fixed interval (default 5 seconds).
R9. The agent withdraws its advertisement after a small number of consecutive failures and re-advertises after a small number of consecutive successes.
R10. The agent withdraws its advertisement before upgrading or draining its server, and re-advertises only after the server is healthy again.
R11. A server's advertisement status is visible per cluster.

**Authentication**

R12. Klustro operates a single OIDC issuer for all clusters. Each cluster is addressed as a distinct audience/client id built from its UUIDv7.
R13. The API server is configured to trust that issuer, to take the subject as the username (prefixed to avoid collisions) and groups from a dedicated claim.
R14. Cluster tokens have a short lifetime — one hour.
R15. A token contains the issuer, the principal's UUIDv7, the cluster as audience, the mapped Kubernetes group or groups, and the expiry. It contains no other identity or permission data.
R16. A token is issued only when: the principal holds a cluster kubeconfig action on that cluster; organisation policies are satisfied; and the account is not suspended.
R17. A token is refused with a distinguishable reason when the principal has cluster access but the account is suspended, and with a different reason when the principal lacks the action.
R18. The API server fetches Klustro's signing keys through standard discovery and caches them, so server nodes require outbound reachability to Klustro's issuer.

**Group mapping**

R19. The highest kubeconfig action a principal holds on a cluster determines its Kubernetes group: admin, edit, view, or a namespace-restricted edit group.
R20. If a principal holds both an unrestricted and a namespace-restricted edit grant, the unrestricted one wins.
R21. A principal with no kubeconfig action receives no token.

**RBAC reconciliation in the cluster**

R22. One active reconciler per cluster maintains the bindings, elected through a lease so that exactly one runs at a time.
R23. The reconciler maintains cluster-wide bindings for admin, edit and view, and namespace-scoped bindings for the namespace-restricted edit group.
R24. Namespace-scoped bindings are created when the namespace exists; the reconciler must converge once a namespace appears.
R25. Reconciliation is idempotent and must restore bindings that are deleted out of band.
R26. The reconciler must never remove or modify customer-authored bindings, roles or subjects outside Klustro's naming.

**Kubeconfig**

R27. The CLI produces a kubeconfig for a cluster, optionally merged into an existing configuration.
R28. The kubeconfig's context alias is a local convenience only and never affects anything server-side.
R29. The kubeconfig authenticates by invoking the CLI to obtain a token, so credentials are never stored in the file.
R30. The kubeconfig works unchanged whether the cluster has one server or five.

**Two-layer access model**

R31. Network reachability and authorization are separate: reaching the API requires membership of or a tagged device in the project's network; acting on the API requires a valid token.
R32. A user with a valid Klustro role but no network membership gets an explicit "join the project's network" explanation rather than a connection error.
R33. A user in the network with no valid token is rejected with an authentication failure, even from inside the network.
R34. Klustro never invites users to the network; the customer manages membership in their provider account.

**Break-glass**

R35. Each server writes a local administrative kubeconfig at the cluster software's standard path.
R36. Anyone with root on a server can administer the cluster independently of Klustro, and this must remain true in every failure mode of the Klustro control plane.

**Visibility**

R37. The cluster page shows the entry point address, per-server advertisement status, and whether the current user can reach the network.
R38. Token issuance, refusal and refusal reasons are audited.

## Acceptance criteria

- A user who joined the project's network runs `kubectl get nodes` against a freshly created cluster using only CLI-produced configuration.
- Deleting a server does not change any user's kubeconfig or reachable address.
- Killing the API on one server makes its advertisement withdraw and the cluster remain reachable through the others, with recovery on health return.
- A suspended account cannot obtain a token while a mere permission gap produces a different, distinguishable error.
- Deleting a Klustro-managed binding out of band is restored automatically; deleting a customer-authored binding is not touched.
- A user without a Klustro role who is in the network is rejected with an authentication failure, and a user with a role but no network membership is told to join the network.
