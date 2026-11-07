---
name: cybersec
description: Use when inspecting codebases for vulnerabilities, auditing host posture, triaging web/auth logs, auditing supply chain dependencies, or applying defensive security hardening patch recipes.
---

# cybersec: Automated Cybersecurity Inspection & Hardening Framework

`cybersec` is a zero-dependency, high-velocity cybersecurity inspection, vulnerability triage, and automated defense framework. Modeled on standard library simplicity and deterministic evidence rigor, it equips AI agents and developers with automated **SAST and secret detection**, **supply chain manifest analysis**, **host posture forensics**, **log intrusion triage**, and **copy-pasteable hardening patch recipes**.

---

## 1. Core Principles & Workflow

### Zero External Dependencies & Standard Library First
The `cybersec` engine requires only Python 3 (`sys`, `os`, `re`, `subprocess`, `socket`, `hashlib`, `json`, `urllib`, `pathlib`). It runs cleanly in air-gapped CI/CD environments, container sandboxes, or minimal Alpine/Debian base images without `pip install`.

### The 3-Turn Security Recon Cycle
Security turns must be evidence-driven and high-velocity. Complete full security triage and remediation in 3 targeted turns:
1. **Turn 1 (Surface Mapping & Automated Discovery):**
   * Run automated discovery tools (`cybersec scan`, `cybersec deps`, `cybersec host`).
   * Inventory exposed attack surfaces, open sockets, hardcoded keys, and suspicious dependencies.
2. **Turn 2 (Deep Exploit Verification & Triaging):**
   * Trace source $\to$ sanitization $\to$ sink data paths.
   * Cross-reference findings with the [OWASP Top 10 Reference](references/owasp-top10.md) and [GTFOBins Catalogue](references/gtfobins.md).
   * Tag all findings under the Deterministic Evidence Classification System.
3. **Turn 3 (Hardened Remediation & Verification Testing):**
   * Apply copy-pasteable defense recipes with `cybersec patch <recipe>`.
   * Re-run security scans and test suites (`bun test` or `pytest`) to verify flaw remediation without functional regressions.

### Deterministic Evidence Classification System
Never speculate, generalize, or hallucinate security vulnerabilities. Every finding in a security report MUST carry one of three deterministic tags:
* **`[CONFIRMED: file:line]`**: Directly verified vulnerable code path, active secret leak, or proven exploit payload with verified sink.
* **`[POTENTIAL: file:line]`**: Suspicious source-to-sink flow or unvalidated input requiring runtime context or integration testing to confirm.
* **`[CLEAN: component]`**: Audited module or component verified safe with active defense controls in place.

---

## 2. Tooling Guide: CLI Reference (`cybersec`)

The standalone engine is located at `.agents/skills/cybersec/scripts/cybersec` (executable directly or via `python3 .agents/skills/cybersec/scripts/cybersec`).

```sh
# 1. Static Analysis & High-Entropy Secret Scanning
cybersec scan <path> [--deep] [--json]

# 2. Dependency Manifest & Supply Chain Hook Auditing
cybersec deps <manifest> [--json]

# 3. Host Posture, SUID Binaries & Privilege Escalation Audit
cybersec host [--json]

# 4. Web Access & Authentication Log Intrusion Triage
cybersec triage <logfile> [--type=auto|web|auth] [--json]

# 5. Production Defense Patch & Hardening Recipes
cybersec patch <safe-path|sql-param|security-headers|timing-safe|rate-limiter> [--json]
```

### CLI Command Matrix

| Subcommand | Target Inputs | Primary Checks | Output Formats |
| :--- | :--- | :--- | :--- |
| `cybersec scan` | Source file, directory | API keys, AWS creds, JWTs, private keys, SQLi, command injection, path traversal, weak crypto | Text table or JSON |
| `cybersec deps` | `package.json`, `requirements.txt`, `Pipfile`, `Cargo.toml` | Malicious install hooks (`curl`, `bash`), unpinned wildcards, untrusted index URLs | Text report or JSON |
| `cybersec host` | Local OS environment | Listening TCP/UDP ports, GTFOBins SUID/SGID binaries, world-writable critical paths, SSH config | Text summary or JSON |
| `cybersec triage` | Web access logs, auth logs (`/var/log/auth.log`) | Top attacking IPs, 4xx/5xx error anomalies, injection probes (`UNION`, `..`), SSH brute force | Forensic report or JSON |
| `cybersec patch` | Recipe identifier | Production-ready Python & TypeScript defensive templates | Formatted code or JSON |

