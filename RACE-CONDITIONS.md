# Race Conditions

A race condition is a flaw where the outcome depends on the timing of two operations that run at the same time. In web applications the classic form is **TOCTOU — time-of-check to time-of-use**: the application checks a condition, then acts on it in a separate step, and a second request can slip between the check and the action.

## The Core Pattern: Check, Then Update (Not Atomically)

A vulnerable flow often looks like this in plain language:

1. Read the current balance, stock level, or "already claimed?" flag.
2. Decide the request is allowed.
3. Update the balance, stock, or flag.

If two identical requests arrive at nearly the same moment, both can pass step 2 before either completes step 3 — so a one-time action happens twice. The fix, at a design level, is to make the check and the update a single atomic operation: a database transaction with a row lock, a unique constraint on the claim, or an idempotency key that the server enforces.

## Where to Look First

Prioritize operations that are meant to happen **once** or that **deduct a finite resource**:

| Priority | Operation class | Examples (generic) |
|----------|-----------------|---------------------|
| 1 | One-time redeem / claim | Coupon redemption, voucher use, welcome bonus, reward claim |
| 2 | Balance / quota / stock deduction | Fund transfer, purchase, reservation, inventory decrement |
| 3 | Invite / referral / signup bonus | Referral claim, invite acceptance |
| 4 | Verification / reset | Token verification, email confirmation, password reset |
| 5 | Anything that "should succeed only once per user" | Vote, rating, single-use link, one-time download |

## Common Patterns

### Coupon / Balance Double-Spend

A single-use coupon or a fixed balance is used by two parallel requests. Both requests observe "still valid / sufficient funds" before either one records the use, so both succeed. Evidence is server-side: two success responses, duplicate ledger rows, a balance that went negative, or a discount applied twice.

### Limit Overrun

A "claim once" reward, a daily limit, or a per-phone rate limit is implemented as a check-then-increment counter rather than an atomic increment. A burst of parallel requests can pass the check before any of them increments the counter, so the limit is exceeded. Compare a burst of parallel attempts with the same number of sequential attempts: if the parallel burst gets further, the limit is not atomic.

### Inventory Oversell

Stock of one item remains, and several parallel purchase requests each see "in stock" before any of them decrements the count. Multiple orders are created for one unit. The same pattern applies when add-to-cart and checkout each check inventory independently.

### Multi-Step Race

A workflow such as create → pay → confirm can be raced across two parallel pipelines. If the confirm step does not cryptographically bind to a completed payment, a confirm started on one channel can complete while the payment on the other channel is still in flight or abandoned — leaving an order marked paid or shipped without a matching payment.

### Verification and Registration Races

Email verification, password-reset tokens, and same-email registration are also race targets: two simultaneous registrations with the same email, a token used twice before it is invalidated, or a verification completing while a privileged action that requires verification is already in progress. These matter because they can affect account ownership, not just money.

## Detection Approach (Your Own Test Account Only)

Work only on a target you are authorized to test, with your own accounts and your own test data.

1. **Pick a state-changing request** — one that should succeed at most once, or that deducts a finite resource. Capture it in a proxy (for example, Burp Suite Repeater).
2. **Record the baseline first.** Run the request once, sequentially, and note the response and the resulting state (balance, claim count, order status). Then reset your test data if the application allows it, or use a fresh one-time test item.
3. **Send a parallel burst.** Send the same request many times as simultaneously as your tooling allows — typically 20–30 copies. Practical options:
   - Burp Suite Repeater's **Send group in parallel** (and single-packet attack where available).
   - A concurrency extension such as Turbo Intruder with a gate, so all queued requests release together.
   - An HTTP/2 single-packet approach, where several requests travel in one burst so the server dispatches them with sub-millisecond spacing.
4. **Classify the outcome.** The expected result is zero or one success. More than one success, duplicate ledger entries, or an inconsistent final state (negative balance, stock below zero, two grants) indicates a race.
5. **Confirm it is reproducible, not flaky.** Repeat the burst with fresh test data. A real race reproduces; a one-off error does not. Correlate the duplicate successes with unique markers (a per-request header or unique field) so you can point to the exact duplicate records in the application's own state.
6. **Stop at the evidence.** Do not keep racing a live balance, coupon pool, or inventory to "see how far it goes." Two duplicate successes plus the server-side artifact are enough for a report.

## What Counts as Evidence

- More than one success response for an operation that should succeed once.
- A server-side artifact: two ledger rows, two grants, two emails, two orders, or a final balance/stock that contradicts the policy.
- A parallel burst that exceeds a limit which sequential attempts respect.

A single surprising response with no state change is not a finding — it may be a display quirk or a retry artifact.

## Remediation Concepts (for Developers and Reviewers)

- Make check-and-update atomic: a single transaction with a row lock (`SELECT … FOR UPDATE` or equivalent), not a read in application code followed by a separate write.
- Enforce uniqueness at the database level: a unique constraint on `(user, coupon)`, `(user, reward)`, or `(order, transaction)` makes a duplicate insert fail no matter how the requests are timed.
- Require and enforce idempotency keys on state-changing endpoints, so a retried or duplicated request is recognized as the same operation.
- Use atomic counters for rate limits and quotas (increment-and-check in one step), not check-then-increment.
- In multi-instance deployments, a lock that only exists in one process's memory does not protect the others — use a shared lock or a database constraint.

## Scope Reminder

Race testing sends bursts of requests. Run it only against targets you own or have explicit written authorization for, with your own test accounts and test data, and keep bursts modest. Never race real payments, real orders, or another user's resources.
