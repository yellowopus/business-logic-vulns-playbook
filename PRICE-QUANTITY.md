# Price & Quantity Manipulation

Price and quantity manipulation is the family of flaws where the server accepts a money or amount value from the client without validating it as a business value. The interface may show the right price, while the request that actually sets the price is editable — and the server trusts it.

All examples below use generic placeholders (`shop.example.com`, invented item IDs). Test only on targets you own or are authorized to test, with your own test accounts, and never manipulate a real order or payment.

## 1. Client-Side Price Trust

**The pattern:** the price, discount, or total is sent by the browser (in a form field, JSON body, or hidden parameter) and the server uses it as-is instead of looking the price up from its own catalog.

**What to check:**

- Does the checkout or cart request contain a `price`, `amount`, `total`, `discount`, or `shipping_fee` field? Change it in your own test request and see whether the server recalculates the total from its own data or accepts your value.
- Does the server trust a client-sent status such as `payment_status: "paid"` or `success: true`? A status flag is not proof of payment; the server should verify payment with the payment provider.
- Is an item identified only by a client-sent ID? Try swapping an item ID for a cheaper item's ID at checkout while keeping the expensive item in the cart, and see which price the server charges.
- Is a coupon or discount a client-sent field (`coupon_id`, `discount`, `free_shipping: true`)? Check whether the server validates the coupon's existence, eligibility, and expiry — or applies whatever value arrives.

**The fix, in principle:** the server must derive price, discount, and total from its own catalog and rules, using only the item ID and quantity from the client — and it must verify payment status with the payment provider, never with a client-sent flag.

## 2. Quantity Tricks: Negative, Zero, and Fractional Values

**Negative quantity.** If the server validates "amount > 0" but not quantity, a negative quantity can turn a purchase into a credit: the total goes negative, or the account is credited instead of charged. Test with your own test account using small values such as `quantity: -1`, and check the resulting balance and order state rather than assuming the response means success.

**Zero quantity.** A zero-quantity line can create an order with no items but a valid order record, or keep a discount attached while the items are gone. Check whether a zero-quantity order can proceed to confirmation.

**Fractional quantity.** If the server multiplies `unit_price × quantity` without requiring quantity to be a whole number, a fractional quantity (for example `0.02`) can pay a fraction of the price while the order still ships the full item. This is a real, documented pattern on e-commerce and food-delivery flows. The check is simple: does the server reject non-integer quantities for items sold by the unit?

**Very large quantity.** Extremely large quantities can overflow a 32-bit integer or exceed floating-point precision, wrapping the total to zero or a negative number. Treat overflow tests cautiously — they can crash a payment service — and only run them where authorized, with the program owner's knowledge if the target is shared.

**The fix, in principle:** validate quantity server-side as a positive integer within a sane range, and validate price and total by recalculation.

## 3. Currency and Rounding Issues

**Currency swap.** If the request carries a currency code, check whether the server validates it against the price. A numeric amount paired with a different currency than the one the price was quoted in can be charged at the wrong value.

**Rounding.** Money should be handled in the smallest currency unit (cents) as integers, not as floating-point decimals. Rounding flaws appear where the charge and the credit round differently — for example, a payment gateway rounding a charge down while the wallet credits the amount rounded up — so a small, repeatable gain appears on every cycle. Also check per-line rounding: a per-item price with more decimals than the currency supports can round to zero when multiplied across many items.

**Exchange-rate lag.** Where deposits and withdrawals can use different currencies, a rate that changes between deposit and withdrawal can be exploited. The design fix is to lock the rate at transaction time and to settle in a single currency per transaction.

**The fix, in principle:** integer cents everywhere, one rounding rule applied once at the end, rates locked at transaction time.

## 4. Cart and Checkout Logic

The cart and the checkout are often two separate checks that do not agree with each other. Test the seams:

- **Apply, then change.** Apply a coupon or discount, then remove the item, change the item, or change the quantity — does the discount stay attached to a different order than the one it was validated for?
- **Stacking.** Can two coupons, a referral credit, and loyalty points combine to more than 100% of the total? Check whether the server enforces a stacking policy or simply adds every discount it receives.
- **Fixed-amount coupon on a cheap item.** A fixed-amount coupon (for example, $5 off) applied to an item worth less than the coupon can produce a negative total or a credit. Check whether the server caps the discount at the order total.
- **Dropped required fields.** If a request omits a field that triggers payment (for example, the list of paid items), does the server fall back to treating the order as free? A missing field should be an error, not a free tier.
- **State changes.** Can an order's status be changed by the client — for example, setting `status` to `refunded` or `shipped` directly? Can a completed transaction ID be reused for a second purchase? Can a limit be reset by cancelling a transaction that already counted toward it?
- **Free shipping and fees.** Negative shipping or fee fields can reduce the total the same way a negative price does. Check every money field in the request, not just the headline price.

## 5. Testing Approach

1. Map the cart → checkout → payment → confirmation flow and note every money-related field in each request.
2. Change one field at a time on your own test account, with small, reversible values, and record the response **and** the resulting order/balance state.
3. Confirm server-side: a changed total in a response is only a finding if the order, invoice, or balance actually reflects it.
4. Never complete a manipulated real payment, and never use a finding to obtain goods or credit. Report the logic gap with the evidence you have.

## Remediation Concepts (for Developers and Reviewers)

- Never trust price, discount, total, currency, or payment status from the client. Recalculate from server-side catalog data and verify payment with the provider.
- Validate quantity as a positive integer in a sane range; validate amounts as integer cents with explicit minimum and maximum.
- Cap discounts at the order total; enforce stacking rules server-side.
- Treat a missing required field as an error, not as a default to the free/cheap path.
- Make order-status transitions server-controlled, with a state machine that only allows legal transitions (for example, `paid → shipped`, never `pending → refunded` by the client).
