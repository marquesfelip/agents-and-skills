---
name: invoice-reconciliation
description: 'Invoice validation and reconciliation processes for SaaS billing. Use when: invoice reconciliation, billing reconciliation, invoice validation, stripe invoice sync, invoice audit, billing audit, stripe invoice database, invoice line items, invoice amount validation, stripe invoice finalized, invoice payment reconciliation, billing discrepancy, invoice mismatch, stripe invoice webhook, invoice storage, invoice history, revenue reconciliation, billing period reconciliation, invoice idempotency, invoice deduplication, invoice reporting, invoice export, invoice dispute, stripe invoice retrieve, invoice status sync.'
argument-hint: 'Describe your invoicing model (Stripe-managed or custom), which Stripe invoice events you process, how you store invoices locally, and any reconciliation gaps you have observed (e.g., missing invoices, amount mismatches, incorrect line items).'
---

# Invoice Reconciliation

## When to Use

Invoke this skill when you need to:
- Store Stripe invoices locally for querying, reporting, and dispute resolution
- Validate that local invoice records match what Stripe has
- Detect and alert on billing discrepancies between your DB and Stripe
- Build a scheduled reconciliation job to catch missed webhook events
- Provide customers with an invoice history page
- Handle invoice finalization, voiding, and credit notes

---

## Step 1 — Local Invoice Schema

Mirror Stripe invoice fields needed for reporting and reconciliation:

```sql
CREATE TABLE invoices (
    id                  UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID        NOT NULL REFERENCES tenants(id),
    subscription_id     UUID        REFERENCES subscriptions(id),
    stripe_invoice_id   TEXT        UNIQUE NOT NULL,              -- inv_...
    stripe_customer_id  TEXT        NOT NULL,
    number              TEXT,                                      -- INV-0001 (Stripe-assigned)
    status              TEXT        NOT NULL,                      -- draft | open | paid | void | uncollectible
    currency            TEXT        NOT NULL,
    subtotal_cents      BIGINT      NOT NULL DEFAULT 0,
    tax_cents           BIGINT      NOT NULL DEFAULT 0,
    amount_due_cents    BIGINT      NOT NULL DEFAULT 0,
    amount_paid_cents   BIGINT      NOT NULL DEFAULT 0,
    amount_remaining_cents BIGINT   NOT NULL DEFAULT 0,
    period_start        TIMESTAMPTZ,
    period_end          TIMESTAMPTZ,
    due_date            TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    voided_at           TIMESTAMPTZ,
    hosted_invoice_url  TEXT,
    invoice_pdf_url     TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoices_tenant_id         ON invoices(tenant_id);
CREATE INDEX idx_invoices_stripe_invoice_id ON invoices(stripe_invoice_id);
CREATE INDEX idx_invoices_status            ON invoices(status);
CREATE INDEX idx_invoices_paid_at           ON invoices(paid_at);

-- Invoice line items for detailed reporting
CREATE TABLE invoice_line_items (
    id                  UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID        NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    stripe_line_item_id TEXT        NOT NULL,
    description         TEXT,
    amount_cents        BIGINT      NOT NULL,
    currency            TEXT        NOT NULL,
    proration           BOOLEAN     NOT NULL DEFAULT false,
    period_start        TIMESTAMPTZ,
    period_end          TIMESTAMPTZ,
    quantity            INT,
    unit_amount_cents   BIGINT
);
```

---

## Step 2 — Invoice Sync via Webhooks

Upsert invoice records when Stripe delivers lifecycle events:

```go
// Handle all invoice events with a single upsert handler
func (h *InvoiceHandler) SyncInvoice(ctx context.Context, event stripe.Event) error {
    var stripeInvoice stripe.Invoice
    if err := json.Unmarshal(event.Data.Raw, &stripeInvoice); err != nil {
        return fmt.Errorf("unmarshal invoice: %w", err)
    }

    tenant, err := h.tenantRepo.GetByStripeCustomerID(ctx, stripeInvoice.Customer.ID)
    if err != nil {
        return fmt.Errorf("resolve tenant: %w", err)
    }

    local := mapStripeInvoice(tenant.ID, &stripeInvoice)

    return h.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // Upsert invoice — safe to call multiple times
        if err := h.invoiceRepo.UpsertTx(ctx, tx, local); err != nil {
            return err
        }

        // Upsert line items
        for _, line := range stripeInvoice.Lines.Data {
            item := mapLineItem(local.ID, line)
            if err := h.lineItemRepo.UpsertTx(ctx, tx, item); err != nil {
                return err
            }
        }

        return nil
    })
}

// Register for all relevant invoice events
var invoiceEvents = []string{
    "invoice.created",
    "invoice.finalized",
    "invoice.paid",
    "invoice.payment_failed",
    "invoice.voided",
    "invoice.marked_uncollectible",
    "invoice.updated",
}
```

---

## Step 3 — Stripe-to-DB Mapping

