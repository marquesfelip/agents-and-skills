---
name: pii-handling
description: 'Secure handling, masking, minimization, and lifecycle management of personal data. Use when: PII handling, personal data protection, GDPR compliance, LGPD compliance, data minimization, PII masking, PII anonymization, pseudonymization, sensitive personal data, data subject rights, right to erasure, right to access, personal data lifecycle, PII encryption, CCPA compliance, personal data review.'
argument-hint: 'Feature, API, or data model that handles personal data — or the regulatory context (GDPR, LGPD, CCPA) and data types involved'
---

# PII Handling Specialist

## When to Use
- Designing or reviewing systems that collect, store, or process personal data
- Implementing GDPR / LGPD / CCPA compliance requirements
- Auditing data flows for PII minimization opportunities
- Implementing data subject rights: access, portability, erasure, rectification
- Designing masking and anonymization strategies for analytics, logging, and dev environments

---

## Step 1 — Identify and Classify PII

Enumerate all personal data fields in the system:

| Category | Legal basis required | Examples |
|---|---|---|
| **Direct identifiers** | Always requires basis | Full name, email, phone, national ID, passport, taxpayer ID (CPF/CNPJ), SSN |
| **Quasi-identifiers** | Often requires basis | IP address, device ID, birth date, ZIP code, gender |
| **Special category (GDPR Art. 9)** | Explicit consent or legal obligation | Health data, racial/ethnic origin, religious beliefs, biometrics, political views, union membership |
| **Financial** | Contractual necessity or consent | Bank account, card numbers, salary, credit score |
| **Behavioural** | Consent or legitimate interest (balancing test) | Purchase history, browsing history, location history, profiling data |
| **Children's data** | Parental consent + stricter rules | Any PII where subject is under-age per jurisdiction |

---

## Step 2 — Apply Data Minimization

Collect the minimum data necessary for each purpose:

**Questions for every field:**
1. Why do we collect this field? What specific purpose does it serve?
2. Is there a less sensitive alternative (e.g., age range instead of exact birth date)?
3. Can this purpose be achieved without storing this field at all?
4. Is collection at registration time, or can it be deferred until actually needed?
5. How long do we actually need it?

**Common minimization opportunities:**

| Instead of | Consider |
|---|---|
| Exact birth date | Age range bucket, or birth year only |
| Full phone number for 2FA | Hashed phone for deduplication; use verification only |
| Full address always | Address only when order/delivery is required |
| Full government ID | Last N digits for verification; hash for deduplication |
| Precise GPS coordinates | City-level location for analytics |

---

## Step 3 — Legal Basis Mapping (LGPD / GDPR)

Every processing activity requires a documented legal basis:

| Basis | When it applies | Key requirement |
|---|---|---|
| Consent | Optional features, marketing, profiling | Freely given, specific, informed, unambiguous; withdrawable at any time |
| Contract | Service delivery (account creation, order fulfillment) | Necessary for contract performance |
| Legal obligation | Tax records, anti-money laundering | Required by law; cannot refuse |
| Legitimate interest | Fraud prevention, security, basic analytics | Must pass balancing test: interest vs. data subject rights |
| Vital interests | Emergency safety situations | Rare; documented reason required |
| Public task | Government entities | Statutory authority required |

**Forbidden bases:**
- Consent cannot be bundled with terms of service acceptance ("take it or leave it")
- Legitimate interest cannot be used for special-category data
- Consent from minors is invalid without parental consent

---

## Step 4 — Data-at-Rest Protection

### Encryption Requirements by Tier

| Data tier | Minimum encryption | Key management |
|---|---|---|
| Special category (health, biometrics) | AES-256-GCM field-level encryption | Dedicated KMS key; access logged |
| Direct identifiers | AES-256 at-rest (disk or field) | Rotated annually; access via KMS |
| Quasi-identifiers | Database-level encryption (TDE) | Standard KMS key rotation |
| Non-PII | Transport + storage TLS sufficient | Default managed keys |

### PII Storage Pattern

```python
# BAD — plaintext PII columns, unencrypted
class User(Base):
    ssn = Column(String)          # plaintext national ID
    date_of_birth = Column(Date)  # unnecessary precision

# GOOD — encrypted at field level; minimize precision
class User(Base):
    ssn_encrypted = Column(LargeBinary)   # AES-256-GCM encrypted, KMS-managed key
    ssn_hash = Column(String)             # HMAC-SHA256 for deduplication lookup
    birth_year = Column(Integer)          # Only year stored — sufficient for business need
```

---

## Step 5 — Masking and Pseudonymization

### Masking by Context

| Context | Technique | Example |
|---|---|---|
| API response | Static masking | `"****@gmail.com"` / `"+55 11 ****-4321"` |
| Logs | Field scrubbing / redaction | Remove field or replace with `[REDACTED]` |
| Dev/Staging environments | Faker substitution | Replace real data with realistic fake data |
| Analytics | k-anonymity / generalization | Age ranges, city-level geo, aggregate counts |
| Inter-service | Tokenization | Opaque token maps to real data in secure vault |
| Backup/Export | Pseudonymization | Replace direct identifiers with pseudonyms; keep mapping in separate secured store |

### Tokenization Pattern (PCI / PII Vault)

