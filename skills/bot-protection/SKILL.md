---
name: bot-protection
description: 'Automated abuse detection, bot mitigation strategies, and behavioral protections. Use when: bot protection, bot detection, automated abuse, CAPTCHA, credential stuffing bots, scraping bots, bot mitigation, browser fingerprinting, headless browser detection, behavioral analysis, account takeover bot, form spam bot, bot traffic, automated attack prevention.'
argument-hint: 'Endpoint, form, or authentication flow targeted by bots — or describe the bot protection strategy being designed'
---

# Bot Protection Specialist

## When to Use
- Protecting login, registration, checkout, or forms from automated abuse
- Detecting and mitigating credential stuffing, scraping, or spam bots
- Designing CAPTCHA or challenge strategies for public-facing flows
- Reviewing bot detection signals and behavioral analysis implementation
- Building bot protection at the application, gateway, or CDN layer

---

## Step 1 — Classify the Bot Threat

Different bots require different defenses:

| Bot type | Target | Motivation |
|---|---|---|
| Credential stuffing | Login endpoint | Account takeover using leaked credentials |
| Account creation bots | Registration | Fake accounts for spam, fraud, abuse |
| Scraping bots | Public content pages, APIs | Steal data, pricing, inventory |
| Carding / card testing | Payment checkout | Test stolen card numbers |
| Comment / form spam | Contact forms, reviews | SEO spam, phishing links |
| Gift card / voucher abuse | Promo code endpoints | Enumerate valid codes |
| Scalper bots | Purchase / inventory | Buy limited items for resale |
| Form fill bots | Any form | Automated submissions |

---

## Step 2 — Detection Signal Layers

No single signal is sufficient — combine multiple layers:

### Layer 1: Passive Signals (no user friction)

| Signal | Indicator of bot |
|---|---|
| Request rate | Abnormally high frequency from single IP/account |
| User-Agent | Empty, headless browser UA, known bot libraries (`python-requests`, `Go-http-client`, `curl`) |
| Header ordering | Bots don't replicate exact browser header order |
| TLS fingerprint (JA3/JA4) | Non-browser TLS fingerprints |
| IP reputation | Datacenter IPs, known bad ASNs, Tor exit nodes, VPN endpoints |
| Cookie behavior | Absence of long-lived cookies; no cookie jar behavior |
| JavaScript execution | Absence of JS events (mouse movement, scroll) or inability to execute JS |
| Timing patterns | Exactly periodic requests; millisecond-precision regularity |
| Geography | Single account logging in from 20 countries in 1 hour |
| Device fingerprint | Canvas, WebGL, AudioContext fingerprinting |

### Layer 2: Behavioral Signals

| Signal | Detection |
|---|---|
| Form fill speed | Human fills forms in seconds; bots in milliseconds |
| Mouse/touch movement | Bots either have none or move in straight lines |
| Scroll behavior | Bots don't scroll before submitting |
| Honeypot interaction | Bots fill hidden fields; humans don't see them |
| Navigation path | Bots skip intermediate pages; go directly to target |
| Interaction with hidden elements | Bots click invisible elements |

### Layer 3: Application-Level Signals

| Signal | Detection |
|---|---|
| High error rate | Credential stuffing generates many failed logins |
| Credential correlation | Same password tried across many accounts |
| Multi-account same device | Many accounts from one browser fingerprint |
| Unusual hours | Bot attacks often happen off-business-hours |
| Impossible velocity | Same user in two distant geolocations minutes apart |

---

## Step 3 — Challenge Strategies

### Honeypot Fields (zero friction)

```html
<!-- Hidden field — humans don't see or fill it; bots do -->
<form method="POST" action="/register">
  <input type="text" name="username" />
  <input type="email" name="email" />

  <!-- Honeypot — hidden from real users via CSS -->
  <input type="text" name="phone_number" style="display:none" tabindex="-1" autocomplete="off" />

  <button type="submit">Register</button>
</form>
```

```
# Server-side: reject if honeypot field is non-empty
if request.form.get('phone_number'):
    return 400   # Bot detected — fail silently or return fake success
```

### Timing Check (zero friction)

```
# Record form render time; reject if submitted too fast
render_time = cache.get(f"form:{session_id}:render_ts")
submit_time = time.time()
if submit_time - render_time < 1.5:   # submitted in < 1.5 seconds
    increment_bot_score(session_id)
    apply_challenge()
```

### CAPTCHA (user friction — use progressive)

