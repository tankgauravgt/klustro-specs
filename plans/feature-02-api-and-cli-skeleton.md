# Feature 02 — Public API and CLI skeleton

| | |
|---|---|
| Increment | 02 |
| Depends on | 01 |
| Unblocks | 03, 05, 07, 09, 13, 15, 28 |
| Spec refs | §20.1, §20.2 |

## Goal

Define and implement the single external contract — the REST API at `/v1` — and the CLI that consumes it. Every later increment adds resources to this contract rather than inventing its own surface. The web UI is built on this API and only this API.

## In scope

- API addressing, pagination, idempotency, concurrency control and error shape.
- Request identity, rate limiting and request correlation.
- CLI shell: authentication, context, output formats.
- Versioning policy for the API.

## Out of scope

- Any concrete resource endpoints beyond what identity (Feature 03) requires.
- SDK generation for other languages.
- Terraform provider and GitHub Action (later).
- Webhooks and event stream (later).

## Requirements

**Addressing and conventions**

R1. All endpoints live under a versioned base path (`/v1`).
R2. Resources are addressed by UUIDv7 only. Display names are never part of a URL.
R3. List endpoints use cursor pagination with a stable sort order and an opaque cursor.
R4. Creating requests require an idempotency key; a repeated key with an identical body returns the original result rather than creating a second resource.
R5. Update requests use entity tags with conditional writes. A stale tag is rejected rather than silently overwriting.

**Errors and correlation**

R6. Every response carries a request id, and the same id appears in audit records and logs for that request.
R7. Errors use a single envelope: a machine code from the shared catalogue, a human message, and the request id.
R8. Errors never leak internal structure, secret material or cross-tenant existence.

**Authentication surface**

R9. The API accepts user sessions, personal access tokens, service principal tokens and workload identity tokens. The API layer authenticates but does not make authorization decisions.
R10. Authentication failures and authorization denials are distinguishable by code.

**Rate limiting and safety**

R11. Rate limits exist and are applied per principal and per organisation.
R12. Organization-wide policies that gate requests (Feature 25) are evaluated at this layer before a request reaches a service.

**CLI**

R13. The CLI provides authentication, session persistence and organisation/project context switching.
R14. The CLI accepts either a display name or an id for a resource. A display name is resolved inside the current context and fails with a clear error if it matches more than one resource.
R15. The CLI exposes `whoami` so a user can see the effective identity and active context.
R16. The CLI supports machine-readable output for scripting.
R17. The CLI uses the public API exclusively — it has no private path.

## Acceptance criteria

- The web UI can be built against this API without any privileged endpoint.
- `Idempotency-Key` replaying an identical create returns one resource, not two.
- A stale conditional update is rejected with a distinguishable error.
- Every error a client sees carries a catalogue code and the request id.
- A CLI list call and the equivalent UI page return the same data.
