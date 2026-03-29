---
name: api-key-management
description: 'API key creation, rotation, and restriction practices for SaaS applications. Use when: API key management, API key security, API key rotation, API key creation, API key revocation, API key scoping, API key hashing, API key prefix, API key expiry, API key permissions, API key audit, API key storage, secure API key design, API key lifecycle, API key restriction, per-tenant API key, API key rate limiting, API key leakage, API key exposure, rotate API key, revoke API key.'
argument-hint: 'Describe your API key use cases (machine-to-machine, CI/CD, webhook delivery, third-party integrations), whether keys are tenant-scoped or user-scoped, and any current gaps (plain-text storage, no expiry, no scoping, no audit trail).'
---

# API Key Management

## When to Use

Invoke this skill when you need to:
- Design the full API key lifecycle: generate → authenticate → rotate → revoke
- Store API keys securely (never in plain text; hash like passwords)
- Implement key scoping and permission boundaries to limit blast radius
- Build a key audit trail: creation, last-used, rotation, revocation events
- Enforce expiry and rate limiting per key
- Handle member removal by revoking all their keys

---

## Threat Model: API Key Attack Vectors

| Attack | Mechanism | Defense |
|---|---|---|
| Key leak from DB | Attacker reads `api_keys` table | Never store plain key — store SHA-256 hash only |
| Key leak from source code | Developer commits key to git | Prefix-based scanning; secrets detection in CI |
| Key leak from logs | Key logged in request headers | Mask `Authorization` header in all log output |
| Key forwarding / sharing | Shared beyond intended user | Key is always personal; scopes limit scope of breach |
| Brute force | Guess short / low-entropy keys | 32-byte random; SHA-256 indexed — brute force computationally infeasible |
| Stolen key — no detection | Key used indefinitely | Expiry + last-used tracking + anomaly detection |
| Overprivileged key | Key can do anything | Scope list limits to minimum required permissions |
| Orphaned key after member removal | Key still works after user leaves | Membership removal revokes all user keys atomically |

---

## Step 1 — API Key Data Model

```sql
CREATE TABLE api_keys (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id         UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    display_name    TEXT        NOT NULL,               -- human label: "CI Pipeline", "Zapier Integration"
    key_prefix      TEXT        NOT NULL,               -- e.g., "sk_live_XXXX" — first 8 chars shown in UI
    key_hash        TEXT        NOT NULL UNIQUE,        -- SHA-256 of the full key; used for auth lookup
    scopes          TEXT[]      NOT NULL DEFAULT '{}',  -- e.g., '{read:projects,write:webhooks}'
    status          TEXT        NOT NULL DEFAULT 'active'
                                CHECK (status IN ('active','revoked','suspended','expired')),
    expires_at      TIMESTAMPTZ,                        -- NULL = no expiry (long-lived keys)
    last_used_at    TIMESTAMPTZ,
    last_used_ip    INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at      TIMESTAMPTZ,
    revoked_by      UUID        REFERENCES users(id)
);

CREATE INDEX idx_api_keys_hash   ON api_keys(key_hash);   -- auth lookup
CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id, status);
CREATE INDEX idx_api_keys_user   ON api_keys(user_id, status);

-- Key event audit log
CREATE TABLE api_key_events (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    key_id      UUID        NOT NULL REFERENCES api_keys(id) ON DELETE CASCADE,
    tenant_id   UUID        NOT NULL,
    event_type  TEXT        NOT NULL,  -- 'created'|'used'|'rotated'|'revoked'|'expired'|'suspended'
    actor_id    UUID        REFERENCES users(id),
    ip_address  INET,
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_key_events_key ON api_key_events(key_id, created_at DESC);
```

---

## Step 2 — Key Generation

