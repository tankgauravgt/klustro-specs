# Feature 08 — Policy pipeline core

| | |
|---|---|
| Increment | 08 |
| Depends on | 01, 04, 07 |
| Unblocks | 09, 10, 11, 12, 15, 16 |
| Spec refs | §10 (all), §8.4, §8.5 |

## Goal

Make Klustro, and only Klustro, the author of the project's network policy. The pipeline is the single path from Klustro's model of the project to the network's live policy, and it must be deterministic, testable, concurrency-safe and reversible.

This increment delivers system-derived policy only: the rules that follow from cluster and node topology. User-authored rules arrive in Feature 15.

## In scope

- Triggers, queueing, coalescing and per-project serialisation.
- Deterministic rendering of the complete policy from current state.
- Test layers and the apply-safety gate.
- Atomic apply with optimistic concurrency.
- Revision records, diffs and barriers.
- Run states, failure handling and visibility.
- Renderer versioning and staged rollout.

## Out of scope

- User rules and their approval lifecycle (Feature 15).
- Drift detection and rollback (Feature 16).
- Service objects' health-awareness on the node side (Feature 12).
- Additional providers (later).

## Requirements

**Triggers and queueing**

R1. The pipeline is triggered by topology changes, label changes, rule activation and retirement, drift, renderer upgrades, manual resync, rollback and initial takeover.
R2. Exactly one run per project may be in progress at any time, enforced by a lease. Different projects run concurrently.
R3. A crashed run's lease expires and the next run re-renders from current state.
R4. Triggers arriving within a short coalescing window are merged into a single run that records every contributing trigger.
R5. Every run renders the complete policy from the state at the moment it runs, never a delta. Stale or out-of-order triggers must be harmless.
R6. If the rendered content hash equals the last applied hash, the run ends as a no-change run and performs no write.

**Rendering**

R7. A renderer is deterministic and versioned; the version is recorded on every revision.
R8. Identical state always produces byte-identical output: canonical ordering of all blocks and stable ordering of rules.
R9. The rendered document contains a header identifying Klustro as the manager, the project, the revision sequence and the renderer version.
R10. The rendered document defines ownership of the root tag and of every tag currently in use, referring to the root tag as owner.
R11. The rendered document contains system rules followed by active user rules (none yet at this increment).
R12. The rendered document contains auto-approval entries for each cluster's service, scoped to that cluster's server tag.
R13. The rendered document contains generated tests.
R14. All blocks not listed above are absent. There are no implicit grants, no SSH rules and no attribute rules.
R15. The content hash is a stable hash of the canonical form.

**System rules**

R16. For each cluster, the following are generated from topology and are neither editable nor deletable by any principal:
  - server-to-server consensus traffic;
  - cluster nodes to the cluster API and supervisor port;
  - cluster nodes to each other for kubelet and overlay networking;
  - cluster nodes and tailnet members to the cluster's service on the API port.
R17. Unassigned nodes receive no grants.
R18. Traffic between clusters in the same project is denied by default, as is everything not explicitly listed.
R19. Each system rule is recorded with its origin so the UI can distinguish it from a user rule.

**Testing**

R20. Three test layers run in order and all must pass before anything is applied:
  - local structural checks: schema validity, tag-name validity, every referenced tag declared, no wildcard sources or destinations, size within the provider's limit;
  - embedded provider-evaluated tests: acceptance and denial assertions generated per tag pair, plus one acceptance assertion per active user rule;
  - the provider's own validation of the complete candidate document.
R21. Tests are generated per tag pair, not per node, so the file's size does not grow with node count.
R22. A test or validation failure is treated as a Klustro defect: nothing is applied, the run is recorded as failed validation, on-call is paged, and the project surfaces a "policy update pending" state.
R23. No failure mode may result in a partially applied policy.

**Applying**

R24. Apply performs, in order: ensure service objects for new clusters exist, write the policy conditionally on the last applied revision's entity tag, then delete service objects for removed clusters after the policy no longer references them.
R25. A conditional write rejection is treated as drift: the live content is recorded, the run re-renders and retries with the new tag, up to a bounded number of attempts per run.
R26. Each policy write is a single atomic operation.

**Verifying and recording**

R27. After writing, the pipeline reads the policy back, canonicalises it and compares hashes with what it rendered. A mismatch is a failure.
R28. Each successful apply records a revision containing: id, project, sequence, run, triggers, renderer version, content, hash, provider tag, human-readable diff, test results, the active rule ids included, approvers, and the applied time.
R29. The diff is human-readable and available in the UI and CLI.

**Barriers**

R30. Operations that depend on policy must wait for a revision that contains what they need:
  - a new cluster becomes eligible for joins only after a revision contains its tags, grants, service object and auto-approval entry;
  - a join token for a cluster cannot be created before that revision, and the customer sees that the network is being prepared rather than an error;
  - a node being assigned or relabelled waits for a revision whose ownership block contains the new tags, and only then is the device retagged.

**Rollout**

R31. New renderer versions roll out progressively across projects with automatic pause if any run fails validation.
R32. Run states are: queued, rendering, testing, awaiting approval, applying, verifying, succeeded — with terminal alternatives no-change, failed validation, blocked credentials and failed apply.
R33. Failures are handled as specified: provider rate limits and server errors back off exponentially with the live policy untouched; invalid credentials block the run and degrade the project; validation failures apply nothing.
R34. Every run, failure, drift reversion and renderer promotion is audited.

## Acceptance criteria

- Declaring a cluster's topology produces a policy containing that cluster's tags, system rules, service auto-approval and tests, applied atomically.
- Rendering the same state twice produces byte-identical output in golden-file tests.
- Concurrent triggers for one project serialise, coalesce where close in time, and never produce two concurrent applies.
- A run whose render fails validation applies nothing and pages on-call.
- A conditional write conflict is recovered by re-render and retry without operator involvement.
- A cluster cannot accept joins before its revision is applied.
