---
name: rce-prevention
description: 'Safe execution practices preventing remote code execution and unsafe command usage. Use when: RCE prevention, remote code execution, command injection, shell injection, exec injection, eval security, unsafe exec, subprocess security, os.system, shell=True, os/exec golang, ProcessBuilder java, child_process node, code execution prevention, unsafe deserialization RCE, template injection RCE.'
argument-hint: 'Code using exec, eval, shell commands, or dynamically executed code to review — or describe the execution pattern being designed'
---

# RCE Prevention Specialist

## When to Use
- Reviewing code that executes system commands, spawns subprocesses, or evaluates code dynamically
- Auditing use of `exec`, `eval`, `system`, `shell_exec`, `os.system`, `child_process.exec`
- Designing safe alternatives to shell command execution
- Reviewing template engines, deserialization paths, or plugin systems for code execution risks
- Auditing server-side scripting, job runners, or build systems that accept user input

---

## Step 1 — Identify RCE Attack Surfaces

RCE occurs when **user-controlled input reaches an execution context**. Entry points:

| Category | Examples |
|---|---|
| Shell execution | `os.system(cmd)`, `subprocess.call(cmd, shell=True)`, `exec(cmd)` in Python/PHP/Ruby |
| Subprocess with string args | `child_process.exec(cmd)` in Node.js, `os/exec` without argument isolation in Go |
| Code evaluation | `eval(userInput)`, `new Function(code)`, `exec(code)` in Python |
| Template injection | Server-side template rendered with user-controlled template string |
| Deserialization | Untrusted bytes deserialized into executable objects |
| Expression languages | SpEL, OGNL, JEXL evaluated with user input (Spring, Struts) |
| Script engines | Rhino, Groovy, Nashorn, Jython executing user-supplied scripts |
| File inclusion | PHP `include($userPath)`, `require($userPath)` with user-controlled path |

---

## Step 2 — Shell Command Execution — Dangerous Patterns

### Most dangerous — shell metacharacter injection

```python
# PYTHON — BAD (shell=True with user input)
import subprocess
filename = request.form['filename']
subprocess.call(f"convert {filename} output.pdf", shell=True)
# Attacker: filename = "x.jpg; rm -rf /; #"

# PYTHON — GOOD (argument list, no shell)
filename = sanitize_filename(request.form['filename'])
subprocess.run(["convert", filename, "output.pdf"], shell=False, timeout=30)
```

```javascript
// NODE.JS — BAD
const { exec } = require('child_process')
exec(`convert ${filename} output.pdf`, callback)
// exec passes to /bin/sh — shell metacharacters interpreted

// NODE.JS — GOOD
const { execFile } = require('child_process')
execFile('convert', [filename, 'output.pdf'], callback)
// execFile bypasses shell — no metacharacter interpretation
```

```go
// GO — BAD (implicitly uses shell when string contains pipe/redirect)
cmd := exec.Command("sh", "-c", "convert " + filename + " output.pdf")

// GO — GOOD (no shell; arguments isolated)
cmd := exec.Command("convert", filename, "output.pdf")
```

```php
// PHP — BAD
system("ffmpeg -i " . $_GET['file'] . " output.mp4");

// PHP — GOOD
$file = escapeshellarg($_GET['file']);
system("ffmpeg -i " . $file . " output.mp4");
// Better: avoid shell entirely; use a PHP FFmpeg library
```

### Shell metacharacters to prevent injection:
`;`, `|`, `&`, `$()`, `` ` `, `>`, `<`, `\n`, `\r`, `$(`, `${`, `&&`, `||`

---

## Step 3 — Eval and Dynamic Code Execution

**Never evaluate user-supplied strings as code.**

| Language | Dangerous | Safe alternative |
|---|---|---|
| JavaScript | `eval(input)`, `new Function(input)`, `setTimeout(input, ms)` | JSON.parse for data; no dynamic code evaluation |
| Python | `eval(input)`, `exec(input)`, `compile(input, ...)` | ast.literal_eval for safe data; no code eval |
| PHP | `eval($input)`, `preg_replace('/e', $input)` (deprecated) | Remove eval entirely; use allowlisted operations |
| Ruby | `eval(input)`, `Kernel.eval(input)` | No eval on external data |
| Java | SpEL, OGNL, Groovy with user input | Disable expression evaluation; use allowlisted operations |

---

## Step 4 — Template Injection → RCE

Server-side template injection (SSTI) can lead to RCE when a template engine evaluates user-controlled template syntax.

