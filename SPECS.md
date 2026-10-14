# Klustro — Platform Foundation Specification

| | |
|---|---|
| **Document** | Klustro platform foundation: hierarchy, identity, authorization, mesh, clusters, billing |
| **Status** | Draft for review |
| **Date** | 2026-10-14 |
| **Scope** | v1 (launch), with v1.x and later items called out explicitly |

---

## Table of contents

1. [Overview](#1-overview)
2. [Decision log](#2-decision-log)
3. [Scope and phasing](#3-scope-and-phasing)
4. [Conventions](#4-conventions)
5. [Management hierarchy and core entities](#5-management-hierarchy-and-core-entities)
6. [Identity and principals](#6-identity-and-principals)
7. [Authorization](#7-authorization)
8. [Mesh provider layer (generic)](#8-mesh-provider-layer-generic)
9. [Tailscale provider](#9-tailscale-provider)
10. [Policy pipeline](#10-policy-pipeline)
11. [Node onboarding and the agent](#11-node-onboarding-and-the-agent)
12. [Cluster lifecycle](#12-cluster-lifecycle)
13. [Billing and plans](#13-billing-and-plans)
14. [Account lifecycle and enforcement](#14-account-lifecycle-and-enforcement)
15. [System architecture](#15-system-architecture)
16. [Key flows](#16-key-flows)
17. [Error handling and resilience](#17-error-handling-and-resilience)
18. [Security](#18-security)
19. [Audit, observability and notifications](#19-audit-observability-and-notifications)
20. [Public API and CLI](#20-public-api-and-cli)
21. [Platform admin console](#21-platform-admin-console)
22. [Testing strategy](#22-testing-strategy)
23. [Verification items (proof-of-concept)](#23-verification-items-proof-of-concept)
24. [Glossary](#24-glossary)

---

## 1. Overview

### 1.1 Product

Klustro is a SaaS **management plane** for k3s clusters running on **customer-owned machines** connected by an **overlay mesh network**. Customers onboard their own servers and workers (cloud VMs, on-prem hardware, edge devices, Raspberry Pis) by running a script downloaded from Klustro. Klustro then:

- joins each machine to the project's mesh network,
- installs and configures k3s (embedded etcd) over that mesh,
- generates and enforces the mesh network policy,
- issues short-lived, role-based cluster credentials,
- meters usage and bills it prepaid or postpaid.

Klustro **never runs customer workloads or the customer's control plane**. Every k3s server and worker runs on customer hardware.

### 1.2 Target customers

| Segment | Priority | Needs |
|---|---|---|
| Individual developers and small teams | **Launch focus** | Self-serve signup, fast first cluster, prepaid billing |
| Mid-size companies with several teams | Supported at launch | Projects, SSO/SCIM, RBAC, postpaid invoicing |
| Enterprises, resellers and MSPs | Data model ready, UI later | Nested tenants, contracts, consolidated billing |

### 1.3 Success criteria

1. A new user can sign up, connect a Tailscale tailnet, onboard three nodes and run `kubectl get nodes` against an HA-capable cluster without contacting anyone.
2. Every action in Klustro is controlled by the scoped role-assignment model and is audited.
3. Prepaid and postpaid accounts use one meter and one ledger. Only how charges are settled differs.
4. Running out of money never takes down customer workloads.
5. Adding a new mesh provider requires a new driver, not changes to the core.

---

## 2. Decision log

| # | Decision | Choice |
|---|---|---|
| D1 | Launch segment | Developers/small teams first. Data model ready for mid-size companies and resellers |
| D2 | Where nodes run | All nodes (servers and workers) on customer machines, onboarded by downloadable script. Cloud connectors later |
| D3 | Pricing model | Plan tiers + metered node-hours |
| D4 | Prepaid at zero balance | Graceful degradation: Grace → Frozen → Suspended → Deleted. Workloads never stopped by Klustro |
| D5 | Auto-recharge | Not in v1 |
| D6 | Postpaid eligibility | Prepaid by default. Postpaid unlocked automatically by payment history or manually by sales |
| D7 | Hierarchy | Organisation → Projects → Clusters → Nodes, with a separate Billing Account |
| D8 | Principal model | Entra ID-style principals with scoped role assignments |
| D9 | Identity features at launch | All: users, groups, service principals, node identity, workload identity, built-in and custom roles, JIT, SSO, SCIM, org policies |
| D10 | Feature gating | All built for launch, sold by plan tier |
| D11 | Group types | `native` or `provider_managed`, fixed when the group is created |
| D12 | Network isolation | A completely separate mesh per project |
| D13 | Action naming | `resource[.sub-resource]:verb` |
| D14 | Currencies | USD and INR from launch |
| D15 | Mesh bandwidth | Klustro does not meter, bill or limit mesh/relay bandwidth on any plan |
| D16 | Mesh providers | Pluggable provider layer. Provider chosen at project creation and fixed. **Tailscale only (default) in v1** |
| D17 | Tailscale account model | **One dedicated Tailscale tailnet per project**, connected with customer-provided OAuth client credentials at project creation |
| D18 | Policy ownership | Klustro owns the tailnet policy file completely, managed through a CI/CD-style policy pipeline |
| D19 | Cluster API access | Over the tailnet only. HA entry point is a Tailscale Service per cluster. Klustro never proxies `kubectl` traffic |
| D20 | Identifiers | UUIDv7 everywhere machine-facing, never slugs |
| D21 | Cluster datastore | k3s embedded etcd on server nodes, always (including single-server clusters) |
| D22 | Web terminal / resource browser | Not offered (Klustro cloud has no path to cluster APIs) |

---

## 3. Scope and phasing

### 3.1 v1 (launch)

- Organisations, projects, clusters, nodes, join tokens, labels
- Full identity and authorization model (Sections 6 and 7)
- Generic mesh provider layer with the **Tailscale** driver
- Policy pipeline with tests, plans, approvals, drift reversion and rollback
- Node agent (Linux amd64/arm64), onboarding script, preflight checks
- k3s cluster lifecycle: create, 1/3/5 servers with embedded etcd, add/remove/drain nodes, rolling upgrades, local and automatic etcd snapshots
- Tailscale Service per cluster for HA API access
- Kubeconfig with short-lived Klustro OIDC tokens, `klustro` CLI
- Plans, prepaid wallets, postpaid invoicing, USD/INR, Stripe/Razorpay, tax invoices
- Account lifecycle enforcement
- Audit log, email and in-app notifications
- REST API `/v1`
- Platform admin console

### 3.2 v1.x

- etcd snapshots to customer-provided S3-compatible storage
- Guided snapshot restore and quorum-loss recovery
- Webhooks and event stream
- Terraform provider, GitHub Action
- HTTP proxy support for agent installs
- Cluster templates
- Mesh DNS integration in the UI (MagicDNS names)

### 3.3 Later

- Additional mesh providers: NetBird, Headscale, Cloudflare (only if it proves L3 node-to-node connectivity) and others
- Cloud connectors (Klustro provisions VMs in the customer's cloud account)
- Reseller/partner UI (nested organisations, consolidated billing)
- Add-on catalogue (ingress, cert-manager, monitoring, storage)
- Auto-recharge for prepaid wallets
- Klustro-hosted snapshot storage (paid add-on)
- Slack/Teams notifications, log/metrics forwarding
- Air-gapped installs, Windows workers
- Nested groups, deny assignments, ABAC conditions on role assignments
- Multi-term (AND) selectors in user mesh rules

### 3.4 Explicit non-goals

- Klustro-hosted compute of any kind
- Proxying or terminating cluster API traffic
- Cross-project networking
- Clusters with more than 5 servers
- External datastores (SQLite/kine, external etcd, PostgreSQL/MySQL)
- Separate etcd-only or control-plane-only node roles

---

## 4. Conventions

### 4.1 Identifiers

- Every resource has an immutable **`id`**: a **UUIDv7** in lowercase canonical form (e.g. `0192f3b0-7c1e-7a4d-9b2f-3e8a1c6d5f40`).
- Every user-facing resource also has a mutable **`display_name`**, used only in the UI and CLI output.
- **Anything machine-facing uses only the `id`**: scope paths, API URLs, Tailscale tags, Tailscale Service names, DNS names, node hostnames, audit records, OIDC claims.
- **Renaming a resource never changes anything outside Klustro's database.** No policy run, no kubeconfig change.
- Fixed values that are not resources keep readable names: `tag:klustro`, built-in role names, action names, plan names, states.
- The CLI accepts either a display name or an ID. A display name is resolved within the current org/project context and fails with an error if it matches more than one resource.

Placeholders in this document such as `<cluster-uuid7>` mean that resource's UUIDv7.

### 4.2 Action names

Actions follow `resource[.sub-resource]:verb` (Section 7.1).

### 4.3 Timestamps, money and configuration

- Timestamps are UTC, RFC 3339.
- Money is stored as integer minor units (cents, paise) with an ISO 4217 currency code.
- Values described as **configurable** (prices, thresholds, timers) live in versioned platform configuration with effective-from dates, not in code. Defaults are given where relevant.

---

## 5. Management hierarchy and core entities

### 5.1 Hierarchy

```
User (global identity)
 └─ Membership ──► Organisation                         (tenant; parent_org_id reserved)
                    ├─ Billing Account                   (may serve many orgs; 1:1 in v1 UI)
                    ├─ Groups                            (native | provider_managed)
                    ├─ Service Principals
                    ├─ Custom Role Definitions
                    ├─ Role Assignments                  (principal + role + scope)
                    ├─ Org Policies, SSO / SCIM configuration
                    └─ Projects                          (one dedicated mesh each)
                         ├─ Mesh Network + Mesh Connection (provider fixed at creation)
                         ├─ Mesh Policy (user rules) + Policy Revisions
                         ├─ Labels
                         ├─ Join Tokens
                         ├─ Unassigned Nodes
                         └─ Clusters
                              ├─ Nodes (role: server | agent)
                              ├─ Tailscale Service (API entry point)
                              └─ Snapshots
```

### 5.2 Entities

#### User
- Global identity: `id`, `email` (unique, verified), `display_name`, authentication methods (password, TOTP, passkeys, SSO links), personal access tokens.
- Can belong to many organisations and switch between them.

#### Organisation
- The **tenant and hard isolation boundary**. No resource, mesh or audit record crosses organisations.
- Fields: `id`, `display_name`, `billing_account_id`, `parent_org_id` (nullable, **reserved for resellers**, always null in v1), `policies`, `sso_config`, `scim_config`, `created_at`, `deleted_at`.
- On signup, a **personal organisation** and a **Free, prepaid billing account** are created automatically. No project is created automatically, because a project needs mesh provider credentials. The user is taken straight to the "Create your first project" wizard (Section 9.2).

#### Membership
- Links a user to an organisation. Removing a membership removes all of that user's role assignments in that org.
- Invited by email (`members:invite`) or provisioned through SCIM.

#### Billing Account
- Separate from the organisation so one account can later pay for many orgs (resellers).
- Fields: `id`, `mode` (`prepaid` | `postpaid`), `plan`, `currency` (fixed at creation), `tax_profile`, `payment_methods`, `wallet` (prepaid), `credit_limit` (postpaid), `lifecycle_state`, `postpaid_terms`.
- The lifecycle state (Section 14) applies to every organisation linked to the account.

#### Project
- Groups clusters and nodes and **owns exactly one dedicated mesh network**.
- Fields: `id`, `org_id`, `display_name`, `mesh_provider_id`, `mesh_connection_id`, `status`, `deleted_at`.
- `mesh_provider_id` and `mesh_connection_id` are **fixed once the project is created**. Moving to another provider means creating a new project and re-onboarding nodes.
- Projects can never communicate with each other at the network level.

#### Cluster
- A k3s cluster inside a project. Fields: `id`, `project_id`, `display_name`, `k3s_version`, `options` (Section 12.2), `status`, `service_name` (`svc:k-<cluster-uuid7>`), `bootstrap_node_id`, `etcd_tuning`.

#### Node
- A customer machine running `klustro-agent`. It belongs to exactly one project, and to at most one cluster.
- Fields: `id`, `project_id`, `cluster_id` (nullable), `role` (`server` | `agent` | null while unassigned), `labels`, `taints`, `facts` (OS, kernel, arch, CPU, RAM), `status`, `mesh_peer_id`, `join_token_id`, `last_heartbeat_at`.
- Moving a node to another project means removing it and onboarding it again.

#### Join Token
- Authorises machines to enrol into a project. Fields: `id`, `project_id`, `cluster_id` (optional), `role` (optional; required if `cluster_id` set), `labels`, `max_uses`, `uses`, `expires_at`, `created_by`, `revoked_at`, `secret_hash`.
- The secret is shown once and stored only as a hash.

#### Label
- A distinct `key=value` pair registered per project, with its own `id`. Used to give user-defined labels safe Tailscale tag names (Section 9.4).
- Keys and values follow Kubernetes label syntax and are synced to k8s node labels.

### 5.3 Deletion semantics

| Resource | Behaviour |
|---|---|
| **Organisation** | Soft delete. Restorable for 7 days (configurable), then purged. Requires all projects deleted and the billing account settled |
| **Project** | Soft delete with 7-day restore window. During the window nodes keep running and Klustro stops changing anything. On purge: agents uninstall themselves and their dedicated `tailscaled`; k3s is uninstalled only if the user chose that when deleting; Klustro removes the devices it created from the tailnet, restores the policy backed up at takeover (revision 0) unless the user opted out, and deletes the stored OAuth credentials |
| **Cluster** | Immediate after typed-name confirmation. Nodes return to **unassigned**. The user chooses whether k3s is uninstalled from the nodes (default: uninstall). Tailscale Service removed by the policy pipeline |
| **Node** | Drain (unless forced) → remove from cluster → agent uninstalls → device removed from tailnet |

---

## 6. Identity and principals

### 6.1 Principal types

Every identity that can act is a **principal** with a `principal_id` (UUIDv7). Permission checks never depend on the principal type.

| Principal | Entra ID equivalent | Lives in | Authenticates with |
|---|---|---|---|
| **User** | User | Global | Password + MFA, passkey, SSO; personal access tokens for API/CLI |
| **Group** | Group | Organisation | Not directly (a container for role assignments) |
| **Service Principal** | App registration / service principal | Organisation | Client secret (≤ 1 year), certificate, or federated OIDC credential |
| **Node Identity** | System-assigned managed identity | Node | mTLS client certificate issued at enrolment |
| **Workload Identity** | Workload identity federation | Cluster | Kubernetes service account token exchanged for a Klustro token |

### 6.2 Users and authentication
- MFA methods: TOTP and passkeys (WebAuthn). Org policy can require MFA.
- Personal access tokens: scoped to one org, expiry ≤ 1 year, revocable. Org policy can disable them.
- **Break-glass login:** an Owner can always sign in with a passkey, even when SSO-only is enforced or the identity provider is down.

### 6.3 Groups
- Each group has `source`, fixed at creation:
  - **`native`**: membership managed in Klustro (`groups:manage`).
  - **`provider_managed`**: membership comes from an external identity provider (Entra ID, Okta, Google Workspace) through SCIM, or through SSO group claims. **Membership is read-only in Klustro.**
- A group's source never changes. To switch, create a new group and move the role assignments.
- Groups can contain users and service principals (native groups only for service principals). Nested groups are later.

*Example:* Entra group `k8s-prod-ops` is synced through SCIM as a provider-managed group and assigned **Cluster Operator** on the `prod` project. When IT removes Alex from the Entra group, the next SCIM sync removes Alex from the Klustro group. Alex's next token refresh no longer contains the role, so `kubectl` stops working within one hour.

### 6.4 Service principals
- Non-human identities for CI/CD, Terraform and automation.
- Credentials:
  - **Client secret:** maximum lifetime 1 year, shown once.
  - **Certificate:** customer uploads a public certificate.
  - **Federated OIDC credential:** trust an external issuer and subject pattern, e.g. issuer `https://token.actions.githubusercontent.com`, subject `repo:acme/app:ref:refs/heads/main`. No stored secret.
- Limits are described in Rule R7 (Section 7.4).

### 6.5 Node identity
- Created when a node enrols (Section 11.3). The agent generates a key pair and receives an mTLS certificate whose subject is the node's UUIDv7.
- Certificates are rotated automatically every 30 days and revoked when the node is removed.
- Used only for agent ↔ Klustro communication. The node's mesh identity is separate (Section 9).

### 6.6 Workload identity
- A pod's Kubernetes service account token can be exchanged at Klustro's token endpoint (RFC 8693 token exchange) for a Klustro access token.
- Klustro cannot reach the cluster API (D19), so the **agent uploads the cluster's service-account issuer public keys (JWKS)** to Klustro whenever they change.
- A **workload identity binding** maps `cluster_id + namespace + service account` to a workload identity principal, which receives role assignments like any other principal.

*Example:* an in-cluster autoscaler running as `kube-system/edge-scaler` is bound to workload identity `0192f4a0-…`, which has the custom role **Node Onboarder** on the `edge` project. It can create join tokens when it detects capacity pressure.

### 6.7 SSO
- Per organisation, OIDC or SAML 2.0 (Entra ID, Okta, Google Workspace, any compliant provider).
- **Domain verification** by DNS TXT record is required before SSO can be enforced for a domain.
- **SSO-only** org policy blocks password login for org members (break-glass excepted).
- Group claims from SSO can feed provider-managed groups when SCIM is not configured.

### 6.8 SCIM
- SCIM 2.0 endpoint per organisation, authenticated with a SCIM bearer token (`scim:configure`).
- Provisions and deprovisions **memberships** and **provider-managed groups**. Deprovisioning removes the org membership and its role assignments; the global user account remains.

### 6.9 Org policies
| Policy | Effect |
|---|---|
| Require MFA | Members without MFA are prompted to enrol at next login and blocked until they do |
| Require SSO | Password login disabled for members (break-glass excepted) |
| IP allowlist | Applies to the Klustro UI, API and **token issuance** (including `klustro kubeconfig token`). Not applied to `kubectl` traffic, which Klustro does not see |
| Maximum session / token lifetime | Caps web sessions, personal access tokens and service principal secrets |
| Disable personal access tokens | — |
| Require approval for mesh rule changes | Enables four-eyes approval for user mesh rules (Section 10.7) |

---

## 7. Authorization

### 7.1 Action format

```
<resource>[.<sub-resource>]:<verb>
```

Wildcards are allowed in **built-in** roles only:

| Pattern | Meaning |
|---|---|
| `clusters:*` | Every verb on clusters |
| `*:read` | Read every resource |
| `billing.*:*` | Everything under billing sub-resources |
| `*` | Everything |

### 7.2 Action catalogue

**Organisation and membership**

| Action | Allows | Example |
|---|---|---|
| `org:read` | View org settings and member list | Opening org settings |
| `org:update` | Rename org, change org policies | Turning on "Require MFA" |
| `org:delete` | Delete the organisation | Closing a company account |
| `members:invite` | Invite users by email | Inviting `priya@acme.com` |
| `members:remove` | Remove a member | Offboarding a contractor |

**Identity**

| Action | Allows | Example |
|---|---|---|
| `groups:read` | View groups and membership | Listing `platform-team` |
| `groups:manage` | Create/delete groups, edit native group membership | Creating native group `sre-oncall` |
| `serviceprincipals:read` | View service principals | — |
| `serviceprincipals:manage` | Create/delete service principals and credentials | Creating `github-deployer` |
| `workloadidentities:manage` | Create/delete workload identity bindings | Binding `kube-system/edge-scaler` |
| `sso:configure` | Configure SSO and domain verification | Adding an Entra tenant |
| `scim:configure` | Enable SCIM, rotate SCIM token | Generating a SCIM token |
| `roles:read` | View role definitions | — |
| `roles:define` | Create/edit/delete custom roles | Creating **Node Onboarder** |
| `roleassignments:read` | View role assignments | "Who has access to `prod`?" |
| `roleassignments:write` | Create/delete role assignments (active or eligible) | Giving `sre-oncall` Cluster Operator on `prod` |
| `jit:request` | Activate an eligible assignment | "Cluster Operator for 2 hours" |
| `jit:approve` | Approve another principal's activation | Team lead approving a request |

**Projects and mesh**

| Action | Allows | Example |
|---|---|---|
| `projects:create` | Create a project, including providing its mesh connection credentials | Creating `staging` |
| `projects:read` | View a project | Opening the `prod` dashboard |
| `projects:update` | Rename, change project settings | Renaming `prod` |
| `projects:delete` | Soft-delete a project | Removing `old-demo` |
| `mesh:read` | View mesh status, peers, policy revisions, runs, drift events | Debugging connectivity |
| `mesh.providers:read` | View the provider catalogue and capabilities | — |
| `mesh.connections:read` | View connection status (credentials always masked) | — |
| `mesh.connections:update` | Rotate the project's provider credentials | Pasting a new OAuth secret |
| `mesh.policy:write` | Create/edit/retire user mesh rules (drafts) | Adding `site=pune → tier=db:5432` |
| `mesh.policy:approve` | Approve pending user rules (not one's own) | Lead approving that rule |
| `mesh.policy:rollback` | Roll back to an earlier revision | Reverting to #14 |
| `mesh.policy:resync` | Trigger a policy pipeline run manually | "Re-sync" button |

A project's mesh connection is created with the project and deleted with it, so there is no separate create or delete action for connections.

**Nodes and labels**

| Action | Allows | Example |
|---|---|---|
| `nodes.jointokens:create` | Generate an onboarding token/script | Script for 3 new Raspberry Pis |
| `nodes.jointokens:read` | List tokens and which nodes used them | — |
| `nodes.jointokens:revoke` | Revoke a token | Token leaked in chat |
| `nodes:read` | View nodes, health, mesh IP, facts | Is `edge-07` online? |
| `nodes:update` | Edit labels and taints | Adding `site=pune` |
| `nodes:assign` | Assign an unassigned node to a cluster as server or agent | Making a node a server |
| `nodes:drain` | Cordon and drain | Before OS patching |
| `nodes:remove` | Remove a node from the project | Retiring hardware |

**Clusters**

| Action | Allows | Example |
|---|---|---|
| `clusters:create` | Create a cluster | Creating `prod-eu` |
| `clusters:read` | View cluster status, version, nodes | Cluster health page |
| `clusters:update` | Change cluster options | Enabling "dedicated servers" |
| `clusters:upgrade` | Start, pause, resume a k3s upgrade | v1.31 → v1.32 |
| `clusters:delete` | Delete a cluster | Tearing down `test` |
| `clusters.kubeconfig:view` | Obtain tokens mapped to k8s `view` | `kubectl get pods` |
| `clusters.kubeconfig:edit` | Obtain tokens mapped to k8s `edit` | `kubectl apply` |
| `clusters.kubeconfig:admin` | Obtain tokens mapped to k8s `cluster-admin` | Installing CRDs |
| `clusters.snapshots:read` | List etcd snapshots | — |
| `clusters.snapshots:create` | Take an on-demand snapshot | Before a risky change |
| `clusters.snapshots:restore` | Reserved for v1.x | — |

**Billing**

| Action | Allows | Example |
|---|---|---|
| `billing:read` | View plan, usage, balance, credit limit | This month's node-hours |
| `billing.wallet:topup` | Add prepaid credit | Adding $50 / ₹4,000 |
| `billing.plan:change` | Upgrade/downgrade the plan | Pro → Business |
| `billing.paymentmethods:manage` | Add/remove payment methods | Replacing an expired card |
| `billing.invoices:read` | Download invoices and receipts | Sending to accounting |
| `billing.taxprofile:update` | Edit tax details (GSTIN, VAT ID, address) | Adding a GSTIN |
| `billing.mode:change` | Accept postpaid offer, switch back to prepaid | "Switch to monthly invoicing" |

**Audit**

| Action | Allows | Example |
|---|---|---|
| `audit:read` | Read and export the audit log | "Who deleted `prod-eu`?" |

### 7.3 Role definitions

A role definition has `actions[]` and `not_actions[]` (Azure-style). Effective actions = `actions − not_actions`.

#### Built-in roles

| Role | `actions` | `not_actions` | Typical holder |
|---|---|---|---|
| **Owner** | `*` | — | Founder / account owner |
| **Admin** | `*` | `org:delete`, `billing.*:*` | Head of platform |
| **Billing Admin** | `billing:read`, `billing.*:*`, `org:read` | — | Finance (no cluster access) |
| **Identity Admin** | `org:read`, `members:*`, `groups:*`, `serviceprincipals:*`, `workloadidentities:*`, `sso:*`, `scim:*`, `roles:*`, `roleassignments:*` | — | IT admin |
| **Contributor** | `projects:*`, `mesh:read`, `mesh.providers:read`, `mesh.connections:*`, `mesh.policy:write`, `mesh.policy:resync`, `nodes:*`, `nodes.*:*`, `clusters:*`, `clusters.snapshots:read`, `clusters.snapshots:create`, `clusters.kubeconfig:edit` | — | Platform engineer |
| **Cluster Operator** | `projects:read`, `mesh:read`, `nodes:*`, `nodes.*:*`, `clusters:read`, `clusters:update`, `clusters:upgrade`, `clusters.snapshots:read`, `clusters.snapshots:create`, `clusters.kubeconfig:admin` | — | SRE |
| **Developer** | `projects:read`, `nodes:read`, `clusters:read`, `clusters.kubeconfig:edit` | — | Application developer |
| **Reader** | `*:read`, `clusters.kubeconfig:view` | — | Stakeholder, support |
| **Auditor** | `*:read`, `audit:read` | — | Compliance (no kubeconfig) |

`mesh.policy:approve` and `mesh.policy:rollback` are held by Owner and Admin by default and can be granted through custom roles.

#### Custom roles (Business and above)
- Defined per organisation, assignable at any scope in that org.
- **Must list explicit actions; wildcards are not allowed.** New actions added to Klustro are therefore never granted silently.
- Cannot contain actions that do not exist in the catalogue.

*Example:* **Node Onboarder** = `projects:read`, `nodes:read`, `nodes.jointokens:create`. A field technician can create join scripts for new edge devices but cannot touch clusters.

### 7.4 Role assignments and rules

```
RoleAssignment {
  id              uuid7
  principal_id    user | group | service_principal | workload_identity
  role_id         built-in or custom role definition
  scope           /orgs/<org-uuid7>
                  /orgs/<org-uuid7>/projects/<project-uuid7>
                  /orgs/<org-uuid7>/projects/<project-uuid7>/clusters/<cluster-uuid7>
  type            active | eligible
  k8s_namespaces  [string]   optional; only meaningful with clusters.kubeconfig:edit
  jit_policy      { max_duration, requires_approval }   eligible only
  created_by, created_at, expires_at?
}
```

**R1. Scope inheritance.** An assignment applies at its scope and every scope below it (org → project → cluster). Checks walk up the scope path.

*Example:* Meera has **Reader** at `/orgs/<acme-uuid7>`, so she can read every project and cluster in Acme.

**R2. Union, no deny.** Effective permissions are the union of all active assignments, direct and through groups. There are no deny assignments in v1.

*Example:* Ravi has **Reader** at the org (via group `all-engineers`) and **Developer** on the `staging` project (direct). He can read everything and deploy to `staging`.

**R3. No privilege escalation.** A principal can create an assignment only if (a) it holds `roleassignments:write` at that scope or above, and (b) it holds every effective action of the role at that scope.

*Example:* an Identity Admin cannot assign themselves Owner, because Owner includes `org:delete`, which they don't hold.

**R4. At least one Owner.** The last active Owner assignment at org scope cannot be removed or demoted, and the last Owner cannot leave the org.

**R5. Group source is fixed** (Section 6.3).

**R6. Custom roles** follow Section 7.3.

**R7. Service principal limits.** Service principals and workload identities cannot hold Owner. They cannot hold `roleassignments:write`, `sso:configure`, `scim:configure` or `roles:define` unless an Owner enables "allow identity management" on that specific principal.

*Example:* `github-deployer` has **Developer** on cluster `prod-eu`. A GitHub Actions job (already on the tailnet) logs in with federated OIDC, gets a kubeconfig token and deploys.

**R8. Just-in-time access (Business and above).** An **eligible** assignment does nothing until activated. Activation requires a justification, lasts 1–8 hours (capped by `jit_policy.max_duration`), and requires approval by a principal with `jit:approve` if `requires_approval` is set. The approver cannot be the requester. Activation, approval and expiry are audited.

*Example:* Sana is eligible for **Cluster Operator** on `prod`. During an incident she requests 2 hours; her lead approves; access expires automatically.

**R9. Kubernetes mapping.** Cluster access is granted through Klustro OIDC tokens (Section 12.6):

| Highest kubeconfig action on the cluster | Token group | Kubernetes binding |
|---|---|---|
| `clusters.kubeconfig:admin` | `klustro:admin` | ClusterRoleBinding → `cluster-admin` |
| `clusters.kubeconfig:edit` | `klustro:edit` | ClusterRoleBinding → `edit` |
| `clusters.kubeconfig:edit` with `k8s_namespaces` | `klustro:edit:ns:<namespace>` | RoleBinding → `edit` in each listed namespace |
| `clusters.kubeconfig:view` | `klustro:view` | ClusterRoleBinding → `view` |

If a principal has both an unrestricted and a namespace-restricted edit grant, the unrestricted one wins.

**R10. Workload identity** principals follow the same assignments and rules (Section 6.6).

**R11. Org policies** apply to every principal in the organisation (Section 6.9).

### 7.5 Authorization check

`check(principal_id, action, scope) → allow | deny` is answered only by the Authz Service:

1. Expand the principal: itself + all groups it belongs to.
2. Collect active assignments (including activated JIT) at the scope and every ancestor scope.
3. Compute effective actions per role (`actions − not_actions`) and union them.
4. Allow if any effective action pattern matches `action`.
5. Apply plan entitlements: an action belonging to a feature not included in the plan is denied with `feature_not_in_plan` (Section 13.3).
6. Apply billing lifecycle restrictions: write actions are denied with `account_frozen` while the account is Frozen or Suspended (Section 14).

---

## 8. Mesh provider layer (generic)

### 8.1 Principle

Klustro owns **intent**; the provider owns **transport**.

| Klustro owns (same for every provider) | Provider owns |
|---|---|
| Project/cluster membership of nodes | Encryption, keys, peer discovery |
| Node roles and labels | NAT traversal and relays |
| Desired traffic policy (selectors, ports) | Policy syntax and enforcement |
| Enrolment workflow and audit | Enrolment credential format |
| Health and status views | IP assignment, interface, DNS |
| k3s configuration over the mesh | — |

Clusters, RBAC, billing and the UI work only with the generic objects below. Each provider is a **driver**.

### 8.2 Generic data model

```
MeshProvider        catalogue entry shipped with Klustro
  id, name, driver_version, capabilities, credential_schema, supported_os_arch

MeshConnection      the project's provider credentials (created with the project)
  id, org_id, project_id, provider_id, ownership (byo | managed), endpoint_url?,
  encrypted_credentials, status (valid | invalid | degraded), last_validated_at,
  provider_account_ref (e.g. tailnet ID; globally unique across Klustro)

MeshNetwork         1:1 with project, immutable
  id, project_id, provider_id, connection_id, provider_network_ref,
  mesh_cidrs (reported by provider), sync_status

MeshPeer            1:1 with node
  id, node_id, provider_device_id, mesh_ipv4, mesh_ipv6?, interface, mtu,
  transport (direct | relayed | unknown), last_handshake_at, tags

MeshRule            user rule (Section 10.7)
  id, project_id, from_selector, to_selector, ports[], protocol, status, author, approver

PolicyRun / PolicyRevision / DriftEvent      (Section 10)
```

In v1 every connection has `ownership = byo`. `managed` is reserved for providers Klustro may host later.

### 8.3 Capabilities

Each driver declares its capabilities. The UI, validation and k3s configuration adapt to them, and no other component contains provider-specific logic.

| Capability | Values | Use |
|---|---|---|
| `l3_node_to_node` | bool | **Required for k3s projects.** Each node has a routable mesh IP reachable by others |
| `transport` | `p2p_with_relay_fallback`, `relayed_only` | Latency warnings for etcd |
| `policy_granularity` | `none`, `node`, `node+port` | Whether port rules can be enforced |
| `tagging` | bool | Mapping selectors to provider objects |
| `enrollment` | `one_time_key`, `reusable_key`, `jwt`, `service_token` | Agent join method |
| `ephemeral_devices` | bool | Cleanup behaviour |
| `self_hosted` | bool | Whether `endpoint_url` is required |
| `services` | bool | Whether the provider offers HA virtual services (needed for the cluster API entry point, Section 12.5) |
| `policy_tests` | bool | Whether the provider evaluates embedded policy tests on save |
| `dns` | bool | Optional display of provider DNS names |
| `ipv6` | bool | Dual-stack option |
| `mtu` | integer | Reported effective MTU |

A provider is offered for new projects only if it declares `l3_node_to_node`, `tagging`, `policy_granularity = node+port` and `services`. Future providers that lack `services` would need an alternative API entry-point design before they can be offered.

### 8.4 Driver interface

**Controller driver** (runs in Klustro cloud, talks to the provider API):

```
ValidateConnection(credentials, endpoint?)           → account ref, detected capabilities, problems[]
PrepareNetwork(project)                              → provider_network_ref
CreateEnrollmentKey(project, tags[], ttl)            → single-use enrollment secret
ListPeers(network) / GetPeer(device_id)              → MeshPeer
SetPeerTags(device_id, tags[])
RemovePeer(device_id)
EnsureService(network, name, port) / DeleteService(network, name)
ReadPolicy(network)                                  → content, etag
ValidatePolicy(network, content)                     → ok | errors[]
WritePolicy(network, content, if_match_etag)         → new etag | precondition_failed
TeardownNetwork(network, restore_backup: bool)
```

**Node driver** (plugin inside `klustro-agent`):

```
Preflight()                          → problems[] (conflicts, kernel, ports)
EnsureClient(version)                → install / verify provider client
Join(enrollment_secret, hostname, tags[])
Status()                             → interface, mesh IPs, MTU, connected, per-peer transport and RTT
AdvertiseService(name, port, target) / WithdrawService(name)
Leave()                              → log out and clean up
```

### 8.5 Adding a provider

A new provider requires: both driver halves, a capability manifest, a credential schema (rendered as the project-creation form) and a passing **driver conformance suite**:

1. Validate good and bad credentials.
2. Join 3 nodes; every node reaches every other on allowed ports.
3. Apply a policy; a blocked port is unreachable; an allowed port is reachable.
4. Create a service; it fails over when a host withdraws.
5. Retag a node; access changes accordingly.
6. Remove a node; it disappears from the provider.
7. Simulate an out-of-band policy edit; drift is detected and reverted.
8. Teardown restores the backed-up state.

Drivers are built into Klustro in v1. A public driver SDK is later.

---

## 9. Tailscale provider

### 9.1 Account model

- **One dedicated Tailscale tailnet per project.** The tailnet should contain only Klustro-managed nodes and the customer's users who need cluster access.
- Klustro connects with an **OAuth client** created by the customer in that tailnet.
- **Klustro owns the tailnet's entire policy file** (Section 10).
- A tailnet can be bound to only one Klustro project, across all organisations.
- Tailscale's control plane and DERP relays are operated by Tailscale. Klustro runs no mesh infrastructure, and relay bandwidth is a matter between the customer and Tailscale (D15).

**Capability manifest:** `l3_node_to_node: true`, `transport: p2p_with_relay_fallback`, `policy_granularity: node+port`, `tagging: true`, `enrollment: one_time_key`, `ephemeral_devices: true` (not used for Klustro nodes), `self_hosted: false`, `services: true`, `policy_tests: true`, `dns: true`, `ipv6: true`, `mtu: 1280`.

### 9.2 Project creation

```
New project
  1. Display name
  2. Mesh provider: [Tailscale ▾]   (only option in v1, preselected)
  3. Tailscale setup wizard
  4. OAuth client ID + secret
  5. Validation
  6. Takeover confirmation
  7. Project ready
```

**Setup wizard steps shown to the customer:**

1. Create a new tailnet for this project.
2. Add the bootstrap snippet to the tailnet's policy file so the OAuth client can be granted Klustro's root tag:
   ```jsonc
   { "tagOwners": { "tag:klustro": ["autogroup:admin"] } }
   ```
3. Create an OAuth client with the scopes below and the tag `tag:klustro`.
4. Paste the client ID and secret into Klustro.

**Required OAuth client scopes** (exact scope names confirmed in V1, Section 23):

| Scope | Purpose |
|---|---|
| Policy file — read and write | Klustro owns the policy |
| Auth keys — write, tag `tag:klustro` | Single-use, tagged enrolment keys |
| Devices — read and write | List, retag and remove nodes |
| Services — read and write | Create and delete the per-cluster API service |
| DNS — read (optional) | Display MagicDNS names |

**Validation** (all must pass before the project is created):

| Check | Failure message |
|---|---|
| Credentials authenticate | "Tailscale rejected these credentials" |
| Each required scope works (non-destructive probe) | "Missing scope: policy file write" |
| The OAuth client can create `tag:klustro` keys | "`tag:klustro` is not owned. Add the bootstrap snippet" |
| Tailnet not bound to another Klustro project | "This tailnet is already used by another Klustro project" |
| Tailscale Services available on the tailnet's plan | "Your Tailscale plan does not include Services" |
| Existing devices without `tag:klustro*` tags | Warning (not a failure): "N existing devices will lose all access when Klustro takes over the policy" |

**Takeover confirmation:** the user must confirm *"Klustro will replace this tailnet's entire policy file and manage it from now on."* Klustro then:

1. Reads and stores the current policy as **revision 0 (backup)**.
2. Runs the policy pipeline to apply revision 1.
3. Recommends enabling Tailscale's setting that prevents policy edits in the admin console, if available (V6).

### 9.3 Credentials

- Stored with envelope encryption under the organisation's data key (Section 18.2). Always masked in UI and API.
- **Never sent to nodes.** Nodes receive only single-use enrolment keys (Section 11.3).
- Validated every 15 minutes. On failure the project status becomes **Mesh management degraded**: the mesh and workloads keep running, while joins, retags and policy runs pause until credentials are fixed. Admins are notified.
- **Rotation** (`mesh.connections:update`): the new secret is validated before replacing the old one. The old one is discarded only after the new one passes.

### 9.4 Tag scheme

Tailscale grant sources and destinations are lists of alternatives and cannot express "has tag A AND tag B". Role and cluster are therefore combined into a single tag.

```
tag:klustro                         root tag; owned by autogroup:admin; owns every other Klustro tag
tag:k-c-<cluster-uuid7>             any node of the cluster
tag:k-c-<cluster-uuid7>-server      server node of the cluster
tag:k-c-<cluster-uuid7>-agent       worker node of the cluster
tag:k-unassigned                    onboarded node not in a cluster
tag:k-l-<label-uuid7>               user label (key=value registered as a Label)
```

| Node state | Tags carried |
|---|---|
| Unassigned, labels `site=pune` | `tag:k-unassigned`, `tag:k-l-<site=pune uuid7>` |
| Server of cluster C | `tag:k-c-<C>`, `tag:k-c-<C>-server`, label tags |
| Worker of cluster C | `tag:k-c-<C>`, `tag:k-c-<C>-agent`, label tags |

Tag names use lowercase letters, digits and dashes only (V5 confirms Tailscale's character and length limits).

### 9.5 Devices
- Tailnet hostname: **`n-<node-uuid7>`**.
- Devices are tagged, so Tailscale node-key expiry does not apply to them (V7 confirms). Servers never drop off the mesh because a key expired.
- Klustro removes a device only when the node is removed, or when the project is purged. **Klustro never removes devices for billing reasons.**

### 9.6 Cluster API service
Each cluster gets a Tailscale Service `svc:k-<cluster-uuid7>` on `tcp:6443` (Section 12.5).

---

## 10. Policy pipeline

### 10.1 Purpose

For each project, Klustro holds the desired state of the tailnet: clusters, nodes, roles, labels and user rules. The policy pipeline is the **only** path that turns this state into the tailnet's policy file and Tailscale Service objects. Nothing else in Klustro writes to Tailscale's policy.

```
             ┌──────────────────────── per project, strictly one at a time ───────────────────────┐
 Triggers ──►│ Queue ─► Render ─► Test ─► Plan ─► Approve? ─► Apply ─► Verify ─► Record ─► Notify  │
             └────────────────────────────────────────────────────────────────────────────────────┘
                                             ▲
                             Drift detector (every 5 min; webhook if available)
```

### 10.2 Triggers

| Trigger | Example |
|---|---|
| `topology` | Cluster created/deleted; node assigned/unassigned; role change |
| `label` | New label value used in the project; label removed from all nodes |
| `rule_activated` / `rule_retired` | User rule approved or retired |
| `drift` | Out-of-band edit detected |
| `renderer_upgrade` | Klustro ships a new renderer version |
| `manual_resync` | `mesh.policy:resync` |
| `rollback` | `mesh.policy:rollback` |
| `takeover` | Project creation (revision 1) |

### 10.3 Queueing and concurrency

- **One lane per project:** a lease-based worker ensures only one run per project is in progress. Different projects run in parallel.
- **Coalescing:** triggers arriving within 2 seconds are merged into one run, which records all of them.
- **Render from current state:** a run renders the complete policy from the state at the moment it runs, never a delta. Stale or out-of-order triggers cannot cause harm.
- **No-op:** if the rendered hash equals the last applied hash, the run ends as `no_change` without writing to Tailscale.

### 10.4 Render

The renderer is deterministic and versioned (e.g. `renderer v1.0`). The rendered file contains:

| Block | Content |
|---|---|
| Header comment | `// Managed by Klustro. Do not edit. Project <project-uuid7>, revision #<seq>, renderer v<x.y>` |
| `tagOwners` | `tag:klustro` → `autogroup:admin`; every Klustro tag in use → `tag:klustro` |
| `grants` | System rules (Section 10.5) followed by active user rules (Section 10.7) |
| `autoApprovers.services` | Each cluster's service → that cluster's `-server` tag |
| `tests` | Generated tests (Section 10.6) |
| All other blocks | Absent: no SSH rules, no `nodeAttrs`, no implicit access |

Output is canonical (sorted keys, stable ordering of grants by rule ID), so identical state always produces byte-identical content. The hash is SHA-256 of the canonical JSON.

### 10.5 System rules

Generated from topology, **not editable or deletable**, recorded with `origin: system`. For each cluster `C`:

| From | To | Ports | Purpose |
|---|---|---|---|
| `tag:k-c-<C>-server` | `tag:k-c-<C>-server` | tcp:2379-2380 | etcd client and peer |
| `tag:k-c-<C>` | `tag:k-c-<C>-server` | tcp:6443 | k3s API and supervisor |
| `tag:k-c-<C>` | `tag:k-c-<C>` | tcp:10250, udp:8472 | kubelet, flannel VXLAN |
| `tag:k-c-<C>`, `autogroup:member` | `svc:k-<C>` | tcp:6443 | Cluster API through the Tailscale Service |

Unassigned nodes receive no grants. Traffic between clusters in the same project is denied by default, and so is everything else not listed.

### 10.6 Tests

Three layers. If any fails, **nothing is applied**.

1. **Local checks:** schema; tag names valid; every referenced tag present in `tagOwners`; no `*` sources or destinations; size within Tailscale's limit.
2. **Embedded policy tests** in the file's `tests` block. Tailscale refuses to save a policy whose tests fail. Generated per tag pair, not per node:
   ```jsonc
   { "src": "tag:k-c-<C>-server", "accept": ["tag:k-c-<C>-server:2379", "tag:k-c-<C>-server:6443"] },
   { "src": "tag:k-c-<C>-agent",  "accept": ["tag:k-c-<C>-server:6443"],
                                  "deny":   ["tag:k-c-<C>-server:2379"] },
   { "src": "tag:k-c-<C>",        "deny":   ["tag:k-c-<D>:10250"] },        // cluster C ↛ cluster D
   { "src": "tag:k-unassigned",   "deny":   ["tag:k-c-<C>-server:6443"] }
   ```
   One accept test is also generated for every active user rule.
3. **Tailscale validate API** on the full candidate file.

A failure is a Klustro defect (the policy is rendered from Klustro's model). The run ends as `failed_validation`, Klustro on-call is paged, and the project shows *"Mesh policy update pending."*

### 10.7 User rules and approval

**Selectors** (v1 allows exactly one term per selector):

| Selector | Renders to | Allowed as |
|---|---|---|
| `label: key=value` | `tag:k-l-<label-uuid7>` | from, to |
| `cluster: <cluster-uuid7>` | `tag:k-c-<C>` | from, to |
| `cluster_role: <cluster-uuid7>/server` or `/agent` | `tag:k-c-<C>-server` / `-agent` | from, to |
| `tailnet_members` | `autogroup:member` | from only |

Rules must list protocol (`tcp`, `udp`, `icmp`) and ports or port ranges; "all ports" is not allowed. Multi-term selectors are rejected when the rule is saved.

*Example:* `from: tailnet_members → to: cluster_role <C>/agent, tcp:80,443` exposes ingress on workers to tailnet users.

**Lifecycle:** `draft → pending_approval → active → retired`

- Without the org policy "Require approval for mesh rule changes", saving a rule makes it `active` immediately.
- With it, a rule goes to `pending_approval` and needs a different principal with `mesh.policy:approve`.
- The pipeline renders **only `active` rules**, so pending rules never block system runs.
- **System changes are never held for approval.**

### 10.8 Plan

Each run stores a human-readable diff against the last applied revision:

```
+ grant  site=pune → tier=db  tcp:5432        (rule 0192f3d1-…, author ravi@…, approver meera@…)
+ test   accept site=pune → tier=db:5432
```

Available in the UI and through `klustro mesh policy plan`.

### 10.9 Apply

Executed in this order:

1. **Ensure service objects:** create Tailscale Services for new clusters (`EnsureService`). The policy refers to them, so they must exist first.
2. **Write the policy** with `If-Match: <ETag of last applied revision>`.
   - `412 Precondition Failed` → handled as drift (Section 10.11): the live content is recorded, then the run re-renders and retries with the new ETag (maximum 3 attempts per run).
3. **Delete service objects** of deleted clusters, after the policy no longer references them.

Each policy write is a single atomic operation. There are no partial applies.

### 10.10 Verify, record and barriers

- **Verify:** read the policy back, canonicalise (HuJSON → sorted JSON), compare the hash with the rendered hash.
- **Record:** store a **PolicyRevision**:
  ```
  id (uuid7), project_id, seq, run_id, triggers[], renderer_version, content, hash, etag,
  diff, test_results, active_rule_ids[], approver_ids[], applied_at
  ```
- **Barriers:** operations that depend on a policy wait until a revision containing what they need has been applied.

| Operation | Waits for |
|---|---|
| New cluster becomes `ready` for joins | A revision containing its tags, grants, service object and `autoApprovers` entry |
| Join token for a cluster can be created | Same as above. Otherwise: *"Cluster network is being prepared"* |
| Node assigned to a cluster or given a new label | A revision whose `tagOwners` contains the new tags; **then** the device is retagged |

### 10.11 Drift detection and reversion

- **Detect:** every 5 minutes, and on Tailscale policy-change webhooks if available (V8), compare the live canonical hash with the last applied hash.
- **On drift:** store a **DriftEvent** (live content, diff, detected_at), run the pipeline with trigger `drift` to **re-apply the current render**, emit audit event `mesh.policy.drift_reverted`, notify project admins: *"Someone edited the tailnet policy in Tailscale; Klustro reverted it. [View diff]"*
- **Loop guard:** more than 3 drift events in one hour → keep reverting, show an **"Isolation at risk"** banner, and email organisation Owners. This usually means an external automation is fighting Klustro.

### 10.12 Rollback

The user picks an earlier revision. Klustro takes that revision's **set of active user rules and renderer version**, re-renders against the **current topology** (so no deleted nodes or clusters are referenced), then tests, plans and applies it as a new revision (`#17 — rollback of #14`). Rules that existed at #14 but have since been retired are reactivated; rules created after #14 are retired. Requires `mesh.policy:rollback`.

Revision 0 (takeover backup) cannot be rolled back to while the project exists, because it lacks Klustro's system rules. It is restored only on project purge.

### 10.13 Renderer upgrades

New renderer versions roll out **canary (1% of projects) → 10% → 50% → 100%**, with automatic pause if any project's run fails validation.

### 10.14 Run states

`queued → rendering → testing → awaiting_approval (user-rule activation only) → applying → verifying → succeeded`

Terminal alternatives: `no_change`, `failed_validation`, `blocked_credentials`, `failed_apply` (retries exhausted).

### 10.15 Failure handling

| Failure | Behaviour |
|---|---|
| Tailscale 429 / 5xx | Exponential backoff up to 15 minutes. Live policy untouched |
| Credentials invalid | `blocked_credentials`; project shows Mesh management degraded |
| Test or validation failure | `failed_validation`; nothing applied; on-call paged |
| Worker crash | Lease expires; next worker re-renders from current state |
| Renderer defect at scale | Canary pause (Section 10.13) |

### 10.16 Visibility

- **UI — Project → Mesh Policy:** current revision, run history and states, diffs, test results, drift events, read-only rendered file, Re-sync and Rollback.
- **CLI:** `klustro mesh policy show | history | diff <rev> <rev> | plan | resync | rollback <rev>`
- **Audit:** every run, rule change, approval, drift event and rollback.

---

## 11. Node onboarding and the agent

### 11.1 Supported platforms (v1)

- Linux amd64 and arm64: Ubuntu, Debian, RHEL-family, Raspberry Pi OS.
- systemd required.

### 11.2 Join tokens and the script

`nodes.jointokens:create` with: project (implicit), optional cluster, role (required with a cluster), labels, `max_uses` (default 1), expiry (default 24 h, max 30 days).

The UI shows a one-line command containing the token secret (shown once):

```
curl -fsSL https://get.klustro.io/install.sh | sudo sh -s -- --token kjt_<secret>
```

The token secret is a Klustro secret only. It contains no Tailscale key and no k3s token.

### 11.3 Enrolment

1. The script installs `klustro-agent` and starts it.
2. **Preflight** (Section 11.4). Failures are reported to the UI and stop enrolment.
3. The agent generates a key pair and calls the Agent Gateway with the token and a CSR.
4. Klustro checks the token (valid, not expired, not revoked, uses remaining) and its own billing state (Section 14), creates the **Node** record, and issues the **node identity certificate**.
5. Klustro asks the Tailscale driver for a **single-use, pre-authorised, non-ephemeral, tagged auth key** with a 10-minute expiry. Tags are determined by the token (`tag:k-unassigned` or the cluster role tags, plus label tags). The key is sent to the agent over mTLS.
6. The agent's node driver starts a **dedicated `tailscaled` instance** and joins:
   ```
   tailscaled --tun=klustro0 --statedir=/var/lib/klustro/tailscale \
              --socket=/run/klustro/tailscaled.sock --port=41642
   tailscale --socket=/run/klustro/tailscaled.sock up \
              --authkey=<single-use key> --hostname=n-<node-uuid7> --advertise-tags=<tags>
   ```
   A separate instance avoids conflicts with any Tailscale client the machine already runs (V3).
7. The agent reports `Status()`: mesh IPv4/IPv6, interface `klustro0`, MTU. A **MeshPeer** is recorded.
8. If the token names a cluster, cluster join begins (Section 12.3). Otherwise the node is **unassigned**.
9. Metering starts; audit events are written for every step.

### 11.4 Preflight checks

| Check | Example message |
|---|---|
| OS/arch supported | "Alpine Linux is not supported in v1" |
| Kernel modules (`br_netfilter`, `overlay`, `vxlan`) | "`br_netfilter` missing. Run `modprobe br_netfilter`" |
| `/dev/net/tun` present | — |
| Ports 6443, 10250, 2379-2380 free | "Port 6443 in use by process `kube-apiserver`" |
| Existing k3s/rke2/kubeadm install | "An existing k3s installation was found" |
| Clock skew < 5 s | "Clock is 42 s off. Enable NTP" |
| Outbound HTTPS to Klustro and Tailscale | — |
| Minimum resources (server: 2 vCPU, 2 GB RAM; agent: 1 vCPU, 1 GB RAM) | Warning only |

### 11.5 Agent responsibilities

| Module | Responsibility |
|---|---|
| Identity | Node certificate, rotation |
| Control stream | Long-lived outbound connection to the Agent Gateway over the internet (not the tailnet); receives tasks, sends heartbeats every 30 s |
| Mesh (node driver) | Dedicated `tailscaled`, join, status, peer RTT and transport reporting, service advertisement |
| k3s manager | Install, configure, upgrade, uninstall k3s; snapshots |
| Health | Local k3s `readyz`, etcd member health, node facts |
| RBAC reconciler | On server nodes: maintains Klustro ClusterRoleBindings/RoleBindings (Section 12.6). One active reconciler per cluster through a Kubernetes Lease |
| JWKS uploader | Uploads the cluster's service-account issuer keys for workload identity |
| Self-update | Staged rollout; can be pinned per project |

### 11.6 Node states

`enrolling → unassigned → installing → ready ⇄ offline → draining → removing → removed`

`offline` is set after 3 missed heartbeats (90 s). `failed` can follow `installing` with logs and retry.

### 11.7 Labels and taints

Edited with `nodes:update`. Labels are synced to k8s node labels and to Tailscale tags (through a policy run, then retag). Taints are synced to k8s.

---

## 12. Cluster lifecycle

### 12.1 States

`network_provisioning → waiting_for_servers → bootstrapping → ready ⇄ degraded → upgrading → deleting`

- `network_provisioning`: waiting for the policy revision containing the cluster (Section 10.10 barriers).
- `degraded`: etcd unhealthy, quorum at risk, or a server's API not ready.

### 12.2 Creation options

| Option | Default |
|---|---|
| k3s version | Latest from Klustro's supported list (latest 3 minor versions) |
| Dedicated servers (taint servers `CriticalAddonsOnly=true:NoExecute`) | Off |
| Disable bundled components (Traefik, ServiceLB, local-path, metrics-server) | None disabled |
| Cluster CIDR / Service CIDR | `10.42.0.0/16` / `10.43.0.0/16`; must not overlap the provider's mesh range (`100.64.0.0/10` for Tailscale) |
| Snapshot schedule / retention | Every 12 h / 5 snapshots |

### 12.3 Embedded etcd bootstrap and joining

- The **first server** assigned becomes the cluster's `bootstrap_node_id` and starts with `cluster-init: true`. This applies to single-server clusters too, so growing to 3 servers never needs a datastore migration.
- **Other servers and workers** wait until the bootstrap server is ready and the Tailscale Service is advertised, then join through it.
- The **k3s cluster token** is generated by Klustro, stored encrypted, and delivered only to enrolled agents over mTLS.

Agent-written `/etc/rancher/k3s/config.yaml` (server example):

```yaml
cluster-init: true                        # bootstrap server only
# server: https://k-<cluster-uuid7>.<tailnet>.ts.net:6443   # all other nodes
token: <k3s cluster token>
node-name: n-<node-uuid7>
node-ip: <mesh IPv4>
flannel-iface: klustro0
tls-san:
  - k-<cluster-uuid7>.<tailnet>.ts.net
  - <service virtual IP>
node-label:
  - klustro.io/node-id=<node-uuid7>
  - klustro.io/cluster-id=<cluster-uuid7>
kube-apiserver-arg:
  - oidc-issuer-url=https://id.klustro.io
  - oidc-client-id=k-<cluster-uuid7>
  - oidc-username-claim=sub
  - oidc-username-prefix=klustro:
  - oidc-groups-claim=groups
etcd-arg:
  - heartbeat-interval=<ms>
  - election-timeout=<ms>
etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 5
```

Flannel derives its MTU from `klustro0` (V9 confirms).

### 12.4 Server count and quorum

| Servers | Failures tolerated | Klustro behaviour |
|---|---|---|
| 1 | 0 | Allowed; shown as "No HA" |
| 2 | 0 | Allowed as a transition; warning to add a third |
| 3 | 1 | Recommended |
| 4 | 1 | Allowed as a transition; warning |
| 5 | 2 | Allowed |
| > 5 | — | Rejected |

**Quorum protection:** removing, draining or upgrading a server is refused if it would leave fewer healthy etcd members than quorum (`⌊n/2⌋ + 1`).

*Example:* 3 servers, server-3 offline. Removing server-2 is refused: *"This would leave 1 of 3 members; etcd would lose quorum."*

**etcd over the tailnet:**
- Agents report RTT and direct/relayed transport to every peer server.
- A relayed path between two servers produces a cluster warning.
- RTT above 50 ms between servers produces a warning.
- `heartbeat-interval` and `election-timeout` are set when servers join, from measured RTT: defaults 100 ms / 1000 ms; when the 95th-percentile RTT exceeds 50 ms, heartbeat = 2 × p95 RTT and election timeout = 10 × heartbeat (V10 validates these values).

### 12.5 HA API access through a Tailscale Service

```
svc:k-<cluster-uuid7>
  hosts: every server of the cluster (tag:k-c-<cluster-uuid7>-server; auto-approved)
  port:  tcp:6443, TCP passthrough (TLS terminated by k3s)
  name:  k-<cluster-uuid7>.<tailnet>.ts.net  + a stable virtual IP
```

- Created by the policy pipeline with the cluster; deleted with it.
- Each server's agent advertises the service:
  `tailscale --socket=/run/klustro/tailscaled.sock serve --service=svc:k-<cluster-uuid7> --tcp=6443 tcp://127.0.0.1:6443`
- **Health-aware:** the agent probes `https://127.0.0.1:6443/readyz` every 5 s, withdraws after 3 consecutive failures, and re-advertises after 3 consecutive successes. The agent also withdraws before upgrading or draining its server.
- Single-server clusters get the service too, so the kubeconfig never changes when servers are added.
- Joining nodes register through the service. After registering, k3s agents load-balance across servers themselves.
- **The cluster API is reachable only over the tailnet.** Users and CI runners must be members of (or tagged devices in) the project's tailnet. Klustro does not invite users to the tailnet; the customer manages tailnet membership in Tailscale.

### 12.6 Cluster authentication and authorization

Two layers, kept separate:

| Layer | Controlled by | Rule |
|---|---|---|
| Network: can a device reach `svc:k-<C>:6443`? | Tailnet policy (system rules) | Tailnet members and cluster nodes |
| Authorization: what can the caller do? | Klustro OIDC token + k8s RBAC | Without a valid token: `401`, even inside the tailnet |

*Example:* Meera is in the tailnet with **Reader** in Klustro: she reaches the API, but `kubectl delete` is forbidden. Ravi has **Developer** but is not in the tailnet: `kubectl` cannot connect, and the UI says *"Join the project's tailnet to access this cluster."*

**Kubeconfig** (`klustro kubeconfig <cluster>`):

```yaml
apiVersion: v1
kind: Config
clusters:
- name: prod-eu                               # context alias = display name (local only)
  cluster:
    server: https://k-<cluster-uuid7>.<tailnet>.ts.net:6443
    certificate-authority-data: <k3s server CA>
users:
- name: klustro-prod-eu
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1
      command: klustro
      args: [kubeconfig, token, --cluster, <cluster-uuid7>]
contexts:
- name: prod-eu
  context: { cluster: prod-eu, user: klustro-prod-eu }
current-context: prod-eu
```

**Token** (issued by `https://id.klustro.io`, lifetime 1 hour):

```json
{
  "iss": "https://id.klustro.io",
  "sub": "<principal-uuid7>",
  "aud": "k-<cluster-uuid7>",
  "groups": ["klustro:edit"],
  "exp": 1792000000
}
```

- Issued only if the principal has a `clusters.kubeconfig:*` action on the cluster, the org policies pass (MFA, SSO, IP allowlist), and the account is not Suspended.
- The **RBAC reconciler** keeps these bindings in each cluster: `klustro:admin` → `cluster-admin`, `klustro:edit` → `edit`, `klustro:view` → `view`, and `klustro:edit:ns:<ns>` → `edit` RoleBinding in `<ns>` (created when the namespace exists).
- The API server fetches Klustro's signing keys through OIDC discovery and caches them. Server nodes need outbound HTTPS to `id.klustro.io`.
- **Break-glass:** k3s writes a local admin kubeconfig at `/etc/rancher/k3s/k3s.yaml` on every server. Anyone with root on a server can always administer the cluster, independent of Klustro.

### 12.7 Adding and removing nodes

- **Assign** (`nodes:assign`): pick an unassigned node and a role. Policy run with new tags → retag device → install k3s → join.
- **Drain** (`nodes:drain`): cordon, evict pods respecting PodDisruptionBudgets, report progress.
- **Remove a worker:** drain → stop k3s → delete k8s node → uninstall k3s (if chosen) → node becomes unassigned or is removed.
- **Remove a server:** quorum check → withdraw service advertisement → drain → **automatic etcd snapshot** → stop k3s → delete k8s node (k3s removes the etcd member) → **verify the member is gone** → uninstall k3s (if chosen).

### 12.8 Upgrades

1. **Automatic etcd snapshot** before starting.
2. Servers **one at a time**: check all etcd members healthy and leader stable → withdraw service advertisement → upgrade → wait `readyz` and etcd healthy → re-advertise.
3. Workers in batches (default: 1 node or 20%, whichever is larger; configurable).
4. **Pause on failure:** the failing node is marked; the user sees agent logs and can retry, skip (workers only) or abort. Already upgraded nodes are not downgraded automatically.

### 12.9 Snapshots

| Feature | Phase |
|---|---|
| Scheduled local snapshots (default every 12 h, keep 5) | v1 |
| Automatic snapshot before every upgrade and every server removal | v1 |
| On-demand snapshot (`clusters.snapshots:create`) | v1 |
| Snapshot listing (`clusters.snapshots:read`) | v1 |
| Snapshots to customer-provided S3-compatible storage | v1.x |
| Guided restore and quorum-loss recovery (`clusters.snapshots:restore`) | v1.x; v1 ships a documented runbook |

---

## 13. Billing and plans

### 13.1 Model

- **Plan tiers** set features, limits and included usage.
- **Metered usage** beyond the included amount is charged per node-hour.
- **Prepaid and postpaid share one meter and one ledger.** Prepaid usage draws down a wallet; postpaid usage builds up into an invoice.

### 13.2 Meters

| Meter | Unit | Rule |
|---|---|---|
| `server_node_hours` | node-hour | Server-role node connected in the hour |
| `agent_node_hours` | node-hour | Worker or unassigned node connected in the hour |

- A node is **connected** in an hour if Klustro received at least one heartbeat during that hour. A node offline for the full hour is not billed.
- If a node held the server role at any point in the hour, the hour is a server hour.
- **Mesh and relay bandwidth are not metered, billed or limited on any plan** (D15).

**Usage event:**

```
id (uuid7), idempotency_key = node_id + hour_start, billing_account_id, org_id,
project_id, cluster_id?, node_id, meter, hour_start, recorded_at
```

- Events are append-only and idempotent: a duplicate never produces a second charge.
- Late events are accepted for 72 hours. Events arriving after a postpaid invoice is finalised are billed on the next invoice.

### 13.3 Plans and entitlements

Prices are defined per currency in the versioned **Price List** (Section 13.9), not in code.

| | Free | Pro | Business | Enterprise |
|---|---|---|---|---|
| Base price per month | 0 | Price List | Price List | Contract |
| Included node-hours per month (pooled) | — | 7,300 (~10 nodes) | 36,500 (~50 nodes) | Contract |
| Connected nodes | **Hard cap: 3** | Unlimited (overage billed) | Unlimited (overage billed) | Contract |
| Overage | Not applicable | Per node-hour, server and agent rates | Per node-hour, server and agent rates | Contract |
| Projects | 1 | 5 | Unlimited | Unlimited |
| Clusters | 1 | 10 | Unlimited | Unlimited |
| Users, groups, service principals, built-in roles, MFA | ✅ | ✅ | ✅ | ✅ |
| Workload identity | — | ✅ | ✅ | ✅ |
| Custom roles, JIT access | — | — | ✅ | ✅ |
| SSO, SCIM, org policies | — | — | ✅ | ✅ |
| Mesh rule approvals (four-eyes) | — | — | ✅ | ✅ |
| Audit log retention | 7 days | 30 days | 1 year | Contract |
| Payment mode | Prepaid | Prepaid, or postpaid if eligible | Prepaid, or postpaid if eligible | Postpaid (contract) |

- The Free plan never incurs charges. A fourth node on Free is refused at enrolment with *"Free plan allows 3 nodes. Upgrade to Pro."*
- Included node-hours are pooled across both meters and consumed in chronological order. Overage is rated per meter.
- The **Billing Service** answers `entitlements(org) → { features[], limits{} }`. Other services ask it and never read plan tables directly.

*Example:* a Pro customer runs 14 nodes (2 servers, 12 workers) for a full 730-hour month = 10,220 node-hours. 7,300 are included; the remaining 2,920 are charged at overage rates, split by meter in the order the hours occurred.

### 13.4 Currencies and payment providers

| Currency | Payment provider | Methods | Tax |
|---|---|---|---|
| **INR** | Razorpay | UPI, cards, netbanking | GST: CGST+SGST (same state) or IGST (different state); GSTIN on invoice when provided |
| **USD** | Stripe | Cards, bank transfer (postpaid contracts) | Per billing country through the tax engine (e.g. EU VAT reverse charge for businesses with a VAT ID) |

- Currency is chosen when the billing account is created and **cannot be changed**. It defaults from the billing country (India → INR, otherwise USD) and can be overridden.
- Prices are set separately per currency. There is no live currency conversion.
- Wallets, ledgers, invoices and credit limits are all in the account currency.
- Payment providers sit behind a **Payments Adapter** interface: `CreateCheckout`, `ChargeSavedMethod`, `Refund`, `HandleWebhook`.

### 13.5 Prepaid (default for every account)

1. **Top-up** (`billing.wallet:topup`): minimum amount configurable per currency (default $10 / ₹800). A tax invoice/receipt is issued immediately.
2. **Credit grants:** the wallet is a set of grants, each with `amount`, `source` (`purchase`, `promo`, `goodwill`), optional `expires_at`. Consumption order: earliest `expires_at` first (grants without expiry last); ties broken by `promo` before `goodwill` before `purchase`; then oldest first.
3. **Hourly drawdown:** rated overage is debited every hour. The plan's base price is debited on the renewal date.
4. **Balance alerts:** when the balance falls below 25%, 10% and 0% of the most recent top-up amount. Email, in-app, and webhook (v1.x).
5. **Zero balance:** account lifecycle (Section 14). The balance may go negative during Grace down to a configurable cap (default: 20% of the most recent top-up). Beyond the cap, usage keeps being recorded and is settled at the next top-up.
6. Promotional and goodwill grants are non-refundable. Refunds of purchased credit are handled by Klustro staff through the admin console.

*Example:* Neha's wallet reaches $0 on 1 March. Her clusters keep running (Grace). On 8 March her management plane becomes read-only (Frozen). On 12 March she tops up $40: $6 of usage from the grace period is settled from it and everything unlocks immediately.

### 13.6 Postpaid

**Eligibility:**
- **Automatic offer:** at least 3 consecutive months on a paid plan in prepaid mode, total spend at or above a configurable threshold, a verified saved payment method, no chargebacks or failed payments. The UI offers "Switch to monthly invoicing" (`billing.mode:change`).
- **Manual:** Klustro staff approve a contract or purchase order with payment terms (net 15/30/45).

**Operation:**
1. Usage builds up on a draft invoice for the calendar month.
2. **Credit limit:** initially 2 × average monthly spend over the last 3 months (configurable floor). Recomputed monthly; it can only rise, and only after 6 consecutive months without failed payments.
3. **Spend alerts** at 80% and 100% of the credit limit. Above 100%, the account enters lifecycle Grace (Section 14).
4. On the 1st, the invoice is finalised with tax and charged to the saved method (card customers) or sent with payment terms (contract customers).
5. **Collections:** failed card charges retry on days 1, 3, 5 and 7. Unpaid 7 days after the due date → Frozen; the lifecycle continues from there.

*Example:* Acme is postpaid with a $2,000 limit. In June it uses $1,600 and receives an 80% alert. The invoice on 1 July is charged automatically. After 6 clean months its limit rises to $3,000.

**Mode switching:**
- Prepaid → postpaid: the remaining wallet balance is consumed first; only then does usage accrue to the invoice.
- Postpaid → prepaid: the final invoice must be paid first.

### 13.7 Plan changes

- **Upgrade:** immediate. Prorated base price for the rest of the period; new limits apply immediately. Prepaid accounts need a wallet balance covering the prorated amount.
- **Downgrade:** takes effect at the next renewal. Blocked if current usage exceeds the target plan's limits, with a list of what to remove (e.g. *"8 clusters; Pro allows 10, Free allows 1"*). Features not in the target plan (e.g. SSO) are disabled at renewal after a warning 7 days before.

### 13.8 Ledger

- **Double-entry and append-only.** Accounts include: customer wallet (liability), accounts receivable (postpaid), revenue, promotional credit, tax payable.
- Every top-up, debit, credit grant, refund, invoice line and adjustment is a ledger entry. **Nothing is updated or deleted.** Corrections are reversing entries.
- The wallet balance is **always computed** from ledger entries (with a cached materialised value that is re-verified on every write).
- **Rating** converts usage events into charges using the Price List version effective at the event's `hour_start`.

### 13.9 Price List

Versioned, with effective-from dates, one set of prices per currency:

```
PriceList { id, version, effective_from,
  currencies: { USD: { pro_base, business_base, server_node_hour, agent_node_hour, min_topup }, 
                INR: { ... } } }
```

---

## 14. Account lifecycle and enforcement

### 14.1 States

Managed only by the **Lifecycle Enforcer**. Every other component only reads the state.

| State | Entered when | Effect |
|---|---|---|
| **Active** | Balance > 0 (prepaid) or within credit limit and no overdue invoice (postpaid) | Everything works |
| **Grace** | Prepaid balance ≤ 0, or postpaid above credit limit. Days 0–7 (configurable) | Everything works. Banner and daily emails |
| **Frozen** | End of Grace, or postpaid invoice unpaid 7 days after due date. Until day 30 | **Management plane read-only:** no new projects, clusters, nodes or join tokens; no upgrades or policy rule changes. Kubeconfig tokens are still issued. **Policy pipeline system runs and drift reversion keep running** so isolation is preserved |
| **Suspended** | Day 30 → day 60 | As Frozen, plus **Klustro stops issuing kubeconfig tokens** (`kubectl` through Klustro stops within 1 hour) and agents stop reconciling (no k3s changes). The tailnet, k3s and workloads keep running. Break-glass local kubeconfig still works |
| **Deleted** | Day 60, after final notices at day 45 and day 55 | Klustro instructs agents to uninstall themselves (k3s and workloads are left running), deletes stored mesh credentials, and deletes org data except billing and tax records kept for legal retention. Klustro does not modify the tailnet policy or devices |

- **Top-up or payment at any stage before Deleted** returns the account to **Active** immediately, after settling the outstanding amount.
- **Klustro never stops customer workloads, removes tailnet devices or rewrites the tailnet policy to destroy access for billing reasons.**
- Lifecycle timers and thresholds are configurable. Payment provider outages pause transitions into Frozen.

### 14.2 Notifications

| When | Channel |
|---|---|
| Balance thresholds, credit-limit thresholds | Email, in-app |
| Entering each state | Email to Owners and Billing Admins, in-app banner |
| Daily during Grace and Frozen | Email |
| Day 45, day 55 | Final notice email |

---

## 15. System architecture

### 15.1 Components

```
                 ┌──────────────────────── Klustro Cloud (management plane) ─────────────────────────┐
 Web UI / CLI ──►│ API Gateway  (authn, rate limits, org policies, request IDs, idempotency keys)      │
 API clients     │   ├─ Identity Service      users, groups, SPs, workload identities, SSO, SCIM,      │
                 │   │                        OIDC issuer (id.klustro.io), token exchange              │
                 │   ├─ Authz Service         role definitions, assignments, JIT, check()              │
                 │   ├─ Resource Service      orgs, projects, clusters, nodes, labels, join tokens     │
                 │   ├─ Mesh Orchestrator     provider registry, controller drivers, Policy Pipeline,  │
                 │   │                        drift detector, connection health                         │
                 │   ├─ Cluster Orchestrator  desired cluster state → agent tasks                       │
                 │   ├─ Agent Gateway         mTLS endpoint, enrolment, control streams, heartbeats    │
                 │   ├─ Metering Service      heartbeats → usage events                                 │
                 │   ├─ Billing Service       rating, ledger, wallets, invoices, entitlements, tax     │
                 │   ├─ Payments Adapter      Stripe (USD), Razorpay (INR)                              │
                 │   ├─ Lifecycle Enforcer    billing lifecycle state machine                           │
                 │   ├─ Audit Service         append-only audit log                                     │
                 │   ├─ Notification Service  email, in-app (webhooks v1.x)                             │
                 │   ├─ Secrets Service       envelope encryption with KMS                              │
                 │   └─ Admin Console         internal staff tools                                      │
                 └──────────▲──────────────────────────────────────────▲──────────────────────────────┘
                            │ mTLS over internet (node identity)        │ Tailscale API (OAuth)
              ┌─────────────┴──────────────┐                 ┌──────────┴──────────┐
              │ Customer node               │                 │ Tailscale           │
              │  klustro-agent              │◄── tailnet ────►│ control plane, DERP │
              │  tailscaled (klustro0)      │                 └─────────────────────┘
              │  k3s server | agent         │
              └─────────────▲──────────────┘
                            │ kubectl over the tailnet → svc:k-<cluster-uuid7>:6443
                 User / CI device in the tailnet (OIDC token from id.klustro.io)
```

### 15.2 Boundaries

| Component | Owns | Never does |
|---|---|---|
| Authz Service | All permission decisions | Store resources |
| Billing Service | Plans, entitlements, ledger | Change lifecycle state |
| Lifecycle Enforcer | Lifecycle state transitions | Rate usage |
| Mesh Orchestrator | All writes to mesh providers | Decide permissions |
| Policy Pipeline | Tailnet policy file and services | Act outside a project's lane |
| Cluster Orchestrator | Cluster desired state, agent tasks | Talk to Tailscale |
| Agent Gateway | Agent connections and identity | Make authorization decisions for users |

### 15.3 Data stores

- **Primary relational database:** all resources, assignments, ledger, policy revisions. UUIDv7 primary keys.
- **Event log:** usage events and audit events (append-only).
- **Object storage:** rendered policy contents, invoices (PDF), agent logs.
- **KMS:** key-encryption keys for per-org data keys.

---

## 16. Key flows

### 16.1 Signup to first cluster

1. Sign up → personal org + Free prepaid billing account.
2. "Create your first project" → Tailscale wizard → OAuth credentials → validation → takeover → revision 1.
3. "Create cluster" → `network_provisioning` → policy revision with cluster tags, grants and service → `waiting_for_servers`.
4. "Add nodes" → join token (cluster, role `server`, max uses 1) → run script on machine A → bootstrap server, `cluster-init` → service advertised → `ready`.
5. Join token (role `agent`, max uses 2) → machines B and C join as workers.
6. User joins their laptop to the tailnet → `klustro kubeconfig <cluster>` → `kubectl get nodes`.

### 16.2 Kubeconfig token

`kubectl` → exec `klustro kubeconfig token --cluster <uuid7>` → Identity Service authenticates the user (session or PAT) → org policies checked → Authz picks the highest of admin/edit/view (including namespaces) → lifecycle not Suspended → 1-hour token → `kubectl` calls `svc:k-<uuid7>:6443` over the tailnet → k3s validates token → RBAC bindings apply.

### 16.3 Prepaid hour

Heartbeats → Metering creates idempotent usage events at hour end → Billing rates them (included hours first) → ledger debit → alert thresholds checked → if balance ≤ 0, Lifecycle Enforcer moves the account to Grace.

### 16.4 Postpaid month

Usage accrues to the draft invoice → credit-limit alerts → on the 1st, finalise with tax → charge or send → collections retries on failure → Frozen 7 days after due date if unpaid.

### 16.5 Label change

`nodes:update` adds `site=pune` → Label record created (if new) → policy run with `label` trigger → revision includes `tag:k-l-<uuid7>` in `tagOwners` → device retagged → k8s node label synced.

---

## 17. Error handling and resilience

| Situation | Behaviour |
|---|---|
| Klustro cloud unreachable | Tailnet, k3s and workloads keep running. Existing tokens work until expiry. Break-glass local kubeconfig always works. Management actions wait |
| Tailscale API unreachable | Policy runs, joins and retags queue with backoff. Existing mesh unaffected |
| Tailscale control plane outage | Existing peer connections continue per Tailscale's behaviour. Klustro shows provider status |
| Node offline | `offline` after 90 s; not billed for full offline hours; server offline affects quorum warnings |
| Duplicate or late heartbeats | Idempotent events; late events accepted for 72 h |
| k3s upgrade failure | Upgrade paused; logs shown; retry, skip (workers) or abort |
| Join token leaked | Revoke instantly; nodes already enrolled remain; UI lists nodes enrolled with that token |
| OAuth credentials revoked | Mesh management degraded; workloads unaffected |
| Policy validation failure | Nothing applied; on-call paged |
| Payment provider outage | Top-ups retried; lifecycle transitions into Frozen paused |
| Identity provider outage | SSO users can't sign in; existing tokens valid until expiry; break-glass Owner passkey login |
| etcd quorum loss | Cluster `degraded`; documented recovery runbook (v1), guided recovery (v1.x) |
| Lost Owner access | Support-assisted recovery with identity verification, audited |

---

## 18. Security

### 18.1 Principles
- Least privilege for every principal, credential and OAuth scope.
- Short-lived credentials: 1-hour kubeconfig tokens, 10-minute single-use Tailscale keys, 30-day node certificates.
- Klustro holds no long-lived credential that grants access to customer workloads.

### 18.2 Secrets
- Envelope encryption: per-org data keys encrypted by KMS key-encryption keys.
- Encrypted: Tailscale OAuth secrets, k3s cluster tokens, SCIM tokens, service principal secret hashes, payment provider references.
- Secrets never appear in logs, audit payloads or API responses. Join token and PAT secrets are stored only as hashes.

### 18.3 Agent channel
- mTLS with node identity certificates; certificate revoked on node removal.
- Agents connect outbound only; no inbound ports opened by the agent.
- Agent tasks are signed by the Cluster Orchestrator and validated by the agent.

### 18.4 Tenant isolation
- Organisation ID on every row; enforced in queries and in the Authz Service.
- One tailnet per project; one tailnet can't be bound twice.
- Workload identity JWKS scoped to its cluster.

---

## 19. Audit, observability and notifications

### 19.1 Audit events

```
id (uuid7), time, org_id, actor_principal_id, actor_type, via (ui | api | cli | scim | system),
action, scope, target_id, result (allowed | denied | failed), reason?, ip, user_agent,
request_id, metadata (no secrets)
```

- Every API action, authorization denial, JIT activation, policy run, drift event, lifecycle transition and staff action is audited.
- Append-only. Retention by plan (Section 13.3). CSV export.

### 19.2 Health

- Node: online/offline, facts, mesh transport and RTT per peer.
- Cluster: k3s component health, etcd member health, quorum status, service advertisement status per server.
- Project: mesh connection status, last policy run, drift status.

### 19.3 Notifications

v1: email and in-app. v1.x: webhooks and event stream. Later: Slack/Teams.

---

## 20. Public API and CLI

### 20.1 REST API

- Base path `/v1`. Every resource addressed by UUIDv7: `/v1/orgs/{org-uuid7}/projects/{project-uuid7}/clusters/{cluster-uuid7}`.
- Cursor pagination; `Idempotency-Key` header required for creating requests; `ETag` / `If-Match` for updates.
- Errors: `{ "error": { "code": "feature_not_in_plan", "message": "...", "request_id": "..." } }`
- Authentication: user sessions, personal access tokens, service principal tokens, workload identity tokens.
- The web UI uses only the public API.

### 20.2 CLI

```
klustro login | logout | whoami
klustro context use --org <org> --project <project>
klustro projects list | create | delete
klustro clusters list | create | upgrade | delete
klustro nodes list | assign | drain | remove | label
klustro tokens create | list | revoke
klustro kubeconfig <cluster> [--merge]
klustro kubeconfig token --cluster <cluster-uuid7>
klustro mesh policy show | history | diff | plan | resync | rollback
klustro mesh rules list | create | approve | retire
klustro snapshots list | create --cluster <cluster>
klustro billing balance | usage | topup
```

---

## 21. Platform admin console

Internal tools for Klustro staff. Every action is audited with the staff member's identity.

| Feature | Detail |
|---|---|
| Customer lookup | By org, user, billing account, tailnet |
| Read-only impersonation | View the customer's UI as them; no write actions |
| Credit grants and refunds | Goodwill grants, refunds of purchased credit, as ledger entries |
| Postpaid approval | Contracts, payment terms, credit limits |
| Price List management | New versions with effective-from dates |
| Lifecycle overrides | Extend Grace, pause enforcement, with reason |
| Renderer and agent rollouts | Canary control, pause, rollback |
| Abuse controls | Suspend accounts for terms-of-service violations |

---

## 22. Testing strategy

| Area | Tests |
|---|---|
| **Authorization** | Table-driven: every built-in role × every action × every scope level. Privilege escalation, last Owner, service principal limits, JIT expiry, plan entitlement denials. Property test: no assignment can grant more than its creator holds |
| **Identity** | SSO (OIDC, SAML), SCIM provisioning/deprovisioning, provider-managed group read-only, federated OIDC credentials, workload identity token exchange |
| **Policy pipeline** | Golden-file tests (state → rendered policy); determinism; coalescing; barriers; ETag conflict handling; drift reversion and loop guard; rollback re-rendering; renderer canary |
| **Tailscale driver** | Driver conformance suite (Section 8.5) against a dedicated test tailnet |
| **Billing** | Rating golden files (USD/INR, included hours, proration, server/agent split); credit grant order; ledger invariants (balance = sum of entries); postpaid credit limit; clock-driven simulation of Active → Deleted |
| **Agent and mesh** | Lab of VMs and Raspberry Pis behind different NAT types; dedicated `tailscaled` coexisting with a personal one; direct vs relayed detection |
| **Clusters** | End-to-end: single server → 3 servers; quorum protection; losing 1 of 3 servers under load; rolling upgrade with pause; server removal with member verification; service failover timing; etcd under induced latency and relay |
| **Resilience** | Cut Klustro cloud for 24 h: workloads, tailnet and break-glass access keep working |

---

## 23. Verification items (proof-of-concept)

These must be confirmed before or at the start of implementation. If a result differs from what this spec assumes, the affected section is revised before building on it.

| ID | Verify | Affects |
|---|---|---|
| V1 | Exact Tailscale OAuth scope names for policy file, auth keys, devices, services, DNS | 9.2 |
| V2 | Tailscale Services: API for creating services, `autoApprovers.services`, TCP passthrough on 6443, advertise/withdraw behaviour, failover time, availability by plan | 9.2, 12.5 |
| V3 | Two `tailscaled` instances side by side (custom socket, state dir, port, TUN name) on Ubuntu, Debian, RHEL and Raspberry Pi OS | 11.3 |
| V4 | Tagged auth keys whose tags are owned by `tag:klustro` can be minted by an OAuth client holding `tag:klustro` | 9.2, 9.4 |
| V5 | Tailscale tag and service name character and length limits for `k-c-<uuid7>-server` | 9.4 |
| V6 | Tailscale setting that prevents policy edits in the admin console, and its plan availability | 9.2 |
| V7 | Node-key expiry is disabled for tagged devices | 9.5 |
| V8 | Tailscale webhooks for policy-file changes | 10.11 |
| V9 | Flannel VXLAN MTU derived correctly from `klustro0` (MTU 1280) | 12.3 |
| V10 | etcd heartbeat/election tuning values under 50–200 ms RTT and relayed paths | 12.4 |
| V11 | k3s API server behaviour when the OIDC issuer is unreachable at startup | 12.6, 17 |
| V12 | Tailscale policy `tests` block evaluated on save and rejection behaviour | 10.6 |

---

## 24. Glossary

| Term | Meaning |
|---|---|
| **Agent** | `klustro-agent`, the Klustro service running on each customer node. (Kubernetes worker nodes are called *workers* or *agent role* nodes; the k3s term `agent` is used in tag names) |
| **Barrier** | A wait for a policy revision that contains what an operation needs |
| **Billing Account** | Entity holding plan, payment mode, currency, wallet or credit limit |
| **Break-glass** | Emergency access path independent of normal controls |
| **Drift** | Difference between the live tailnet policy and Klustro's last applied revision |
| **Join token** | Klustro credential allowing machines to enrol into a project |
| **Mesh connection** | The project's credentials for its mesh provider account |
| **Node-hour** | One node connected for (part of) one clock hour |
| **Policy pipeline** | The process that renders, tests, applies and records the tailnet policy |
| **Principal** | Anything that can act: user, group, service principal, node identity, workload identity |
| **Revision** | One applied version of a project's tailnet policy |
| **Scope** | A path in the hierarchy (`/orgs/…/projects/…/clusters/…`) where a role assignment applies |
| **Tailnet** | A Tailscale private network; one per Klustro project |
| **UUIDv7** | Time-ordered UUID used for every Klustro resource ID |

