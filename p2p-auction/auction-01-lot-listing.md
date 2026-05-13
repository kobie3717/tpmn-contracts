# lot-listing

**Workflow:** P2P Auction Deal Lifecycle
**Domain:** P2P Marketplace
**Contract:** `lot-listing: A → B | P`

---

## A: Input

```yaml
seller_id:        string      # Unique identifier for the seller
title:            string      # Lot title
description:      string      # Lot description
starting_bid:     number      # Starting bid amount (USD)
increment:        number      # Minimum bid step (USD)
reserve_price:    number?     # Optional reserve price (USD), nullable
end_time:         datetime    # Auction end timestamp
```

## F: Processing Logic

1. **Identity gate** — verify seller.is_verified = true, seller.is_banned = false. Reject if either fails.
2. **Input validation** — starting_bid > 0, increment > 0, end_time > now. Reject with reason if fails.
3. **Content filter** — title must not contain banned keywords (weapons, drugs, firearms, explosives, counterfeit). Reject if match.
4. **Lot initialization** — create lot with: lot_id (uuid), seller, title, description, starting_bid, increment, reserve_price (null if not provided), current_bid=0, high_bidder=null, status=OPEN, timer=end_time, snipe_extension_count=0, snipe_extended=false, buyer_premium_pct=5, proxy_bids=[], bid_history=[], created_at=now.
5. **Notify** — broadcast new lot to all registered buyers.

## B: Output

```yaml
lot_id:                 string      # UUID for the lot
seller:                 string      # Seller ID
title:                  string      # Lot title
description:            string      # Lot description
starting_bid:           number      # Starting bid (USD)
increment:              number      # Minimum bid step (USD)
reserve_price:          number?     # Reserve price (USD), null if not provided
current_bid:            number      # Current high bid (initialized to 0)
high_bidder:            string?     # Current high bidder ID (null)
status:                 enum        # {OPEN}
timer:                  datetime    # Auction end time
snipe_extension_count:  number      # Anti-snipe extension count (0)
snipe_extended:         bool        # Whether timer was extended (false)
buyer_premium_pct:      number      # Buyer premium percentage (5)
created_at:             datetime    # Creation timestamp
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] lot_id is non-empty uuid
- [ ] status = OPEN
- [ ] current_bid = 0
- [ ] high_bidder = null
- [ ] snipe_extension_count = 0
- [ ] snipe_extended = false
- [ ] buyer_premium_pct = 5
- [ ] lot.seller = input seller_id
- [ ] lot.timer = input end_time
- [ ] If reserve_price provided → lot.reserve_price = input reserve_price
- [ ] If reserve_price not provided → lot.reserve_price = null
- [ ] eligible = true ⟹ seller.is_verified ∧ ¬seller.is_banned (identity gate invariant)
- [ ] eligible = true ⟹ starting_bid > 0 ∧ increment > 0 (input validation invariant)
- [ ] eligible = true ⟹ end_time > created_at (timer invariant)
- [ ] No SPT: lot represents THIS seller's specific item only
