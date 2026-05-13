# dispute-handling

**Workflow:** P2P Auction Deal Lifecycle
**Domain:** P2P Marketplace
**Contract:** `dispute-handling: A → B | P`

---

## A: Input

```yaml
invoice_id:    string      # UUID of the invoice
raised_by:     string      # Buyer ID or seller ID who raised dispute
dispute_type:  enum        # {NON_PAYMENT, ITEM_NOT_AS_DESCRIBED, NON_DELIVERY, DAMAGED_ITEM}
reason:        string      # Sanitized text description (max 2000 chars)
raised_at:     datetime    # Dispute raised timestamp
```

## F: Processing Logic

1. **Invoice gate** — verify invoice exists and invoice.status IN [PAID, SHIPPED, DELIVERED].
2. **Window gate** — verify raised_at is within 30 days of the most recent invoice event date (PAID date, SHIPPED date, or DELIVERED date — whichever is most recent). Reject if outside window.
3. **Content gate** — sanitize reason against SQL injection and XSS patterns (RULE_13).
4. **Escrow freeze** — immediately freeze escrow (no funds can be released while dispute is OPEN).
5. **Create dispute** — {dispute_id: uuid, invoice_id, raised_by, type: dispute_type, reason, status: OPEN, created_at: now}.
6. **Notify admin** — admin receives dispute for review.
7. **Admin resolution** — admin reviews evidence: buyer wins → refund buyer, status=RESOLVED. Seller wins → release escrow, status=RESOLVED. Escalate → status=ESCALATED → senior admin.

## B: Output

```yaml
dispute_id:     string      # UUID for the dispute
invoice_id:     string      # UUID of the invoice
raised_by:      string      # Buyer ID or seller ID who raised dispute
type:           enum        # {NON_PAYMENT, ITEM_NOT_AS_DESCRIBED, NON_DELIVERY, DAMAGED_ITEM}
reason:         string      # Sanitized text description
status:         enum        # {OPEN, RESOLVED, ESCALATED}
escrow_frozen:  bool        # Escrow freeze status (true)
created_at:     datetime    # Dispute creation timestamp
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

- [ ] dispute_id is non-empty uuid
- [ ] escrow_frozen = true immediately on OPEN (RULE_7: escrow cannot be released during active dispute)
- [ ] raised_at within 30 days of most recent invoice event
- [ ] reason is sanitized (no SQL injection, no XSS — RULE_13)
- [ ] status = OPEN at creation
- [ ] invoice.status IN [PAID, SHIPPED, DELIVERED] at time of dispute (cannot dispute PENDING_PAYMENT)
- [ ] If status = RESOLVED ∧ buyer_wins → refund issued ∧ escrow released to buyer
- [ ] If status = RESOLVED ∧ seller_wins → escrow released to seller ∧ no refund
- [ ] Escrow release only occurs when dispute status = RESOLVED (never while OPEN or ESCALATED)
- [ ] No SPT: dispute outcome is specific to THIS invoice and parties only
