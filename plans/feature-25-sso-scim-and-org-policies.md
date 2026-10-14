# Feature 25 — SSO, SCIM and organisation policies

| | |
|---|---|
| Increment | 25 |
| Depends on | 01–04, 23, 24 |
| Unblocks | — |
| Spec refs | §6.7, §6.8, §6.9, §7.4 R11, §13.3 |

## Goal

Let an organisation take control of how its people sign in and how their access is granted and removed, without Klustro becoming an identity provider intermediary. This is the increment that makes Klustro acceptable to a company's IT and security function.

It is deliberately after groups and service principals (Feature 24), because provider-managed groups are the main consumer of both SSO group claims and SCIM.

## In scope

- SSO configuration per organisation.
- Domain verification.
- SCIM provisioning and deprovisioning.
- The organisation policy set and its enforcement points.

## Out of scope

- Klustro acting as an identity provider for customers (it issues cluster tokens only, not user logins).
- Just-in-time user provisioning from SSO claims.
- Multiple simultaneous SSO connections per organisation.
- Emailing administrators of customer IdPs.

## Requirements

**SSO**

R1. An organisation can configure SSO with an OIDC or SAML 2.0 provider and can change or remove it.
R2. The configuration is validated before it is saved, and a configuration that cannot complete a login is not saved.
R3. Domain verification by a DNS record is required before SSO can be enforced for a domain.
R4. An organisation can enforce SSO-only, which disables password login for its members.
R5. Enforcing SSO-only never removes the Owner break-glass path defined in Feature 03; the console states this explicitly so it is not mistaken for a lockout.
R6. When SSO is enforced and the identity provider is unreachable, existing sessions and tokens remain valid until expiry, and the failure is explained rather than appearing as a generic login error.
R7. Group claims from SSO can feed provider-managed groups where SCIM is not configured.
R8. SSO configuration and enforcement are gated by plan.

**SCIM**

R9. An organisation can enable a SCIM 2.0 endpoint and issue a bearer token for it.
R10. Enabling, rotating and disabling the SCIM token requires the SCIM configuration action, and the token is shown once and stored hashed.
R11. SCIM provisions and deprovisions organisation memberships and provider-managed groups.
R12. Deprovisioning removes the organisation membership and, with it, all of that member's role assignments in the organisation. The global user account remains and the user's other memberships are untouched.
R13. Deprovisioning must take effect on the next authorization check, so access ends without waiting for the next login.
R14. SCIM-driven changes are attributed to SCIM as the acting channel in the audit log, naming the token used.
R15. SCIM operations are idempotent, so a retried or replayed provisioning request does not duplicate a membership or a group.
R16. Invalid or unauthorised SCIM requests are rejected without partial application.
R17. Rotating the SCIM token invalidates the previous token immediately.

**Organisation policies**

R18. The following policies exist per organisation and are changeable by a principal holding the organisation update action:
  - require MFA;
  - require SSO;
  - IP allowlist;
  - maximum session and token lifetime;
  - disable personal access tokens;
  - require approval for mesh rule changes.
R19. Require MFA: members without a second factor are prompted to enrol at next login and blocked until they do, with the break-glass path excepted.
R20. Require SSO: password login is disabled for members, with break-glass excepted.
R21. IP allowlist: enforced on the console, the API and on token issuance. It explicitly does not apply to cluster API traffic, which Klustro never sees, and the console must say so rather than implying it does.
R22. Maximum session and token lifetime: caps web sessions, personal access tokens and service principal secret lifetimes. A policy can only tighten, never extend, a hard platform maximum.
R23. Disable personal access tokens: existing tokens are rejected for new requests; the tokens themselves are revoked rather than merely ignored.
R24. Require approval for mesh rule changes: activates the four-eyes flow defined in Feature 15.
R25. Policies apply to every principal in the organisation, including non-human principals, without exception.
R26. Policy changes take effect on the next authorization check and are audited with before and after values.
R27. Policy changes that would immediately lock out every administrator are refused with an explanation, consistent with the last-Owner invariant.
R28. SSO, SCIM and the organisation policy set are gated by plan; on lower plans the controls are shown as unavailable rather than hidden.

## Acceptance criteria

- An organisation connects its identity provider, verifies its domain and enforces SSO without locking out its Owner.
- A user removed from a group in the customer's directory loses their Klustro access on the next authorization check, and the removal is visible in the audit log as a SCIM action.
- Replaying a SCIM provisioning request does not create a second membership.
- Rotating the SCIM token immediately invalidates the old one.
- With an IP allowlist set, requests from outside it are refused on the console, the API and token issuance, while cluster traffic is explicitly unaffected.
- Setting a maximum token lifetime does not extend or silently ignore tokens already beyond it.
- Every policy change is attributable to a principal with the before and after values recorded.
