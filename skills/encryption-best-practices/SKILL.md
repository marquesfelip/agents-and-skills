---
name: encryption-best-practices
description: 'Encryption at rest, in transit, key management, hashing strategies, and safe cryptographic practices. Use when: encryption design, encryption at rest, encryption in transit, TLS configuration, key management, symmetric encryption, asymmetric encryption, AES, RSA, ECDH, cryptographic hashing, SHA-256, data encryption, field-level encryption, envelope encryption, crypto review, insecure cryptography, weak cipher.'
argument-hint: 'Code, config, or architecture with cryptographic operations to review — or describe the encryption requirement being designed'
---

# Encryption Best Practices Specialist

## When to Use
- Reviewing cryptographic code or configurations for weaknesses
- Designing encryption at rest for databases, files, or backup data
- Configuring TLS for APIs, services, or internal communication
- Choosing algorithms for symmetric/asymmetric encryption or hashing
- Implementing key management, envelope encryption, or field-level encryption

---

## Step 1 — Understand the Cryptographic Context

Before advising, identify:
- What is being encrypted: data at rest, data in transit, or both
- Sensitivity and regulatory requirements (PII, PCI-DSS, HIPAA, LGPD/GDPR)
- Key management infrastructure: managed (AWS KMS, GCP Cloud KMS) vs self-managed
- Performance constraints (real-time encryption vs batch)
- Who needs to decrypt: same system, another service, external party

---

## Step 2 — Algorithm Selection Guide

### Symmetric Encryption (data at rest, bulk data)

| Algorithm | Mode | Use | Avoid |
|---|---|---|---|
| AES-256 | GCM | Preferred — authenticated encryption | ECB (no semantic security), CBC without MAC |
| AES-128 | GCM | Acceptable when 128-bit key is required | |
| ChaCha20-Poly1305 | — | Alternative to AES-GCM; faster on systems without AES-NI | |

**Never use:** DES, 3DES, RC4, Blowfish (legacy), AES-ECB.

### Asymmetric Encryption (key exchange, digital signatures)

| Algorithm | Key size | Use |
|---|---|---|
| RSA-OAEP | ≥2048 bit (prefer 4096) | Key encryption, legacy compatibility |
| ECDH | P-256 or X25519 | Key exchange; prefer over RSA for performance |
| RSA-PSS | ≥2048 bit | Digital signatures |
| ECDSA | P-256 or Ed25519 | Digital signatures; prefer Ed25519 for modern systems |

