# timer-progression

**Workflow:** P2P Auction Deal Lifecycle
**Domain:** P2P Marketplace
**Contract:** `timer-progression: A → B | P`

---

## A: Input

```yaml
lot_id:           string      # UUID of the lot
lot.status:       enum        # {OPEN, GOING_ONCE, GOING_TWICE}
lot.timer:        datetime    # Current auction end time
now:              datetime    # Current timestamp
new_bid_arrived:  bool        # Whether a bid arrived this tick
```

## F: Processing Logic

1. **Bid reset** — if new_bid_arrived AND lot.status IN [GOING_ONCE, GOING_TWICE]: reset lot.status = OPEN.
2. **Timer expiry check** — if now > lot.timer:
   - If lot.status = OPEN → set lot.status = GOING_ONCE, set lot.timer = now + 10s. Broadcast.
   - If lot.status = GOING_ONCE AND ¬new_bid_arrived → set lot.status = GOING_TWICE, set lot.timer = now + 10s. Broadcast.
   - If lot.status = GOING_TWICE AND ¬new_bid_arrived → trigger auction-close.
3. **Broadcast on every status change**.

## B: Output

```yaml
lot_id:          string      # UUID of the lot
status:          enum        # {OPEN, GOING_ONCE, GOING_TWICE, CLOSED}
timer:           datetime    # Updated timer (extended if status changed)
close_triggered: bool        # True if auction-close was called
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] If new_bid_arrived AND prior status IN [GOING_ONCE, GOING_TWICE] → status = OPEN (bid reset invariant)
- [ ] GOING_ONCE timer = now + 10s at moment of transition
- [ ] GOING_TWICE timer = now + 10s at moment of transition
- [ ] close_triggered = true ⟹ prior status was GOING_TWICE ∧ ¬new_bid_arrived
- [ ] Status transitions are one-directional absent bids: OPEN → GOING_ONCE → GOING_TWICE → CLOSED
- [ ] Timer is only ever extended, never shortened (anti-snipe invariant carries through)
