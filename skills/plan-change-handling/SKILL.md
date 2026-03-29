---
name: plan-change-handling
description: 'Upgrades, downgrades, and proration handling for SaaS subscription plans. Use when: plan change handling, subscription upgrade, subscription downgrade, plan upgrade stripe, plan downgrade stripe, proration stripe, subscription proration, immediate upgrade, end of period downgrade, plan change proration, stripe subscription update price, subscription item update, proration behavior stripe, proration credit, upgrade billing, downgrade billing, plan tier change, switch plan, change subscription plan, seat change billing, plan change webhook, subscription schedule stripe, prorated invoice, change plan immediately, change plan next period.'
argument-hint: 'Describe your upgrade/downgrade UX (immediate vs end-of-period), whether you prorate, how seat or usage changes affect billing, and which plan change scenarios you need to handle (e.g., monthly → annual, Pro → Enterprise, seat increase).'
---

# Plan Change Handling

## When to Use

Invoke this skill when you need to:
- Implement upgrade and downgrade flows with correct Stripe API calls
- Configure proration behavior (immediate credit, none, or customer choice)
- Handle plan changes for monthly↔annual billing interval switches
- Manage seat quantity changes on per-seat plans
- Ensure entitlements update instantly on upgrade, at period end on downgrade
- Test proration calculations before charging

---

## Step 1 — Proration Strategy Decision

Choose the right proration behavior for your product:

| Scenario | Recommended `proration_behavior` | When to Use |
|---|---|---|
| Upgrade (higher tier) | `create_prorations` | Charge/credit immediately; fair to customer |
| Downgrade (lower tier) | `none` + apply at period end | Avoid retroactive credits; simpler billing |
| Annual → Monthly | `create_prorations` | Customer switching to flexibility |
| Monthly → Annual | `create_prorations` + collect now | Lock in commitment; charge difference |
| Seat increase | `create_prorations` | Immediate access to new seats |
| Seat decrease | `none` + apply at next renewal | Don't issue credits mid-period |

---

## Step 2 — Upgrade (Immediate)

Upgrade takes effect immediately; proration invoice generated:

```go
func (s *BillingService) Upgrade(ctx context.Context, tenantID, newStripePriceID string) error {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    // Find the SubscriptionItem to replace
    stripeSub, err := subscription.Get(sub.StripeSubscriptionID, nil)
    if err != nil {
        return fmt.Errorf("fetch stripe subscription: %w", err)
    }
    if len(stripeSub.Items.Data) == 0 {
        return errors.New("subscription has no items")
    }
    itemID := stripeSub.Items.Data[0].ID

    // Update the price with proration
    _, err = subscription.Update(sub.StripeSubscriptionID, &stripe.SubscriptionParams{
        Items: []*stripe.SubscriptionItemsParams{
            {
                ID:    stripe.String(itemID),
                Price: stripe.String(newStripePriceID),
            },
        },
        ProrationBehavior: stripe.String("create_prorations"),
    })
    if err != nil {
        return fmt.Errorf("upgrade subscription: %w", err)
    }

    // DB sync happens via webhook: customer.subscription.updated
    // Entitlements updated via webhook: see entitlement-synchronization skill
    return nil
}
```

**Preview proration before charging:**
```go
func (s *BillingService) PreviewUpgradeProration(ctx context.Context, tenantID, newStripePriceID string) (*ProrationPreview, error) {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil {
        return nil, err
    }

    stripeSub, err := subscription.Get(sub.StripeSubscriptionID, nil)
    if err != nil {
        return nil, err
    }

    prorationDate := time.Now().Unix()

    invoice, err := invoice.GetNext(&stripe.InvoiceParams{
        Customer:        stripe.String(sub.StripeCustomerID),
        Subscription:    stripe.String(sub.StripeSubscriptionID),
        SubscriptionProrationDate: stripe.Int64(prorationDate),
        SubscriptionItems: []*stripe.SubscriptionItemsParams{
            {
                ID:    stripe.String(stripeSub.Items.Data[0].ID),
                Price: stripe.String(newStripePriceID),
            },
        },
        SubscriptionProrationBehavior: stripe.String("create_prorations"),
    })
    if err != nil {
        return nil, fmt.Errorf("preview proration: %w", err)
    }

    return &ProrationPreview{
        AmountDueCents: invoice.AmountDue,
        Currency:       string(invoice.Currency),
        Lines:          mapInvoiceLines(invoice.Lines.Data),
    }, nil
}
```

---

## Step 3 — Downgrade (End of Period)

Downgrade applies at the end of the current billing period to avoid issuing prorated credits:

