---
name: organization-membership-security
description: 'Secure team and organization membership management for SaaS applications. Use when: organization membership security, team membership management, member role security, membership authorization, remove member security, role change authorization, last owner protection, membership audit, forced logout on removal, member offboarding, role escalation prevention, tenant membership model, membership lifecycle, team access control, org membership design, membership permission boundary, membership state machine.'
argument-hint: 'Describe your membership model (roles available, whether you have owners/admins/members/viewers), how membership changes are authorized, and any known gaps (e.g., removed members retain active sessions, role escalation not gated).'
---

# Organization Membership Security

## When to Use

Invoke this skill when you need to:
- Design a secure membership state machine (invite → active → suspended → removed)
- Enforce authorization boundaries on role changes (who can promote/demote whom)
- Prevent last-owner removal that locks tenants out of their account
- Revoke all active sessions when a member is removed or suspended
- Audit membership changes with a tamper-evident log
- Handle member offboarding: access revocation, API key invalidation, data reassignment

---

## Step 1 — Membership Data Model

```sql
CREATE TABLE memberships (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id     UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role        TEXT        NOT NULL DEFAULT 'member'
                            CHECK (role IN ('owner','admin','member','viewer')),
    status      TEXT        NOT NULL DEFAULT 'active'
                            CHECK (status IN ('active','suspended','removed')),
    invited_by  UUID        REFERENCES users(id),
    added_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by  UUID        REFERENCES users(id),
    removed_at  TIMESTAMPTZ,
    removed_by  UUID        REFERENCES users(id),
    UNIQUE (tenant_id, user_id)  -- one membership per user per tenant
);

CREATE INDEX idx_memberships_tenant_status ON memberships(tenant_id, status);
CREATE INDEX idx_memberships_user         ON memberships(user_id);

-- Membership change history (immutable append-only log)
CREATE TABLE membership_changes (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    membership_id   UUID        NOT NULL REFERENCES memberships(id),
    user_id         UUID        NOT NULL REFERENCES users(id),
    changed_by      UUID        NOT NULL REFERENCES users(id),
    change_type     TEXT        NOT NULL,  -- 'role_changed' | 'suspended' | 'restored' | 'removed'
    old_role        TEXT,
    new_role        TEXT,
    old_status      TEXT,
    new_status      TEXT,
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_membership_changes_tenant ON membership_changes(tenant_id, created_at DESC);
```

---

## Step 2 — Role Hierarchy and Change Authorization

Not all actors can change all roles. Enforce a strict hierarchy:

```go
// Role hierarchy: owner > admin > member > viewer
var roleOrdinal = map[string]int{
    "owner":  4,
    "admin":  3,
    "member": 2,
    "viewer": 1,
}

// Rules:
// 1. You cannot change the role of someone with an equal or higher role than your own
// 2. You cannot assign a role higher than your own
// 3. Only owners can promote to owner / demote another owner
// 4. You cannot change your own role

func (s *MembershipService) ChangeRole(ctx context.Context, req ChangeRoleRequest) error {
    actor := auth.UserFromCtx(ctx)

    // Self-change protection
    if actor.UserID == req.TargetUserID {
        return ErrCannotChangeOwnRole
    }

    actorMembership, err := s.repo.Get(ctx, req.TenantID, actor.UserID)
    if err != nil {
        return ErrNotMember
    }
    targetMembership, err := s.repo.Get(ctx, req.TenantID, req.TargetUserID)
    if err != nil {
        return ErrTargetNotMember
    }

    // Actor cannot act on users with equal or higher rank
    if roleOrdinal[targetMembership.Role] >= roleOrdinal[actorMembership.Role] {
        return ErrInsufficientRank
    }

    // Actor cannot assign a role higher than their own
    if roleOrdinal[req.NewRole] > roleOrdinal[actorMembership.Role] {
        return ErrCannotEscalateRole
    }

    // Owner-to-owner transfer requires explicit confirmation
    if req.NewRole == "owner" && actorMembership.Role != "owner" {
        return ErrOnlyOwnerCanGrantOwner
    }

    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        if err := s.repo.UpdateRoleTx(ctx, tx, req.TenantID, req.TargetUserID, req.NewRole, actor.UserID); err != nil {
            return err
        }
        return s.auditRepo.LogTx(ctx, tx, MembershipChange{
            TenantID:     req.TenantID,
            MembershipID: targetMembership.ID,
            UserID:       req.TargetUserID,
            ChangedBy:    actor.UserID,
            ChangeType:   "role_changed",
            OldRole:      &targetMembership.Role,
            NewRole:      &req.NewRole,
        })
    })
}
```

---

## Step 3 — Last-Owner Protection

Prevent all owners from being removed or demoted — that would lock the tenant:

```go
func (s *MembershipService) guardLastOwner(ctx context.Context, tenantID, targetUserID, action string) error {
    targetMembership, err := s.repo.Get(ctx, tenantID, targetUserID)
    if err != nil {
        return err
    }
    if targetMembership.Role != "owner" {
        return nil // not an owner — guard does not apply
    }

    ownerCount, err := s.repo.CountByRole(ctx, tenantID, "owner")
    if err != nil {
        return err
    }
    if ownerCount <= 1 {
        return ErrCannotRemoveLastOwner
    }
    return nil
}

// Call guardLastOwner BEFORE any remove or demote-from-owner operation
func (s *MembershipService) RemoveMember(ctx context.Context, req RemoveMemberRequest) error {
    if err := s.guardLastOwner(ctx, req.TenantID, req.TargetUserID, "remove"); err != nil {
        return err
    }
    return s.removeMember(ctx, req)
}
```

