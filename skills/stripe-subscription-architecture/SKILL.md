---
name: stripe-subscription-architecture
description: 'Subscription modeling using Stripe billing for SaaS products. Use when: stripe subscription architecture, subscription modeling, Stripe Product Price, Stripe Customer object, subscription database schema, stripe tenant mapping, stripe plan modeling, recurring billing setup, stripe subscription creation, Stripe billing setup, subscription data model, stripe price tiers, stripe billing interval, stripe trial setup, stripe coupon, stripe payment method, stripe billing cycle, create subscription stripe, stripe checkout session, subscription entity design, stripe customer ID, multi-plan subscription, subscription metadata.'
argument-hint: 'Describe your pricing model (flat-rate, per-seat, usage-based), billing intervals (monthly/annual), trial requirements, and whether you use Stripe Checkout or direct API integration.'
---

# Stripe Subscription Architecture

## When to Use

Invoke this skill when you need to:
- Design the Stripe object model (Customer → Subscription → Invoice) for your SaaS
- Map Stripe objects to your database schema
- Model pricing tiers, billing intervals, and trials in Stripe
- Implement the subscription creation flow end-to-end
- Handle Stripe Customer creation and tenant linkage

---

## Step 1 — Understand the Stripe Object Hierarchy

Every recurring billing implementation maps to this hierarchy:

```
Customer (1)
  └─ Subscription (1..N)
       └─ SubscriptionItem (1..N) — one per Price
            └─ Price (references a Product)
  └─ PaymentMethod (1..N)
  └─ Invoice (1..N)  ← generated per billing cycle
       └─ InvoiceItem (1..N)
```

**Key objects to understand:**

| Stripe Object | Purpose | Your DB mirrors it? |
|---|---|---|
| Customer | One per tenant/user; holds payment methods | Yes — store `stripe_customer_id` |
| Product | Represents a plan tier (Free, Pro, Enterprise) | Yes — map to your `plans` table |
| Price | Billing interval + amount for a Product | Yes — map to your `prices` table |
| Subscription | Active recurring billing agreement | Yes — mirror full state |
| SubscriptionItem | Links a Price to a Subscription | Reference via subscription |
| Invoice | Record of a billing period charge | Yes — store for reconciliation |
| PaymentMethod | Card/bank details attached to Customer | Reference only (never store raw) |

---

## Step 2 — Database Schema

Mirror the Stripe object hierarchy in your database so billing state is queryable locally without API calls:

```sql
-- One tenant has exactly one Stripe customer
ALTER TABLE tenants
    ADD COLUMN stripe_customer_id  TEXT UNIQUE,
    ADD COLUMN billing_email       TEXT;

-- Products map to plan definitions
CREATE TABLE plans (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT        NOT NULL,           -- "Pro", "Enterprise"
    stripe_product_id TEXT      UNIQUE NOT NULL,
    active          BOOLEAN     NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Prices: each plan × billing interval combination
CREATE TABLE prices (
    id                  UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id             UUID        NOT NULL REFERENCES plans(id),
    stripe_price_id     TEXT        UNIQUE NOT NULL,
    billing_interval    TEXT        NOT NULL CHECK (billing_interval IN ('month', 'year')),
    amount_cents        INT         NOT NULL,
    currency            TEXT        NOT NULL DEFAULT 'usd',
    active              BOOLEAN     NOT NULL DEFAULT true
);

-- Subscriptions: one active per tenant at a time (enforce in application)
CREATE TABLE subscriptions (
    id                      UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID        NOT NULL REFERENCES tenants(id),
    plan_id                 UUID        REFERENCES plans(id),
    price_id                UUID        REFERENCES prices(id),
    stripe_subscription_id  TEXT        UNIQUE NOT NULL,
    stripe_customer_id      TEXT        NOT NULL,
    status                  TEXT        NOT NULL,   -- mirrors Stripe status
    trial_start             TIMESTAMPTZ,
    trial_end               TIMESTAMPTZ,
    current_period_start    TIMESTAMPTZ NOT NULL,
    current_period_end      TIMESTAMPTZ NOT NULL,
    cancel_at_period_end    BOOLEAN     NOT NULL DEFAULT false,
    canceled_at             TIMESTAMPTZ,
    ended_at                TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_tenant_id ON subscriptions(tenant_id);
CREATE INDEX idx_subscriptions_status    ON subscriptions(status);
CREATE INDEX idx_subscriptions_stripe_id ON subscriptions(stripe_subscription_id);
```

---

## Step 3 — Stripe Object Creation Flow

Follow this sequence to create a subscription:

```
1. Create/retrieve Stripe Customer
2. Attach PaymentMethod to Customer
3. Set as default PaymentMethod on Customer
4. Create Subscription with the desired Price
5. Handle SCA (3D Secure) if payment_intent requires action
6. Persist subscription record to DB
```

