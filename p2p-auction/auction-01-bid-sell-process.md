# bid-sell-process

**Workflow:** P2P Auction Deal Lifecycle  
**Domain:** P2P Marketplace  
**Contract:** `bid-sell-process: A → B | P`

---

## A: Input

```yaml
seller_id:        string    # registered seller identifier
title:            string    # item title (sanitized, no banned keywords)
description:      string    # item description (sanitized)
starting_bid:     number    # minimum first bid (USD, > 0)
increment:        number    # minimum bid step (USD, > 0)
reserve_price:    number?   # minimum acceptable price (null = no reserve)
end_time:         datetime  # scheduled auction close (must be > now)
buyer_id:         string    # bidder identifier (per bid submission)
bid_amount:       number    # bid amount (USD, per bid submission)
nonce:            string    # client-generated unique ID per bid (replay prevention)
proxy_max:        number?   # proxy bid ceiling (null = no proxy)
```

## F: Processing Logic

1. **Seller validation** — verify seller.is_verified = true, seller.is_banned = false. Reject if fails.
2. **Content filter** — title/description must not contain banned keywords (weapons, drugs, firearms, explosives, counterfeit). Sanitize against XSS/SQL injection (RULE_13).
3. **Lot initialization** — create lot: {lot_id, seller, title, description, starting_bid, increment, reserve_price, current_bid=0, high_bidder=null, status=OPEN, timer=end_time, snipe_extension_count=0, snipe_extended=false, buyer_premium_pct=5, proxy_bids=[], bid_history=[]}.
4. **Bid validation** — for each bid: verify lot.status IN [OPEN, GOING_ONCE, GOING_TWICE], buyer ≠ seller (RULE_2), buyer not banned (RULE_3), nonce unique (RULE_12), first bid ≥ starting_bid OR subsequent bid ≥ current_bid + increment (RULE_1), now ≤ timer.
5. **Anti-snipe** — if (timer - now) ≤ 30s AND snipe_extension_count < 3: timer += 120s, snipe_extension_count++, snipe_extended=true. If count ≥ 3: no extension (RULE_11).
6. **Proxy auto-bid** — fire proxy if proxy.max_amount ≥ current_bid + increment AND proxy.buyer ≠ current_bidder.
7. **Timer progression** — OPEN→GOING_ONCE (10s no bid)→GOING_TWICE (10s no bid)→close.
8. **Auction close** — UNSOLD if no bids. PENDING_SELLER if reserve not met (24h seller decision). SOLD: set winner=high_bidder, hammer_price=current_bid.

## B: Output

```yaml
lot_id:                 string    # uuid assigned at creation
seller:                 string    # seller_id
title:                  string    # sanitized title
description:            string    # sanitized description
starting_bid:           number    # as input
increment:              number    # as input
reserve_price:          number?   # as input or null
current_bid:            number    # final bid amount
high_bidder:            string?   # winner (null if UNSOLD)
status:                 enum      # {SOLD, UNSOLD, PENDING_SELLER}
hammer_price:           number?   # = current_bid at close (null if UNSOLD)
winner:                 string?   # = high_bidder at close (null if UNSOLD)
buyer_premium_pct:      number    # 5 (platform constant)
snipe_extension_count:  number    # 0–3
snipe_extended:         bool      # true if any extension fired
bid_history:            object[]  # [{buyer_id, amount, timestamp, nonce}]
closed_at:              datetime  # auction close timestamp
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] status ∈ {SOLD, UNSOLD, PENDING_SELLER}
- [ ] status = SOLD ⟹ winner ≠ null ∧ hammer_price > 0
- [ ] status = SOLD ⟹ winner ≠ seller (self-bid invariant — RULE_2)
- [ ] hammer_price = current_bid at close (immutable — RULE_6)
- [ ] snipe_extension_count ≤ 3 (hard cap — RULE_11)
- [ ] Each nonce in bid_history is unique (replay prevention — RULE_12)
- [ ] buyer_premium_pct = 5 (platform constant)
- [ ] title and description contain no banned keywords and no injection patterns (RULE_13)
- [ ] If reserve_price set AND current_bid < reserve_price → status ≠ SOLD without seller acceptance (RULE_5)
- [ ] Terminal states SOLD/UNSOLD are irreversible (RULE_9)
- [ ] No SPT: auction outcome is specific to this lot only
