# Feature 03 — Identity, users and sessions

| | |
|---|---|
| Increment | 03 |
| Depends on | 01, 02 |
| Unblocks | 04, 05, 24, 25 |
| Spec refs | §5.2 (User, Membership), §6.1, §6.2, §6.7 |

## Goal

Give a human being a single global identity they can authenticate with, and a membership in an organisation that everything else is scoped to. This is the first principal type and the foundation of the authorization model.

## In scope

- Global user identity and credentials.
- Multi-factor authentication and passkeys.
- Web sessions and personal access tokens.
- Organisation membership and invitations.
- Organisation switching.
- Break-glass sign-in.

## Out of scope

- Groups and provider-managed identities (Feature 24).
- Service principals, workload identity and node identity (Features 10, 24).
- SSO and SCIM (Feature 25).
- Organization-level policy enforcement (Feature 25), except where noted.
- Role assignments (Feature 04).

## Requirements

**User identity**

R1. A user is a global identity, not owned by any organisation. The same user can belong to many organisations.
R2. Email is unique across the platform and must be verified before the account is usable for organisation actions.
R3. A user has a display name and one or more authentication methods.
R4. A user's principal id is a UUIDv7 and is the only identifier used by authorization.

**Authentication**

R5. Password authentication is supported.
R6. TOTP and passkeys (WebAuthn) are supported as second factors. Both are available on every plan.
R7. A user can register more than one passkey and more than one TOTP device.
R8. Recovery from a lost second factor must exist and must not be self-service bypass of MFA by a user who cannot prove identity.
R9. Authentication events (success, failure, factor enrolment, factor removal) are audited.

**Sessions**

R10. Sessions are time-limited and revocable. A user can see and revoke their active sessions.
R11. Session lifetime must be capped by organisation policy when that policy is lower than the platform default (Feature 25).

**Personal access tokens**

R12. A personal access token is scoped to exactly one organisation.
R13. Lifetime is bounded to a maximum of one year and is always set explicitly at creation.
R14. The secret is shown once at creation and stored only as a hash.
R15. Tokens are revocable, and revocation takes effect for new requests immediately.
R16. A token's lifetime must be capped by organisation policy, and personal access tokens as a class must be disableable by organisation policy.

**Membership**

R17. A membership links a user to an organisation. Authorization scope paths always start at an organisation.
R18. A user can be invited to an organisation by email by a principal holding the invitation action.
R19. Removing a membership removes all of that user's role assignments in that organisation.
R20. The global user account survives removal from an organisation.
R21. A user can switch between their organisations; the active organisation is context, not identity.
R22. Membership changes are audited.

**Break-glass**

R23. An Owner must always be able to sign in with a passkey, even when password login is disabled for the organisation or the external identity provider is unavailable.
R24. Break-glass sign-in is audited as such.

## Acceptance criteria

- A user can register, verify, enrol MFA and sign in.
- A user can belong to two organisations and switch between them without re-authenticating.
- Removing a user from an organisation immediately removes their access to that organisation's resources, while their account and other memberships survive.
- A revoked personal access token fails on its next use.
- Break-glass sign-in succeeds while SSO-only is enforced.
