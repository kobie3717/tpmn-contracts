# payment

**Workflow:** P2P Auction Deal Lifecycle  
**Domain:** P2P Marketplace  
**Contract:** `payment: A → B | P` *(MOCKUP — visualized only)*

---

## A: Input

```yaml
invoice_id:        string    # from auction-04
buyer:             string    # payer
total:             number    # amount due (USD)
payment_method:    enum      # {credit_card, bank_transfer, crypto, escrow}
payment_deadline:  datetime  # must be > now
```

## F: Processing Logic

*(Simulated — not executed in demo)*

1. **Deadline check** — verify payment submitted before payment_deadline. Auto-cancel if past.
2. **Payment processing** — buyer pays via payment provider (Stripe/PayFast/crypto).
3. **Webhook confirmation** — payment provider sends signed webhook. System validates signature.
4. **Escrow hold** — funds held in escrow (not released to seller until delivery confirmed).
5. **Status update** — invoice.status = PAID.
6. **Notify seller** — payment received, ship the item.

## B: Output

```yaml
payment_id:      string    # payment provider transaction ID
invoice_id:      string    # passthrough
buyer:           string    # payer
amount_paid:     number    # must equal invoice.total
payment_method:  enum      # method used
status:          enum      # {PAID}
escrow_held:     bool      # true (funds in escrow)
paid_at:         datetime  # payment confirmation timestamp
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

*(Mockup assertions — verified in UI only)*

- [ ] amount_paid = invoice.total (no underpayment)
- [ ] status = PAID
- [ ] escrow_held = true (funds not yet released — RULE_7)
- [ ] paid_at < payment_deadline (on-time payment)
- [ ] Escrow cannot be released while any dispute is OPEN (RULE_7)
- [ ] invoice.total immutable after PAID (RULE_8)
- [ ] Webhook signature validated before PAID status set