---

## 3. Reference Manuals

* **[OWASP Top 10 Reference Guide](references/owasp-top10.md):** Deep breakdown of all 10 vulnerability categories, attack payloads, detection signatures, and concrete Python/TypeScript mitigations.
* **[GTFOBins Privilege Escalation Manual](references/gtfobins.md):** Catalogue of 24+ Unix binaries (`bash`, `python`, `find`, `vim`, `tar`, `node`, `php`, etc.) abused for SUID/sudo breakout, with step-by-step hardening instructions.

---

## 4. Defensive Playbooks

### Playbook A: Remediating Leaked Secrets
1. **Immediate Revocation:** Invalidate the exposed token at the service provider immediately (e.g. AWS IAM, GitHub Token Settings, Stripe Dashboard).
2. **Purge from Version Control:**
   ```bash
   # Remove secret completely from git history
   git filter-repo --invert-paths --path <file-with-secret>
   ```
3. **Environment Isolation:** Move credentials to environment variables loaded via `.env` files protected by `.gitignore`:
   ```bash
   echo ".env" >> .gitignore
   chmod 600 .env
   ```
4. **Validation:** Re-run `cybersec scan <repo-root>` to confirm zero leaked tokens.

---

### Playbook B: Remediating Injections (SQL, Command, Path Traversal)
1. **SQL Injection:**
   * Replace string formatting with engine-native parameterization.
   * Consult: `cybersec patch sql-param`
2. **Command Injection:**
   * Replace `shell=True` or `child_process.exec()` with explicit argument arrays passed to `subprocess.run(shell=False)` or `child_process.spawn()`.
   * Enforce strict input allowlisting (e.g. alphanumeric only) before arguments are passed.
3. **Path Traversal:**
   * Canonicalize the destination path with `os.path.realpath` / `path.resolve` and verify that the target path begins strictly with the base directory.
   * Consult: `cybersec patch safe-path`

---

### Playbook C: Remediating Supply Chain Risks
1. **Strip Lifecycle Execution in Automation:**
   * In CI pipelines, always execute `npm ci --ignore-scripts`.
   * In `.npmrc`, set `ignore-scripts=true`.
2. **Pin Concrete Versions:**
   * Replace wildcards (`*`) and unpinned semver prefixes (`^`, `~`) with exact versions.
   * Maintain committed lockfiles (`package-lock.json`, `Pipfile.lock`, `Cargo.lock`).
3. **Audit Manifests:**
   * Execute `cybersec deps package.json` prior to committing any package updates.

---

### Playbook D: Hardening Host Posture & SUID Breakouts
1. **Audit SUID Binaries:**
   * Run `cybersec host --json` and inspect `suid_binaries`.
   * Strip SUID bits from non-essential binaries:
     ```bash
     chmod u-s /usr/bin/find /usr/bin/vim /usr/bin/python3
     ```
2. **Secure `/etc/sudoers`:**
   * Never grant `NOPASSWD: ALL` to standard users.
   * Ensure `Defaults env_reset` is active to prevent `LD_PRELOAD` library injection.
3. **Harden OpenSSH (`/etc/ssh/sshd_config`):**
   * Set `PermitRootLogin no`.
   * Set `PasswordAuthentication no` (require SSH public key authentication).
   * Set `X11Forwarding no`.
4. **Restrict File Permissions:**
   * Ensure critical paths are not world-writable:
     ```bash
     chmod 644 /etc/passwd
     chmod 600 /etc/shadow
     ```

---

## 5. Pre-Submission Quality Checklist

Before submitting a cybersecurity report or PR review:
- [ ] Have all findings been gathered using automated scans (`cybersec scan`, `cybersec deps`, `cybersec host`, `cybersec triage`)?
- [ ] Is every vulnerability tagged strictly with `[CONFIRMED: file:line]`, `[POTENTIAL: file:line]`, or `[CLEAN: component]`?
- [ ] Are all proposed remediations backed by concrete, runnable code recipes (or validated via `cybersec patch <recipe>`)?
- [ ] Has path traversal logic been protected with canonical boundary validation?
- [ ] Have all SQL queries been converted to parameterized statements?
- [ ] Did you avoid theoretical speculation or unverified vulnerability claims?
