# invoicing

**Workflow:** P2P Auction Deal Lifecycle  
**Domain:** P2P Marketplace  
**Contract:** `invoicing: A → B | P`

---

## A: Input

```yaml
lot_id:             string    # from auction-01
winner:             string    # buyer identifier
seller:             string    # seller identifier
hammer_price:       number    # final bid price (USD)
buyer_premium_pct:  number    # 5 (platform constant)
proceed:            bool      # must be true (from auction-02 variable-check)
```

## F: Processing Logic

1. **Gate check** — verify proceed = true from variable-check. Halt if false.
2. **Calculate buyer_premium** — buyer_premium = round(hammer_price × buyer_premium_pct / 100).
3. **Calculate total** — total = hammer_price + buyer_premium.
4. **Create invoice** — {invoice_id: uuid, lot_id, buyer: winner, seller, hammer_price, buyer_premium, total, status: PENDING_FINALIZATION, created_at: now}.
5. **Send checkout link** — send to winner. 72h deadline: if no checkout → CANCELLED, lot may be re-listed.

## B: Output

```yaml
invoice_id:         string    # uuid
lot_id:             string    # passthrough
buyer:              string    # winner
seller:             string    # seller
hammer_price:       number    # passthrough
buyer_premium:      number    # round(hammer_price × 0.05)
total:              number    # hammer_price + buyer_premium
status:             enum      # {PENDING_FINALIZATION}
created_at:         datetime  # invoice creation timestamp
checkout_deadline:  datetime  # created_at + 72h
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] invoice_id is non-empty uuid
- [ ] buyer_premium = round(hammer_price × buyer_premium_pct / 100) (arithmetic correctness)
- [ ] total = hammer_price + buyer_premium (arithmetic correctness)
- [ ] total > hammer_price (premium is positive and additive)
- [ ] status = PENDING_FINALIZATION at creation
- [ ] buyer ≠ seller (self-trade invariant)
- [ ] checkout_deadline = created_at + 72h exactly
- [ ] proceed = true was verified before invoice creation
- [ ] Invoice only created when auction-02 verdict = ALLOW
