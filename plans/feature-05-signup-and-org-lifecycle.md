# Feature 05 — Signup and organisation lifecycle

| | |
|---|---|
| Increment | 05 |
| Depends on | 01, 02, 03, 04 |
| Unblocks | 06, 07 |
| Spec refs | §5.1, §5.2 (Organisation), §5.3, §16.1 |

## Goal

Take a brand-new person from landing page to a usable tenant in one self-service step, with no human on the Klustro side involved. This is success criterion 1's first half.

## In scope

- Self-service signup.
- Personal organisation creation.
- Billing account creation in its Free, prepaid default shape.
- Owner assignment.
- The "create your first project" entry point.
- Organisation settings, rename and soft delete.

## Out of scope

- Project creation itself (Feature 07).
- Plan upgrades and billing behaviour (Features 06, 18, 19).
- SSO, SCIM and organisation policies (Feature 25).
- Nested organisations and resellers (later).

## Requirements

**Signup**

R1. Signup requires no Klustro staff involvement and no sales contact.
R2. Signup creates, in one atomic outcome: the user, a personal organisation, a billing account, a membership, and an active Owner assignment for that user.
R3. An organisation is the tenant and the hard isolation boundary. It is created with a display name derived from the user, editable later.
R4. The billing account is created in prepaid mode on the Free plan, owned by that organisation, with the currency defaulted from the billing country (India → INR, otherwise USD).
R5. Currency is fixed at billing account creation and is immutable thereafter.
R6. No project is created automatically, because a project requires mesh provider credentials.
R7. Immediately after signup the user is taken to the "create your first project" flow.
R8. Signup must be idempotent against retries and must never produce two organisations for one signup.

**Organisation**

R9. An organisation has an id, a display name, a billing account reference, a reserved nullable parent organisation reference (always null at launch), policies, SSO configuration, SCIM configuration, creation time and deletion time.
R10. The reserved parent organisation field exists in the model from the start and is never populated in this phase.
R11. Organisation settings are readable with the read action and changeable with the update action.

**Deletion**

R12. Deleting an organisation requires that all its projects are already deleted and its billing account is settled.
R13. Organisation deletion is a soft delete with a restore window (default 7 days, configurable).
R14. During the restore window the organisation is restorable by an Owner.
R15. After the restore window the organisation is purged, retaining billing and tax records for their legal retention period.
R16. Deletion, restore and purge are audited.

**Membership interaction**

R17. The organisation's member list is visible to principals holding the read action.
R18. Inviting and removing members is controlled by distinct actions.
R19. The last-Owner invariant from Feature 04 is enforced against organisation membership operations here.

## Acceptance criteria

- A new user completes signup and lands in an organisation they own, on the Free plan, in prepaid mode, with the correct default currency, with no staff involvement.
- Replaying signup does not create a second organisation.
- An organisation with an active project cannot be deleted; the error names the blocking projects.
- A deleted organisation is restorable within the window and purged after it, with tax and billing records retained.
- The Owner who signed up sees their own membership and cannot remove their own last-Owner assignment.
