# Feature 18 — Rating, ledger and prepaid wallets

| | |
|---|---|
| Increment | 18 |
| Depends on | 01–06, 17 |
| Unblocks | 19, 21, 22, 29 |
| Spec refs | §13.1, §13.3, §13.5, §13.7, §13.8, §13.9, §16.3 |

## Goal

Turn metered usage into money, and money into a balance a customer can trust. This is the financial core: one meter, one ledger, one truth, with prepaid and postpaid differing only in how a balance is settled.

## In scope

- The versioned price list.
- Rating of usage into charges.
- The append-only double-entry ledger.
- Prepaid grants, drawdown, balance and alerts.
- Proration and plan changes as money.
- Balance and usage visibility.

## Out of scope

- Payment provider integration, invoice documents and tax (Feature 19).
- Postpaid credit limits, invoices and collections (Feature 22).
- Lifecycle state transitions (Feature 21).
- Auto-recharge (later).

## Requirements

**Price list**

R1. Prices live in a versioned price list with effective-from dates, one price set per currency.
R2. A price list version covers, per currency: plan base prices, server and agent node-hour rates, and the minimum top-up amount.
R3. Prices are defined per currency independently. There is no live currency conversion anywhere in Klustro.
R4. A price change never alters already-rated history.

**Rating**

R5. Rating converts usage events into charges using the price list version effective at the event's hour start, not at the time rating runs.
R6. Each plan defines an included node-hour allowance per month, pooled across both meters.
R7. Included hours are consumed in chronological order across the meters.
R8. Only usage beyond the included allowance is rated, and it is rated per meter at that meter's rate.
R9. The Free plan never produces a charge of any kind.
R10. Rating is idempotent with respect to usage events: re-rating the same event never produces a second charge.
R11. Rating records, per charge, which price list version and which meter produced it.

**Ledger**

R12. The ledger is double-entry.
R13. The ledger is append-only. Nothing is updated or deleted; corrections are reversing entries.
R14. Ledger accounts include at least: customer wallet (a liability), accounts receivable for postpaid, revenue, promotional credit and tax payable.
R15. Every top-up, usage debit, credit grant, refund, invoice line and adjustment is a ledger entry.
R16. A balance is always computed from ledger entries. Any cached or materialised value must be re-verified against the entries on every write, and a mismatch must be treated as a defect rather than silently corrected.
R17. Ledger invariants — balances equal the sum of their entries, and every entry balances — are machine-checked continuously.

**Prepaid wallet**

R18. A wallet is a set of grants, each with an amount, a source (purchase, promotional or goodwill) and an optional expiry.
R19. Consumption order is fixed and testable: earliest expiry first, grants without expiry last; ties broken by promotional before goodwill before purchase; then oldest first.
R20. A top-up has a per-currency minimum and immediately produces a receipt.
R21. Metered charges are debited on an hourly cadence; the plan's base price is debited on the renewal date.
R22. Balance alerts fire when the balance crosses fixed proportions of the most recent top-up amount (default 25%, 10% and 0%), through email and in-app notification.
R23. A wallet may go negative during the grace state, down to a configurable cap expressed as a proportion of the most recent top-up (default 20%).
R24. Beyond that cap, usage keeps being recorded and settled at the next top-up; it must never be lost.
R25. Promotional and goodwill grants are non-refundable. Refunds of purchased credit are performed by Klustro staff as ledger entries, never by the customer.

**Plan changes as money**

R26. An upgrade takes effect immediately, prorates the base price for the remainder of the period, and applies the new limits immediately.
R27. A prepaid account must have a wallet balance covering the prorated amount for an upgrade to complete.
R28. A downgrade takes effect at the next renewal and is blocked while current usage exceeds the target plan's limits, with a list of what must be removed.
R29. Features absent from the target plan are disabled at renewal, with a warning ahead of time.
R30. Plan changes are ledger events, not silent edits to a plan field.

**Visibility**

R31. A customer can see plan, current-period usage, included allowance consumed, balance or amount owed, and the entries that produced them.
R32. The CLI exposes balance, usage and top-up.
R33. Every ledger entry is traceable to the usage events, price list version or staff action that produced it.

## Acceptance criteria

- A month of usage for a Pro customer rates correctly: included hours pooled and consumed chronologically, the remainder split by meter at the right rates.
- Rating the same period twice produces the same charges and no duplicates.
- Wallet consumption follows the specified grant order, proven by a test with overlapping expiries and mixed sources.
- The materialised balance always equals the sum of ledger entries, including after refunds and reversing entries.
- An upgrade prorates correctly and is refused when the wallet cannot cover it.
- Changing a price list version changes new charges only, never history.