| CAPTCHA type | Friction | Bot resistance | Use when |
|---|---|---|---|
| Invisible reCAPTCHA v3 | None | Medium | First layer; score-based |
| reCAPTCHA v2 checkbox ("I'm not a robot") | Low | Medium | On suspicious sessions |
| Image/text challenge reCAPTCHA | High | High | On high-risk actions or bad scores |
| hCaptcha | Low–Medium | Medium-High | Privacy-preserving alternative |
| Cloudflare Turnstile | None–Low | High | CDN-integrated; good privacy |
| Math / text challenge (custom) | Medium | Low | Avoid — easily solved by bots |

**Progressive CAPTCHA strategy:**
1. Default: invisible challenge (score-based, no friction)
2. Suspicious: visible checkbox challenge
3. High risk (repeated failures, bad score): image challenge or block
4. Reset score after legitimate activity

---

## Step 4 — Credential Stuffing Defense

Credential stuffing is the #1 bot attack on login endpoints. Defense layers:

```
Layer 1: Rate limiting (see rate-limiting-strategies skill)
  - 5 attempts / 15 min per account
  - 20 attempts / min per IP

Layer 2: IP reputation blocking
  - Block or challenge known datacenter/VPN/proxy IPs on login
  - Use IP threat intelligence feeds (MaxMind, IPQualityScore, Cloudflare)

Layer 3: Device fingerprinting
  - Track browser fingerprint per account
  - Alert on new device login; offer re-auth

Layer 4: Breached password detection
  - Check submitted password against HaveIBeenPwned on login
  - Force reset if breached: "Your password was found in a data breach"

Layer 5: Anomaly detection
  - Flag accounts with login attempts from many IPs
  - Flag impossible velocity (same account, two distant countries, < 1 hour)

Layer 6: MFA
  - Even if credentials are correct, 2FA blocks account takeover
```

---

## Step 5 — Scraping Defense

| Defense | Implementation |
|---|---|
| Rate limiting by IP + user | Limit page or API requests per minute |
| Login wall | Require auth to access valuable data |
| Randomize HTML structure | Makes scraper selectors brittle |
| Anti-scraping headers | `X-Robots-Tag: noindex, nofollow` (polite bots) |
| robots.txt | Defines allowed crawl paths (only respected by polite bots) |
| CDN bot management | Cloudflare Bot Management, Akamai, Fastly — fingerprint + challenge |
| Response delay | Introduce artificial delay after N requests — costly for scrapers, invisible to users |
| Data watermarking | Embed unique identifiers in responses to trace leaked scrapes |

---

## Step 6 — API Bot Protection

For API endpoints with no browser context:

| Mechanism | Description |
|---|---|
| API keys | All API access requires a key; revoke abusive keys |
| JWT with short expiry | Tokens not easily shared between bot instances |
| Request signing (HMAC) | Sign request parameters; bots can't forge signatures without the secret |
| IP allowlisting (B2B) | Known partner IPs only |
| Mutual TLS (mTLS) | Certificate-authenticated clients only |
| Rate limiting per key | See rate-limiting-strategies skill |

---

## Step 7 — Bot Protection at the Edge (CDN/Gateway)

Prefer applying bot detection as close to the edge as possible:

| Layer | Tools |
|---|---|
| CDN | Cloudflare Bot Management, Akamai Bot Manager, AWS WAF Bot Control |
| API Gateway | Kong with rate-limit / bot plugins, AWS API Gateway WAF |
| Application | In-process signal collection + challenge; CAPTCHA SDKs |

**Advantage of edge detection**: blocks bots before they reach your origin servers, reducing load and cost.

---

## Step 8 — Output Report

```
## Bot Protection Review: <application/endpoint>

### Critical
- POST /login — no CAPTCHA, no rate limiting, no bot signals collected
  Credential stuffing attacks succeed without friction
  Fix: add sliding window rate limit (5/15min per account) + invisible CAPTCHA + IP reputation check

- POST /register — no honeypot fields, no timing check
  Mass fake account creation freely possible
  Fix: add honeypot field + form timing check + email verification requirement

### High
- No IP reputation filtering on auth endpoints
  Bot attacks from known datacenter IPs undetected
  Fix: integrate IP threat intelligence; apply CAPTCHA challenge for datacenter IPs

### Medium
- reCAPTCHA v3 score threshold set to 0 (effectively disabled)
  All score thresholds passing; no challenges triggered
  Fix: set threshold at 0.5; trigger v2 checkbox for scores below 0.5

- No breached password check on login
  Credential stuffing succeeds with valid leaked credentials
  Fix: integrate HaveIBeenPwned API; force password reset on match

### Low
- No new-device login notification to users
  Account takeover not surfaced to legitimate user
  Fix: send email on first login from new device/browser fingerprint

### Passed
- Honeypot fields on contact form ✓
- API endpoints require signed requests (HMAC) ✓
- Cloudflare Bot Management enabled on public pages ✓
```