**Implementation (Go):**
```go
import (
    stripe "github.com/stripe/stripe-go/v76"
    "github.com/stripe/stripe-go/v76/customer"
    "github.com/stripe/stripe-go/v76/subscription"
)

func (s *BillingService) CreateSubscription(ctx context.Context, tenantID, priceID, paymentMethodID string) (*Subscription, error) {
    tenant, err := s.tenantRepo.Get(ctx, tenantID)
    if err != nil {
        return nil, err
    }

    // Step 1 — Ensure Stripe Customer exists
    stripeCustomerID, err := s.ensureStripeCustomer(ctx, tenant)
    if err != nil {
        return nil, fmt.Errorf("create stripe customer: %w", err)
    }

    // Step 2 — Attach payment method
    pm, err := paymentmethod.Attach(paymentMethodID, &stripe.PaymentMethodAttachParams{
        Customer: stripe.String(stripeCustomerID),
    })
    if err != nil {
        return nil, fmt.Errorf("attach payment method: %w", err)
    }

    // Step 3 — Set as default
    _, err = customer.Update(stripeCustomerID, &stripe.CustomerParams{
        InvoiceSettings: &stripe.CustomerInvoiceSettingsParams{
            DefaultPaymentMethod: stripe.String(pm.ID),
        },
    })
    if err != nil {
        return nil, fmt.Errorf("set default payment method: %w", err)
    }

    // Step 4 — Create Stripe Subscription
    price, err := s.priceRepo.GetByStripeID(ctx, priceID)
    if err != nil {
        return nil, err
    }

    stripeSub, err := subscription.New(&stripe.SubscriptionParams{
        Customer: stripe.String(stripeCustomerID),
        Items: []*stripe.SubscriptionItemsParams{
            {Price: stripe.String(price.StripePriceID)},
        },
        PaymentSettings: &stripe.SubscriptionPaymentSettingsParams{
            SaveDefaultPaymentMethod: stripe.String("on_subscription"),
        },
        Expand: []*string{stripe.String("latest_invoice.payment_intent")},
    })
    if err != nil {
        return nil, fmt.Errorf("create stripe subscription: %w", err)
    }

    // Step 5 — Handle incomplete payment (SCA required)
    if stripeSub.Status == stripe.SubscriptionStatusIncomplete {
        pi := stripeSub.LatestInvoice.PaymentIntent
        if pi.Status == stripe.PaymentIntentStatusRequiresAction {
            return nil, &RequiresSCAError{ClientSecret: pi.ClientSecret}
        }
    }

    // Step 6 — Persist to database
    return s.subscriptionRepo.Create(ctx, mapStripeSubscription(tenantID, price.ID, stripeSub))
}
```

---

## Step 4 — Stripe Customer Idempotency

Never create duplicate Stripe Customers for the same tenant:

```go
func (s *BillingService) ensureStripeCustomer(ctx context.Context, tenant *Tenant) (string, error) {
    if tenant.StripeCustomerID != "" {
        return tenant.StripeCustomerID, nil
    }

    c, err := customer.New(&stripe.CustomerParams{
        Email:    stripe.String(tenant.BillingEmail),
        Metadata: map[string]string{"tenant_id": tenant.ID},
    })
    if err != nil {
        return "", fmt.Errorf("create stripe customer: %w", err)
    }

    if err := s.tenantRepo.SetStripeCustomerID(ctx, tenant.ID, c.ID); err != nil {
        // Customer created but we failed to persist — idempotence via metadata lookup
        return "", fmt.Errorf("persist stripe customer id: %w", err)
    }

    return c.ID, nil
}
```

**Recovery when `stripe_customer_id` is lost:** search Stripe by `metadata["tenant_id"]` before creating a new Customer.

---

## Step 5 — Trial Configuration

Configure trials at the Price or Subscription level:

```go
// Trial via Subscription creation
stripeSub, err := subscription.New(&stripe.SubscriptionParams{
    Customer: stripe.String(stripeCustomerID),
    Items: []*stripe.SubscriptionItemsParams{
        {Price: stripe.String(stripePriceID)},
    },
    TrialPeriodDays: stripe.Int64(14),         // Days from now
    // OR use trial_end for a fixed end date:
    // TrialEnd: stripe.Int64(time.Now().Add(14 * 24 * time.Hour).Unix()),
})
```

**Trial without payment method (card-optional trial):**
```go
stripeSub, err := subscription.New(&stripe.SubscriptionParams{
    Customer:            stripe.String(stripeCustomerID),
    Items:               ...,
    TrialPeriodDays:     stripe.Int64(14),
    PaymentBehavior:     stripe.String("default_incomplete"),
    TrialSettings: &stripe.SubscriptionTrialSettingsParams{
        EndBehavior: &stripe.SubscriptionTrialSettingsEndBehaviorParams{
            MissingPaymentMethod: stripe.String("cancel"), // or "pause"
        },
    },
})
```

---

## Quality Checks

- [ ] Each tenant maps to exactly one Stripe Customer; `stripe_customer_id` is unique-constrained in DB
- [ ] Products and Prices are pre-created in Stripe Dashboard / via Stripe CLI; never created per-customer
- [ ] Subscription status in DB is always derived from Stripe webhooks — never set manually
- [ ] `stripe_subscription_id`, `stripe_customer_id` are indexed for fast webhook lookups
- [ ] Trial `trial_end` is stored in DB for local expiry checks without Stripe API calls
- [ ] SCA incomplete flow handled — return `client_secret` to frontend for 3DS confirmation
- [ ] Stripe Customer metadata includes `tenant_id` for recovery/lookup