**Dangerous pattern:**
```python
# Jinja2 — NEVER render user-supplied template strings
from jinja2 import Template
template_str = request.form['message']      # attacker supplies: {{ ''.__class__.__mro__[1].__subclasses__() }}
Template(template_str).render(user=user)    # → RCE
```

**Safe pattern:**
```python
# GOOD — use a predefined template; user input is DATA only
from jinja2 import Environment
env = Environment(autoescape=True)
template = env.from_string("Hello, {{ name }}!")    # template is fixed
result = template.render(name=user_supplied_name)   # user input = data, not template
```

**Rule:** Templates must come from trusted sources (code, filesystem, DB with strict validation). User input must only populate **variables into templates**, never become the template itself.

---

## Step 5 — File Inclusion RCE

File inclusion vulnerabilities allow loading arbitrary code:

```php
// PHP — DANGEROUS (Remote File Inclusion if allow_url_include=On)
include($_GET['module'] . '.php');

// SAFE — allowlist of permitted modules
$allowed = ['home', 'profile', 'settings'];
$module = $_GET['module'];
if (!in_array($module, $allowed, true)) {
    die('Invalid module');
}
include($module . '.php');
```

```python
# Python — DANGEROUS (path traversal + code execution)
module_name = request.args.get('plugin')
importlib.import_module(module_name)

# SAFE — allowlist
allowed_plugins = {'csv_exporter', 'pdf_renderer'}
if module_name not in allowed_plugins:
    abort(400)
importlib.import_module(module_name)
```

---

## Step 6 — Deserialization → RCE

Deserializing untrusted data from Java, Python pickle, PHP unserialize, .NET BinaryFormatter, or YAML can trigger arbitrary code execution via gadget chains.

| Format | Risk | Safe alternative |
|---|---|---|
| Java `ObjectInputStream` | Gadget chains → RCE | JSON + explicit type mapping; if unavailable: use serial-killer, notSoSerial denylist |
| Python `pickle` | `__reduce__` → arbitrary exec | Use `json`, `msgpack`, or `dataclasses` |
| PHP `unserialize()` | Magic method chains → RCE | Use `json_decode()` |
| .NET `BinaryFormatter` | Gadget chains → RCE | Use `System.Text.Json`, `XmlSerializer` with type restrictions |
| YAML (PyYAML) | `!!python/object` tag → exec | Use `yaml.safe_load()` instead of `yaml.load()` |
| Ruby `Marshal` | Gadget chains | Use JSON |

**General rule:** Never deserialize data from user-controlled sources into polymorphic objects. If deserialization is unavoidable, use an allowlist of permitted classes.

---

## Step 7 — Principle of Least Privilege for Execution

Even if RCE occurs, limit blast radius:

- [ ] Application runs as non-root, non-privileged OS user
- [ ] No `setuid` or `setgid` on application binaries
- [ ] Sandboxed subprocess execution (seccomp, AppArmor, containers without `--privileged`)
- [ ] No write access to web root or code directories
- [ ] `PATH` explicitly set in subprocess environments; never inherited from user
- [ ] Subprocess environments cleared of sensitive environment variables

---

## Step 8 — Detection Signals

Indicators of RCE attempts in logs:
- Shell metacharacters (`;`, `|`, `&&`, `$(`) in input fields or URL parameters
- Long encoded strings (`%3B`, `%7C`, base64 blobs) in unexpected parameters
- Unusual outbound network connections from application servers
- New processes spawned by the application with unexpected names
- File modifications in unexpected directories

---

## Step 9 — Output Report

```
## RCE Prevention Review: <service/file>

### Critical
- workers/converter.py:44 — subprocess.call(f"ffmpeg -i {filename}", shell=True)
  shell=True + unvalidated filename → command injection
  Fix: subprocess.run(["ffmpeg", "-i", filename, ...], shell=False, timeout=60)

- templates/email.py:21 — Template(user_subject).render(...) — SSTI vulnerability
  User controls template string → Jinja2 sandbox escape → RCE
  Fix: use a fixed template string; inject user input as data variables only

### High
- plugins/loader.py:8 — importlib.import_module(request.args['plugin'])
  Arbitrary module import via URL param
  Fix: validate against allowlist of permitted plugin names

### Medium
- api/export.js:55 — child_process.exec(`zip ${outputDir} ${files.join(' ')}`)
  exec() uses /bin/sh; filename metacharacters could inject commands
  Fix: use execFile with explicit argument array; validate all filenames

### Low
- Application runs as root inside Docker container
  Fix: add USER directive in Dockerfile; run as non-root

### Passed
- Go exec.Command used with argument arrays — no shell interpolation ✓
- yaml.safe_load() used throughout; yaml.load() not present ✓
- No eval() usage in codebase ✓
```