**Never use:** RSA-PKCS1v1.5 for encryption (PKCS#1 padding oracle), MD5 or SHA-1 in signatures.

### Hashing (data integrity, fingerprinting — not password storage)

| Algorithm | Use |
|---|---|
| SHA-256 | General purpose integrity checks |
| SHA-384/SHA-512 | Higher security margin |
| BLAKE2 / BLAKE3 | Performance-sensitive non-password hashing |
| SHA-3 (Keccak) | Where algorithm diversity from SHA-2 family is needed |

**Never use for integrity:** MD5, SHA-1.
**Password hashing:** use Argon2id, bcrypt, or scrypt — see `password-storage-security` skill.

---

## Step 3 — Encryption at Rest

### Principles
- Encrypt data at the storage layer AND at the application layer for sensitive fields.
- Use **envelope encryption**: encrypt data with a Data Encryption Key (DEK); protect DEK with a Key Encryption Key (KEK) from a KMS.
- Never store the encryption key alongside the data it protects.
- Use authenticated encryption (AES-GCM or ChaCha20-Poly1305) — provides both confidentiality AND integrity.

### Envelope Encryption Pattern
```
1. Generate a random DEK (e.g., AES-256 key)
2. Encrypt data with DEK → encrypted_data
3. Encrypt DEK with KEK (from KMS) → encrypted_dek
4. Store: { encrypted_data, encrypted_dek, key_id }
5. Decryption: fetch KEK from KMS → decrypt encrypted_dek → use DEK to decrypt data
```

### Field-Level Encryption (for sensitive columns)
- Encrypt specific fields (SSN, credit card, health data) at the application layer before storing.
- Use separate keys per field type or per tenant.
- Indexed searches on encrypted fields require format-preserving encryption or bloom filters.

### Key areas to audit:
- [ ] Encryption algorithm is AES-256-GCM or equivalent
- [ ] IV/nonce is random and unique per encryption operation (never reused with the same key)
- [ ] Authentication tag is verified on decryption
- [ ] Backups are encrypted with the same or stronger protection as live data
- [ ] Database encryption (TDE) is enabled but is NOT a substitute for application-layer encryption

---

## Step 4 — Encryption in Transit (TLS)

### TLS Configuration Checklist
- [ ] TLS 1.2 minimum; TLS 1.3 preferred
- [ ] Disabled: TLS 1.0, TLS 1.1, SSLv2, SSLv3
- [ ] Cipher suites: prefer ECDHE + AES-GCM or ChaCha20; disable RC4, DES, 3DES, EXPORT ciphers
- [ ] Certificate: valid, not self-signed in production, minimum 2048-bit RSA or P-256 ECDSA
- [ ] Certificate chain: complete (intermediate certs included)
- [ ] HSTS: `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
- [ ] Certificate pinning: consider for high-value mobile apps

### Internal Service Communication
- Encrypt all internal service-to-service traffic, even within a VPC (zero-trust model).
- Use mTLS for service mesh authentication (Istio, Linkerd, Consul Connect).
- Never assume network-level security is sufficient for sensitive data.

---

## Step 5 — Key Management

| Practice | Requirement |
|---|---|
| Key generation | Use CSPRNG; length matches algorithm requirements |
| Key storage | KMS (AWS KMS, GCP Cloud KMS, Azure Key Vault, HashiCorp Vault) — never alongside data |
| Key rotation | Rotate DEKs periodically; re-encrypt data with new DEK; rotate KEKs on a schedule |
| Key access control | Least privilege — only the service that encrypts/decrypts should have key access |
| Key deletion | Soft delete with recovery window; confirm all data re-encrypted before hard delete |
| Audit | Log all key access, creation, rotation, and deletion events |

**Key hierarchy:**
```
Root Key (HSM/KMS, rarely used)
  └── KEK (Key Encryption Key, rotated annually)
        └── DEK (Data Encryption Key, rotated per dataset or more frequently)
```

---

## Step 6 — Common Cryptographic Pitfalls

| Pitfall | Problem | Fix |
|---|---|---|
| AES-ECB mode | Identical plaintext blocks produce identical ciphertext (pattern leakage) | Use AES-GCM |
| IV/nonce reuse | With the same key + nonce, GCM is completely broken | Generate a random 96-bit IV per encryption |
| Unauthenticated encryption (CBC without MAC) | Ciphertext can be tampered without detection | Use AES-GCM or add HMAC (Encrypt-then-MAC) |
| Storing key with data | Key compromise = data compromise | Separate key store from data store |
| Self-implemented crypto | Timing side-channels, padding oracle, logic errors | Use well-audited libraries (libsodium, Bouncy Castle, Go `crypto`) |
| MD5/SHA-1 for security | Collision vulnerabilities | Upgrade to SHA-256+ |
| Hardcoded keys | Key rotation is impossible | Use KMS with programmatic access |

---

## Step 7 — Output Report

```
## Encryption Review: <component/service>

### Critical
- AES-ECB used for field encryption — identical inputs produce identical ciphertexts
  Fix: migrate to AES-256-GCM; re-encrypt existing data

- Encryption key stored in same database table as encrypted data
  Fix: move key to AWS KMS / HashiCorp Vault; use envelope encryption

### High
- TLS 1.0 still enabled on API gateway
  Fix: disable TLS 1.0 and 1.1; enforce TLS 1.2+ only

- IV hardcoded to all-zeros — nonce reuse across all encryptions
  Fix: generate crypto.randomBytes(12) per encryption operation

### Medium
- No key rotation policy defined
  Fix: schedule DEK rotation every 90 days; document re-encryption process

### Low
- SHA-1 used for non-security checksum
  Fix: upgrade to SHA-256 (low risk but good hygiene)

### Passed
- AES-256-GCM with random IV used for backup encryption ✓
- TLS 1.3 enforced on public API ✓
- Envelope encryption with AWS KMS implemented ✓
```
