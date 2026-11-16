# Business-Logic Testing Checklist

Copy-paste checklist for testing a business flow on a target you own or are authorized to test. Use your own test accounts and test data only. For each item, mark: **N/A** / **Not vulnerable** / **Vulnerable** / **Needs recheck** — and record the exact endpoint, parameter, and value that proved it either way.

## Before You Start

- [ ] Authorization confirmed in writing; scope, accounts, and test data identified
- [ ] Intended (happy-path) flow mapped: steps in order, checks at each step
- [ ] Baseline recorded: normal flow works, normal balances / states noted
- [ ] Test accounts created (at least two, to test cross-account access safely)

## Race Conditions

- [ ] One-time coupon / voucher redeemed with parallel requests — only one succeeds
- [ ] Reward / bonus claim sent in parallel — only one grant recorded
- [ ] Balance transfer / withdrawal sent in parallel — balance never goes negative, no double credit
- [ ] Purchase of last-stock item sent in parallel — only one order created
- [ ] "Claim once" / first-order bonus sent in parallel — only one claim recorded
- [ ] Rate limit / lockout tested with a parallel burst vs sequential attempts — limit holds
- [ ] Password-reset / verification token used twice — second use rejected
- [ ] Same-email registration sent in parallel — only one account created

## Price & Quantity

- [ ] Price / total / discount field changed in checkout request — server recalculates from its own catalog
- [ ] Client-sent `payment_status` / `success` flag — server verifies payment with provider, not the flag
- [ ] Item ID swapped for a cheaper item at checkout — server charges the item actually ordered
- [ ] Quantity `-1` — rejected; no credit issued
- [ ] Quantity `0` — rejected or handled; no empty order confirmed
- [ ] Quantity fractional (e.g. `0.02`) — rejected for unit-sold items
- [ ] Quantity very large — rejected or handled; total does not wrap to zero/negative
- [ ] Coupon stacked with another coupon / credit / points — stacking policy enforced, total never negative
- [ ] Fixed-amount coupon on an item worth less than the coupon — discount capped at order total
- [ ] Coupon applied, then item removed / changed — discount does not survive onto a different order
- [ ] Required paid-item field omitted from request — error, not a free order
- [ ] Currency code changed — server validates currency against the quoted price
- [ ] Shipping / fee field set negative — rejected; total not reduced
- [ ] Amount `0`, `0.01`, or `0.001` — handled per business rules, not as a free purchase

## Workflow Bypass

- [ ] Final step (confirm / set password / complete) called directly, skipping earlier steps — rejected
- [ ] Protected page accessed directly after password step, before second factor — rejected
- [ ] Order status changed by client (`pending` → `shipped` / `refunded`) — rejected
- [ ] Completed transaction ID reused for a second purchase — rejected
- [ ] Email changed after verification — account requires re-verification
- [ ] Premium endpoint called directly with a free-tier account — rejected
- [ ] Premium feature still active after downgrade from paid trial — disabled on downgrade
- [ ] Free trial still usable after expiry — enforced server-side

## Access Logic

- [ ] One test account reads / changes another test account's order or profile by ID swap — rejected
- [ ] Regular test account sets role / level field to admin — rejected
- [ ] Regular test account calls merchant / admin endpoint — rejected
- [ ] Same action tested via web and via mobile/API entry point — both enforce the same check
- [ ] Permission revoked in UI — API access actually revoked
- [ ] Resource IDs predictable / sequential — enumeration does not expose other users' data

## Type Juggling (Loosely-Typed Comparison)

- [ ] Secrets / tokens / hashes compared with strict equality or `hash_equals()` — no loose `==` on secrets
- [ ] Signature / HMAC compared strictly — never loosely against `"0"` or `0`
- [ ] Language and version noted — coercion behavior confirmed for that version
- [ ] Unexpected types (array, boolean, null) supplied for a string secret — rejected, not coerced to a match

## Account & Verification

- [ ] Verification code empty / fixed / reused — rejected
- [ ] Verification code from one account used on another — rejected
- [ ] Password-reset token predictable, reusable after use, or valid after email change — rejected
- [ ] Reset flow step skipped (final reset endpoint called directly) — rejected
- [ ] Session / token still valid after logout or expiry — rejected
- [ ] OAuth / SSO linking with another account's email — no account merge without verification

## Coupons & Referrals

- [ ] Same coupon applied twice to one order — rejected
- [ ] Coupon applied, item removed, different item added — discount revalidated
- [ ] Referral loop (account A refers B refers C…) — abuse controls in place
- [ ] Coupon for a fixed amount on a variable-price item — no negative total

## After Testing

- [ ] Every "Vulnerable" item reproduced at least once with fresh test data
- [ ] Server-side evidence recorded for each finding (ledger row, balance, order state — not just a response)
- [ ] Test data cleaned up where the application allows it
- [ ] Findings written up with [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md)
