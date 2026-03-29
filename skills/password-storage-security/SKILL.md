---
name: password-storage-security
description: 'Password hashing, salting, algorithm selection, credential storage safety, and breach mitigation strategies. Use when: password storage, password hashing, bcrypt, Argon2, scrypt, PBKDF2, password security, salt generation, credential storage, password breach, credential stuffing prevention, HaveIBeenPwned, password policy, MD5 password, SHA1 password, insecure password storage, password database.'
argument-hint: 'Password hashing/storage code to review — or describe the credential management system being designed'
---

# Password Storage Security Specialist

## When to Use
- Reviewing password hashing implementation for correctness and strength
- Migrating from insecure hashing (MD5, SHA-1, unsalted SHA-256) to a proper algorithm
- Designing a credential storage system from scratch
- Implementing breach detection (HaveIBeenPwned integration)
- Auditing password policy, reset flow, and credential lifecycle

---

## Step 1 — Algorithm Selection

Use a **password hashing function** (slow by design) — never a general-purpose hash:

| Algorithm | Recommendation | Parameters |
|---|---|---|
| **Argon2id** | First choice (OWASP recommended) | m=64MB, t=3, p=4 minimum; tune to ~500ms on target hardware |
| **bcrypt** | Acceptable; well-supported | cost factor ≥12 (aim for ~300ms); max input 72 bytes |
| **scrypt** | Acceptable alternative | N=32768, r=8, p=1 minimum |
| **PBKDF2-SHA256** | Use only when FIPS compliance required | ≥600,000 iterations (NIST SP 800-132 2023) |

**Never use for passwords:**
- MD5, SHA-1, SHA-256, SHA-512 (even with salt) — these are fast; GPUs crack them trivially.
- RC4, DES, any encryption algorithm — passwords must be hashed, not encrypted.
- Unsalted hashes of any kind — enable rainbow table attacks.
- Custom or home-grown hashing schemes.

---

## Step 2 — Salt Requirements

Salts prevent two threats: rainbow table attacks and identical-password correlation.

| Property | Requirement |
|---|---|
| Generation | CSPRNG — never `rand()`, sequential, or user-derived |
| Length | ≥128 bits (16 bytes minimum; 32 bytes recommended) |
| Uniqueness | One unique salt per password, per user — regenerated on every password change |
| Storage | Store alongside the hash in the same record — salts are not secret |
| Format | Most libraries manage salt internally (bcrypt encodes salt in the hash string) |

```
# GOOD — bcrypt handles salt internally
hash = bcrypt.GenerateFromPassword([]byte(password), 12)

# BAD — manual static salt
hash = sha256(static_salt + password)
```

---

## Step 3 — Secure Implementation Checklist

### Storage
- [ ] Hash stored as a string in the DB (e.g., `$2a$12$...` for bcrypt, `$argon2id$...` for Argon2id)
- [ ] Password field is NOT nullable — an empty string is a valid (bad) password, not null
- [ ] No plaintext or encrypted passwords stored — **hash only, irreversibly**
- [ ] Old hash algorithm version tracked per record for migration (e.g., `hash_version` column)

### Password Comparison
- [ ] Use the library's built-in comparison function (e.g., `bcrypt.CompareHashAndPassword`)
- [ ] Comparison is **constant-time** — never compare hash strings with `==` or `string.Compare`
- [ ] Return generic error on mismatch ("Invalid credentials") — never indicate which field is wrong

### Password Input
- [ ] Accept passwords up to 72 bytes for bcrypt (truncates silently beyond this) — enforce or strip at boundary
- [ ] For Argon2id/scrypt: no practical limit; still set a reasonable max (e.g., 1024 chars) to prevent DoS
- [ ] Never log passwords — not even "password attempt" debug logs
- [ ] Strip leading/trailing whitespace only if your policy explicitly allows it; otherwise don't

---

## Step 4 — Algorithm Migration (Upgrading Legacy Hashes)

Migrating from MD5/SHA-1/unsalted hashes without forcing password resets:

### Online Migration (transparent to users)
```
1. Add hash_version column to users table
2. On successful login:
   a. Verify password against OLD hash (MD5/SHA-1)
   b. If valid: immediately re-hash with Argon2id; store new hash; update hash_version
   c. Complete login normally
3. After transition period: force password reset for accounts still on old version
```

### Strengthening Existing bcrypt Hashes (without passwords)
If bcrypt cost factor is too low:
1. Hash the existing bcrypt hash with Argon2id: `argon2id(bcrypt_hash)`
2. Store wrapped hash and mark as `wrapped_version`
3. On login: compute `argon2id(bcrypt(password, old_salt))`; update to full Argon2id on success

---

## Step 5 — Breach Detection

Integrate with Have I Been Pwned (HIBP) Pwned Passwords API to check if a password appears in known breaches:

### k-Anonymity API (privacy-preserving):
```
1. SHA1(password) → full_hash
2. prefix = full_hash[:5]
3. GET https://api.pwnedpasswords.com/range/{prefix}
4. Search response for full_hash[5:] (suffix)
5. If found: count indicates breach exposure; reject or warn
```

When to check:
- At registration and password change: block passwords found in breaches
- At login (optional, lower priority): warn user if their password has since been breached

---

## Step 6 — Password Policy

Effective password policy minimizes user friction while maximizing security:

| Rule | Recommendation |
|---|---|
| Minimum length | 12 characters minimum; longer is better |
| Maximum length | 64–128 characters (set a limit to prevent DoS on hashing) |
| Complexity rules | Avoid mandatory complexity rules (NIST SP 800-63B); focus on length instead |
| Common password blocking | Check against top-10K or top-100K password lists |
| Breach checking | Block passwords found in HIBP database |
| Expiration | Do not force periodic expiration (NIST 800-63B guidance); change only on suspected breach |
| History | Prevent reuse of last 5–12 passwords |
| Username similarity | Reject passwords that are too similar to username/email |

---

## Step 7 — Password Reset Flow

| Step | Security requirement |
|---|---|
| Request reset | Rate limit requests per account and IP; do not confirm whether email exists |
| Token generation | CSPRNG; ≥128 bits; single-use; expire in ≤1 hour |
| Token delivery | Email only; never via SMS alone for password reset |
| Token storage | Hash the token before storing (same as passwords — treat as credentials) |
| Token validation | Constant-time comparison; immediately invalidate on use or expiry |
| Post-reset | Invalidate all existing sessions; notify user of password change |

---

## Step 8 — Output Report

```
## Password Storage Review: <service/file>

### Critical
- Passwords hashed with MD5 without salt → trivially crackable
  Fix: migrate to Argon2id using online migration pattern; force reset for unmigrated accounts

- Custom XOR-based obfuscation used instead of hashing
  Fix: replace entirely with Argon2id; treat all stored passwords as compromised

### High
- bcrypt cost factor set to 4 (default) → too fast; crackable with consumer hardware
  Fix: increase to cost=12; monitor hashing time and adjust to ~300ms on production hardware

### Medium
- No HIBP integration at registration
  Fix: add k-anonymity API check; block passwords found in breach databases

### Low
- Password max length not enforced → potential DoS via oversized input to bcrypt
  Fix: enforce max 72 bytes for bcrypt inputs or upgrade to Argon2id with max 1024 chars

### Passed
- CSPRNG salt generation ✓
- Constant-time comparison used ✓
- Passwords never logged ✓
- Password reset tokens are single-use and expire in 1 hour ✓
```
