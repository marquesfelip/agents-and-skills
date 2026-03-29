---
name: deserialization-security
description: 'Secure serialization/deserialization patterns and unsafe object execution prevention. Use when: deserialization security, unsafe deserialization, Java deserialization, pickle security, PHP unserialize, YAML unsafe load, BinaryFormatter, gadget chain, object injection, deserialization RCE, serialization format, JSON type safety, deserialization vulnerability, untrusted data deserialization.'
argument-hint: 'Serialization/deserialization code, data format handling, or message consumer to review — or describe the data exchange format being designed'
---

# Deserialization Security Specialist

## When to Use
- Reviewing code that deserializes data from external or user-controlled sources
- Auditing queue consumers, API request parsers, or file readers for unsafe deserialization
- Choosing a safe serialization format for a data exchange or caching layer
- Investigating a known deserialization vulnerability in a dependency
- Hardening Java, Python, PHP, .NET, or Ruby object deserialization

---

## Step 1 — Understand the Deserialization Context

Before advising, identify:
- **Format**: Java serialization, Python pickle, PHP `unserialize`, .NET BinaryFormatter, YAML, XML, MessagePack, custom binary
- **Source of data**: user HTTP request, message queue, cache, file upload, third-party API, inter-service call
- **Trust level**: is the data source fully trusted (internal service with strong auth), partially trusted (authenticated user), or untrusted (public)?
- **Language/runtime**: Java, Python, PHP, .NET, Ruby, Go, Node.js

---

## Step 2 — Risk Classification by Format

| Format | Risk level | Reason |
|---|---|---|
| Java `ObjectInputStream` | Critical | Gadget chains in classpath → RCE |
| Python `pickle` / `cPickle` | Critical | `__reduce__` executes arbitrary code |
| PHP `unserialize()` | Critical | Magic methods (`__wakeup`, `__destruct`) → RCE/SQLi |
| .NET `BinaryFormatter` | Critical | Gadget chains → RCE (deprecated in .NET 5+) |
| Ruby `Marshal` | Critical | Gadget chains → RCE |
| YAML (`yaml.load()`) | High | Language-specific tags (`!!python/object`) → code execution |
| XML with DTD | High | XXE → SSRF, file read, DoS |
| JSON | Low | Type-safe if parsed into expected schema — use typed parsing |
| MessagePack | Low | Schema-less but no native code execution; validate structure |
| Protobuf | Low | Schema-enforced; safe if schema is controlled |

---

## Step 3 — Language-Specific Safe Alternatives

### Java

```java
// BAD — native Java deserialization
ObjectInputStream ois = new ObjectInputStream(untrustedInputStream);
Object obj = ois.readObject();   // gadget chain exploit possible

// GOOD — use JSON with strict typing
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(/* NEVER enable this */);  // BAD — enables polymorphic types
MyDTO dto = mapper.readValue(json, MyDTO.class);       // GOOD — fixed target type

// If Java deserialization is unavoidable: use a class filter
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.myapp.dto.*;java.lang.*;!*"  // allowlist; deny all others
);
ois.setObjectInputFilter(filter);
```

**Jackson-specific risks:**
- `enableDefaultTyping()` → polymorphic deserialization → RCE gadget chains
- `@JsonTypeInfo(use = Id.CLASS)` with untrusted input → same risk
- **Fix**: never enable default typing on untrusted input; use `@JsonTypeInfo(use = Id.NAME)` with an explicit allowlist

### Python

```python
# BAD — pickle executes arbitrary code during deserialization
import pickle
data = pickle.loads(untrusted_bytes)   # attacker crafts __reduce__ → os.system("rm -rf /")

# GOOD — use JSON or dataclasses
import json
data = json.loads(untrusted_str)       # data only; no code execution

# If pickle is required (ML models, internal caching):
# — Only accept from trusted, authenticated internal sources
# — Never expose pickle deserialization to external input
# — Consider alternative: joblib with signature verification, or ONNX for ML models
```

```python
# YAML — BAD
import yaml
data = yaml.load(untrusted_str)        # !!python/object tags → RCE

# YAML — GOOD
data = yaml.safe_load(untrusted_str)   # restricts to safe types only
```

### PHP

```php
// BAD — unserialize triggers __wakeup, __destruct, __toString magic methods
$data = unserialize($_COOKIE['user_data']);

// GOOD — use JSON
$data = json_decode($_COOKIE['user_data'], true);

// If unserialize is unavoidable: use allowed_classes option (PHP 7.0+)
$data = unserialize($input, ['allowed_classes' => ['MyApp\DTO\UserPrefs']]);
```

### .NET (C#)

