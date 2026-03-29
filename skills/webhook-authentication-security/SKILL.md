---
name: webhook-authentication-security
description: 'Secure webhook verification and validation for SaaS applications. Use when: webhook authentication security, webhook signature verification, webhook HMAC, webhook secret rotation, webhook replay prevention, secure webhook endpoint, webhook payload validation, webhook origin validation, webhook timing attack, webhook secret management, outbound webhook security, inbound webhook security, webhook signing, webhook shared secret, webhook delivery security, webhook receiver security, webhook forgery prevention.'
argument-hint: 'Clarify whether you are building an inbound webhook receiver (validating payloads from a third party like Stripe or GitHub) or an outbound webhook sender (delivering events to customer-configured URLs). Describe current validation mechanism, if any.'
---

# Webhook Authentication Security

## When to Use

Invoke this skill when you need to:
- Verify the authenticity of inbound webhook payloads from third-party providers (Stripe, GitHub, etc.)
- Build a secure outbound webhook delivery system with HMAC signatures for your customers
- Prevent replay attacks, timing attacks, and spoofed webhook payloads
- Design webhook secret rotation without service interruption
- Validate webhook endpoint URLs to prevent SSRF via customer-configured targets
- Monitor webhook delivery health and implement secure retry logic

---

## Part A — Inbound Webhook Security (receiving from third parties)

### Step A1 — Signature Verification Pattern

Every inbound webhook MUST be verified before processing. Raw body must be captured before parsing.

```go
// Middleware: verify Stripe-style HMAC-SHA-256 signature
// Works for: Stripe, GitHub, Linear, Shopify, and most major providers
func WebhookSignatureMiddleware(secret string, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. Read raw body (must happen before json.Decode)
        body, err := io.ReadAll(io.LimitReader(r.Body, 1<<20)) // 1 MB max
        if err != nil {
            http.Error(w, "failed to read body", http.StatusBadRequest)
            return
        }
        r.Body = io.NopCloser(bytes.NewReader(body)) // restore for next handler

        // 2. Extract signature header (format varies by provider — adapt as needed)
        sigHeader := r.Header.Get("Stripe-Signature") // or X-Hub-Signature-256, X-Signature, etc.
        if sigHeader == "" {
            http.Error(w, "missing signature", http.StatusUnauthorized)
            return
        }

        // 3. Verify HMAC
        if err := verifyStripeSignature(body, sigHeader, secret); err != nil {
            http.Error(w, "invalid signature", http.StatusUnauthorized)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func verifyStripeSignature(payload []byte, sigHeader, secret string) error {
    // Stripe format: "t=1716000000,v1=abc123..."
    parts := strings.Split(sigHeader, ",")
    var timestamp, signature string
    for _, p := range parts {
        kv := strings.SplitN(p, "=", 2)
        if len(kv) != 2 { continue }
        switch kv[0] {
        case "t":  timestamp = kv[1]
        case "v1": signature = kv[1]
        }
    }
    if timestamp == "" || signature == "" {
        return errors.New("malformed signature header")
    }

    // Replay prevention: reject events older than 5 minutes
    ts, err := strconv.ParseInt(timestamp, 10, 64)
    if err != nil || time.Since(time.Unix(ts, 0)) > 5*time.Minute {
        return errors.New("webhook timestamp too old — possible replay attack")
    }

    // Reconstruct signed payload
    signed := fmt.Sprintf("%s.%s", timestamp, payload)

    // Constant-time HMAC comparison (prevents timing attacks)
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write([]byte(signed))
    expected := hex.EncodeToString(mac.Sum(nil))

    if !hmac.Equal([]byte(expected), []byte(signature)) {
        return errors.New("signature mismatch")
    }
    return nil
}
```

**Provider-specific header names:**

| Provider | Signature Header | Algorithm | Timestamp field |
|---|---|---|---|
| Stripe | `Stripe-Signature` | HMAC-SHA-256, `v1=` | `t=` in same header |
| GitHub | `X-Hub-Signature-256` | HMAC-SHA-256, `sha256=` prefix | Not included — use `X-GitHub-Delivery` for dedup |
| Shopify | `X-Shopify-Hmac-Sha256` | HMAC-SHA-256, base64 | Not included |
| Linear | `Linear-Signature` | HMAC-SHA-256 | Not included |
| Custom | `X-Webhook-Signature` | HMAC-SHA-256 | Include timestamp in payload |

