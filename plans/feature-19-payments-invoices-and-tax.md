# Feature 19 — Payments, invoices and tax

| | |
|---|---|
| Increment | 19 |
| Depends on | 01–06, 18 |
| Unblocks | 21, 22, 29 |
| Spec refs | §13.4, §13.5, §13.6, §19.3 |

## Goal

Let money actually change hands, in both launch currencies, and produce documents a customer's finance team will accept. This is the last piece needed before a paying customer can exist.

## In scope

- The payment provider abstraction.
- USD and INR payment paths.
- Top-up checkout and saved-method charging.
- Refunds.
- Invoice and receipt documents.
- Tax calculation and tax identity.

## Out of scope

- Credit limits, postpaid invoicing cadence and collections (Feature 22).
- Currency conversion, which never happens.
- Additional currencies beyond USD and INR.
- Tax engine replacement; tax rules are consumed, not invented, here.

## Requirements

**Provider abstraction**

R1. All payment provider interaction goes through a single interface with a fixed set of operations: create a checkout, charge a saved method, refund, and handle an inbound provider notification.
R2. No billing logic outside the adapter knows which provider is in use.
R3. Adding a provider later must require only a new adapter, not changes to billing, ledger or lifecycle logic.
R4. Provider notifications are authenticated, idempotent and replay-safe; a duplicated notification must never produce a second top-up or a second charge.
R5. Provider references are stored as secret-encrypted references and never expose card or account details to Klustro.

**Currency and provider mapping**

R6. The account currency is fixed at billing account creation and never changes.
R7. INR accounts are served by the Indian payment provider, supporting the Indian payment methods specified.
R8. USD accounts are served by the international provider, supporting cards and, for contracted postpaid accounts, bank transfer.
R9. The default currency follows the billing country at creation and can be overridden at that moment only.
R10. Wallets, ledgers, invoices and credit limits are all denominated in the account currency.

**Top-up and charging**

R11. A customer can top up their prepaid wallet by a minimum amount per currency, choosing a payment method at checkout.
R12. A successful top-up immediately produces both a ledger entry and a receipt.
R13. A failed or abandoned checkout produces no ledger entry.
R14. A saved method can be charged without customer interaction, for subscription renewal and for postpaid settlement.
R15. A charge that the provider declines is recorded as a failed attempt with the provider's reason, and does not silently retry forever.

**Refunds**

R16. A refund reverses the appropriate ledger entries rather than editing or deleting them.
R17. Promotional and goodwill credit is never refundable.
R18. Refunds are performed by Klustro staff, are audited, and record a reason.

**Tax and documents**

R19. Tax is calculated from the billing account's tax profile and the customer's billing country using the platform's tax rules.
R20. Indian accounts are taxed as intra-state or inter-state supply based on the customer's state relative to Klustro's, with the customer's tax identifier printed on the invoice when provided.
R21. Non-Indian accounts are taxed per the destination rules for the billing country, including reverse-charge treatment for business customers that supply a valid tax identifier.
R22. A tax profile can be updated by a principal holding the tax profile action, and changes affect future documents only.
R23. Invoices and receipts are generated as retrievable documents and stored durably.
R24. A document records enough detail to be reconciled: line items, the price list version used, tax breakdown, currency and the payment reference.
R25. Documents are retrievable by any principal holding the invoices read action.

**Failure handling**

R26. A payment provider outage must not corrupt the ledger, lose a top-up or cause an incorrect lifecycle transition. Retries are safe and bounded.
R27. A top-up that succeeds at the provider but is not yet recorded by Klustro is reconciled automatically from provider notifications rather than requiring the customer to re-pay.

## Acceptance criteria

- A customer in India tops up in INR via UPI and receives a GST-compliant receipt immediately.
- A customer outside India tops up in USD by card and receives an appropriate document for their country.
- A replayed provider notification produces no duplicate ledger entry.
- A refund reverses the correct entries and never mutates history.
- A provider outage during a top-up leaves the wallet and ledger consistent, and reconciliation resolves the in-flight payment without double charging.
- No code path outside the payment adapters references a specific provider.
