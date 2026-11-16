# Business-Logic Finding — Report Template

Use this template for one finding per report. Fill in every section; if a field does not apply, write "N/A" and say why. Use generic placeholders (`shop.example.com`, `YOUR_TEST_ACCOUNT`) — never include real user data, real credentials, or another person's account details.

## Summary

- **Title:** [Short, specific title — e.g. "Coupon can be redeemed twice via parallel requests"]
- **Severity (your assessment):** [Critical / High / Medium / Low — with a one-line reason]
- **Target:** [Authorized target only, e.g. `shop.example.com` — the system named in your authorization]
- **Date tested:** [YYYY-MM-DD]
- **Tester account(s):** [Your own test account identifier(s) only, e.g. `test-user-01`]

## Business Impact

Explain the impact in business terms, not just technical terms:

- What could an attacker gain — money, goods, credit, access, account takeover?
- Is the gain repeatable, and does it scale (once per account, once per coupon, unlimited)?
- Who is affected — the business, other users, or both?
- Keep the impact honest: state what you proved, and label anything you did not test as untested.

## Affected Flow

Describe the intended flow in a few lines:

1. [Step 1 — e.g. user adds an item to the cart]
2. [Step 2 — e.g. user applies a one-time coupon]
3. [Step 3 — e.g. user checks out and pays]

Then state the business rule that is broken: [e.g. "A one-time coupon must be usable at most once per account."]

## Reproduction Steps

Use your own test account and test data. Number every step so the reader can follow it exactly.

**Setup**

1. [e.g. Create test account `test-user-01` with a starting balance of `100` test credits.]
2. [e.g. Obtain a one-time coupon `TEST-COUPON-01` valid for one use.]

**Steps**

1. [e.g. Log in as `test-user-01` and capture the coupon-redemption request in a proxy.]
2. [e.g. Send the same redemption request `N` times in parallel (e.g. 20 copies via Burp Repeater "Send group in parallel").]
3. [e.g. Record the responses: `X` of `N` returned success.]
4. [e.g. Check the resulting state: the account ledger shows `X` redemptions; the balance is `<value>`.]

**Request (generic placeholder — adapt to your target)**

```http
POST /api/redeem-coupon HTTP/1.1
Host: shop.example.com
Content-Type: application/json

{"coupon_code": "TEST-COUPON-01"}
```

**Expected result**

[What the business rules say should happen — e.g. "Exactly one redemption succeeds; all other attempts are rejected."]

**Actual result**

[What actually happened — e.g. "Two redemptions succeeded; the ledger shows two entries and the discount was applied twice."]

## Evidence

- [Response excerpts, status codes, and timestamps — redact tokens and session values]
- [Server-side state: ledger rows, balance before/after, order status — this is what proves the finding]
- [Screenshots or logs, if any — with sensitive values redacted]

Note what you did **not** do: [e.g. "No real payment was made; no real order was placed; only my own test account was used."]

## Remediation

Suggest a fix at the design level (the program owner decides the implementation):

- [e.g. Make the check-and-update atomic: a single database transaction with a row lock, or a unique constraint on `(account, coupon)` so a duplicate insert fails.]
- [e.g. Recalculate price and discount server-side from the catalog; never trust a client-sent price, total, or payment-status flag.]
- [e.g. Enforce the workflow state machine server-side: the confirm step must require a verified completed payment, and order status may only move through legal transitions.]
- [e.g. Compare secrets, tokens, and signatures with strict equality or a timing-safe comparison function; validate input types before comparing.]

## References

- [Link to the relevant playbook file, e.g. RACE-CONDITIONS.md]
- [Any public reference that explains the pattern — no links to real targets]

## Retest Notes (fill in after the fix)

- **Retest date:** [YYYY-MM-DD]
- **Result:** [Fixed / Partially fixed / Not fixed — with the evidence]
