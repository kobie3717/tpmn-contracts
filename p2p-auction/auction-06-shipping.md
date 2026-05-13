# shipping

**Workflow:** P2P Auction Deal Lifecycle  
**Domain:** P2P Marketplace  
**Contract:** `shipping: A → B | P` *(MOCKUP — visualized only)*

---

## A: Input

```yaml
invoice_id:        string    # from auction-05
seller:            string    # shipper
tracking_number:   string    # non-empty alphanumeric carrier tracking ID
carrier:           enum      # {DHL, FedEx, UPS, USPS, PostNet, other_approved}
delivery_address:  object    # from auction-04 {street, city, postal_code, country}
payment_status:    enum      # must be PAID
```

## F: Processing Logic

*(Simulated — not executed in demo)*

1. **Payment gate** — verify invoice.status = PAID before allowing shipping update.
2. **Tracking validation** — tracking_number must be non-empty alphanumeric. carrier must be in approved_carriers list.
3. **Status update** — invoice.status = SHIPPED.
4. **Notify buyer** — send tracking number + carrier + estimated delivery.
5. **Delivery monitoring** — system polls carrier API for delivery confirmation.
6. **Auto-confirm** — if no dispute raised within 5 days of SHIPPED: buyer auto-confirms receipt → invoice.status = DELIVERED → escrow released to seller.
7. **Manual confirm** — buyer can click "Confirm Received" at any time → same flow.

## B: Output

```yaml
invoice_id:        string    # passthrough
tracking_number:   string    # validated tracking ID
carrier:           enum      # validated carrier
status:            enum      # {SHIPPED, DELIVERED}
shipped_at:        datetime  # when SHIPPED status set
delivered_at:      datetime? # when DELIVERED confirmed (null until confirmed)
escrow_released:   bool      # true only when status = DELIVERED
auto_confirm_at:   datetime  # shipped_at + 5 days (auto-confirm deadline)
```

## P: Postcondition Checklist

AI verification checks — all must pass for CONTRACT satisfaction:

*(Mockup assertions — verified in UI only)*

- [ ] tracking_number is non-empty alphanumeric
- [ ] carrier ∈ approved_carriers list
- [ ] status = SHIPPED before DELIVERED (one-directional)
- [ ] escrow_released = true ⟹ status = DELIVERED
- [ ] escrow_released = false while any dispute is OPEN (RULE_7)
- [ ] invoice.status = PAID verified before SHIPPED status set
- [ ] auto_confirm_at = shipped_at + 5 days exactly
- [ ] Seller identity on shipment matches lot.seller (RULE_10)