```go
const (
    KeyPrefix       = "sk_live_"     // visible in UI: identifies key type
    KeyPrefixTest   = "sk_test_"     // test environment keys
    KeyEntropyBytes = 32
)

type GeneratedKey struct {
    ID          string   // UUID stored in DB
    FullKey     string   // "sk_live_{base62(32-random-bytes)}" — shown ONCE to user
    DisplayPrefix string // first 10 chars of FullKey — stored for UI display
    KeyHash     string   // SHA-256(FullKey) — stored in DB for auth
}

func generateAPIKey(prefix string) (*GeneratedKey, error) {
    b := make([]byte, KeyEntropyBytes)
    if _, err := rand.Read(b); err != nil {
        return nil, fmt.Errorf("generating api key entropy: %w", err)
    }
    // Use base62 encoding: URL-safe, copyable, no ambiguous chars
    fullKey := prefix + base62Encode(b)

    hash := sha256.Sum256([]byte(fullKey))
    hashHex := hex.EncodeToString(hash[:])

    return &GeneratedKey{
        FullKey:       fullKey,
        DisplayPrefix: fullKey[:len(prefix)+8], // prefix + first 8 chars
        KeyHash:       hashHex,
    }, nil
}

func (s *APIKeyService) CreateKey(ctx context.Context, req CreateAPIKeyRequest) (*CreateAPIKeyResult, error) {
    actor := auth.UserFromCtx(ctx)

    // 1. Validate scopes are within the actor's own permissions
    if err := s.validateScopes(ctx, actor.TenantID, actor.UserID, req.Scopes); err != nil {
        return nil, err
    }

    // 2. Enforce limit: max N keys per user per tenant
    count, err := s.repo.CountActiveForUser(ctx, actor.TenantID, actor.UserID)
    if err != nil {
        return nil, err
    }
    if count >= s.config.MaxKeysPerUser {
        return nil, ErrKeyLimitReached
    }

    // 3. Generate
    gen, err := generateAPIKey(KeyPrefix)
    if err != nil {
        return nil, err
    }

    // 4. Persist — store ONLY the hash and display prefix
    key, err := s.repo.Create(ctx, APIKey{
        TenantID:    actor.TenantID,
        UserID:      actor.UserID,
        DisplayName: req.DisplayName,
        KeyPrefix:   gen.DisplayPrefix,
        KeyHash:     gen.KeyHash,
        Scopes:      req.Scopes,
        ExpiresAt:   req.ExpiresAt,
    })
    if err != nil {
        return nil, err
    }

    s.auditLog.Log(ctx, APIKeyEvent{
        KeyID: key.ID, TenantID: actor.TenantID,
        EventType: "created", ActorID: actor.UserID,
    })

    return &CreateAPIKeyResult{
        Key:      key,
        FullKey:  gen.FullKey,  // returned ONCE — never retrievable again
    }, nil
}
```

---

## Step 3 — Authentication Lookup

```go
// API key authentication middleware
func APIKeyAuth(repo APIKeyRepository, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        raw := extractAPIKey(r) // from Authorization: Bearer sk_live_... or X-API-Key header
        if raw == "" {
            http.Error(w, "missing api key", http.StatusUnauthorized)
            return
        }

        // Validate prefix before DB lookup (fast reject)
        if !strings.HasPrefix(raw, KeyPrefix) && !strings.HasPrefix(raw, KeyPrefixTest) {
            http.Error(w, "invalid api key format", http.StatusUnauthorized)
            return
        }

        hash := sha256KeyHex(raw)
        key, err := repo.GetByHash(r.Context(), hash)
        if err != nil || key == nil {
            http.Error(w, "invalid api key", http.StatusUnauthorized)
            return
        }

        // Validate status and expiry
        if key.Status != "active" {
            http.Error(w, "api key revoked or suspended", http.StatusUnauthorized)
            return
        }
        if key.ExpiresAt != nil && time.Now().After(*key.ExpiresAt) {
            repo.SetStatus(r.Context(), key.ID, "expired") //nolint: errcheck — best effort
            http.Error(w, "api key expired", http.StatusUnauthorized)
            return
        }

        // Update last_used async — don't add latency to the hot path
        go repo.UpdateLastUsed(context.Background(), key.ID, realIP(r))

        ctx := auth.WithAPIKey(r.Context(), key)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func sha256KeyHex(key string) string {
    h := sha256.Sum256([]byte(key))
    return hex.EncodeToString(h[:])
}
```

---

## Step 4 — Scope Enforcement

