# Feature 16 — Policy drift detection and rollback

| | |
|---|---|
| Increment | 16 |
| Depends on | 01–15 |
| Unblocks | — |
| Spec refs | §10.11, §10.12, §10.13, §19.1, §21 |

## Goal

Guarantee that the network policy Klustro believes it applied is the one actually in force. Klustro owns the policy file outright; anything else editing it is a defect that must be detected, reported and undone automatically.

## In scope

- Continuous drift detection.
- Automatic reversion and its reporting.
- Loop guarding against an external automation fighting Klustro.
- Rollback to a prior revision with correct re-rendering.
- Renderer rollout safety as it relates to drift.

## Out of scope

- Preventing out-of-band edits at the provider (recommended in Feature 07, owned by the provider).
- Rollback of anything other than policy.
- Rollback to the takeover backup while the project exists.

## Requirements

**Detection**

R1. Klustro compares the live policy's canonical hash with the hash of the last applied revision on a fixed cadence (default every 5 minutes).
R2. Where the provider offers change notifications, Klustro also reacts to them, so detection is not bounded by the polling interval.
R3. A difference is recorded as a drift event containing the live content, a human-readable difference, and when it was detected.
R4. Drift detection must not itself write to the provider.

**Reversion**

R5. On drift, the pipeline runs with a drift trigger and re-applies the current render for that project.
R6. Reversion must not be queued behind unrelated work indefinitely; it is a normal pipeline run subject to the same per-project serialisation.
R7. Reversion emits a distinct audit event.
R8. Project administrators are notified that the policy was edited externally and has been reverted, with a route to view the difference.
R9. If reversion cannot complete because credentials are invalid or the provider is unreachable, the drift event remains open and visible, and reversion is retried with backoff.

**Loop guard**

R10. More than three drift events for one project within one hour raises an "isolation at risk" condition.
R11. In that condition Klustro keeps reverting, must not stop protecting isolation, displays a prominent warning, and notifies organisation Owners.
R12. The condition clears automatically once drift stops recurring.

**Rollback**

R13. A principal holding the rollback action selects an earlier revision by sequence.
R14. Rollback takes that revision's set of active user rules and its renderer version, and re-renders them against the project's **current** topology, so no deleted node, cluster or label is ever referenced.
R15. The result is tested, planned and applied as a new revision, clearly labelled as a rollback of the named earlier revision. Rollback never rewrites history.
R16. Rules that existed in the target revision but were retired since are reactivated; rules created after it are retired.
R17. Rollback is audited, including who performed it and which revision was the target.
R18. The takeover backup cannot be rolled back to while the project exists, because it lacks Klustro's system rules. The UI states this rather than offering the option.
R19. Rollback is refused while credentials are invalid or a run is in progress; the user is told why.

**Interactions**

R20. Drift reversion and rollback both go through the normal test-and-apply path; neither may bypass validation.
R21. If a renderer version rolls out and immediately fails validation for a project, the rollout pauses; drift handling must remain functional on the last known-good renderer during that pause.
R22. Drift, reversion, loop-guard raising and clearing, and rollback are visible on the project's mesh policy page alongside run history.
R23. The CLI can show drift events, roll back to a revision, and show the difference between two revisions.

## Acceptance criteria

- An out-of-band policy edit is detected within the polling interval, reverted without operator action, and reported to administrators with the difference.
- A repeated external edit pattern raises the isolation-at-risk warning and emails Owners.
- Rollback to a revision that referenced a now-deleted cluster succeeds and produces a policy that references nothing stale.
- Rollback to the takeover backup is not offered while the project exists.
- Drift reversion never applies content that fails validation.
