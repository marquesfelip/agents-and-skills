---
name: ssrf-protection
description: 'Outbound request validation, internal network protection, and SSRF attack mitigation. Use when: SSRF, server-side request forgery, outbound request validation, internal network access, cloud metadata endpoint, URL validation, allowlist outbound, SSRF prevention, DNS rebinding, open redirect SSRF, webhook SSRF, URL fetch security, curl security, HTTP client security.'
argument-hint: 'Code making outbound HTTP requests or accepting URLs as input — or describe the URL-fetch feature being designed'
---

# SSRF Protection Specialist

## When to Use
- Reviewing any code that accepts user-supplied URLs and makes outbound HTTP requests
- Auditing webhook receivers, URL preview features, image proxies, or link unfurlers
- Designing safe outbound request patterns for integrations and crawlers
- Protecting cloud-hosted services against metadata endpoint exposure
- Preventing internal network enumeration via SSRF

---

## Step 1 — Identify SSRF Attack Surface

SSRF exists wherever **user-controlled input influences a server-side outbound request**.

Common entry points:
| Feature | SSRF vector |
|---|---|
| Webhook registration | User provides callback URL |
| URL preview / link unfurl | User shares URL; server fetches it |
| Image/avatar from URL | Server downloads image from user-provided URL |
| PDF/screenshot generation | Headless browser fetches user-supplied URL |
| Import from URL | Server fetches file from user-specified source |
| Proxy endpoint | Server fetches and returns remote resource |
| XML/SVG processing | External entities reference URLs (XXE-to-SSRF) |
| OAuth redirect | Poorly validated redirect URLs |

---

## Step 2 — What Attackers Target via SSRF

| Target | URL | Risk |
|---|---|---|
| Cloud metadata (AWS) | `http://169.254.169.254/latest/meta-data/iam/security-credentials/` | IAM credentials, instance role tokens |
| Cloud metadata (GCP) | `http://metadata.google.internal/computeMetadata/v1/` | Service account tokens, project info |
| Cloud metadata (Azure) | `http://169.254.169.254/metadata/instance` | Managed identity tokens |
| Kubernetes API | `https://kubernetes.default.svc/` | Cluster control plane access |
| Internal services | `http://internal-db:5432`, `http://redis:6379` | Read internal data, pivot attacks |
| Localhost | `http://localhost:8080/admin` | Bypass authentication on local admin UIs |
| Internal DNS records | `http://billing.internal/api/admin` | Access internal-only services |

---

## Step 3 — Defense Strategy

Use **both** an allowlist and an IP-level block — neither alone is sufficient.

### Layer 1: URL Allowlist (best for known targets)

```go
// GOOD — explicit allowlist of known providers
var allowedHosts = map[string]bool{
    "api.github.com":  true,
    "hooks.slack.com": true,
}

func validateURL(raw string) error {
    u, err := url.Parse(raw)
    if err != nil {
        return errors.New("invalid URL")
    }
    if u.Scheme != "https" {
        return errors.New("only HTTPS allowed")
    }
    host := strings.ToLower(u.Hostname())
    if !allowedHosts[host] {
        return fmt.Errorf("host not allowed: %s", host)
    }
    return nil
}
```

### Layer 2: IP Blocklist (for open-ended URL acceptance)

When you can't maintain a strict allowlist (e.g., user webhooks):

```go
func isSafeIP(ip net.IP) bool {
    // Block all private, loopback, link-local, and metadata ranges
    blockedCIDRs := []string{
        "127.0.0.0/8",       // loopback
        "10.0.0.0/8",        // RFC 1918
        "172.16.0.0/12",     // RFC 1918
        "192.168.0.0/16",    // RFC 1918
        "169.254.0.0/16",    // link-local / cloud metadata
        "100.64.0.0/10",     // shared address space
        "::1/128",           // IPv6 loopback
        "fc00::/7",          // IPv6 unique local
        "fe80::/10",         // IPv6 link-local
        "0.0.0.0/8",         // "this" network
    }
    for _, cidr := range blockedCIDRs {
        _, network, _ := net.ParseCIDR(cidr)
        if network.Contains(ip) {
            return false
        }
    }
    return true
}
```

### Layer 3: DNS Resolution Check (Prevent DNS Rebinding)

Resolve the hostname at validation time AND at request time or use the resolved IP directly:

