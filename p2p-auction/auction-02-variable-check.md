# variable-check

**Workflow:** P2P Auction Deal Lifecycle  
**Domain:** P2P Marketplace  
**Contract:** `variable-check: A → B | P`

---

## A: Input

```yaml
lot_id:                 string    # from auction-01 output
status:                 enum      # {SOLD, UNSOLD, PENDING_SELLER}
seller:                 string    # seller identifier
winner:                 string?   # buyer identifier (null if UNSOLD)
hammer_price:           number?   # final sale price (null if UNSOLD)
buyer_premium_pct:      number    # must be 5
snipe_extension_count:  number    # must be 0–3
bid_history:            object[]  # all bids with nonces
reserve_price:          number?   # original reserve (null if none)
starting_bid:           number    # original starting bid
increment:              number    # original increment
title:                  string    # lot title
description:            string    # lot description
closed_at:              datetime  # close timestamp
```

## F: Processing Logic

1. **RULE_1 check** — verify each bid in bid_history: first bid ≥ starting_bid; each subsequent bid ≥ previous_bid + increment.
2. **RULE_2 check** — verify winner ≠ seller (no self-sale).
3. **RULE_3 check** — verify no bid from banned user in bid_history.
4. **RULE_5 check** — verify reserve_price unchanged from lot creation to close.
5. **RULE_6 check** — verify hammer_price = current_bid at close moment (immutable).
6. **RULE_9 check** — verify no state transition from SOLD/UNSOLD back to OPEN.
7. **RULE_11 check** — verify snipe_extension_count ≤ 3.
8. **RULE_12 check** — verify all nonces in bid_history are unique (no duplicates).
9. **RULE_13 check** — verify title/description pass XSS and SQL injection scan.
10. **Premium check** — verify buyer_premium_pct = 5.
11. **GEM² truth-filter** — route lot summary through GEM² gate: POST truth-filter with lot context. Parse: truth_score, verdict (ALLOW/REVIEW/BLOCK), spt_issues.
12. **Verdict** — ALLOW if all 10 rule checks pass AND truth_score ≥ 70. REVIEW if any soft flag. BLOCK if any hard rule fails OR truth_score < 40.

## B: Output

```yaml
lot_id:           string    # passthrough
all_rules_pass:   bool      # true if RULE_1–13 all satisfied
failed_rules:     string[]  # list of failed rule IDs (empty if all pass)
truth_score:      number    # GEM² score 0–100
verdict:          enum      # {ALLOW, REVIEW, BLOCK}
spt_issues:       string[]  # SPT flags from GEM² (empty if none)
checked_at:       datetime  # verification timestamp
proceed:          bool      # true = advance to invoicing; false = hold
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] all_rules_pass = true ⟹ failed_rules = [] (empty list)
- [ ] all_rules_pass = false ⟹ failed_rules contains at least one rule ID
- [ ] verdict = ALLOW ⟹ proceed = true
- [ ] verdict = BLOCK ⟹ proceed = false
- [ ] verdict = REVIEW ⟹ proceed = false (manual review required)
- [ ] truth_score ∈ [0, 100]
- [ ] If RULE_2 fails (winner = seller) → verdict = BLOCK regardless of truth_score
- [ ] If RULE_6 fails (hammer_price tampered) → verdict = BLOCK
- [ ] If RULE_11 fails (snipe_extension_count > 3) → verdict = BLOCK
- [ ] checked_at > closed_at (verification happens after close)
- [ ] No SPT: verdict applies to THIS lot only — no generalization across lots
