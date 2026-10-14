# Feature 09 — Join tokens and the onboarding script

| | |
|---|---|
| Increment | 09 |
| Depends on | 01–08 |
| Unblocks | 10 |
| Spec refs | §5.2 (Join Token), §11.2, §16.1 |

## Goal

Give a customer a one-line command they can paste on a fresh machine to bring it under Klustro management. Everything about the operator experience at scale depends on this being safe to hand out and safe to revoke.

## In scope

- Join token creation, constraints and revocation.
- The token secret format and storage.
- The one-line onboarding command.
- Token usage attribution.
- The preconditions a token creation must satisfy.

## Out of scope

- The agent itself and enrolment (Feature 10).
- Device creation in the mesh provider (Feature 10).
- Cluster join (Feature 11).

## Requirements

**Token creation**

R1. Creating a join token requires the dedicated create action and is scoped to a project.
R2. A token may optionally name a cluster and a node role. A role is mandatory when a cluster is named.
R3. A token may carry a set of labels to be applied to machines enrolling with it.
R4. A token has a maximum-use count (default 1) and an expiry (default 24 hours, maximum 30 days).
R5. A token records who created it and when.
R6. Creation is refused while the target cluster's policy revision has not been applied, with a message explaining that the cluster network is being prepared — not a generic error.
R7. Creation is refused when the Free plan node hard cap would be exceeded at enrolment, with a message naming the limit and the plan.
R8. Creation is refused while the account's lifecycle state forbids it.

**The secret**

R9. The token secret is a Klustro credential only. It must not contain, embed or imply a mesh provider key or a cluster token.
R10. The secret is displayed exactly once at creation and stored only as a hash thereafter.
R11. The secret must be unguessable and must not encode the project, cluster or role in a readable form.

**The command**

R12. The UI presents a single copy-pasteable command that downloads and runs the onboarding script with the token.
R13. The command is identical in shape regardless of role, labels or cluster.
R14. The command must not require the operator to edit a file or supply additional Klustro credentials.

**Lifecycle**

R15. A token can be revoked immediately by a principal holding the revoke action.
R16. Revocation takes effect for new enrolments immediately.
R17. Nodes that already enrolled with a revoked token remain enrolled and unaffected.
R18. The system can list, for any token, every node that enrolled using it.
R19. Tokens that are expired, exhausted or revoked are rejected at enrolment with distinguishable reasons.
R20. Token creation, revocation and each successful use are audited.

**Visibility**

R21. Principals holding the read action can list tokens with their use counts, remaining uses, expiry and revocation time, and can see which nodes used each token.
R22. The secret is never visible again after creation, by any principal, including platform staff.

## Acceptance criteria

- A one-line command on a fresh supported machine with a valid token begins enrolment.
- The token secret is retrievable exactly once.
- Revoking a token stops new enrolments immediately while already-enrolled machines keep running.
- A leaked-token scenario is answerable: the exact set of machines enrolled with that token is enumerable.
- Token creation for a cluster whose policy is not yet applied produces the explanatory message, not a failure.
