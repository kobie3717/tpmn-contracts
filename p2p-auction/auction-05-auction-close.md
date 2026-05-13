# auction-close

**Workflow:** P2P Auction Deal Lifecycle
**Domain:** P2P Marketplace
**Contract:** `auction-close: A → B | P`

---

## A: Input

```yaml
lot_id:              string      # UUID of the lot
lot.high_bidder:     string?     # Current high bidder ID (null if no bids)
lot.current_bid:     number      # Current high bid amount (USD)
lot.reserve_price:   number?     # Reserve price (USD), null if no reserve
lot.seller:          string      # Seller ID
lot.buyer_premium_pct: number    # Buyer premium percentage (5)
lot.hammer_price:    number?     # Final sale price (null until close)
```

## F: Processing Logic

1. **No-bid close** — if lot.high_bidder = null: set lot.status = UNSOLD. Notify seller. END.
2. **Reserve not met** — if lot.reserve_price set AND lot.current_bid < lot.reserve_price: set lot.status = PENDING_SELLER. Start 24h timeout: if seller does not respond → auto-reject → lot.status = UNSOLD. If seller accepts → proceed to step 3. If seller rejects → lot.status = UNSOLD.
3. **Assert winner** — ASSERT lot.high_bidder ≠ null.
4. **Finalize** — set lot.status = SOLD, lot.winner = lot.high_bidder, lot.hammer_price = lot.current_bid.
5. **Generate invoice** — create invoice record (triggers auction-06-invoice-generation).
6. **Notify winner and seller**.

## B: Output

```yaml
lot_id:       string      # UUID of the lot
status:       enum        # {SOLD, UNSOLD, PENDING_SELLER}
winner:       string?     # Winner buyer ID (null if UNSOLD)
hammer_price: number?     # Final sale price (USD, null if UNSOLD)
invoice_id:   string?     # Invoice UUID (null if not SOLD)
closed_at:    datetime    # Auction close timestamp
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] status ∈ {SOLD, UNSOLD, PENDING_SELLER}
- [ ] status = UNSOLD ⟹ high_bidder = null OR reserve not met AND seller rejected/timed out
- [ ] status = SOLD ⟹ winner ≠ null ∧ hammer_price > 0 ∧ invoice_id ≠ null
- [ ] status = SOLD ⟹ winner ≠ lot.seller (self-bid invariant preserved through close)
- [ ] hammer_price is immutable after SOLD (RULE_6: no post-close modification)
- [ ] status is terminal: SOLD and UNSOLD cannot transition to OPEN (RULE_9)
- [ ] If reserve_price set AND current_bid < reserve_price → status ≠ SOLD without seller acceptance