---

### Step A2 — Idempotent Event Processing

```go
// Prevent duplicate processing — store processed delivery IDs
// (for Stripe: event.ID; for GitHub: X-GitHub-Delivery UUID)
func (h *WebhookHandler) IsAlreadyProcessed(ctx context.Context, deliveryID string) (bool, error) {
    _, err := h.db.QueryRowContext(ctx,
        `SELECT 1 FROM processed_webhooks WHERE delivery_id = $1`, deliveryID).Scan(new(int))
    if errors.Is(err, sql.ErrNoRows) {
        return false, nil
    }
    return err == nil, err
}

func (h *WebhookHandler) MarkProcessed(ctx context.Context, deliveryID, provider string) error {
    _, err := h.db.ExecContext(ctx,
        `INSERT INTO processed_webhooks (delivery_id, provider, processed_at)
         VALUES ($1, $2, NOW())
         ON CONFLICT (delivery_id) DO NOTHING`,
        deliveryID, provider)
    return err
}
```

---

## Part B — Outbound Webhook Security (sending to customers)

### Step B1 — Webhook Secret Management

```sql
CREATE TABLE webhook_endpoints (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    url             TEXT        NOT NULL,
    display_url     TEXT,                   -- masked: "https://example.com/hook/****"
    secret_hash     TEXT        NOT NULL,   -- SHA-256 of the raw secret; raw secret shown once
    secret_prefix   TEXT        NOT NULL,   -- first 8 chars of secret for UI identification
    event_types     TEXT[]      NOT NULL DEFAULT '{}',  -- subscribed events; empty = all
    status          TEXT        NOT NULL DEFAULT 'active'
                                CHECK (status IN ('active','paused','disabled')),
    failure_count   INT         NOT NULL DEFAULT 0,
    last_success_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      UUID        REFERENCES users(id)
);
```

```go
// Generate a webhook signing secret (shown once to customer; stored as hash)
func generateWebhookSecret() (rawSecret, secretHash, prefix string, err error) {
    b := make([]byte, 32)
    if _, err = rand.Read(b); err != nil {
        return
    }
    rawSecret = "whsec_" + base64.StdEncoding.EncodeToString(b) // "whsec_" prefix identifies type
    hash := sha256.Sum256([]byte(rawSecret))
    secretHash = hex.EncodeToString(hash[:])
    prefix = rawSecret[:14] // "whsec_" + 8 chars
    return
}
```

### Step B2 — Outbound Payload Signing

```go
type WebhookDelivery struct {
    endpointURL string
    secretHash  string // looked up by computing sha256 of candidate — or store secret encrypted
    payload     []byte
    eventType   string
    deliveryID  string
}

func (s *WebhookSender) SignAndSend(ctx context.Context, d WebhookDelivery) error {
    timestamp := strconv.FormatInt(time.Now().Unix(), 10)

    // Reconstruct secret from hash is not possible — store encrypted secret or use KMS
    // Pattern: store secret in encrypted column (AES-GCM with KMS-managed key)
    rawSecret, err := s.secretStore.Decrypt(ctx, d.secretHash)
    if err != nil {
        return fmt.Errorf("decrypting webhook secret: %w", err)
    }

    // Build signed payload: "{timestamp}.{body}"
    signed := fmt.Sprintf("%s.%s", timestamp, d.payload)
    mac := hmac.New(sha256.New, []byte(rawSecret))
    mac.Write([]byte(signed))
    sig := hex.EncodeToString(mac.Sum(nil))

    req, err := http.NewRequestWithContext(ctx, http.MethodPost, d.endpointURL, bytes.NewReader(d.payload))
    if err != nil {
        return err
    }
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Webhook-Timestamp", timestamp)
    req.Header.Set("X-Webhook-Signature", "v1="+sig)
    req.Header.Set("X-Webhook-ID", d.deliveryID)
    req.Header.Set("X-Webhook-Event", d.eventType)
    req.Header.Set("User-Agent", "YourApp-Webhooks/1.0")

    // Timeout: never let customer endpoints block delivery workers
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return fmt.Errorf("delivering webhook: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return fmt.Errorf("endpoint returned %d", resp.StatusCode)
    }
    return nil
}
```