```go
func mapStripeInvoice(tenantID string, inv *stripe.Invoice) *Invoice {
    var paidAt, voidedAt *time.Time
    if inv.StatusTransitions.PaidAt > 0 {
        t := time.Unix(inv.StatusTransitions.PaidAt, 0)
        paidAt = &t
    }
    if inv.StatusTransitions.VoidedAt > 0 {
        t := time.Unix(inv.StatusTransitions.VoidedAt, 0)
        voidedAt = &t
    }

    return &Invoice{
        TenantID:             tenantID,
        StripeInvoiceID:      inv.ID,
        StripeCustomerID:     inv.Customer.ID,
        Number:               inv.Number,
        Status:               string(inv.Status),
        Currency:             string(inv.Currency),
        SubtotalCents:        inv.Subtotal,
        TaxCents:             inv.Tax,
        AmountDueCents:       inv.AmountDue,
        AmountPaidCents:      inv.AmountPaid,
        AmountRemainingCents: inv.AmountRemaining,
        PeriodStart:          nullableTime(inv.PeriodStart),
        PeriodEnd:            nullableTime(inv.PeriodEnd),
        DueDate:              nullableTime(inv.DueDate),
        PaidAt:               paidAt,
        VoidedAt:             voidedAt,
        HostedInvoiceURL:     inv.HostedInvoiceURL,
        InvoicePDFURL:        inv.InvoicePDF,
    }
}
```

---

## Step 4 — Reconciliation Job

Run a periodic job to detect and repair invoices that were missed due to webhook failures:

```go
// Runs daily — catches invoices missed by webhooks
func (j *InvoiceReconciliationJob) Run(ctx context.Context) error {
    // Fetch last 90 days of invoices from Stripe
    params := &stripe.InvoiceListParams{}
    params.Filters.AddFilter("created", "gte", strconv.FormatInt(
        time.Now().AddDate(0, 0, -90).Unix(), 10,
    ))
    params.Limit = stripe.Int64(100)
    params.Expand = []*string{stripe.String("data.lines")}

    iter := invoice.List(params)

    var missing, updated, errors int
    for iter.Next() {
        stripeInv := iter.Invoice()

        local, err := j.invoiceRepo.GetByStripeID(ctx, stripeInv.ID)
        if errors.Is(err, ErrNotFound) {
            // Missing invoice — sync it now
            tenant, err := j.tenantRepo.GetByStripeCustomerID(ctx, stripeInv.Customer.ID)
            if err != nil {
                slog.Error("reconciliation: unknown customer",
                    slog.String("customer_id", stripeInv.Customer.ID))
                errors++
                continue
            }
            if err := j.invoiceRepo.Upsert(ctx, mapStripeInvoice(tenant.ID, stripeInv)); err != nil {
                errors++
                continue
            }
            missing++
            continue
        }

        // Check for status mismatch
        if local.Status != string(stripeInv.Status) ||
            local.AmountPaidCents != stripeInv.AmountPaid {
            if err := j.invoiceRepo.Upsert(ctx, mapStripeInvoice(local.TenantID, stripeInv)); err != nil {
                errors++
                continue
            }
            updated++
        }
    }

    slog.Info("invoice reconciliation complete",
        slog.Int("missing_synced", missing),
        slog.Int("updated", updated),
        slog.Int("errors", errors),
    )

    return iter.Err()
}
```

---

## Step 5 — Invoice Validation Rules

Validate invoices before considering billing data trustworthy:

```go
type InvoiceValidator struct{}

func (v *InvoiceValidator) Validate(local *Invoice, stripe *stripe.Invoice) []ValidationError {
    var errs []ValidationError

    if local.AmountDueCents != stripe.AmountDue {
        errs = append(errs, ValidationError{
            Field:    "amount_due_cents",
            Expected: stripe.AmountDue,
            Got:      local.AmountDueCents,
        })
    }

    if local.Status != string(stripe.Status) {
        errs = append(errs, ValidationError{
            Field:    "status",
            Expected: string(stripe.Status),
            Got:      local.Status,
        })
    }

    if local.Currency != string(stripe.Currency) {
        errs = append(errs, ValidationError{
            Field:    "currency",
            Expected: string(stripe.Currency),
            Got:      local.Currency,
        })
    }

    return errs
}
```

---

## Quality Checks

- [ ] All invoice webhook event types (`invoice.paid`, `invoice.finalized`, `invoice.voided`, etc.) registered and handled
- [ ] Invoice upsert is idempotent — safe to apply the same event multiple times
- [ ] Line items stored separately for detailed reporting and dispute evidence
- [ ] Reconciliation job runs daily and covers at least 90 days back to catch webhook gaps
- [ ] Discrepancies between DB and Stripe alerts are emitted and actionable (not silently corrected)
- [ ] `stripe_invoice_id` has a UNIQUE constraint — prevents duplicate invoice records
- [ ] `hosted_invoice_url` stored locally so customers can access it from your UI without a Stripe API call
- [ ] Invoice data retained for at least 7 years for tax and financial compliance
