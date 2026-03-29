---
name: distributed-workflows
description: 'Long-running distributed workflow coordination for SaaS systems. Use when: distributed workflow, saga pattern, saga orchestration, saga choreography, long-running process, workflow state machine, multi-step workflow, compensating transaction, rollback saga, workflow step, workflow failure, workflow retry, workflow timeout, stuck workflow, workflow recovery, distributed transaction, two-phase workflow, provisioning saga, payment workflow, tenant provisioning workflow, workflow coordinator, workflow engine, step idempotency, workflow compensation, workflow audit, distributed saga, workflow persistence.'
argument-hint: 'Describe the workflow being orchestrated (e.g., tenant provisioning, payment + entitlement activation, order fulfillment), list the steps in order, identify which steps are reversible (compensating action exists) and which are not, and describe the failure mode if the workflow does not complete.'
---

# Distributed Workflows

## When to Use

Invoke this skill when you need to:
- Coordinate multi-step processes that span multiple services or external systems (payment, DB, email, Stripe)
- Handle partial failures: some steps succeed, then a later step fails — need rollback
- Implement compensating transactions for each reversible step
- Detect stuck or timed-out workflows and resume or escalate them
- Audit every step transition with full context for debugging and compliance
- Choose between saga choreography (event-driven, decentralized) and orchestration (coordinator, centralized)

---

## Choreography vs Orchestration

| | Choreography | Orchestration |
|---|---|---|
| Control | Distributed — each service reacts to events | Centralized — coordinator drives every step |
| Observability | Hard — trace spans multiple independent services | Easy — state in one table |
| Coupling | Services know event contracts only | Services know coordinator API |
| Failure detection | Each service must detect and publish failure events | Coordinator detects missing callbacks / timeouts |
| Best for | Simple, stable workflows with few steps | Complex, dynamic workflows with branching and retries |
| Recommended for SaaS | Choreography for < 3 steps | Orchestration for 3+ steps or compensation logic |

---

## Step 1 — Workflow State Schema

```sql
CREATE TYPE workflow_step_status AS ENUM ('pending', 'running', 'completed', 'failed', 'compensated', 'skipped');
CREATE TYPE workflow_status AS ENUM ('running', 'completed', 'failed', 'compensating', 'compensation_failed');

CREATE TABLE workflows (
    id             UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    type           TEXT        NOT NULL,  -- 'tenant_provisioning' | 'payment_activation'
    tenant_id      UUID,
    correlation_id TEXT        NOT NULL UNIQUE,  -- idempotency key (e.g., checkout session ID)
    status         workflow_status NOT NULL DEFAULT 'running',
    current_step   TEXT,
    context        JSONB       NOT NULL DEFAULT '{}',  -- shared state across steps
    started_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at   TIMESTAMPTZ,
    timeout_at     TIMESTAMPTZ NOT NULL,  -- deadline for stuck-workflow detection
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_steps (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id   UUID        NOT NULL REFERENCES workflows(id) ON DELETE CASCADE,
    step_name     TEXT        NOT NULL,
    status        workflow_step_status NOT NULL DEFAULT 'pending',
    attempt_count INT         NOT NULL DEFAULT 0,
    input         JSONB,
    output        JSONB,        -- result stored for compensation and next steps
    error         TEXT,
    started_at    TIMESTAMPTZ,
    completed_at  TIMESTAMPTZ,
    UNIQUE (workflow_id, step_name)
);

CREATE INDEX idx_workflows_stuck ON workflows(timeout_at)
    WHERE status = 'running';
CREATE INDEX idx_workflows_tenant ON workflows(tenant_id, started_at);
```

---

## Step 2 — Workflow Definition (Tenant Provisioning Example)