```python
# Store PII in vault, use token everywhere else
def store_pii(national_id: str) -> str:
    token = secrets.token_urlsafe(32)
    pii_vault.store(token, encrypt(national_id, key=KMS.get_key('pii')))
    return token

# Retrieve only when explicitly needed (logged, audited)
def get_pii(token: str, reason: str, requestor_id: str) -> str:
    audit_log.write(token=token, reason=reason, requestor=requestor_id)
    return decrypt(pii_vault.get(token), key=KMS.get_key('pii'))
```

---

## Step 6 — Implement Data Subject Rights

### Right of Access (GDPR Art. 15 / LGPD Art. 18-I)

```python
def export_user_data(user_id: str) -> dict:
    # Collect ALL personal data across ALL systems
    return {
        'account': users_db.get(user_id),
        'orders': orders_db.get_by_user(user_id),
        'events': events_db.get_by_user(user_id),
        'consents': consents_db.get_by_user(user_id),
        # ... every table/service that holds this user's data
    }
```

### Right to Erasure (GDPR Art. 17 / LGPD Art. 18-VI) — Right to Be Forgotten

```python
def erase_user(user_id: str):
    # 1. Anonymize where deletion would break referential integrity
    orders_db.anonymize_user_fields(user_id)        # zero out: name, email, address
    # 2. Delete where records are not needed
    consents_db.delete_by_user(user_id)
    events_db.delete_by_user(user_id)
    # 3. Invalidate all active sessions
    sessions_db.revoke_all(user_id)
    # 4. Mark account as erased (soft delete — for audit trail only)
    users_db.mark_erased(user_id)
    # 5. Trigger deletion in downstream systems (email provider, analytics, CRM)
    integrations.propagate_erasure(user_id)
    # 6. Log the erasure event for regulatory evidence (keep 3–5 years)
    audit_log.record_erasure(user_id)
```

**Erasure checklist:**
- [ ] All databases and tables holding user PII
- [ ] File storage (uploads, profile photos, exports)
- [ ] Backups — have a documented process; erasure may take until backup rotation
- [ ] Third-party integrations (CRM, email, analytics, support tools)
- [ ] Cache layers (Redis, CDN) — flush cached personal data
- [ ] Search indexes (Elasticsearch) — delete indexed personal fields

### Right to Rectification

- [ ] Users can update all their personal data, not just some fields
- [ ] Changes propagated to downstream systems/replicas

---

## Step 7 — Consent Management

```python
class Consent(Base):
    user_id = Column(String, nullable=False)
    purpose = Column(String, nullable=False)        # 'marketing', 'analytics', 'profiling'
    granted_at = Column(DateTime, nullable=False)
    withdrawn_at = Column(DateTime, nullable=True)  # null = still active
    version = Column(String, nullable=False)        # privacy policy / terms version
    source = Column(String, nullable=False)         # 'signup_form', 'settings_v2'
    ip_address = Column(String, nullable=False)     # evidence of consent action

# Check before processing
def can_process(user_id: str, purpose: str) -> bool:
    consent = db.get_active_consent(user_id, purpose)
    return consent is not None and consent.withdrawn_at is None
```

**Consent requirements:**
- [ ] Purpose-specific (one consent per distinct processing purpose)
- [ ] Granular (marketing ≠ analytics ≠ profiling — separate checkboxes)
- [ ] Pre-ticked checkboxes are **invalid**
- [ ] "Accept all" must have an equivalent "reject all" mechanism
- [ ] Withdrawal is as easy as granting (one-click opt-out)
- [ ] Consent record stored with timestamp, IP, version of notice

---

## Step 8 — Dev/Test Environment PII Controls

- [ ] Production PII **never** copied to dev or staging — use synthetic or anonymized data
- [ ] Data anonymization script runs as part of staging refresh pipeline
- [ ] Dev database credentials rotated separately from production
- [ ] Anonymization verified periodically — check for re-identification risk
- [ ] Contractor/vendor access to dev does not expose real personal data

---

## Output Report

```
## PII Handling Review: <service/feature>

### Critical
- Production PII database snapshot used in staging environment
  Real user PII (name, email, SSN) exposed to all developers and contractors
  Fix: implement anonymization pipeline in staging refresh procedure

- National ID (CPF) stored in plaintext in users table
  Breach exposes government IDs with no encryption protection
  Fix: encrypt at field level with AES-256-GCM; store HMAC hash for lookups

### High
- No consent recorded for marketing email processing
  Sending marketing without legal basis — LGPD/GDPR violation
  Fix: add consent capture and consent table; gate marketing queue on active consent

- Right to erasure not implemented — user account deletion leaves PII in orders, events, logs
  Fix: implement cascading anonymization across all PII-holding tables

### Medium
- Email address returned in all API list endpoints even when not needed by caller
  Fix: apply data minimization; omit email from list responses; include only in profile detail

- Birth date stored at full precision (day/month/year) — only birth year needed for business logic
  Fix: migrate to birth_year column; discard day/month after migration

### Low
- Consent UI uses single "I accept all" checkbox for three distinct processing purposes
  Fix: separate checkboxes per purpose; no pre-ticked by default

### Passed
- Passwords stored using Argon2id ✓
- Field-level encryption on health data ✓
- IP logging with documented legitimate-interest basis ✓
```
