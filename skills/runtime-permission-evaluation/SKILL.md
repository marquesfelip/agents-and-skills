---
name: runtime-permission-evaluation
description: 'Define dynamic permission evaluation at runtime combining entitlements, roles, and contextual signals for SaaS. Use when: runtime permission evaluation, dynamic permission check, permission evaluation runtime, contextual permission, role entitlement combination, permission resolver, access policy evaluation, RBAC entitlement combination, attribute based permission, runtime access policy, dynamic access check, permission evaluation engine, combined permission check, user role feature check, resource permission evaluation, permission evaluation design, context aware permission.'
argument-hint: 'Describe your permission dimensions: e.g., user roles (admin/member/viewer), resource ownership (own vs. shared vs. tenant-wide), entitlement gating (plan features), and whether you have row-level or field-level access distinctions.'
---

# Runtime Permission Evaluation

## When to Use

Invoke this skill when you need to:
- Combine multiple permission signals: user role + tenant entitlement + resource ownership
- Build a policy evaluator that accepts context (user, resource, action, tenant) and returns allow/deny
- Distinguish between "tenant can use this feature" vs. "this user can perform this action on this resource"
- Handle admin/impersonation bypasses safely and auditably
- Evaluate permissions in background jobs where there is no HTTP request context
- Design a permission evaluation order that is fast, deterministic, and auditable

---

## Core Concept: Permission Evaluation Dimensions

Runtime permission evaluation combines three independent axes:

```
DECISION = f(
    tenant_entitlement,   // Can the tenant's plan do this at all?
    user_role,            // Does the user's role allow this action?
    resource_ownership,   // Does the user own or share access to this resource?
    [contextual signals]  // IP restriction, time-of-day, MFA state, etc.
)
```

**Evaluation order (fail-fast):**
1. **Tenant entitlement check** — If the feature is not in the plan, deny immediately.
2. **User role check** — If the user role does not allow the action, deny.
3. **Resource ownership/sharing check** — If the user does not have access to the specific resource, deny.
4. **Contextual signals** — Additional policy constraints (IP allowlisting, MFA, etc.).

---

## Step 1 — Permission Request / Response Model

```go
// PermissionRequest captures all inputs the evaluator needs.
type PermissionRequest struct {
    TenantID    string
    UserID      string
    UserRole    string           // 'owner' | 'admin' | 'member' | 'viewer'
    Action      string           // 'read' | 'write' | 'delete' | 'invite' | 'export'
    Resource    string           // 'project' | 'report' | 'member' | 'billing' | 'settings'
    ResourceID  *string          // Specific resource ID, nil = collection-level
    FeatureKey  *string          // Entitlement gate (e.g. "pdf_export"); nil = no entitlement gate
    // Contextual signals
    IPAddress   net.IP
    MFAVerified bool
    ActorKind   string           // 'user' | 'api_key' | 'service_account' | 'platform_admin'
}

type PermissionResult struct {
    Allowed bool
    Reason  DenyReason          // populated when Allowed=false
    Signals []string            // audit log evidence: which checks passed/failed
}

type DenyReason string
const (
    DenyReasonFeatureNotInPlan  DenyReason = "feature_not_in_plan"
    DenyReasonRoleInsufficient  DenyReason = "role_insufficient"
    DenyReasonResourceForbidden DenyReason = "resource_forbidden"
    DenyReasonIPRestricted      DenyReason = "ip_restricted"
    DenyReasonMFARequired       DenyReason = "mfa_required"
)
```

---

## Step 2 — Role-Action Matrix

Define which roles can perform which actions on which resources. Store as config or code — NOT in the DB (rarely changes at runtime).

```go
// RolePolicy defines role-based action permissions per resource type.
type RolePolicy map[string]map[string][]string
// rolePolicy[resource][action] = []allowedRoles

var DefaultRolePolicy = RolePolicy{
    "project": {
        "read":   {"owner", "admin", "member", "viewer"},
        "write":  {"owner", "admin", "member"},
        "delete": {"owner", "admin"},
        "share":  {"owner", "admin"},
    },
    "member": {
        "read":   {"owner", "admin"},
        "invite": {"owner", "admin"},
        "remove": {"owner", "admin"},
        "update_role": {"owner"},
    },
    "billing": {
        "read":   {"owner", "admin"},
        "write":  {"owner"},
    },
    "settings": {
        "read":   {"owner", "admin", "member"},
        "write":  {"owner", "admin"},
    },
    "report": {
        "read":   {"owner", "admin", "member", "viewer"},
        "export": {"owner", "admin", "member"},  // + entitlement gate: "pdf_export"
    },
    "api_key": {
        "read":   {"owner", "admin"},
        "create": {"owner", "admin"},
        "delete": {"owner", "admin"},
    },
}

func (p RolePolicy) IsAllowed(resource, action, role string) bool {
    actions, ok := p[resource]
    if !ok {
        return false
    }
    allowedRoles, ok := actions[action]
    if !ok {
        return false
    }
    for _, r := range allowedRoles {
        if r == role {
            return true
        }
    }
    return false
}
```

---

## Step 3 — Permission Evaluator

