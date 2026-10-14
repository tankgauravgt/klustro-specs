# Feature 22 — Postpaid billing

| | |
|---|---|
| Increment | 22 |
| Depends on | 01–21 |
| Unblocks | — |
| Spec refs | §13.4, §13.6, §13.7, §16.4, §13.3 |

## Goal

Support mid-size customers who want invoicing rather than a wallet, without forking the billing engine. Prepaid and postpaid share one meter, one rating path and one ledger; only settlement differs — and this increment is that difference.

## In scope

- Postpaid eligibility, automatic and manual.
- Mode switching in both directions.
- Draft invoice accrual.
- Credit limits and spend alerts.
- Invoice finalisation, charging and payment terms.
- Collections and its hand-off to the lifecycle state machine.

## Out of scope

- Tax calculation and document generation, which are reused from Feature 19.
- Lifecycle state effects, which are defined in Feature 21.
- Consolidated billing across multiple organisations (later, for resellers).
- Contracts and purchase orders beyond recording their terms.

## Requirements

**Eligibility**

R1. Prepaid is the default for every account.
R2. An account becomes eligible for an automatic offer after: at least three consecutive months on a paid plan in prepaid mode; total spend at or above a configurable threshold; a verified saved payment method; and no chargebacks or failed payments.
R3. When eligible, the customer is offered a switch to monthly invoicing in the console.
R4. Klustro staff can also enable postpaid manually for a contracted customer or purchase order, recording the agreed payment terms.
R5. Enterprise contracted customers are postpaid by contract rather than by the automatic offer.
R6. Postpaid requires a positive decision by the customer or by staff; it is never entered silently.

**Mode switching**

R7. Prepaid to postpaid: the remaining wallet balance is consumed first, and only usage beyond it accrues to the invoice.
R8. Switching must not lose, duplicate or re-rate usage that crosses the boundary.
R9. Postpaid to prepaid: the final invoice must be settled before the switch completes.
R10. Mode switching is a distinct audited action, gated by its own permission.

**Accrual and invoicing**

R11. Postpaid usage accrues to a draft invoice for the calendar month.
R12. The draft is visible to the customer during the month, with usage detail.
R13. On the first day of the following month the invoice is finalised with tax applied.
R14. A card customer's saved method is charged automatically at finalisation.
R15. A contracted customer receives the invoice with their agreed payment terms — net 15, 30 or 45.
R16. Once finalised, an invoice is immutable; late usage is billed on the next invoice.
R17. Finalisation and charging are idempotent, so a retried job never double charges.

**Credit limit**

R18. An initial credit limit is derived as a multiple of average monthly spend over the previous three months, with a configurable floor.
R19. The limit is recomputed monthly.
R20. The limit can only rise, and only after a sustained period without failed payments (default six consecutive months).
R21. The current limit is always visible to the customer.

**Alerts and enforcement hand-off**

R22. Spend alerts fire at fixed proportions of the credit limit (default 80% and 100%).
R23. Crossing the credit limit moves the account into the lifecycle Grace state.
R24. Postpaid billing decides only that a threshold was crossed; it never itself changes lifecycle state.

**Collections**

R25. A failed automatic charge is retried on a fixed schedule (default days 1, 3, 5 and 7).
R26. Each failure notifies the customer and becomes visible on the account.
R27. An invoice unpaid seven days after its due date moves the account into Grace, from which the standard lifecycle sequence applies.
R28. Collections must never take an action that violates the lifecycle invariants — no workload is stopped, no device is removed, no policy is rewritten.

**Visibility**

R29. A customer can see the draft period's usage, past invoices, their payment terms and their credit limit.
R30. The customer can see exactly which usage produced each invoice line.
R31. Eligibility evaluation, mode switches, credit limit changes, finalisation and each collection attempt are audited.

## Acceptance criteria

- An eligible prepaid customer is offered, and can accept, monthly invoicing.
- A month's usage accrues to a draft, finalises on the first with correct tax, and is charged to the saved method once.
- A contracted customer receives an invoice with the agreed terms instead of an automatic charge.
- Crossing 80% and 100% of the credit limit each produce exactly one alert, and crossing 100% enters Grace.
- Failed charges retry on schedule and hand off to Grace seven days after the due date.
- Switching prepaid → postpaid consumes the wallet first and does not double-bill the crossed usage.
- Switching postpaid → prepaid is blocked until the final invoice is settled.
- No collection action removes a device, stops a workload or edits policy.
