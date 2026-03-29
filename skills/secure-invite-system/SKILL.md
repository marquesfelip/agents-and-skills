---
name: secure-invite-system
description: 'Secure organization invitation workflows for SaaS applications. Use when: secure invite system, organization invitation, team invite security, invite token security, invite link expiry, invite email validation, invite flow hardening, invite abuse prevention, invite replay prevention, invite takeover, organization invite design, token-based invite, guest invite security, invite rate limiting, invite acceptance flow, pending invite management, bulk invite security, invite revocation.'
argument-hint: 'Describe your invitation model (email-based, link-based, or both), whether invites are tenant-scoped or workspace-scoped, and any known abuse scenarios (invite link forwarding, domain spoofing, mass invite spam).'
---

# Secure Invite System

## When to Use

Invoke this skill when you need to:
- Design or audit a token-based invitation flow for organizations or teams
- Harden invite tokens against replay, forwarding, and takeover attacks
- Enforce invite email validation (invitee must accept with the invited address)
- Rate-limit invite creation to prevent spam and abuse
- Handle edge cases: expired invites, re-inviting, role changes before acceptance
- Build invite revocation for offboarding and security incidents

---

## Threat Model: Invite Attack Vectors

| Attack | Mechanism | Defense |
|---|---|---|
| Invite link forwarding | Victim forwards invite email to attacker | Bind token to invitee email; verify at acceptance |
| Invite token brute force | Guess short/weak tokens | 32-byte random token; hashed storage |
| Replay after acceptance | Reuse accepted invite link | Single-use: invalidate on acceptance |
| Expired link reuse | Old invite link still works | Hard expiry enforced server-side |
| Email spoofing | Attacker claims to be from a trusted domain | Verify invited email domain vs. allowed domains |
| Mass-invite spam | Invite bot sends thousands of invite emails | Rate limit per tenant + per inviter |
| Invite role escalation | Accept invite into a higher role than granted | Role stored server-side; not in token or URL |
| Open invite links | Shareable "join" link with no email binding | Limit scope; require email confirmation |
| Pre-registration takeover | Attacker registers email before victim accepts | Check invite ownership at acceptance; see Step 5 |

---

## Step 1 — Invite Data Model

```sql
CREATE TABLE invitations (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    invited_by      UUID        NOT NULL REFERENCES users(id),
    invitee_email   TEXT        NOT NULL,
    invitee_role    TEXT        NOT NULL DEFAULT 'member'
                                CHECK (invitee_role IN ('admin','member','viewer')),
    token_hash      TEXT        NOT NULL UNIQUE,  -- SHA-256 of the raw token; raw token sent via email only
    status          TEXT        NOT NULL DEFAULT 'pending'
                                CHECK (status IN ('pending','accepted','expired','revoked')),
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '72 hours',
    accepted_by     UUID        REFERENCES users(id),  -- user who accepted
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invitations_token_hash   ON invitations(token_hash);
CREATE INDEX idx_invitations_tenant_email ON invitations(tenant_id, invitee_email);
CREATE INDEX idx_invitations_status       ON invitations(status, expires_at)
    WHERE status = 'pending';

-- Constraint: one pending invite per email per tenant
CREATE UNIQUE INDEX idx_invitations_pending_unique
    ON invitations(tenant_id, LOWER(invitee_email))
    WHERE status = 'pending';
```

---

## Step 2 — Invite Token Generation and Storage