```
1. Resolve hostname to IP(s)
2. Check IP(s) against blocklist
3. Make request using resolved IP (bypass DNS rebinding)
   — Set `Host` header to original hostname for SNI/virtual hosting
```

---

## Step 4 — Scheme and Protocol Restrictions

Block non-HTTP(S) schemes that can read local files or internal resources:

```
Blocked schemes: file://, gopher://, ftp://, dict://, ldap://, sftp://, tftp://, jar://

Only allow: https:// (prefer) and http:// if required
```

```python
# GOOD
from urllib.parse import urlparse

def is_safe_url(url: str) -> bool:
    parsed = urlparse(url)
    return parsed.scheme in ('http', 'https') and bool(parsed.netloc)

# BAD — no scheme check
requests.get(user_url)   # attacker passes "file:///etc/passwd"
```

---

## Step 5 — Safe HTTP Client Configuration

Configure the HTTP client to enforce SSRF constraints at the transport level:

```go
// Go — safe HTTP client for outbound user-supplied requests
transport := &http.Transport{
    DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
        host, port, _ := net.SplitHostPort(addr)
        ips, err := net.DefaultResolver.LookupHost(ctx, host)
        if err != nil {
            return nil, err
        }
        for _, ipStr := range ips {
            ip := net.ParseIP(ipStr)
            if !isSafeIP(ip) {
                return nil, fmt.Errorf("blocked: %s resolves to private IP %s", host, ipStr)
            }
        }
        return (&net.Dialer{Timeout: 5 * time.Second}).DialContext(ctx, network, addr)
    },
    DisableKeepAlives:   true,
    MaxIdleConns:        10,
    IdleConnTimeout:     30 * time.Second,
}
client := &http.Client{
    Transport: transport,
    Timeout:   10 * time.Second,
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
        // Validate each redirect target — attackers chain redirects to bypass initial check
        return validateURL(req.URL.String())
    },
}
```

**Key settings:**
- [ ] Request timeout (prevent hanging connections to slow internal services)
- [ ] Redirect validation (each hop must pass SSRF checks)
- [ ] Max redirect count (prevent redirect loops)
- [ ] Response size limit (prevent memory exhaustion from large responses)

---

## Step 6 — Redirect Chain SSRF

Attackers bypass initial URL validation by redirecting to an internal target:

```
Attacker registers: https://attacker.com/redirect → 302 → http://169.254.169.254/
Server validates: https://attacker.com ✓ (passes)
Server follows redirect → reaches metadata endpoint ✗
```

**Defense:**
- Validate every redirect target, not just the initial URL.
- Prefer following redirects only within the original allowlisted domain.
- Cap redirect chain at 3–5 hops; abort on policy violation.

---

## Step 7 — Cloud-Specific Mitigations

### AWS
- Enable IMDSv2 (token-required metadata access) — SSRF alone cannot retrieve credentials.
- Attach minimal IAM roles to compute instances.
- Use VPC endpoint policies to restrict metadata access.

### GCP
- Require `Metadata-Flavor: Google` header for metadata access (not exploitable via SSRF without custom headers).
- Use Workload Identity Federation instead of service account key files.

### Azure
- Restrict IMDS with network policies.
- Use Managed Identity with minimal RBAC assignments.

---

## Step 8 — Output Report

```
## SSRF Review: <service/feature>

### Critical
- POST /api/webhooks — user-supplied URL fetched server-side with no host validation
  Attacker can target: http://169.254.169.254/ to retrieve AWS IAM credentials
  Fix: resolve hostname; block RFC1918, loopback, and link-local IP ranges

- /proxy?url= — accepts file:// scheme
  `file:///etc/passwd` readable by server process
  Fix: allowlist http:// and https:// only; reject all other schemes

### High
- Redirect following not validated — attacker can bypass initial host check via 302
  Fix: validate each redirect target through the same IP blocklist

### Medium
- No request timeout set on outbound HTTP client — internal service enumeration possible via timing
  Fix: set 5-second timeout; log and alert on timeouts to internal-range IPs

### Low
- AWS EC2 instance still uses IMDSv1 (no token required)
  Fix: enable IMDSv2 on all instances (aws ec2 modify-instance-metadata-options --http-tokens required)

### Passed
- Webhook URLs validated against scheme allowlist (https only) ✓
- Private IP ranges blocked at transport layer ✓
- Max 3 redirects; each redirect validated ✓
```
