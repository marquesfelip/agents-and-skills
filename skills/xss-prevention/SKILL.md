---
name: xss-prevention
description: 'Output encoding, sanitization, CSP usage, and frontend/backend XSS prevention strategies. Use when: XSS prevention, cross-site scripting, output encoding, HTML escaping, sanitization, Content Security Policy, CSP, DOM XSS, reflected XSS, stored XSS, innerHTML, dangerouslySetInnerHTML, template injection, script injection, frontend security, XSS defense.'
argument-hint: 'Template, component, API response, or HTML-rendering code to review — or describe the rendering strategy being designed'
---

# XSS Prevention Specialist

## When to Use
- Reviewing server-side templates, React/Vue/Angular components, or HTML-rendering code for XSS
- Auditing API responses that are rendered in a browser context
- Configuring Content Security Policy (CSP) headers
- Reviewing sanitization libraries and their configuration
- Designing a frontend rendering strategy that is XSS-safe by default

---

## Step 1 — Classify the XSS Type

| Type | Where input is stored/reflected | Exploitation |
|---|---|---|
| **Reflected XSS** | Input reflected in the same response (search results, error messages) | User clicks a crafted URL |
| **Stored XSS** | Input saved to DB and rendered later (comments, profiles, messages) | Payload persists and fires for all viewers |
| **DOM XSS** | Input flows through JavaScript to a dangerous sink (no server round-trip) | Attacker controls URL fragment, `postMessage`, etc. |

---

## Step 2 — Output Encoding Rules

The core rule: **encode output in the context where it will be rendered**.

| Output context | Required encoding | What to escape |
|---|---|---|
| HTML body / attribute | HTML entity encoding | `<`, `>`, `&`, `"`, `'` |
| JavaScript string | JavaScript escaping | `\`, `"`, `'`, newline, `/` |
| URL parameter | URL encoding (percent encoding) | All non-alphanumeric chars |
| CSS value | CSS escaping | Non-alphanumeric chars |
| JSON in `<script>` | JSON encoding + HTML-safe variant | `<`, `>`, `&`, `'` within JSON |

> **Context determines encoding.** Applying HTML encoding inside a `<script>` block is wrong. Applying JS escaping in an HTML attribute is wrong. Match the encoding to the delivery context.

---

## Step 3 — Dangerous Sinks — Review Every Instance

### Server-Side

| Pattern | Language | Risk | Fix |
|---|---|---|---|
| `{{ raw_var \| safe }}` | Jinja2/Django | Disables auto-escaping | Remove `safe`; use default auto-escaping |
| `<%- variable %>` | EJS | Raw/unescaped output | Use `<%= variable %>` |
| `{!! $variable !!}` | Blade (Laravel) | Raw output | Use `{{ $variable }}` |
| `response.write(input)` | ASP.NET | Direct output | Use `HttpUtility.HtmlEncode(input)` |
| `text/html` with user data | any | Rendered as HTML | Template auto-escaping or explicit encoding |

### Client-Side (DOM XSS Sinks)

| Dangerous sink | Safe alternative |
|---|---|
| `element.innerHTML = userInput` | `element.textContent = userInput` |
| `document.write(userInput)` | DOM manipulation with `textContent` |
| `eval(userInput)` | Never use `eval` with external data |
| `setTimeout(userInput, ms)` | Always pass a function, never a string |
| `location.href = userInput` | Validate URL; use `encodeURIComponent` for params |
| `dangerouslySetInnerHTML={{ __html: input }}` | Sanitize with DOMPurify before using |
| `v-html="userInput"` (Vue) | Sanitize with DOMPurify; avoid where possible |
| `[innerHTML]="userInput"` (Angular) | Use Angular's `DomSanitizer.sanitize()` |

```js
// BAD
document.getElementById('output').innerHTML = userInput

// GOOD
document.getElementById('output').textContent = userInput

// If HTML rendering is unavoidable — sanitize first
import DOMPurify from 'dompurify'
element.innerHTML = DOMPurify.sanitize(userInput, { ALLOWED_TAGS: ['b', 'i', 'a'] })
```

---

## Step 4 — Sanitization vs Encoding

| Approach | When to use | Library |
|---|---|---|
| **Output encoding** (preferred) | Rendering plain text or data as HTML | Framework auto-escaping, `htmlspecialchars`, `he` (JS) |
| **Sanitization** | User-supplied HTML that must be rendered (rich text, markdown output) | DOMPurify, bleach (Python), sanitize-html (Node.js) |
| **Reject** | No legitimate reason for HTML tags | Strip all HTML; validate input as plain text |