```go
func (s *BillingService) Downgrade(ctx context.Context, tenantID, newStripePriceID string) error {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    stripeSub, err := subscription.Get(sub.StripeSubscriptionID, nil)
    if err != nil {
        return fmt.Errorf("fetch stripe subscription: %w", err)
    }

    // Schedule the price change for the next billing period
    _, err = subscription.Update(sub.StripeSubscriptionID, &stripe.SubscriptionParams{
        Items: []*stripe.SubscriptionItemsParams{
            {
                ID:    stripe.String(stripeSub.Items.Data[0].ID),
                Price: stripe.String(newStripePriceID),
            },
        },
        ProrationBehavior: stripe.String("none"),    // No prorations
        BillingCycleAnchor: stripe.String("unchanged"), // Keep current period
    })
    if err != nil {
        return fmt.Errorf("schedule downgrade: %w", err)
    }

    // Store pending downgrade in DB for UI feedback
    return s.subRepo.SetPendingDowngrade(ctx, tenantID, newStripePriceID, sub.CurrentPeriodEnd)
}
```

**Track pending downgrades:**
```sql
ALTER TABLE subscriptions
    ADD COLUMN pending_price_id         TEXT,       -- price takes effect at next renewal
    ADD COLUMN pending_change_at        TIMESTAMPTZ; -- = current_period_end
```

---

## Step 4 — Seat Quantity Changes

For per-seat plans, update the quantity on the SubscriptionItem:

```go
func (s *BillingService) UpdateSeatCount(ctx context.Context, tenantID string, newSeatCount int) error {
    if newSeatCount < 1 {
        return errors.New("seat count must be at least 1")
    }

    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    stripeSub, err := subscription.Get(sub.StripeSubscriptionID, nil)
    if err != nil {
        return err
    }

    currentSeats := int(stripeSub.Items.Data[0].Quantity)
    prorationBehavior := "create_prorations" // default: prorate seat additions

    if newSeatCount < currentSeats {
        prorationBehavior = "none" // No credit for seat removals mid-period
    }

    _, err = subscription.Update(sub.StripeSubscriptionID, &stripe.SubscriptionParams{
        Items: []*stripe.SubscriptionItemsParams{
            {
                ID:       stripe.String(stripeSub.Items.Data[0].ID),
                Quantity: stripe.Int64(int64(newSeatCount)),
            },
        },
        ProrationBehavior: stripe.String(prorationBehavior),
    })
    if err != nil {
        return fmt.Errorf("update seat count: %w", err)
    }

    return s.subRepo.UpdateSeatCount(ctx, tenantID, newSeatCount)
}
```

---

## Step 5 — Billing Interval Switch (Monthly ↔ Annual)

Switching billing intervals requires changing the Price, as Stripe Prices are interval-locked:

```go
func (s *BillingService) SwitchToAnnual(ctx context.Context, tenantID string) error {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    // Find the annual price for the current plan
    plan, err := s.planRepo.GetByPriceID(ctx, sub.PriceID)
    if err != nil {
        return err
    }
    annualPrice, err := s.priceRepo.GetByPlanAndInterval(ctx, plan.ID, "year")
    if err != nil {
        return fmt.Errorf("no annual price for plan %s: %w", plan.Name, err)
    }

    stripeSub, err := subscription.Get(sub.StripeSubscriptionID, nil)
    if err != nil {
        return err
    }

    // Immediately switch + prorate: customer pays annual price minus remaining monthly credit
    _, err = subscription.Update(sub.StripeSubscriptionID, &stripe.SubscriptionParams{
        Items: []*stripe.SubscriptionItemsParams{
            {
                ID:    stripe.String(stripeSub.Items.Data[0].ID),
                Price: stripe.String(annualPrice.StripePriceID),
            },
        },
        ProrationBehavior:  stripe.String("create_prorations"),
        BillingCycleAnchor: stripe.String("now"), // Reset the billing cycle anchor to today
    })
    if err != nil {
        return fmt.Errorf("switch to annual: %w", err)
    }

    return nil
}
```

---

## Quality Checks

- [ ] Upgrades use `create_prorations` — customer receives credit for unused time on old plan
- [ ] Downgrades use `none` proration + scheduled price change — no retroactive credits issued
- [ ] Proration preview endpoint exists so frontend can show the customer the exact charge before confirming
- [ ] Seat increases prorate; seat decreases apply at next renewal with `none`
- [ ] Pending downgrades stored in DB so UI can inform the customer what changes at period end
- [ ] Billing interval switches use `BillingCycleAnchor: "now"` to reset the billing cycle
- [ ] Entitlements update immediately on upgrade — not deferred until webhook
- [ ] Plan change events are emitted for analytics (upgrade vs downgrade distinction tracked)
