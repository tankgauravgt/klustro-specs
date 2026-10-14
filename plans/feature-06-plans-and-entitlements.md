# Feature 06 — Plans and entitlements

| | |
|---|---|
| Increment | 06 |
| Depends on | 01, 04, 05 |
| Unblocks | 07, 09, 18, 24, 25, 26 |
| Spec refs | §13.1, §13.3, §10 (D10), §7.5 step 5 |

## Goal

Make features and limits sellable. From this increment onward, every capability a customer can reach is gated by a single entitlements answer rather than by scattered checks, so a plan change changes behaviour everywhere at once.

## In scope

- Plan tiers and their limits.
- The entitlements answer and its consumers.
- Feature gating inside the authorization check.
- Hard caps enforcement at the point of creation.
- Assignment of a plan to a billing account.

## Out of scope

- Prices, rating, ledgers and invoices (Features 18, 19).
- Upgrade/downgrade money mechanics and proration (Features 18, 19).
- Usage-based overage calculation (Feature 17, 18).
- Enforcement of lifecycle restrictions (Feature 21).

## Requirements

**Plans**

R1. Four plan tiers exist: Free, Pro, Business, Enterprise.
R2. A billing account has exactly one plan at any time.
R3. The Free plan never incurs charges of any kind.
R4. Every plan defines, as data rather than code:
  - a base price per month, per currency (contract for Enterprise);
  - included node-hours per month, pooled across meters;
  - a connected-node allowance, which may be a hard cap;
  - a project allowance, which may be unlimited;
  - a cluster allowance, which may be unlimited;
  - a feature set (workload identity; custom roles and JIT access; SSO, SCIM and organisation policies; mesh rule approvals);
  - an audit log retention period;
  - the permitted billing modes.
R5. The Free plan permits exactly 3 connected nodes, 1 project and 1 cluster.
R6. Pro permits 5 projects and 10 clusters with unlimited nodes subject to overage.
R7. Business permits unlimited projects and clusters.

**Entitlements**

R8. Exactly one component answers `entitlements(organisation) → { features[], limits{} }`.
R9. No other component reads plan definitions or plan limits directly. Every consumer asks this component.
R10. The answer is derived from the organisation's billing account plan, so a plan change propagates without any per-service change.
R11. Entitlement answers are evaluated at the moment of use, not cached across a plan change in a way that grants revoked features.

**Gating**

R12. The authorization check consults entitlements as a distinct step after permission resolution.
R13. An action belonging to a feature absent from the plan is denied with a distinct, stable error code — not a generic permission denial.
R14. Feature gating is applied to reads as well as writes, so a UI cannot offer what the API will refuse.
R15. Limits are enforced at the point of creation with a message that names the limit and the plan.
R16. A creation that would exceed a hard cap is refused before any resource, policy run or provider call occurs.
R17. Overage beyond a soft allowance is permitted and metered; overage beyond a hard cap is refused.

**Plan changes (structural only)**

R18. The model must accommodate an immediate upgrade and a renewal-boundary downgrade without changing the entitlements contract.
R19. A downgrade that would breach the target plan's limits must be blockable with a list of what the customer must remove first.

## Acceptance criteria

- A Free organisation is refused at the fourth connected node, the second project and the second cluster, each with a message naming the limit and the plan.
- A Business feature (for example SSO configuration) is denied on Pro with the distinct `feature_not_in_plan` code, while a plain permission denial uses a different code.
- Changing a billing account's plan changes gating behaviour everywhere with no service-specific change.
- No service other than the entitlements component contains a plan limit comparison.
