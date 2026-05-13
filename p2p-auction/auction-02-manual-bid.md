# manual-bid

**Workflow:** P2P Auction Deal Lifecycle
**Domain:** P2P Marketplace
**Contract:** `manual-bid: A → B | P`

---

## A: Input

```yaml
lot_id:     string      # UUID of the lot
buyer_id:   string      # Unique identifier for the buyer
amount:     number      # Bid amount (USD)
nonce:      string      # Client-generated unique ID for replay prevention
```

## F: Processing Logic

1. **Lot gate** — verify lot exists, lot.status IN [OPEN, GOING_ONCE, GOING_TWICE]. Reject if not.
2. **Buyer gate** — verify buyer.is_verified = true, buyer.is_banned = false. Reject if fails.
3. **Self-bid gate** — verify buyer_id ≠ lot.seller. Reject with "Seller cannot bid on own lot".
4. **Replay gate** — verify nonce not in processed_nonces. Reject if duplicate.
5. **Timer gate** — verify now ≤ lot.timer. Reject if expired.
6. **Amount gate** — if lot.high_bidder = null: amount ≥ lot.starting_bid (first bid). Else: amount ≥ lot.current_bid + lot.increment (subsequent bid). Reject with minimum required amount if fails.
7. **State update** — record nonce, update lot.current_bid = amount, lot.high_bidder = buyer_id. If status was GOING_ONCE or GOING_TWICE → reset to OPEN.
8. **Anti-snipe check** — if (lot.timer - now) ≤ 30s AND lot.snipe_extension_count < 3: lot.timer += 120s, lot.snipe_extension_count++, lot.snipe_extended = true. Else if count ≥ 3: no extension.
9. **Proxy trigger** — for each active proxy where proxy.buyer_id ≠ buyer_id AND proxy.max_amount ≥ lot.current_bid + lot.increment: auto-place bid at lot.current_bid + lot.increment.
10. **Notifications** — notify previous high_bidder (outbid), notify seller (new high bid).

## B: Output

```yaml
lot_id:                 string      # UUID of the lot
current_bid:            number      # Updated current high bid (USD)
high_bidder:            string      # Buyer ID of current high bidder
status:                 enum        # {OPEN, GOING_ONCE, GOING_TWICE}
timer:                  datetime    # Auction end time (may be extended)
snipe_extended:         bool        # Whether timer was extended this bid
snipe_extension_count:  number      # Total anti-snipe extensions applied
bid_record:             object      # {buyer_id, amount, timestamp, nonce}
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] lot.current_bid = input amount
- [ ] lot.high_bidder = input buyer_id
- [ ] nonce recorded in processed_nonces (replay prevention active)
- [ ] If previous status was GOING_ONCE or GOING_TWICE → lot.status = OPEN
- [ ] If (timer - now) was ≤ 30s at bid time AND prior snipe_extension_count < 3 → lot.timer extended by exactly 120s
- [ ] If prior snipe_extension_count ≥ 3 → lot.timer unchanged (max extensions enforced)
- [ ] snipe_extension_count ≤ 3 (hard cap invariant)
- [ ] lot.seller ≠ lot.high_bidder (self-bid invariant)
- [ ] No SPT: bid amount and outcome are specific to THIS lot and buyer
