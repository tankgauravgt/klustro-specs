# Feature 07 — Project and mesh connection

| | |
|---|---|
| Increment | 07 |
| Depends on | 01–06 |
| Unblocks | 08, 09, 14 |
| Spec refs | §5.2 (Project), §5.3, §8.1, §8.2, §8.4 (controller), §9.1–9.3, §16.1 |

## Goal

Let a customer create their first project by connecting their own mesh provider account, in one self-service wizard, and hand Klustro ownership of that network's policy. This is the gate between "signed up" and "can add machines".

## In scope

- Project entity and its immutable mesh binding.
- Mesh provider catalogue and capability declaration.
- The Tailscale connection wizard, credential capture and validation.
- Takeover confirmation and the revision-0 backup.
- Credential storage, masking, ongoing validation and rotation.
- Connection status and degradation.

## Out of scope

- Rendering and applying the policy (Feature 08).
- Node enrolment and device creation (Features 09, 10).
- Additional mesh providers (later).
- Any cluster or node concept.

## Requirements

**Project**

R1. A project belongs to exactly one organisation and groups clusters and nodes.
R2. A project owns exactly one dedicated mesh network.
R3. The mesh provider and the mesh connection are chosen at project creation and fixed for the project's life. Changing provider means creating a new project and re-onboarding nodes.
R4. Two projects can never communicate at the network level.
R5. A project has an id, organisation reference, display name, mesh provider reference, mesh connection reference, status and deletion time.
R6. A project cannot be created without a successfully validated mesh connection.

**Provider catalogue**

R7. Mesh providers are catalogue entries shipped with Klustro, each declaring: identity, driver version, capabilities and a credential schema.
R8. Each provider declares capabilities including node-to-node layer-3 reachability, transport characteristics, policy granularity, tagging, enrolment method, ephemeral device handling, self-hosting requirement, service support, policy test support, DNS and IPv6 support, and effective MTU.
R9. Only providers declaring layer-3 node-to-node reachability, tagging, node+port policy granularity and service support may be offered for new projects.
R10. Tailscale is the only offered provider in this phase and is preselected in the wizard.
R11. Provider-specific behaviour is confined to the driver. The UI, validation and later cluster configuration adapt only to declared capabilities.

**Connection and credentials**

R12. The wizard collects the provider's credentials and an optional endpoint where the provider requires one.
R13. Credentials are stored under envelope encryption with the organisation's data key and are always masked in UI and API responses.
R14. Credentials are never sent to nodes.
R15. A connection records its provider, ownership (bring-your-own in this phase), status, last validation time and a provider-side account reference.
R16. A provider-side account reference may be bound to only one Klustro project across all organisations. An attempt to bind it twice is refused with a clear message.
R17. Project creation is refused while any validation fails. Each failure maps to a specific, actionable message rather than a generic error.

**Validation**

R18. On submission, validation confirms that the credentials authenticate, that every required scope works via a non-destructive probe, that Klustro can mint enrolment credentials with its root tag, that the provider-side account is not already bound, and that required provider features are available on the customer's provider plan.
R19. Validation reports existing devices that would lose access at takeover as a warning, not a failure.
R20. Each missing scope is reported individually so the customer knows what to change.

**Takeover**

R21. Before takeover the user must explicitly confirm that Klustro will replace the entire policy file and manage it from then on.
R22. The wizard states the prerequisite bootstrap configuration the customer must add to their provider policy so Klustro's root tag is owned.
R23. On confirmation, Klustro reads and stores the current provider policy as revision 0 (the backup).
R24. Revision 0 is never rendered or rolled back to while the project exists; it exists only to be restored if the project is purged.
R25. Klustro recommends, where the provider supports it, disabling further out-of-band policy edits in the provider's own console.

**Ongoing health**

R26. Credentials are re-validated on a fixed cadence (default 15 minutes).
R27. On validation failure the project moves to a degraded mesh-management state: the mesh and workloads keep running, while joins, retags and policy runs pause, and administrators are notified.
R28. Credential rotation validates the new secret before replacing the old one and discards the old only after the new one passes.

**Deletion**

R29. Project deletion is a soft delete with a restore window (default 7 days).
R30. During the restore window nodes keep running and Klustro stops changing anything.
R31. On purge, Klustro removes the devices it created from the network, restores the takeover backup unless the customer opted out, and deletes the stored credentials. Agent and cluster teardown behaviour is defined in Feature 13.

## Acceptance criteria

- A customer with no Klustro involvement creates a project using their own provider account.
- Each validation failure produces its own message; a project is never created in an unvalidated state.
- The second attempt to bind the same provider account to a different project is refused.
- The original policy content is retrievable as revision 0 after takeover.
- Credentials are masked everywhere and never appear in logs, audits or node communication.
- Invalidating credentials mid-flight pauses joins and policy runs while already-running clusters are unaffected.