```go
// Scope constants — define all valid scopes
const (
    ScopeReadProjects  = "read:projects"
    ScopeWriteProjects = "write:projects"
    ScopeReadMembers   = "read:members"
    ScopeWriteWebhooks = "write:webhooks"
    ScopeReadBilling   = "read:billing"
    ScopeAdmin         = "admin"            // full access — require explicit justification
)

// Scope check middleware
func RequireScope(scope string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := auth.APIKeyFromCtx(r.Context())
            if key == nil {
                // User session — scopes don't apply; defer to RBAC
                next.ServeHTTP(w, r)
                return
            }
            for _, s := range key.Scopes {
                if s == scope || s == ScopeAdmin {
                    next.ServeHTTP(w, r)
                    return
                }
            }
            http.Error(w, "insufficient api key scope", http.StatusForbidden)
        })
    }
}

// Usage: router.With(RequireScope(ScopeWriteProjects)).Post("/projects", handler)
```

---

## Step 5 — Key Rotation

```go
// Rotation generates a new key, revokes the old one atomically
func (s *APIKeyService) RotateKey(ctx context.Context, tenantID, keyID string) (*CreateAPIKeyResult, error) {
    actor := auth.UserFromCtx(ctx)

    old, err := s.repo.GetByID(ctx, tenantID, keyID) // scoped by tenantID — no IDOR
    if err != nil {
        return nil, ErrKeyNotFound
    }
    if old.UserID != actor.UserID && !actor.HasRole("admin") {
        return nil, ErrForbidden
    }

    gen, err := generateAPIKey(KeyPrefix)
    if err != nil {
        return nil, err
    }

    var newKey *APIKey
    err = s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // Create new key with same config
        nk, err := s.repo.CreateTx(ctx, tx, APIKey{
            TenantID: old.TenantID, UserID: old.UserID,
            DisplayName: old.DisplayName + " (rotated)",
            KeyPrefix: gen.DisplayPrefix, KeyHash: gen.KeyHash,
            Scopes: old.Scopes, ExpiresAt: old.ExpiresAt,
        })
        if err != nil {
            return err
        }
        newKey = nk

        // Revoke old key
        return s.repo.SetStatusTx(ctx, tx, old.ID, "revoked", actor.UserID)
    })
    if err != nil {
        return nil, err
    }

    s.auditLog.Log(ctx, APIKeyEvent{
        KeyID: old.ID, TenantID: tenantID,
        EventType: "rotated", ActorID: actor.UserID,
        Metadata: map[string]string{"new_key_id": newKey.ID},
    })

    return &CreateAPIKeyResult{Key: newKey, FullKey: gen.FullKey}, nil
}
```

---

## Step 6 — Key Display in UI

```
What to show in the API keys list:
  ✅ display_name          "CI Pipeline"
  ✅ key_prefix            "sk_live_aB3d..."   (first 8 chars after prefix)
  ✅ scopes                ['read:projects', 'write:webhooks']
  ✅ created_at            2026-01-15
  ✅ last_used_at          2 hours ago
  ✅ expires_at            2027-01-15 (or "Never")
  ✅ status                Active / Revoked

What NEVER to show after initial creation:
  ❌ full key              (shown exactly once at creation — not retrievable)
  ❌ key_hash              (internal; never exposed)
```

---

## Quality Checks

- [ ] Keys: 32-byte random, prefixed (`sk_live_`), SHA-256 hashed — never stored plain
- [ ] Full key shown to user exactly ONCE at creation — not retrievable via any API endpoint
- [ ] Auth lookup hashes the incoming key before querying — constant-time comparison via indexed hash
- [ ] Status and expiry validated on every auth request — expired keys auto-marked
- [ ] Scopes validated at creation: actor cannot grant scopes beyond their own permissions
- [ ] `RequireScope` middleware used on every API key-accessible route
- [ ] Key rotation is atomic: new key created + old key revoked in a single transaction
- [ ] Member removal revokes ALL user keys in the tenant atomically (see `organization-membership-security`)
- [ ] `last_used_at` updated asynchronously — no latency impact on auth hot path
- [ ] Audit log records: created, used (on anomaly), rotated, revoked, expired events

## After Completion

Recommended next steps:

- Use **`secrets-exposure-detection`** to scan for keys accidentally committed to git or logs
- Use **`organization-membership-security`** to ensure key revocation is wired to member removal
- Use **`rate-limiting-strategies`** to apply per-key rate limits
- Use **`audit-trail-design`** to make the key event log tamper-evident