```go
// NEVER store the raw token — store only its SHA-256 hash
func generateInviteToken() (rawToken string, tokenHash string, err error) {
    b := make([]byte, 32)
    if _, err = rand.Read(b); err != nil {
        return "", "", fmt.Errorf("generating invite token: %w", err)
    }
    rawToken = base64.URLEncoding.EncodeToString(b) // URL-safe, no padding issues
    hash := sha256.Sum256([]byte(rawToken))
    tokenHash = hex.EncodeToString(hash[:])
    return rawToken, tokenHash, nil
}

func (s *InviteService) CreateInvite(ctx context.Context, req CreateInviteRequest) (*Invitation, error) {
    // 1. Authorization: inviter must have permission to invite at the requested role
    if err := s.canInviteAtRole(ctx, req.InviterID, req.TenantID, req.Role); err != nil {
        return nil, err
    }

    // 2. Rate limit: max 20 invites per tenant per hour, max 5 per inviter per hour
    if err := s.rateLimiter.Check(ctx, "invite_tenant:"+req.TenantID, 20, time.Hour); err != nil {
        return nil, ErrInviteRateLimitExceeded
    }
    if err := s.rateLimiter.Check(ctx, "invite_user:"+req.InviterID, 5, time.Hour); err != nil {
        return nil, ErrInviteRateLimitExceeded
    }

    // 3. Check for existing pending invite (upsert or reject)
    existing, err := s.inviteRepo.GetPendingForEmail(ctx, req.TenantID, req.InviteeEmail)
    if err == nil {
        // Re-invite: revoke old token, issue new one (updates expiry)
        if err := s.inviteRepo.Revoke(ctx, existing.ID); err != nil {
            return nil, err
        }
    }

    // 4. Generate token
    rawToken, tokenHash, err := generateInviteToken()
    if err != nil {
        return nil, err
    }

    // 5. Persist (store ONLY the hash)
    inv, err := s.inviteRepo.Create(ctx, Invitation{
        TenantID:     req.TenantID,
        InvitedBy:    req.InviterID,
        InviteeEmail: strings.ToLower(req.InviteeEmail),
        Role:         req.Role,
        TokenHash:    tokenHash,
        ExpiresAt:    time.Now().Add(72 * time.Hour),
    })
    if err != nil {
        return nil, err
    }

    // 6. Send email with rawToken (never persisted, only in email)
    return inv, s.mailer.SendInvite(ctx, req.InviteeEmail, rawToken, req.TenantName)
}
```

---

## Step 3 — Invite Acceptance Flow

```go
func (s *InviteService) AcceptInvite(ctx context.Context, rawToken string, acceptingUserID string) error {
    // 1. Resolve token to invite record
    tokenHash := hashToken(rawToken)
    inv, err := s.inviteRepo.GetByTokenHash(ctx, tokenHash)
    if err != nil || inv == nil {
        return ErrInviteInvalid // same error for not found OR wrong token
    }

    // 2. Validate state
    if inv.Status != "pending" {
        return ErrInviteAlreadyUsed
    }
    if time.Now().After(inv.ExpiresAt) {
        s.inviteRepo.Expire(ctx, inv.ID) //nolint: errcheck
        return ErrInviteExpired
    }

    // 3. Validate email binding — accepting user MUST match invitee email
    acceptingUser, err := s.userRepo.GetByID(ctx, acceptingUserID)
    if err != nil {
        return err
    }
    if !strings.EqualFold(acceptingUser.Email, inv.InviteeEmail) {
        // Log: potential invite forwarding attempt
        s.securityLog.Log(ctx, SecurityEvent{
            Type:     "invite_email_mismatch",
            TenantID: inv.TenantID,
            Metadata: map[string]string{
                "invite_email":   inv.InviteeEmail,
                "accepting_user": acceptingUser.Email,
                "invite_id":      inv.ID,
            },
        })
        return ErrInviteEmailMismatch
    }

    // 4. Check invitee is not already a member
    if s.memberRepo.IsMember(ctx, inv.TenantID, acceptingUserID) {
        return ErrAlreadyMember
    }

    // 5. Atomic: add member + mark invite accepted
    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        if err := s.memberRepo.AddTx(ctx, tx, Membership{
            TenantID: inv.TenantID,
            UserID:   acceptingUserID,
            Role:     inv.InviteeRole, // role from DB record — NOT from token or URL
            AddedBy:  inv.InvitedBy,
        }); err != nil {
            return err
        }
        return s.inviteRepo.MarkAcceptedTx(ctx, tx, inv.ID, acceptingUserID)
    })
}
```

---

## Step 4 — Invite for Unregistered Users

