# Business Logic Vulnerabilities Playbook

A practical playbook for finding business-logic flaws in web applications — the bugs that scanners miss because the application is doing exactly what its code says, just not what its business rules intend.

Business-logic flaws live in workflows, not in syntax: a coupon that can be redeemed twice, a price the server trusts from the client, a checkout step that can be skipped, a limit that only exists in the user interface. Finding them takes reasoning about the intended flow, then testing what happens when that flow is bent, repeated, reordered, or run in parallel.

## Who It's For

- Security researchers and bug-bounty hunters testing applications they are authorized to test.
- Application developers and reviewers who want to understand how business rules break in practice, so they can design checks that hold server-side.
- Students learning web security beyond injection flaws, using a lab or a deliberately vulnerable training application.

## Contents

| File | What it covers |
|------|----------------|
| [RACE-CONDITIONS.md](RACE-CONDITIONS.md) | Race conditions and TOCTOU: parallel-request flaws, coupon and balance double-spend, detection approach |
| [PRICE-QUANTITY.md](PRICE-QUANTITY.md) | Price and quantity manipulation: client-side price trust, negative and fractional values, currency and rounding issues |
| [WORKFLOW-BYPASS.md](WORKFLOW-BYPASS.md) | Workflow bypass: skipped steps, state manipulation, access-logic flaws, and weak-comparison (type juggling) concepts |
| [CHECKLIST.md](CHECKLIST.md) | Copy-paste checklist for business-logic testing |
| [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md) | Report template for a business-logic finding |

## Scope & Authorization

**Read this before testing anything.**

- Only test targets you own or have explicit written authorization for. Stay strictly in scope: test only the systems, accounts, and features named in that authorization.
- Test with your own accounts only. Never manipulate real orders or payments, never touch another user's data, and never use a finding to obtain goods, credit, or access you are not entitled to.
- If a test could move real money, ship a real item, or change another person's account, do not run it. Report the logic gap from the evidence you already have, or ask the program owner for a safe test procedure first.
- All examples in this playbook use generic placeholders such as `shop.example.com` and do not target any real third-party site or company. Replace them with your own authorized target and your own test accounts.
- When in doubt, stop and ask. A clean, well-evidenced report of a logic gap is worth more than a risky live demonstration.

## How to Use This Playbook

1. **Map the intended flow first.** For each business process — checkout, coupon redemption, password reset, subscription change — write down the happy path: the steps, in order, and the checks that are supposed to happen at each step.
2. **Ask the bending questions.** What if a step is skipped? Repeated? Reordered? Done twice at the same time? Done with a negative, zero, fractional, or very large value? Done by a different role or a different user?
3. **Test one variable at a time** on your own test account, and record the exact request, response, and resulting state (balance, order status, ledger entry) for each attempt.
4. **Confirm server-side.** A surprising response is not a finding until the resulting state confirms it: a second ledger row, a balance that moved twice, an order that shipped without payment. Reproduce it, then write it up with the template in [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md).

## The Core Mental Model

Most business-logic flaws fall into a few patterns:

- **The server trusts the client.** A price, discount, status flag, or role sent by the browser is accepted without re-checking it server-side.
- **Check and update are not atomic.** A balance or coupon is checked, then updated in a separate step — and a second request can slip between the two (see [RACE-CONDITIONS.md](RACE-CONDITIONS.md)).
- **A step is optional in practice.** The interface enforces an order the API does not (see [WORKFLOW-BYPASS.md](WORKFLOW-BYPASS.md)).
- **A value is not validated as a business value.** Negative, fractional, or overflowing quantities and amounts pass a check that only looked for "is this a number?" (see [PRICE-QUANTITY.md](PRICE-QUANTITY.md)).

If you keep those four patterns in mind while mapping a flow, most findings follow naturally.

## License

MIT — see [LICENSE](LICENSE).