---

## Step 4 — Member Removal: Full Offboarding

Removing a member must trigger a cascade of access revocation:

```go
func (s *MembershipService) removeMember(ctx context.Context, req RemoveMemberRequest) error {
    actor := auth.UserFromCtx(ctx)

    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // 1. Mark membership as removed
        if err := s.repo.SetStatusTx(ctx, tx, req.TenantID, req.TargetUserID,
            "removed", actor.UserID); err != nil {
            return err
        }

        // 2. Revoke all active sessions for this user in this tenant
        if err := s.sessionRepo.RevokeForUserInTenantTx(ctx, tx, req.TenantID, req.TargetUserID); err != nil {
            return err
        }

        // 3. Revoke all API keys belonging to user in this tenant
        if err := s.apiKeyRepo.RevokeForUserInTenantTx(ctx, tx, req.TenantID, req.TargetUserID); err != nil {
            return err
        }

        // 4. Revoke pending invitations sent by this user
        if err := s.inviteRepo.RevokePendingByInviterTx(ctx, tx, req.TenantID, req.TargetUserID); err != nil {
            return err
        }

        // 5. Reassign or orphan owned resources (configurable per resource type)
        if err := s.resourceReassigner.ReassignTx(ctx, tx, req.TenantID, req.TargetUserID, req.ReassignToUserID); err != nil {
            return err
        }

        // 6. Audit log
        return s.auditRepo.LogTx(ctx, tx, MembershipChange{
            TenantID:   req.TenantID,
            UserID:     req.TargetUserID,
            ChangedBy:  actor.UserID,
            ChangeType: "removed",
            Reason:     req.Reason,
        })
    })
}
```

---

## Step 5 — Member Suspension (Reversible Access Block)

Suspension preserves data ownership while blocking all access — useful before deletion confirmation:

```go
// Suspension vs. removal:
// Suspension:  status='suspended' | sessions revoked | API keys disabled | can be restored
// Removal:     status='removed'   | sessions revoked | API keys revoked  | cannot be restored without re-invite

func (s *MembershipService) SuspendMember(ctx context.Context, req SuspendRequest) error {
    if err := s.guardLastOwner(ctx, req.TenantID, req.TargetUserID, "suspend"); err != nil {
        return err
    }

    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        if err := s.repo.SetStatusTx(ctx, tx, req.TenantID, req.TargetUserID, "suspended", req.ActorID); err != nil {
            return err
        }
        // Revoke sessions but NOT API keys (reversible — keys are just disabled)
        if err := s.sessionRepo.RevokeForUserInTenantTx(ctx, tx, req.TenantID, req.TargetUserID); err != nil {
            return err
        }
        if err := s.apiKeyRepo.SuspendForUserInTenantTx(ctx, tx, req.TenantID, req.TargetUserID); err != nil {
            return err
        }
        return s.auditRepo.LogTx(ctx, tx, MembershipChange{
            TenantID: req.TenantID, UserID: req.TargetUserID,
            ChangedBy: req.ActorID, ChangeType: "suspended", Reason: req.Reason,
        })
    })
}
```

---

## Step 6 — Membership Authorization Matrix

| Action | Owner | Admin | Member | Viewer | Notes |
|---|---|---|---|---|---|
| Invite member | ✅ | ✅ | ❌ | ❌ | |
| Invite admin | ✅ | ❌ | ❌ | ❌ | Admins cannot invite admins |
| Change member → viewer | ✅ | ✅ | ❌ | ❌ | |
| Change member → admin | ✅ | ❌ | ❌ | ❌ | |
| Change admin → member | ✅ | ❌ | ❌ | ❌ | Cannot demote equal rank |
| Grant owner | ✅ | ❌ | ❌ | ❌ | Owner-only action |
| Suspend member/viewer | ✅ | ✅ | ❌ | ❌ | |
| Suspend admin | ✅ | ❌ | ❌ | ❌ | |
| Remove any member | ✅ | ✅ | ❌ | ❌ | With last-owner guard |
| Remove last owner | ❌ | ❌ | ❌ | ❌ | Blocked for all |
| Change own role | ❌ | ❌ | ❌ | ❌ | Never allowed |

---

## Quality Checks

- [ ] Role hierarchy enforced: actors cannot change roles higher than or equal to their own rank
- [ ] Role escalation blocked: actors cannot assign roles above their own
- [ ] Self-role-change blocked: no actor can change their own role
- [ ] Last-owner guard runs BEFORE every remove and owner-demote operation
- [ ] Member removal revokes: sessions, API keys, pending invites (atomic TX)
- [ ] Member suspension disables (not revokes) API keys — reversible on restore
- [ ] All membership changes written to `membership_changes` audit log with actor ID and reason
- [ ] Membership lookup always scoped by `tenant_id` — no global `user_id` only queries
- [ ] Removed members receive no references to tenant data in subsequent API calls (ErrNotMember)

## After Completion

Recommended next steps:

- Use **`secure-invite-system`** for the invitation lifecycle that precedes membership
- Use **`api-key-management`** to design the API key revocation flow wired to member removal
- Use **`audit-trail-design`** to make the membership change log tamper-evident
- Use **`authorization-models`** for RBAC enforcement patterns across the broader application
