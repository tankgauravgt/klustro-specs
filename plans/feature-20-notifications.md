# Feature 20 — Notifications

| | |
|---|---|
| Increment | 20 |
| Depends on | 01, 03, 04 |
| Unblocks | 21, 22 |
| Spec refs | §14.2, §19.3, §9.3, §10.11, §12.1 |

## Goal

Make sure a customer learns about anything that needs their attention before it costs them — a low balance, a degraded connection, a reverted policy, an upcoming restriction — and that whoever is responsible within the organisation receives it.

Notifications are treated as a real subsystem from here because billing enforcement (Feature 21) is only acceptable if the customer is warned properly first.

## In scope

- The trigger catalogue.
- Recipient resolution within an organisation.
- Channels available at launch.
- Delivery guarantees, preferences and audit.
- Template and content requirements.

## Out of scope

- Webhooks and event stream (later).
- Slack and Teams channels (later).
- Log, metric or alert forwarding (later).
- Paging of Klustro's own on-call, which is internal operational tooling (Feature 30).

## Requirements

**Channels**

R1. Launch channels are email and in-app notification.
R2. The channel set is extensible: adding a channel must not require changes to the components that raise events.
R3. A notification's channel is chosen by the raising event's severity and the recipient's preferences, not hardcoded at the call site.

**Recipients**

R4. Notifications are addressed to roles, not individuals: an event that concerns an organisation's ability to pay goes to Owners and Billing Admins; an event that concerns a project's network goes to principals with administrative rights on that project.
R5. Recipient resolution uses the authorization model, so a customer's own role changes take effect for future notifications.
R6. A notification is never sent to a principal who has no access to the object it concerns.
R7. If an organisation has no resolvable recipient for a critical notification, that fact must itself be visible to Klustro staff, since it otherwise fails silently.

**Trigger catalogue**

R8. At minimum these events produce notifications:
  - prepaid balance crossing each configured threshold;
  - postpaid spend crossing each configured proportion of the credit limit;
  - entering each billing lifecycle state, with the consequence stated plainly;
  - the final notices before deletion;
  - daily reminders while in grace and while frozen;
  - policy drift detected and automatically reverted, routed to project administrators;
  - mesh connection credentials invalid, so that joins, retags and policy runs are paused;
  - a policy update that could not be applied, pending a fix;
  - cluster degraded, including datastore health deterioration and quorum risk;
  - a server connected over a relayed path, or latency between servers above the warning threshold;
  - a mesh rule awaiting approval, routed to principals able to approve it;
  - an eligible access assignment activated or about to expire;
  - node offline beyond the threshold for a server node;
  - automated agent version rollout paused for a project.

R9. Each trigger declares its severity, its recipient rule, its default channels and whether it can be opted out of.
R10. Security-relevant notifications (credential changes, lifecycle transitions, ownership changes) cannot be opted out of.

**Content**

R11. Every notification states what happened, what it means for the customer, and what they can do about it.
R12. Lifecycle and balance notifications state the concrete consequence and the deadline, not just the state name.
R13. Notifications never contain secret material, credentials or token values.
R14. Notification links land on the relevant object in the console, not a generic landing page.

**Delivery and preferences**

R15. Delivery is at-least-once and idempotent per event and recipient, so a retry does not spam a recipient.
R16. Repeated identical events within a short window are grouped rather than sent individually.
R17. A recipient can set preferences per category and per channel, within the limits of R10.
R18. In-app notifications are marked read per recipient and are clearly distinguishable from unread ones.
R19. Delivery, failure and suppression are recorded, so that "was the customer warned?" is answerable after the fact for any account.

## Acceptance criteria

- A customer approaching zero balance receives balance-threshold, state-entry and daily reminders, each stating the actual consequence and deadline.
- Removing an Owner who was the only recipient causes the missing-recipient condition to be visible to staff rather than silently swallowing messages.
- A policy drift reversion reaches project administrators with a link to the difference.
- No notification payload contains a secret.
- A delivery retry does not produce duplicate notifications for the same event and recipient.
- "Was this customer warned, and when?" is answerable from stored delivery records alone.