When an invitee does not yet have an account:

```go
// Flow:
// 1. Invite created → token sent to email (same as above)
// 2. Invitee clicks link → redirected to registration page with token in URL parameter
// 3. Registration pre-fills the email from the invite (display only; not editable)
// 4. On registration completion: call AcceptInvite with the new user ID
// 5. Email MUST match the invite — validated in AcceptInvite (Step 3, check #3)

// Pre-registration takeover prevention:
// If an attacker registers the email BEFORE the victim accepts:
// → AcceptInvite email check catches the mismatch ONLY if attacker used a different email
// → If attacker registers the exact invited email first:
//    - Mitigate with email verification: require verified email before any account is usable
//    - The invite token is still single-use — attacker cannot use the victim's token
//    - Monitor for: email registered but invite not accepted within short window
```

---

## Step 5 — Invite Revocation and Expiry

```go
// Revoke invite (inviter or admin, before acceptance)
func (s *InviteService) RevokeInvite(ctx context.Context, tenantID, inviteID, revokerID string) error {
    // Verify revoker has permission (must be admin/owner in tenant)
    if err := s.authz.RequireRole(ctx, tenantID, revokerID, "admin"); err != nil {
        return err
    }
    inv, err := s.inviteRepo.GetByID(ctx, inviteID)
    if err != nil || inv.TenantID != tenantID { // explicit tenant check
        return ErrInviteNotFound
    }
    if inv.Status != "pending" {
        return ErrInviteNotRevocable
    }
    return s.inviteRepo.SetStatus(ctx, inviteID, "revoked")
}

// Scheduled job: expire pending invites past their deadline
func (s *InviteExpireJob) Run(ctx context.Context) error {
    n, err := s.inviteRepo.ExpireOlderThan(ctx, time.Now())
    if err != nil {
        return err
    }
    slog.Info("expired invitations cleaned up", slog.Int64("count", n))
    return nil
}
```

---

## Step 6 — Domain Allowlisting (Enterprise)

For tenants that want to restrict invites to specific email domains:

```sql
CREATE TABLE tenant_allowed_domains (
    tenant_id   UUID    NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    domain      TEXT    NOT NULL,  -- e.g., 'acme.com'
    PRIMARY KEY (tenant_id, domain)
);
```

```go
func (s *InviteService) validateDomain(ctx context.Context, tenantID, email string) error {
    allowedDomains, err := s.domainRepo.GetForTenant(ctx, tenantID)
    if err != nil || len(allowedDomains) == 0 {
        return nil // No restriction configured
    }
    parts := strings.SplitN(email, "@", 2)
    if len(parts) != 2 {
        return ErrInvalidEmail
    }
    domain := strings.ToLower(parts[1])
    for _, allowed := range allowedDomains {
        if domain == strings.ToLower(allowed) {
            return nil
        }
    }
    return ErrDomainNotAllowed
}
```

---

## Quality Checks

- [ ] Invite tokens: 32-byte random, URL-safe base64, SHA-256 hash stored — raw token never persisted
- [ ] Token expiry enforced server-side: `expires_at` checked at acceptance — not trusted from URL
- [ ] Single-use: token marked as accepted/revoked atomically before membership is created
- [ ] Email binding: accepting user's verified email must match `invitee_email` — rejects forwarded links
- [ ] Role from DB record: `inv.InviteeRole` used at membership creation — never from token payload or query param
- [ ] Rate limit: per-tenant AND per-inviter limits prevent spam invite campaigns
- [ ] Re-invite: old pending invite revoked before new one issued — no duplicate pending invites
- [ ] Invite revocation requires admin/owner role — verified with explicit tenant scope check
- [ ] Security event logged when email mismatch occurs at acceptance
- [ ] Unique index prevents two pending invites to the same email in the same tenant

## After Completion

Recommended next steps:

- Use **`organization-membership-security`** for secure membership management post-invite
- Use **`authentication-patterns`** for the registration flow invitees land on
- Use **`audit-trail-design`** to make invite creation, acceptance, and revocation auditable