### Step B3 — Customer Endpoint URL Validation (SSRF Prevention)

```go
// Validate customer-supplied webhook URLs before registering
func validateWebhookURL(rawURL string) error {
    u, err := url.Parse(rawURL)
    if err != nil {
        return ErrInvalidURL
    }

    // Allowed schemes only
    if u.Scheme != "https" {
        return errors.New("webhook URL must use HTTPS")
    }

    // Block private/loopback/cloud-metadata addresses
    host := u.Hostname()
    if isPrivateHost(host) {
        return ErrPrivateAddressNotAllowed
    }

    return nil
}

func isPrivateHost(host string) bool {
    privateRanges := []*net.IPNet{
        cidr("10.0.0.0/8"),
        cidr("172.16.0.0/12"),
        cidr("192.168.0.0/16"),
        cidr("127.0.0.0/8"),
        cidr("169.254.0.0/16"),  // link-local / AWS metadata
        cidr("::1/128"),          // IPv6 loopback
        cidr("fc00::/7"),         // IPv6 private
    }
    // Also block: "localhost", "metadata.google.internal", "169.254.169.254"
    if host == "localhost" || strings.HasSuffix(host, ".internal") {
        return true
    }
    ip := net.ParseIP(host)
    if ip == nil {
        return false // domain — DNS-based SSRF needs separate protection (redirect checking)
    }
    for _, r := range privateRanges {
        if r.Contains(ip) {
            return true
        }
    }
    return false
}
```

### Step B4 — Secret Rotation Without Interruption

```go
// Rotation strategy: dual-secret grace period (old + new both valid for 24h)
type WebhookRotationResult struct {
    NewSecret string      // raw secret — shown once
    ExpiresAt time.Time   // when old secret expires
}

func (s *WebhookService) RotateSecret(ctx context.Context, tenantID, endpointID string) (*WebhookRotationResult, error) {
    // Generate new secret
    rawNew, hashNew, prefixNew, err := generateWebhookSecret()
    if err != nil {
        return nil, err
    }

    // Store new secret alongside old one with 24-hour grace window
    // Customer has 24h to update their verification code to the new secret
    gracePeriod := time.Now().Add(24 * time.Hour)
    if err := s.repo.SetPendingRotation(ctx, tenantID, endpointID, hashNew, prefixNew, gracePeriod); err != nil {
        return nil, err
    }

    // During delivery: try new secret first; fall back to old if still in grace period
    return &WebhookRotationResult{NewSecret: rawNew, ExpiresAt: gracePeriod}, nil
}
```

---

## Quality Checks

**Inbound (receiving):**
- [ ] Raw body captured BEFORE `json.Decode` — signature computed over literal bytes
- [ ] Timestamp extracted and validated: reject events older than 5 minutes (replay prevention)
- [ ] HMAC comparison uses `hmac.Equal` (constant-time) — never `==` string comparison
- [ ] Delivery ID stored and checked for idempotency before processing
- [ ] Return HTTP 200 immediately after signature verification; process asynchronously

**Outbound (sending):**
- [ ] Customer webhook secrets: 32-byte random, `whsec_` prefixed, stored encrypted (not plain, not as hash alone)
- [ ] Signature includes timestamp in signed payload — customers can reject replays
- [ ] Customer endpoint URLs validated against private IP/hostname blocklist before storage
- [ ] HTTPS enforced for all customer endpoint URLs
- [ ] HTTP client timeout set (10s max) — never let customer endpoints block delivery workers
- [ ] Failed deliveries retried with exponential backoff; endpoint disabled after N consecutive failures
- [ ] Secret rotation uses dual-secret grace period — customers have time to update

## After Completion

Recommended next steps:

- Use **`ssrf-protection`** for deeper SSRF defense on customer-supplied URLs
- Use **`api-key-management`** to apply similar signing secret lifecycle practices
- Use **`idempotency-patterns`** to make webhook event processing fully replay-safe
- Use **`secrets-management`** for KMS-backed encrypted storage of webhook signing secrets
