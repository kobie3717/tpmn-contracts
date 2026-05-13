# proxy-bid

**Workflow:** P2P Auction Deal Lifecycle
**Domain:** P2P Marketplace
**Contract:** `proxy-bid: A → B | P`

---

## A: Input

```yaml
lot_id:      string      # UUID of the lot
buyer_id:    string      # Unique identifier for the buyer
max_amount:  number      # Maximum amount buyer will auto-bid up to (USD)
```

## F: Processing Logic

1. **Lot gate** — verify lot exists and lot.status IN [OPEN, GOING_ONCE, GOING_TWICE].
2. **Amount gate** — max_amount ≥ lot.starting_bid. Reject if below.
3. **Tie resolution** — if existing proxy for same buyer: update max_amount. If two different buyers register proxies with equal max_amount: first registered (by timestamp) wins; second proxy.active = false (deactivated).
4. **Store proxy** — {buyer_id, max_amount, active: true, registered_at: now}.
5. **Immediate trigger** — if lot.current_bid > 0 AND max_amount ≥ lot.current_bid + lot.increment AND buyer_id ≠ lot.high_bidder: auto-fire bid at lot.current_bid + lot.increment (runs manual-bid F steps 7-10).
6. **Future trigger** — on each subsequent manual bid by another buyer: auto-fire if proxy conditions met.

## B: Output

```yaml
lot_id:        string      # UUID of the lot
buyer_id:      string      # Buyer ID who registered the proxy
max_amount:    number      # Maximum auto-bid amount (USD)
active:        bool        # Whether proxy is active
registered_at: datetime    # Proxy registration timestamp
proxy_id:      string      # UUID for the proxy bid
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] proxy stored in lot.proxy_bids
- [ ] max_amount ≥ lot.starting_bid (amount gate invariant)
- [ ] If two proxies same max_amount: earlier registered_at proxy has active=true, later has active=false
- [ ] proxy.buyer_id ≠ lot.seller (proxy buyer cannot be seller)
- [ ] No proxy fires bid on behalf of its own buyer_id against itself
- [ ] active proxies only fire when proxy.max_amount ≥ lot.current_bid + lot.increment