```go
// WorkflowDefinition declares steps, their order, and compensation actions
type WorkflowDefinition struct {
    Type  string
    Steps []StepDefinition
}

type StepDefinition struct {
    Name      string
    Execute   StepFunc        // forward action
    Compensate StepFunc       // rollback action (nil if not reversible)
    Timeout   time.Duration
}

type StepFunc func(ctx context.Context, wf *WorkflowContext) (StepOutput, error)
type StepOutput map[string]any

// TenantProvisioningWorkflow defines the 5-step tenant provisioning saga
var TenantProvisioningWorkflow = WorkflowDefinition{
    Type: "tenant_provisioning",
    Steps: []StepDefinition{
        {
            Name:      "create_tenant_record",
            Execute:   createTenantRecord,
            Compensate: deleteTenantRecord,  // reversible
            Timeout:   5 * time.Second,
        },
        {
            Name:      "create_stripe_customer",
            Execute:   createStripeCustomer,
            Compensate: deleteStripeCustomer, // reversible via Stripe API
            Timeout:   10 * time.Second,
        },
        {
            Name:      "provision_default_entitlements",
            Execute:   provisionEntitlements,
            Compensate: revokeEntitlements,  // reversible
            Timeout:   5 * time.Second,
        },
        {
            Name:      "send_welcome_email",
            Execute:   sendWelcomeEmail,
            Compensate: nil,                 // NOT reversible — email already sent
            Timeout:   15 * time.Second,
        },
        {
            Name:      "mark_tenant_active",
            Execute:   markTenantActive,
            Compensate: markTenantFailed,    // reversible
            Timeout:   5 * time.Second,
        },
    },
}
```

---

## Step 3 — Workflow Coordinator

```go
type WorkflowCoordinator struct {
    def   WorkflowDefinition
    store WorkflowRepository
}

// Start creates a new workflow — idempotent via correlationID
func (c *WorkflowCoordinator) Start(ctx context.Context, correlationID string, tenantID uuid.UUID, input map[string]any) (*Workflow, error) {
    wf, err := c.store.GetByCorrelationID(ctx, correlationID)
    if err == nil {
        return wf, nil // already started — return existing workflow (idempotent)
    }
    if !errors.Is(err, ErrNotFound) {
        return nil, err
    }

    wf = &Workflow{
        Type:          c.def.Type,
        TenantID:      tenantID,
        CorrelationID: correlationID,
        Status:        WorkflowStatusRunning,
        Context:       input,
        TimeoutAt:     time.Now().Add(30 * time.Minute),
    }
    if err := c.store.Create(ctx, wf); err != nil {
        return nil, err
    }
    go c.execute(context.Background(), wf) // async execution
    return wf, nil
}

func (c *WorkflowCoordinator) execute(ctx context.Context, wf *Workflow) {
    for _, stepDef := range c.def.Steps {
        if err := c.executeStep(ctx, wf, stepDef); err != nil {
            slog.Error("workflow step failed — starting compensation", "workflow", wf.ID, "step", stepDef.Name, "err", err)
            c.compensate(ctx, wf, stepDef)
            c.store.MarkFailed(ctx, wf.ID, err.Error())
            return
        }
    }
    c.store.MarkCompleted(ctx, wf.ID)
    slog.Info("workflow completed", "workflow", wf.ID, "type", wf.Type)
}

func (c *WorkflowCoordinator) executeStep(ctx context.Context, wf *Workflow, def StepDefinition) error {
    // Mark step as running
    c.store.StartStep(ctx, wf.ID, def.Name)

    stepCtx := &WorkflowContext{WorkflowID: wf.ID, TenantID: wf.TenantID, Data: wf.Context}
    stepCtx2, cancel := context.WithTimeout(ctx, def.Timeout)
    defer cancel()

    output, err := def.Execute(stepCtx2, stepCtx)
    if err != nil {
        c.store.FailStep(ctx, wf.ID, def.Name, err.Error())
        return err
    }

    // Merge step output into workflow context for use by later steps
    for k, v := range output {
        wf.Context[k] = v
    }
    c.store.CompleteStep(ctx, wf.ID, def.Name, output)
    return nil
}

// compensate runs compensation (rollback) in REVERSE step order
func (c *WorkflowCoordinator) compensate(ctx context.Context, wf *Workflow, failedAt StepDefinition) {
    c.store.MarkCompensating(ctx, wf.ID)

    completedSteps := c.store.GetCompletedSteps(ctx, wf.ID)
    // Reverse order: undo last completed step first
    for i := len(completedSteps) - 1; i >= 0; i-- {
        stepName := completedSteps[i]
        stepDef := c.findStep(stepName)
        if stepDef == nil || stepDef.Compensate == nil {
            slog.Info("step not reversible — skipping compensation", "step", stepName)
            continue
        }
        stepCtx := &WorkflowContext{WorkflowID: wf.ID, TenantID: wf.TenantID, Data: wf.Context}
        if _, err := stepDef.Compensate(ctx, stepCtx); err != nil {
            slog.Error("compensation failed", "workflow", wf.ID, "step", stepName, "err", err)
            c.store.MarkCompensationFailed(ctx, wf.ID, stepName, err.Error())
            // Stop compensation and escalate — manual intervention required
            return
        }
        c.store.MarkStepCompensated(ctx, wf.ID, stepName)
        slog.Info("step compensated", "workflow", wf.ID, "step", stepName)
    }
}
```

