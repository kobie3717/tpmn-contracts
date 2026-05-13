# customer-relations

**Workflow:** P2P Auction Deal Lifecycle  
**Domain:** P2P Marketplace  
**Contract:** `customer-relations: A → B | P`

---

## A: Input

```yaml
invoice_id:         string    # invoice being addressed
raised_by:          string    # buyer or seller identifier
interaction_type:   enum      # {dispute, feedback, follow_up, refund_request}
dispute_type:       enum?     # {NON_PAYMENT, ITEM_NOT_AS_DESCRIBED, NON_DELIVERY, DAMAGED_ITEM} (null if feedback/follow_up)
reason:             string    # sanitized text (max 2000 chars)
invoice_status:     enum      # must be IN {PAID, SHIPPED, DELIVERED} for dispute
raised_at:          datetime  # submission timestamp
```

## F: Processing Logic

1. **Content gate** — sanitize reason against XSS/SQL injection (RULE_13).
2. **Window check** (disputes only) — verify raised_at within 30 days of most recent invoice event (PAID/SHIPPED/DELIVERED date). Reject if outside window.
3. **Escrow freeze** (disputes only) — immediately freeze escrow on OPEN dispute. Funds cannot be released while dispute is active (RULE_7).
4. **Create record** — {record_id: uuid, invoice_id, raised_by, type: interaction_type, reason, status: OPEN, created_at: now}.
5. **Route** — disputes → admin queue. Feedback → CRM system. Follow-up → scheduled outreach. Refund request → admin queue.
6. **Admin resolution** (disputes) — admin reviews evidence: buyer wins → refund + release escrow to buyer, status=RESOLVED. Seller wins → release escrow to seller, status=RESOLVED. Escalate → status=ESCALATED.
7. **CSAT collection** — after resolution: send satisfaction survey to both parties.

## B: Output

```yaml
record_id:          string    # uuid
invoice_id:         string    # passthrough
raised_by:          string    # initiating party
interaction_type:   enum      # {dispute, feedback, follow_up, refund_request}
status:             enum      # {OPEN, RESOLVED, ESCALATED, CLOSED}
escrow_frozen:      bool      # true if dispute OPEN or ESCALATED
resolution:         string?   # outcome description (null until resolved)
refund_issued:      bool?     # true if buyer won dispute
escrow_released_to: string?   # 'buyer' or 'seller' (null until resolved)
csat_sent:          bool      # true if survey sent post-resolution
closed_at:          datetime? # null until RESOLVED or CLOSED
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] record_id is non-empty uuid
- [ ] reason is sanitized (no XSS, no SQL injection — RULE_13)
- [ ] If interaction_type = dispute → escrow_frozen = true immediately (RULE_7)
- [ ] Escrow release only occurs when status = RESOLVED (never while OPEN or ESCALATED — RULE_7)
- [ ] If dispute raised → invoice_status ∈ {PAID, SHIPPED, DELIVERED} at time of submission
- [ ] raised_at within 30 days of most recent invoice event (dispute window)
- [ ] If status = RESOLVED ∧ buyer wins → refund_issued = true ∧ escrow_released_to = 'buyer'
- [ ] If status = RESOLVED ∧ seller wins → refund_issued = false ∧ escrow_released_to = 'seller'
- [ ] csat_sent = true ⟹ status ∈ {RESOLVED, CLOSED}
- [ ] No SPT: resolution is specific to THIS invoice and parties only — no generalization
