# invoice-generation

**Workflow:** P2P Auction Deal Lifecycle
**Domain:** P2P Marketplace
**Contract:** `invoice-generation: A → B | P`

---

## A: Input

```yaml
lot_id:             string      # UUID of the lot
winner:             string      # Winner buyer ID
seller:             string      # Seller ID
hammer_price:       number      # Final sale price (USD)
buyer_premium_pct:  number      # Buyer premium percentage (5)
```

## F: Processing Logic

1. **Calculate buyer_premium** — buyer_premium = round(hammer_price × buyer_premium_pct / 100).
2. **Calculate total** — total = round(hammer_price × (1 + buyer_premium_pct / 100)).
3. **Create invoice** — {invoice_id: uuid, lot_id, buyer: winner, seller, hammer_price, buyer_premium, total, status: PENDING_PAYMENT, created_at: now}.
4. **Send checkout link to winner** — winner has 72h to complete checkout or invoice is CANCELLED.

## B: Output

```yaml
invoice_id:         string      # UUID for the invoice
lot_id:             string      # UUID of the lot
buyer:              string      # Winner buyer ID
seller:             string      # Seller ID
hammer_price:       number      # Final sale price (USD)
buyer_premium:      number      # Buyer premium amount (USD, 5% of hammer_price rounded)
total:              number      # Total amount due (USD, hammer_price + buyer_premium)
status:             enum        # {PENDING_PAYMENT}
created_at:         datetime    # Invoice creation timestamp
checkout_deadline:  datetime    # Payment deadline (created_at + 72h)
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] invoice_id is non-empty uuid
- [ ] buyer_premium = round(hammer_price × buyer_premium_pct / 100) (arithmetic correctness)
- [ ] total = hammer_price + buyer_premium (arithmetic correctness)
- [ ] total > hammer_price (premium is additive)
- [ ] status = PENDING_PAYMENT at creation
- [ ] buyer ≠ seller (self-trade invariant)
- [ ] invoice total is immutable after status reaches PAID (RULE_8)
- [ ] checkout_deadline = created_at + 72h exactly
