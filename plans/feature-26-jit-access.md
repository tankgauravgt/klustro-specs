# Feature 26 — Just-in-time access

| | |
|---|---|
| Increment | 26 |
| Depends on | 01–04, 20, 21, 24 |
| Unblocks | — |
| Spec refs | §7.2, §7.4 R8, §13.3 |

## Goal

Let an organisation grant standing eligibility rather than standing access, so that powerful permissions exist only for the hours they are needed and every use has a stated reason and a named approver.

Eligible assignments have existed in the model since Feature 04 but are inert. This increment makes them usable.

## In scope

- The distinction between active and eligible assignments.
- Requesting, approving, rejecting and expiring an activation.
- Approval authority and separation of duties.
- The effect of an activation on cluster tokens.
- Visibility and audit.

## Out of scope

- ABAC conditions on assignments (later).
- Deny assignments (later).
- Break-glass emergency access, which remains the passkey Owner path from Feature 03.
- Automatic activation triggered by incidents or alerts.

## Requirements

**Semantics**

R1. An eligible assignment grants nothing until it is activated.
R2. An eligible assignment carries a maximum activation duration and a flag for whether approval is required.
R3. Activation grants exactly the actions of the assignment's role, at its scope, for the activation window and nothing more.
R4. Deactivating or expiring returns the principal to exactly its previous effective permissions, with no residual access.
R5. A principal may hold both an active and an eligible assignment for the same role and scope; the active one is unaffected by the eligible one's lifecycle.
R6. Eligible assignments are subject to the same scope inheritance, union, plan-gating and escalation rules as active ones.

**Requesting**

R7. A principal may request activation of an eligible assignment it holds, and only that.
R8. A request must include a justification. A request without one is rejected.
R9. A request states a requested duration, which must not exceed the assignment's maximum or the platform maximum.
R10. A principal cannot request activation on behalf of another principal.
R11. A request is refused while the account's lifecycle state forbids the action, and while an existing activation for the same assignment is still in force (extension is handled by requesting again after expiry, unless an explicit extend is offered, in which case it is audited as a distinct act).

**Approval**

R12. Where approval is required, an activation stays pending until a principal holding the approval action approves or rejects it.
R13. The approver must be a different principal from the requester. This must hold through group membership as well: a principal must not be able to approve an activation it requested by any route.
R14. Rejection states a reason, which the requester can see.
R15. A pending request has no effect on permissions.
R16. A request that is neither approved nor rejected within a bounded window expires as rejected, so it cannot be approved days later out of context.
R17. Approval grants the activation for the requested duration, starting from approval, not from request.
R18. Where approval is not required, the activation begins immediately on request, still with a justification recorded.

**Expiry and revocation**

R19. An activation expires automatically at the end of its window, without any operator action.
R20. An approver or an administrator can end an activation early; early termination is audited with who ended it and why.
R21. Expiry and early termination apply immediately to the authorization check.
R22. For cluster access, an activation's effect is bounded by both the window and the short token lifetime, so expiry is never more than one token lifetime away in practice.
R23. A principal is notified shortly before its activation expires and once when it expires.

**Visibility and audit**

R24. A principal can see its own assignments, which are eligible, which are currently active, and the remaining time.
R25. Administrators can see who holds eligible assignments, what is currently activated, and a history of activations with justifications and approvers.
R26. Requests, approvals, rejections, expiries, early terminations and the resulting permission grants are all audited.
R27. The activation history is answerable to a question such as "who had administrative access to production last month, and why?".

**Gating**

R28. Just-in-time access is gated by plan.
R29. Custom roles, also gated by the same plan tier, remain the way the underlying role is defined; JIT controls when it is held.

## Acceptance criteria

- An eligible assignment grants nothing until activated, proven by a denied action before activation and an allowed one after.
- An activation with approval required has no effect until a different principal approves it.
- A principal cannot approve its own activation, including via a shared group.
- An activation expires on schedule and the principal's permissions revert exactly, with a cluster token obtained during the window no longer being obtainable after it.
- An unapproved request expires as rejected rather than remaining actionable indefinitely.
- The full activation history — who, what, when, why, approved by whom — is retrievable for any past period.