```go
type PermissionEvaluator struct {
    entitlements EntitlementResolver
    rolePolicy   RolePolicy
    resourceRepo ResourceOwnershipRepository
    ipAllowlist  IPAllowlistService    // nil = no IP restriction
    auditLogger  AuditLogger
}

func (e *PermissionEvaluator) Evaluate(ctx context.Context, req PermissionRequest) PermissionResult {
    signals := make([]string, 0, 4)

    // ── 1. Platform admin bypass (impersonation / ops) ────────────────────────
    if req.ActorKind == "platform_admin" {
        e.auditLogger.Log(ctx, AuditEvent{
            TenantID: req.TenantID, UserID: req.UserID,
            Action: req.Action, Resource: req.Resource,
            Outcome: "allowed_platform_admin_bypass",
        })
        return PermissionResult{Allowed: true, Signals: []string{"platform_admin_bypass"}}
    }

    // ── 2. Entitlement gate (tenant plan check) ───────────────────────────────
    if req.FeatureKey != nil {
        enabled, err := e.entitlements.IsEnabled(ctx, req.TenantID, *req.FeatureKey)
        if err != nil || !enabled {
            return PermissionResult{
                Allowed: false,
                Reason:  DenyReasonFeatureNotInPlan,
                Signals: append(signals, "entitlement_denied:"+*req.FeatureKey),
            }
        }
        signals = append(signals, "entitlement_ok:"+*req.FeatureKey)
    }

    // ── 3. Role-action check ──────────────────────────────────────────────────
    if !e.rolePolicy.IsAllowed(req.Resource, req.Action, req.UserRole) {
        return PermissionResult{
            Allowed: false,
            Reason:  DenyReasonRoleInsufficient,
            Signals: append(signals, fmt.Sprintf("role_denied:%s_on_%s", req.Action, req.Resource)),
        }
    }
    signals = append(signals, fmt.Sprintf("role_ok:%s", req.UserRole))

    // ── 4. Resource ownership / sharing check ────────────────────────────────
    if req.ResourceID != nil {
        accessible, err := e.resourceRepo.IsAccessible(ctx, req.TenantID, req.UserID, req.Resource, *req.ResourceID)
        if err != nil || !accessible {
            return PermissionResult{
                Allowed: false,
                Reason:  DenyReasonResourceForbidden,
                Signals: append(signals, "resource_forbidden:"+*req.ResourceID),
            }
        }
        signals = append(signals, "resource_ok:"+*req.ResourceID)
    }

    // ── 5. Contextual signals ─────────────────────────────────────────────────
    if e.ipAllowlist != nil {
        if !e.ipAllowlist.IsAllowed(ctx, req.TenantID, req.IPAddress) {
            return PermissionResult{
                Allowed: false,
                Reason:  DenyReasonIPRestricted,
                Signals: append(signals, "ip_blocked:"+req.IPAddress.String()),
            }
        }
        signals = append(signals, "ip_ok")
    }

    return PermissionResult{Allowed: true, Signals: signals}
}
```

---

## Step 4 — Middleware and Handler Integration

```go
// HTTP middleware wrapping evaluate — injects result into context for handlers
func PermissionMiddleware(evaluator *PermissionEvaluator, resource, action string, featureKey *string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user := UserFromCtx(r.Context())
            resourceID := chi.URLParam(r, "id") // or nil for collection routes
            var rid *string
            if resourceID != "" {
                rid = &resourceID
            }

            result := evaluator.Evaluate(r.Context(), PermissionRequest{
                TenantID:    user.TenantID,
                UserID:      user.ID,
                UserRole:    user.Role,
                Action:      action,
                Resource:    resource,
                ResourceID:  rid,
                FeatureKey:  featureKey,
                IPAddress:   realIP(r),
                MFAVerified: user.MFAVerified,
                ActorKind:   user.ActorKind,
            })

            if !result.Allowed {
                writePermissionError(w, result)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

func writePermissionError(w http.ResponseWriter, result PermissionResult) {
    w.Header().Set("Content-Type", "application/json")
    code := http.StatusForbidden
    if result.Reason == DenyReasonFeatureNotInPlan {
        code = http.StatusPaymentRequired // 402 with upgrade prompt
    }
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(map[string]any{
        "error":       string(result.Reason),
        "upgrade_url": "/settings/billing",
    })
}
```

---

## Step 5 — Background Job Evaluation

When there is no HTTP request, reconstruct the principal explicitly:

```go
// For background jobs — fetch user principal from DB
func (j *ExportJob) Run(ctx context.Context, payload ExportPayload) error {
    user, err := j.userRepo.Get(ctx, payload.UserID)
    if err != nil {
        return err
    }

    result := j.evaluator.Evaluate(ctx, PermissionRequest{
        TenantID:   payload.TenantID,
        UserID:     payload.UserID,
        UserRole:   user.Role,
        Action:     "export",
        Resource:   "report",
        ResourceID: &payload.ReportID,
        FeatureKey: ptr("pdf_export"),
        ActorKind:  "user",
        // No IP or MFA — background jobs skip contextual signals unless explicitly required
    })

    if !result.Allowed {
        return fmt.Errorf("export permission denied: %s", result.Reason)
    }
    // ... export logic
}
```

---

## Quality Checks

- [ ] Evaluation order is always: entitlement → role → resource → context
- [ ] Platform admin bypass is logged to the audit trail with actor identity
- [ ] `PermissionResult.Reason` propagates to the HTTP response — clients know whether to show upgrade UI or access-denied UI
- [ ] Role-action policy is defined centrally — not scattered across individual handlers
- [ ] Background jobs reconstruct principal from DB — do not skip permission evaluation
- [ ] `DenyReasonFeatureNotInPlan` maps to HTTP 402 — not 403 — for UX-friendly upgrade prompts
- [ ] Resource ownership check is tenant-scoped (prevents cross-tenant access via resource ID)
- [ ] All platform admin bypasses appear in the audit log with `"allowed_platform_admin_bypass"` outcome

## After Completion

You have a full runtime permission evaluator. Recommended next steps:

- Use **`quota-enforcement-runtime`** to reject actions that exceed numeric limits (seats, API calls)
- Use **`feature-access-resolution`** if you need the lightweight entitlement-only check path
- Use **`auditability-patterns`** to design the full audit trail for permission evaluation events