```csharp
// BAD — BinaryFormatter (deprecated, still dangerous in legacy code)
var formatter = new BinaryFormatter();
var obj = formatter.Deserialize(stream);

// GOOD — use System.Text.Json with explicit types
var dto = JsonSerializer.Deserialize<MyDto>(json);

// If XML: disable DTD processing
var settings = new XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit };
```

### Go

```go
// Go encoding/json — safer than native serialization; still validate structure
var data MyStruct
if err := json.Unmarshal(raw, &data); err != nil {
    return err
}
// Validate fields after unmarshaling — don't trust all fields blindly

// encoding/gob — only use with trusted internal sources
// Never expose gob deserialization to external/user input
```

---

## Step 4 — XML and XXE (Related Deserialization Risk)

XML parsers that process Document Type Definitions (DTDs) can be exploited to read files or perform SSRF:

```xml
<!-- Attacker-supplied XML with XXE payload -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root><data>&xxe;</data></root>
```

**Fix — disable DTD/external entity processing (by language):**

```java
// Java (DocumentBuilderFactory)
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature("http://xml.apache.org/saxparser/features/external-general-entities", false);

// Python (defusedxml library)
import defusedxml.ElementTree as ET
tree = ET.parse(xml_source)   // safe; blocking by default

// .NET
var settings = new XmlReaderSettings {
    DtdProcessing = DtdProcessing.Prohibit,
    XmlResolver = null
};
```

---

## Step 5 — Safe Deserialization Architecture Principles

| Principle | Implementation |
|---|---|
| Use data formats, not object formats | JSON, Protobuf, MessagePack — no code execution possible |
| Always deserialize into explicit types | Define a strict schema; reject unknown fields |
| Validate after deserialization | Values must make semantic sense, not just parse successfully |
| Never trust the format to enforce invariants | A valid JSON structure can still contain logically invalid data |
| Authenticate before deserializing | Verify the sender's identity before processing any message |
| Sign serialized data if reused | HMAC-sign tokens/cookies; verify signature before deserializing |
| Isolate deserialization | Run in a sandbox or worker with minimal privileges if format is risky |

---

## Step 6 — Signed Serialized Data (Cookies, Tokens)

When serialized data must survive round-trips through the client (e.g., session cookies):

```python
# GOOD — HMAC-signed JSON (e.g., Flask itsdangerous)
from itsdangerous import URLSafeTimedSerializer
s = URLSafeTimedSerializer(SECRET_KEY)
token = s.dumps({'user_id': 42})
data = s.loads(token, max_age=3600)   # validates signature + expiry before deserializing

# BAD — unsigned base64 data
import base64, json
data = json.loads(base64.b64decode(cookie))   # user can modify freely
```

---

## Step 7 — Detection and Mitigations for Known Gadget Chains

If using Java ObjectInputStream, .NET BinaryFormatter, or PHP unserialize in legacy code that cannot be refactored immediately:

1. **Java**: Apply [serial-killer](https://github.com/nicowillis/SerialKiller) or [notSoSerial](https://github.com/kantega/notsoserial) as a JVM agent to block known gadget chains.
2. **Java**: Keep classpath clean — remove unused libraries (commons-collections, commons-beanutils, spring-core) if not needed.
3. **.NET**: Enable `AppContext.SetSwitch("Switch.System.Runtime.Serialization.EnableUnsafeBinaryFormatterSerialization", false)`.
4. **PHP**: Use `allowed_classes` option in `unserialize()` to allowlist permitted classes.
5. **All**: Monitor for deserialization exceptions — a flood of them signals active exploitation.

---

## Step 8 — Output Report

```
## Deserialization Security Review: <service/file>

### Critical
- cache/session.py:33 — pickle.loads(redis.get(session_key))
  Session data deserialized with pickle — attacker who can write to Redis achieves RCE
  Fix: replace with json.loads(); store only primitive session fields

- api/import.php:19 — unserialize($_POST['data']) with no class restriction
  PHP magic method gadget chain possible
  Fix: use json_decode(); if unserialize unavoidable, add allowed_classes allowlist

### High
- xml_parser.java:55 — DocumentBuilderFactory with default settings
  XXE injection possible; external entities not disabled
  Fix: setFeature("disallow-doctype-decl", true); setFeature external-general-entities false

- config/loader.py:12 — yaml.load(f) without Loader argument (defaults to unsafe)
  !!python/object tags can execute arbitrary code
  Fix: yaml.safe_load(f)

### Medium
- Jackson ObjectMapper with enableDefaultTyping() in UserService
  Polymorphic deserialization → potential gadget chain RCE
  Fix: remove enableDefaultTyping(); use explicit target types for all deserialization

### Passed
- Queue consumers deserialize into explicit Protobuf types ✓
- Cookie values are HMAC-signed before serialization ✓
- JSON used for all inter-service communication ✓
```
