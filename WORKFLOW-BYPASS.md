# Workflow Bypass

A workflow bypass is a flaw where the application enforces an order of steps in its interface, but not in its API. The interface says "you must do A, then B, then C" — the server, however, accepts C on its own. This file covers skipped steps, state manipulation, access-logic flaws, and the weak-comparison (type juggling) concept at a conceptual level.

All examples use generic placeholders (`shop.example.com`, invented paths). Test only on targets you own or are authorized to test, with your own test accounts.

## 1. Skipped Steps

### Payment flow bypass

The intended flow is: add to cart → enter shipping → pay → confirm. The question to test is whether the confirm step actually requires a completed payment, or whether it accepts a direct request to confirm an order. Similarly, a password-reset flow (enter email → receive token → enter token → set new password) may accept a direct request to the final step without a validated token, and a login flow with a second factor may create a usable session after the password step alone, letting the second-factor step be skipped by going directly to a protected page.

**How to test:** walk the intended flow once with your own account, capture each step's request, then replay the final step's request on a fresh flow without completing the earlier steps. Check the resulting state — did the order confirm, the password change, or the session become fully active?

### Verification and subscription steps

- **Email verification:** register with one email, verify it, then change the email to another address. Does the account stay "verified" without re-verification? Can an account be claimed by changing its email to someone else's without verification?
- **Subscription tiers:** sign up for a paid trial, enable a premium feature, then downgrade to the free tier. Is the feature disabled on downgrade, or does it stay active? Can a free-tier account call a premium endpoint directly, even though the interface hides it?
- **Free trial:** does the trial actually expire? Check whether the time limit is enforced server-side or only displayed in the interface.

## 2. State Manipulation

Business objects move through states — an order goes `pending → paid → shipped → delivered`, a claim goes `unused → used`. A state-manipulation flaw lets the client move an object to a state it should not reach from its current state.

**What to check:**

- **Illegal transitions.** Can your own test order be moved directly from `pending` to `shipped` or `refunded`, skipping `paid`? Can a reward be claimed without paying, or shipping be triggered before payment?
- **Client-set status fields.** If a request contains a `status` field, does the server accept a value the client chooses, or does it derive the next state from its own state machine?
- **Transaction reuse.** Can a completed transaction ID be submitted again for a second purchase? Can a one-time token be used twice?
- **Replayed steps.** Can a step that should happen once — claiming a reward, applying a coupon, verifying a code — be replayed for a second effect? (See also [RACE-CONDITIONS.md](RACE-CONDITIONS.md) for the parallel version of this.)
- **Downgrade and cancellation residue.** Does cancelling a transaction restore a limit or refund a balance it should not? Does a limit count a payment type that another payment type bypasses?

**The fix, in principle:** the server should own the state machine. Every transition is validated against the current state and the caller's rights, and the client can request a transition — it cannot declare the resulting state.

## 3. Access-Logic Flaws

Access-logic flaws are authorization checks that are missing, incomplete, or inconsistent — the application checks "are you logged in?" where it should check "are you allowed to do this, to this object?"

**What to check (your own accounts and test data only):**

- **Horizontal access:** can one test account read or change another test account's order, profile, or resource by changing an ID in the request (`user_id`, `order_id`, `resource_id`)? Use two accounts you control, and never touch a real user's data.
- **Vertical access:** can a regular test account call an endpoint or set a field (role, type, level) that belongs to an admin or merchant role?
- **Hidden endpoints:** endpoints visible in client-side code or API documentation may lack the authorization check the interface implies. Check whether the server enforces the check, not whether the button is hidden.
- **Multi-entry inconsistency:** the web interface may block an action that the mobile API, or another endpoint for the same function, allows. Test each entry point, not just the one the interface shows.
- **Permission residue:** when a permission is revoked in the interface, is it actually revoked at the API level? A stale grant in the underlying data can leave an endpoint accessible after the UI says it is not.
- **Filter and routing gaps:** an access filter that matches URLs by string can be bypassed by path tricks (for example, encoded or suffixed path segments) if it inspects the raw request path instead of the normalized route. When reviewing code, an access check that relies on string-matching the request URI is a red flag; the check should use the framework's normalized route.

**The fix, in principle:** enforce authorization server-side on every endpoint, for every object, on every entry point — check both authentication and the specific permission for that object and action.

## 4. Type Juggling Concepts (Weak Comparison)

Type juggling is a class of logic flaw in loosely-typed languages (most famously PHP) where the comparison operator coerces values to a common type before comparing them, so two values that are different as strings can compare as equal.

**The concept, in plain terms:**

- **Loose vs strict comparison.** A loose comparison (`==` in PHP) converts its operands before comparing; a strict comparison (`===`) requires the same type and value. Secrets, tokens, and hashes must always be compared strictly — or with a timing-safe comparison function such as `hash_equals()` — never loosely.
- **Magic hashes.** Some hash digests consist of `0e` followed only by digits. Under loose comparison, such a string is read as a number in scientific notation — `0e…` is zero — so two different digests that both look like `0e` followed by digits compare as equal (`0.0 == 0.0`), even though the digests differ. This can defeat a check of the form "these two different inputs must have the same hash."
- **String-to-number coercion.** In older PHP versions, a non-numeric string compared loosely to `0` was treated as `0`, so `'abc' == 0` was true; PHP 8 changed this, so always note the language version when assessing whether a coercion applies.
- **Null and array coercion.** Passing an unexpected type (for example, an array where a string is expected) can make a comparison function return `NULL`, and `NULL == 0` is true under loose comparison. The same applies to JSON input where a boolean `true` is supplied for a password field that is then loosely compared to a stored string.
- **HMAC and signature checks.** Comparing a computed signature loosely against a constant such as `"0"` has the same problem: a signature that happens to be a magic-hash-shaped string can compare equal to zero. Signatures must be compared with a strict, timing-safe function.

**How to assess it (concept level):**

1. Find where the application compares secrets, tokens, passwords, or signatures. Look for loose comparison operators and for string-comparison functions whose return value is loosely compared to zero.
2. Note the language and version — coercion rules differ between versions.
3. Check whether both compared values can be influenced by the caller, and whether a magic-hash-shaped value can be produced for one side.
4. Report the pattern with the exact comparison and the language version; do not claim exploitability from the pattern alone without confirming the behavior on the authorized target.

**The fix, in principle:** compare secrets with strict equality or `hash_equals()`, validate input types before comparing, and never compare a hash or signature loosely against a number, a boolean, or zero.

## 5. Testing Approach

1. Map the intended flow as a state machine: steps, states, and who is allowed to do what at each step.
2. For each step, ask: "What if I skip this? Repeat it? Do it out of order? Do it as a different role? Do it for a different object?"
3. Test one deviation at a time on your own test accounts, and record the request, the response, and the resulting state.
4. Confirm server-side — a 200 response is not a finding unless the state actually changed in a way the business rules forbid.
5. Never use a bypass to access another user's data, obtain goods or credit, or change a real account. Report the logic gap.
