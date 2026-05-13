# invoice-check

**Workflow:** P2P Auction Deal Lifecycle  
**Domain:** P2P Marketplace  
**Contract:** `invoice-check: A → B | P`

---

## A: Input

```yaml
invoice_id:          string    # from auction-03
buyer:               string    # winner confirming details
hammer_price:        number    # must match auction-03 output
buyer_premium:       number    # must match auction-03 output
total:               number    # must match auction-03 output
shipping_method:     enum      # {standard, express, overnight, pickup}
delivery_address:    object    # {street, city, postal_code, country}
confirmed:           bool      # buyer confirms all details
```

## F: Processing Logic

1. **Amount integrity** — verify hammer_price, buyer_premium, total match invoice record exactly (no tampering — RULE_8).
2. **Total arithmetic** — verify total = hammer_price + buyer_premium.
3. **Shipping validation** — verify shipping_method is from approved list. Validate delivery_address has all required fields.
4. **Buyer confirmation** — require confirmed = true to proceed.
5. **Status update** — set invoice.status = PENDING_PAYMENT.
6. **Payment link** — send payment link to buyer. 72h payment deadline.

## B: Output

```yaml
invoice_id:           string    # passthrough
hammer_price:         number    # verified amount
buyer_premium:        number    # verified amount
total:                number    # verified amount
shipping_method:      enum      # selected method
delivery_address:     object    # {street, city, postal_code, country}
status:               enum      # {PENDING_PAYMENT}
payment_deadline:     datetime  # now + 72h
confirmed:            bool      # true
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] total = hammer_price + buyer_premium (arithmetic re-verified)
- [ ] hammer_price unchanged from invoice creation (RULE_8: no post-creation edit)
- [ ] buyer_premium unchanged from invoice creation (RULE_8)
- [ ] total unchanged from invoice creation (RULE_8)
- [ ] status = PENDING_PAYMENT
- [ ] confirmed = true (buyer explicitly confirmed)
- [ ] shipping_method ∈ {standard, express, overnight, pickup}
- [ ] delivery_address.postal_code is non-empty
- [ ] payment_deadline = confirmation_time + 72h exactly
- [ ] No SPT: invoice amounts are specific to this transaction only
