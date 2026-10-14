# Feature 24 — Groups, service principals and workload identity

| | |
|---|---|
| Increment | 24 |
| Depends on | 01–04, 12 |
| Unblocks | 25, 26 |
| Spec refs | §6.1, §6.3, §6.4, §6.6, §6.8, §7.3, §7.4 R7, §7.4 R10 |

## Goal

Extend the principal model beyond human users so that access can be managed the way organisations actually manage it: by team, and for automation and in-cluster software that must act without a person.

Groups make access administrable; service principals make CI/CD possible; workload identity lets in-cluster software act in Klustro without a stored secret. Together they are what makes Klustro usable by a mid-size company rather than only by an individual.

## In scope

- Group principals and their two sources.
- Service principals and their three credential types.
- Workload identity bindings and token exchange.
- The limits and safety rules that apply to non-human principals.
- Cluster key-upload path that makes workload identity possible.

## Out of scope

- SCIM and SSO provisioning, which create and maintain provider-managed groups (Feature 25).
- Nested groups (later).
- Role assignment mechanics, which reuse Feature 04 unchanged.
- Any path that lets Klustro reach the cluster API; it never does.

## Requirements

**Groups**

R1. A group is a principal with its own id and lives in exactly one organisation.
R2. A group has a source, fixed at creation: native, or provider-managed.
R3. Membership of a native group is manageable in Klustro through the group management action.
R4. Membership of a provider-managed group is read-only in Klustro and is maintained by SCIM or by SSO group claims.
R5. A group's source never changes. Switching source means creating a new group and moving its role assignments — the UI must state this rather than offering an in-place conversion.
R6. A native group may contain users and service principals. A provider-managed group contains users provisioned to that organisation.
R7. Groups cannot contain other groups in this phase; the attempt is rejected with a clear message.
R8. A group that holds role assignments can be deleted only with the consequence of losing that access clearly stated, and deletion removes its assignments.
R9. Groups never authenticate. They are containers for role assignments only.
R10. Membership changes take effect on the next authorization check, not at the next login, and any resulting access loss is visible to administrators.

**Service principals**

R11. A service principal is a non-human principal in an organisation, for CI/CD and automation.
R12. A service principal can hold three credential types: a client secret, an uploaded public certificate, or a federated OIDC credential.
R13. A client secret has a maximum lifetime of one year, is shown once, and is stored only as a hash.
R14. Certificates are uploaded as public material; Klustro never holds the private key.
R15. A federated OIDC credential trusts an external issuer and a subject pattern, storing no secret at all.
R16. Each credential type can be listed, added and revoked independently, and revocation takes effect immediately for new authentications.
R17. Credential changes are audited.
R18. Creating, deleting and managing credentials requires the service principal management action.

**Workload identity**

R19. A workload identity binding maps a cluster, a namespace and a service account name to a workload identity principal.
R20. The binding is created, and the principal is assigned roles, using the existing authorization model with no special cases.
R21. A workload presenting a Kubernetes service account token can exchange it at Klustro's token endpoint for a Klustro access token for the bound principal.
R22. The exchange validates the token's signature, issuer, audience, cluster, namespace and service account against the binding, and rejects an unbound combination.
R23. Because Klustro cannot reach the cluster API, the cluster's agent uploads the service account issuer's public keys whenever they change, and Klustro caches them scoped to that cluster.
R24. A key upload from one cluster must never be usable to validate a token from another cluster.
R25. Key rotation in the cluster is reflected in Klustro automatically, and the upload path is idempotent and tolerant of reordering.
R26. If a cluster's keys are unavailable, exchange fails closed with a distinguishable error rather than accepting an unverifiable token.
R27. Workload identity is gated by plan.

**Non-human principal limits**

R28. Service principals and workload identities can never hold the Owner role.
R29. They cannot hold the assignment-writing, SSO configuration, SCIM configuration or role definition actions unless an Owner has explicitly enabled an identity-management allowance on that specific principal.
R30. This allowance is per-principal, is visible, and its grant and revocation are audited.
R31. Non-human principals are subject to the same scope inheritance, union, plan-gating and lifecycle rules as any other principal, with no exceptions.

## Acceptance criteria

- A provider-managed group's membership cannot be edited in Klustro; the API and console both refuse it.
- Removing a user from a native group removes the access the group granted on the next authorization check.
- A CI job authenticates with a federated OIDC credential and no stored Klustro secret, and receives only the access its assignments grant.
- An in-cluster component exchanges its service account token for a Klustro token and can perform exactly the action its binding's role allows.
- A token from an unbound namespace or service account is refused.
- Keys uploaded from one cluster cannot validate a token from another.
- A service principal cannot be assigned Owner, and cannot manage identities without an explicit, audited allowance.
