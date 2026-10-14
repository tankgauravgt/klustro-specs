# Klustro — Incremental delivery plan

Requirements-only feature documents, ordered so that each increment delivers something usable and depends only on increments before it.

**Ordering principle:** reach the end-to-end critical path first (signup → project → mesh → node → cluster → `kubectl get nodes`), then deepen each subsystem. Billing enforcement is deliberately late, so that warnings, ledger correctness and audit exist before anything can be restricted.

| # | Feature | Doc | Depends on |
|---|---|---|---|
| 01 | Platform foundation | [feature-01-platform-foundation.md](feature-01-platform-foundation.md) | — |
| 02 | Public API and CLI skeleton | [feature-02-api-and-cli-skeleton.md](feature-02-api-and-cli-skeleton.md) | 01 |
| 03 | Identity, users and sessions | [feature-03-identity-and-sessions.md](feature-03-identity-and-sessions.md) | 01, 02 |
| 04 | Authorization core | [feature-04-authorization-core.md](feature-04-authorization-core.md) | 01–03 |
| 05 | Signup and organisation lifecycle | [feature-05-signup-and-org-lifecycle.md](feature-05-signup-and-org-lifecycle.md) | 01–04 |
| 06 | Plans and entitlements | [feature-06-plans-and-entitlements.md](feature-06-plans-and-entitlements.md) | 01, 04, 05 |
| 07 | Project and mesh connection | [feature-07-project-and-mesh-connection.md](feature-07-project-and-mesh-connection.md) | 01–06 |
| 08 | Policy pipeline core | [feature-08-policy-pipeline-core.md](feature-08-policy-pipeline-core.md) | 01, 04, 07 |
| 09 | Join tokens and onboarding script | [feature-09-join-tokens-and-onboarding-script.md](feature-09-join-tokens-and-onboarding-script.md) | 01–08 |
| 10 | Node agent and enrolment | [feature-10-node-agent-and-enrolment.md](feature-10-node-agent-and-enrolment.md) | 01–09 |
| 11 | Cluster creation and bootstrap | [feature-11-cluster-create-and-bootstrap.md](feature-11-cluster-create-and-bootstrap.md) | 01–10 |
| 12 | Cluster API access and kubeconfig | [feature-12-cluster-api-access-and-kubeconfig.md](feature-12-cluster-api-access-and-kubeconfig.md) | 01–11 |
| 13 | Node and cluster operations | [feature-13-node-and-cluster-operations.md](feature-13-node-and-cluster-operations.md) | 01–12 |
| 14 | Labels and taints | [feature-14-labels-and-taints.md](feature-14-labels-and-taints.md) | 01–13 |
| 15 | Mesh user rules and approvals | [feature-15-mesh-user-rules-and-approvals.md](feature-15-mesh-user-rules-and-approvals.md) | 01–14 |
| 16 | Policy drift detection and rollback | [feature-16-policy-drift-and-rollback.md](feature-16-policy-drift-and-rollback.md) | 01–15 |
| 17 | Metering | [feature-17-metering.md](feature-17-metering.md) | 01, 10 |
| 18 | Rating, ledger and prepaid wallets | [feature-18-billing-ledger-and-wallets.md](feature-18-billing-ledger-and-wallets.md) | 01–06, 17 |
| 19 | Payments, invoices and tax | [feature-19-payments-invoices-and-tax.md](feature-19-payments-invoices-and-tax.md) | 01–06, 18 |
| 20 | Notifications | [feature-20-notifications.md](feature-20-notifications.md) | 01, 03, 04 |
| 21 | Account lifecycle and enforcement | [feature-21-account-lifecycle-enforcement.md](feature-21-account-lifecycle-enforcement.md) | 01–20 |
| 22 | Postpaid billing | [feature-22-postpaid-billing.md](feature-22-postpaid-billing.md) | 01–21 |
| 23 | Audit log and export | [feature-23-audit-log-and-export.md](feature-23-audit-log-and-export.md) | 01, 04–22 |
| 24 | Groups, service principals, workload identity | [feature-24-groups-service-principals-and-workload-identity.md](feature-24-groups-service-principals-and-workload-identity.md) | 01–04, 12 |
| 25 | SSO, SCIM and organisation policies | [feature-25-sso-scim-and-org-policies.md](feature-25-sso-scim-and-org-policies.md) | 01–04, 23, 24 |
| 26 | Just-in-time access | [feature-26-jit-access.md](feature-26-jit-access.md) | 01–04, 20, 21, 24 |
| 27 | Cluster upgrades | [feature-27-cluster-upgrades.md](feature-27-cluster-upgrades.md) | 01–13, 28 |
| 28 | Datastore snapshots | [feature-28-etcd-snapshots.md](feature-28-etcd-snapshots.md) | 01–13 |
| 29 | Platform admin console | [feature-29-platform-admin-console.md](feature-29-platform-admin-console.md) | 01–28 |
| 30 | Observability and resilience | [feature-30-observability-and-resilience.md](feature-30-observability-and-resilience.md) | 01–29 |

## Milestones

| Milestone | Increments | Customer-visible outcome |
|---|---|---|
| **M1 — Tenancy** | 01–06 | A user signs up, owns an organisation, is gated by a plan. |
| **M2 — Cluster on customer hardware** | 07–12 | Success criterion 1: self-service project, three nodes, `kubectl get nodes`. |
| **M3 — Operable clusters** | 13–16 | Nodes can be added, drained, retired; customers can open their own network paths; the policy is guaranteed correct. |
| **M4 — Paying customers** | 17–19 | Usage is metered, rated, charged; wallets and invoices work in USD and INR. |
| **M5 — Enforcement** | 20–23 | Billing consequences are applied predictably, with warnings and an answerable record. Covers all of v1's launch scope. |
| **M6 — Company-grade access** | 24–26 | Teams, CI/CD, in-cluster automation, SSO/SCIM, just-in-time access. |
| **M7 — Lifecycle operations** | 27–28 | Upgrades and snapshots on running clusters. |
| **M8 — Operate the platform** | 29–30 | Staff tooling and proven degradation behaviour. |

## Notes

- Feature 28 (snapshots) is numbered after 27 (upgrades) to keep the reading order sensible, but it must be delivered **before** 27 — upgrades depend on pre-operation snapshots.
- Feature 17 (metering) depends only on 01 and 10, so it can be developed in parallel with 11–16.
- Feature 20 (notifications) depends only on 01, 03 and 04, so it can be developed in parallel with 05–19; it is sequenced before 21 because enforcement requires it.
- Increments 24–26 and 27–28 are independent of each other and can be reordered or parallelised.
- v1.x and later items (snapshots to customer object storage, guided restore, webhooks and event streams, Terraform provider, GitHub Action, proxy support, cluster templates, additional mesh providers, resellers, add-on catalogue, auto-recharge, Slack/Teams, nested groups, deny assignments, ABAC conditions, multi-term selectors) are deliberately absent from this plan.