---

## Step 4 — Stuck Workflow Detection

```go
// Background job: find workflows that exceeded their timeout and have not completed
func (d *StuckWorkflowDetector) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            stuckWorkflows, _ := d.store.FindStuck(ctx)
            for _, wf := range stuckWorkflows {
                slog.Error("stuck workflow detected",
                    "workflow_id", wf.ID,
                    "type", wf.Type,
                    "tenant_id", wf.TenantID,
                    "current_step", wf.CurrentStep,
                    "started_at", wf.StartedAt,
                    "timeout_at", wf.TimeoutAt,
                )
                d.alerts.FireStuckWorkflowAlert(ctx, wf)
            }
        }
    }
}

// SQL query for stuck workflows
const findStuckWorkflowsSQL = `
    SELECT id, type, tenant_id, current_step, started_at, timeout_at
    FROM workflows
    WHERE status = 'running'
      AND timeout_at < now()
    ORDER BY timeout_at ASC
    LIMIT 50
`
```

---

## Step Reference: Compensation Eligibility

| Step type | Reversible? | Compensation strategy |
|---|---|---|
| Insert DB record | Yes | `DELETE WHERE id = ?` |
| Create Stripe customer | Yes | `stripe.DeleteCustomer` |
| Send email | No | Log only — cannot unsend |
| Publish event | No | Publish compensating event |
| Charge credit card | Partial | Issue refund |
| Create cloud resource | Yes | Destroy resource via API |
| Grant entitlement | Yes | Revoke entitlement |
| Update existing record | Yes | Restore original values (store in step output) |

---

## Quality Checks

- [ ] Each step in the definition has a `Timeout` — no step blocks indefinitely
- [ ] Compensation runs in reverse step order — later effects undone before earlier ones
- [ ] Non-reversible steps (email, event publish) are placed LAST in the workflow — minimize uncompensable work
- [ ] `correlation_id` UNIQUE constraint — `Start()` is idempotent; second call returns existing workflow
- [ ] Step output saved to DB — compensation step can read what forward step created (e.g., Stripe customer ID)
- [ ] `timeout_at` set on workflow creation — stuck workflow detector can find orphans
- [ ] Stuck workflow alert fires when `timeout_at < now()` for running workflows
- [ ] `compensation_failed` status triggers immediate on-call alert — requires manual intervention

## After Completion

- Use **`job-idempotency`** to make each individual step idempotent on retry
- Use **`dead-letter-handling`** for workflows that reach `compensation_failed` state
- Use **`event-driven-saas-patterns`** for choreography-based workflow signaling via events
- Use **`audit-trail-design`** to record every workflow step transition for compliance
- Use **`tenant-provisioning`** for the specific provisioning saga that uses this pattern
