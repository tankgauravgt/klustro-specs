# Feature 01 — Platform foundation

| | |
|---|---|
| Increment | 01 |
| Depends on | — |
| Unblocks | Everything |
| Spec refs | §4, §15.2, §15.3, §18.2, §19.1 |

## Goal

Establish the substrate that every later increment builds on: how resources are identified, how tenants are kept apart, how configuration and secrets are handled, and how state changes are recorded and announced. Nothing user-visible ships in this increment except the guarantees it provides.

## In scope

- Identifier, naming and timestamp conventions.
- Tenant isolation as a structural guarantee.
- Versioned platform configuration.
- Secret storage and masking.
- Audit event spine (emission only).
- Component boundary definitions.
- Event delivery backbone.
- Stable machine error codes.

## Out of scope

- Any user-facing resource (orgs, projects, clusters, nodes).
- Authentication and authorization decisions.
- Billing, plans and pricing.
- Any provider integration.
- Audit read/export surface (Feature 23).

## Requirements

**Identifiers and conventions**

R1. Every resource has an immutable `id` that is a UUIDv7 in lowercase canonical form.
R2. Every user-facing resource has a mutable `display_name` used only for UI and CLI output.
R3. Anything machine-facing uses only `id`: scope paths, API URLs, provider tags, service names, DNS names, hostnames, audit records and token claims.
R4. Renaming a resource must change nothing outside Klustro's own database — no policy run, no credential change, no external side effect.
R5. Fixed non-resource values keep readable names: `tag:klustro`, built-in role names, action names, plan names, lifecycle state names.
R6. Timestamps are UTC and RFC 3339. Money is integer minor units plus an ISO 4217 currency code.
R7. Every value the spec calls *configurable* lives in versioned platform configuration with effective-from dates. No configurable value is embedded in code.

**Tenancy**

R8. Every stored record carries its owning organisation id.
R9. Tenant scoping is applied as an invariant of data access, not as an optional filter a caller may forget. A query that is not scoped to exactly one organisation must fail closed.
R10. No resource, event, secret or audit record may cross organisations in any read path.
R11. Isolation must be verifiable by a test that attempts cross-tenant access through every service boundary.

**Secrets**

R12. Secrets are stored with envelope encryption: per-organisation data keys, themselves encrypted by a KMS-held key-encryption key.
R13. Secret material never appears in logs, audit payloads, error responses or API responses. Only masked representations are ever returned.
R14. Types covered from the start include provider OAuth credentials, cluster tokens, SCIM tokens, service principal secret hashes and payment provider references.
R15. Key material is never readable by any service that does not need it.

**Audit spine**

R16. A single audit event schema exists and is fixed: id, time, org_id, actor principal id, actor type, channel (`ui`/`api`/`cli`/`scim`/`system`), action, scope, target id, result (`allowed`/`denied`/`failed`), reason, source address, user agent, request id, metadata.
R17. Audit records are append-only. Nothing updates or deletes them.
R18. The spine must accept an event from every service, including system-initiated actions with no human actor.
R19. Metadata must be structurally incapable of carrying secret values.

**Change propagation**

R20. Every meaningful state change is published to a shared event backbone, transactionally coupled to its own write so that a change can never be visible without its event.
R21. Consumers are idempotent; at-least-once delivery is acceptable and duplicates must be harmless.
R22. Services communicate through their published contracts, not each other's storage.

**Boundaries**

R23. The following ownership boundaries are enforced and must not be violated by later increments:
- Authorization owns all permission decisions and stores no resources.
- Billing owns plans, entitlements and the ledger, and never changes lifecycle state.
- Lifecycle ownership changes lifecycle state and never rates usage.
- Mesh ownership performs all writes to mesh providers and never decides permissions.
- Policy pipeline owns the mesh policy file and services, and never acts outside one project's lane.
- Cluster orchestration owns cluster desired state and agent tasks, and never talks to a mesh provider.
- Agent gateway owns agent connections and agent identity, and never makes authorization decisions for users.

**Errors**

R24. A shared catalogue of machine-readable error codes exists with stable identifiers, and is the only source of error codes for API, CLI and UI.

## Acceptance criteria

- A cross-tenant access attempt is rejected and audited.
- No secret value can be produced by any read path, log query or audit query.
- A configuration value can be changed with an effective-from date without a code change or restart.
- An event is published for every state change and replaying it is harmless.
- Every error surfaced to a client carries a code from the shared catalogue.