**Sanitization configuration:**
```js
// DOMPurify — strict allowlist
DOMPurify.sanitize(html, {
  ALLOWED_TAGS: ['b', 'strong', 'i', 'em', 'a', 'p', 'br', 'ul', 'ol', 'li'],
  ALLOWED_ATTR: ['href', 'title'],
  ALLOW_DATA_ATTR: false,
  FORBID_SCRIPTS: true,
  FORBID_TAGS: ['style', 'script', 'iframe', 'object', 'embed'],
})
```

---

## Step 5 — Content Security Policy (CSP)

CSP is the last line of defense — it limits what can execute even if XSS occurs.

### Strong CSP Directives

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{random_nonce}';
  style-src 'self' 'nonce-{random_nonce}';
  img-src 'self' data: https://cdn.example.com;
  connect-src 'self' https://api.example.com;
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
  form-action 'self';
  upgrade-insecure-requests;
```

| Directive | Purpose | Avoid |
|---|---|---|
| `script-src 'nonce-...'` | Allow only scripts with matching nonce | `'unsafe-inline'`, `'unsafe-eval'` |
| `object-src 'none'` | Block Flash/plugin execution | Any whitelist here |
| `base-uri 'self'` | Prevent base tag injection | `*` |
| `frame-ancestors 'none'` | Clickjacking protection | `*` |
| `upgrade-insecure-requests` | Force HTTPS for sub-resources | — |

**Generate nonces properly:**
```
nonce = base64(crypto.randomBytes(16))   # unique per response, not per page
```

**Never use:**
- `'unsafe-inline'` — defeats CSP for scripts
- `'unsafe-eval'` — allows `eval()`, `setTimeout(string)`, etc.
- Wildcard `*` in `script-src` or `default-src`

### CSP Deployment Approach
1. Start with `Content-Security-Policy-Report-Only` to observe violations without breaking the app.
2. Collect reports via `report-uri` or `report-to`.
3. Fix violations; tighten policy.
4. Switch to `Content-Security-Policy` enforcement mode.

---

## Step 6 — Framework-Specific Protections

| Framework | Default safe? | Watch out for |
|---|---|---|
| React | Yes — JSX auto-encodes | `dangerouslySetInnerHTML`, `ref` + direct DOM mutation |
| Vue | Yes — `{{ }}` encodes | `v-html` directive |
| Angular | Yes — template engine encodes | `bypassSecurityTrustHtml()`, `DomSanitizer` misuse |
| Django | Yes — `{{ }}` encodes | `{% autoescape off %}`, `mark_safe()` |
| Go `html/template` | Yes | Using `text/template` for HTML output |
| Jinja2 | Yes (by default) | `| safe` filter, `Markup()` |
| Handlebars | Yes — `{{ }}` | `{{{ rawOutput }}}` triple-braces |

---

## Step 7 — URL and Redirect XSS

`javascript:` URIs in `href` or redirect targets are an XSS vector:

```js
// BAD
window.location.href = userInput   // attacker passes "javascript:alert(1)"
<a href={userInput}>click</a>      // same risk

// GOOD
const url = new URL(userInput)
if (!['http:', 'https:'].includes(url.protocol)) throw new Error('invalid URL')
window.location.href = url.toString()
```

---

## Step 8 — Output Report

```
## XSS Prevention Review: <component/service>

### Critical
- comments.html:42 — `{{ comment.body | safe }}` — auto-escaping disabled; stored comment body rendered raw
  Fix: remove `safe` filter; content is plain text, no HTML needed

- admin/search.js:18 — `results.innerHTML = query` — search term injected into DOM
  Fix: use `results.textContent = query` for display; escape before use

### High
- No Content-Security-Policy header set on any response
  Fix: implement CSP with nonce-based script allowlist; start with Report-Only mode

- profile.vue:31 — `v-html="user.bio"` without sanitization
  Fix: pipe through DOMPurify with tag allowlist before binding

### Medium
- CSP present but includes `'unsafe-inline'` in script-src — policy ineffective against inline XSS
  Fix: replace inline scripts with nonce-tagged script blocks; remove `'unsafe-inline'`

### Low
- X-Content-Type-Options header missing — MIME-sniffing XSS possible on file downloads
  Fix: add `X-Content-Type-Options: nosniff` to all responses

### Passed
- React JSX used throughout frontend — auto-encoding active ✓
- `dangerouslySetInnerHTML` usage: 0 occurrences ✓
- URL redirects validate protocol against https allowlist ✓
```
