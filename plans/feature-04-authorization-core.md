# Feature 04 — Authorization core

| | |
|---|---|
| Increment | 04 |
| Depends on | 01, 02, 03 |
| Unblocks | 05, 06, 07, 09, 13, 15, 22, 24, 25, 26 |
| Spec refs | §7 (all), §6.9 |

## Goal

Make every action in Klustro controllable by an explicit, auditable permission decision. This increment defines the action catalogue, roles, the assignment model with scope inheritance, and the single decision point that answers "may this principal do this here?".

No resource created in later increments may exist without an action name defined here or added here.

## In scope

- Action naming format and catalogue.
- Role definitions: built-in and custom.
- Role assignments, scopes and inheritance.
- The authorization check algorithm.
- Assignment safety rules.
- JIT assignment representation (activation deferred to Feature 26).

## Out of scope

- JIT activation and approval workflow (Feature 26).
- Plan entitlement enforcement (Feature 06) and lifecycle restriction (Feature 21) — both are invoked by the check but defined elsewhere.
- Deny assignments and ABAC conditions (later).
- Nested groups (later).

## Requirements

**Action catalogue**

R1. Actions use the form `resource[.sub-resource]:verb`.
R2. The catalogue covers organisation, membership, identity, projects, mesh, nodes, labels, clusters, billing and audit, as enumerated in the specification.
R3. Wildcards are permitted in built-in role definitions only: `resource:*`, `*:verb`, `resource.*:*`, `*`.
R4. Adding a new action to the catalogue is a deliberate act that never silently grants permission to existing custom roles.

**Role definitions**

R5. A role definition has `actions[]` and `not_actions[]`; effective actions are `actions` minus `not_actions`.
R6. The built-in roles are shipped as listed in the specification: Owner, Admin, Billing Admin, Identity Admin, Contributor, Cluster Operator, Developer, Reader, Auditor.
R7. Approving and rolling back mesh policy changes are not part of any built-in role except Owner and Admin, and can be granted through custom roles.
R8. Custom roles list explicit actions only — wildcards are rejected.
R9. Custom roles cannot reference actions absent from the catalogue.
R10. Custom roles are organisation-scoped and assignable at any scope within that organisation.
R11. Custom roles are gated by plan entitlement.

**Assignments**

R12. A role assignment binds a principal, a role, and a scope.
R13. Valid scopes are organisation, project and cluster paths built from UUIDv7s.
R14. An assignment has a type of `active` or `eligible`.
R15. An assignment may carry an optional list of Kubernetes namespaces, meaningful only with the cluster kubeconfig edit action.
R16. An assignment may carry a JIT policy (maximum duration, requires-approval) when eligible.
R17. An assignment records who created it and when, and may have an expiry.
R18. Scope inheritance applies downward: organisation → project → cluster. A check walks up the scope path.
R19. Effective permissions are the union of all active assignments, direct and through group membership. There is no deny in this increment.

**Safety rules**

R20. No privilege escalation: a principal may create an assignment only if it holds the write action at that scope or above **and** holds every effective action of the role being assigned at that scope.
R21. At least one active Owner must always remain at organisation scope. The last Owner cannot be removed or demoted, and the last Owner cannot leave the organisation.
R22. A group's source is fixed at creation and never changes.
R23. Service principals and workload identities can never hold Owner. They cannot hold assignment-writing, SSO configuration, SCIM configuration or role definition actions unless an Owner has explicitly enabled identity management on that specific principal. (Enforced once those principals exist; the rule must be expressed here.)

**The check**

R24. Exactly one component answers `check(principal, action, scope) → allow | deny`. No other component may make a permission decision.
R25. The check expands the principal into itself plus all groups it belongs to.
R26. The check collects active assignments at the scope and every ancestor scope.
R27. The check computes effective actions per role and unions them.
R28. The check allows if any effective action pattern matches the requested action.
R29. The check then applies plan entitlements; an action belonging to a feature not in the plan is denied with a distinct code.
R30. The check then applies billing lifecycle restrictions; write actions are denied with a distinct code while the account is Frozen or Suspended.
R31. Every denial is audited with the actor, action, scope and reason.

## Acceptance criteria

- Every built-in role × every catalogue action × every scope level is covered by table-driven tests.
- Team-scoped access behaves as specified: read granted at organisation scope applies to all projects and clusters beneath it.
- A direct project-scoped role adds to, and never subtracts from, a broader one.
- An Identity Admin cannot assign themselves Owner.
- The final Owner cannot be removed or leave.
- Plan-gated and lifecycle-restricted denials are distinguishable from a plain missing-permission denial.
- No component other than the authorization component contains permission logic.
