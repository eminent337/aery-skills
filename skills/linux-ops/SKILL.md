---
name: linux-ops
description: Use when generating systemd background services, setting up process watchdogs, validating cron schedules, or inspecting Linux service health.
---

# Linux Resident System Services & Daemon Automation (`linux-ops`)

## Overview

The `linux-ops` skill provides a zero-dependency toolkit for managing background daemons, generating sandboxed `systemd` service units, supervising processes with automatic crash recovery, and validating cron automation schedules on Linux.

---

## 1. Quick Start

All capabilities are available via the standalone CLI engine at:
`.agents/skills/linux-ops/scripts/linux-ops`

```bash
# 1. Generate a production-hardened systemd service unit
python3 .agents/skills/linux-ops/scripts/linux-ops daemon --name my-app --exec "/usr/bin/node /app/index.js" --memory-max 512M [--json]

# 2. Inspect status of running systemd services
python3 .agents/skills/linux-ops/scripts/linux-ops status [service_name] [--json]

# 3. Supervise a command with an automatic crash restart watchdog
python3 .agents/skills/linux-ops/scripts/linux-ops watchdog --cmd "python3 worker.py" --max-restarts 3 --cooldown 2.0

# 4. Validate and translate a 5-field cron schedule
python3 .agents/skills/linux-ops/scripts/linux-ops cron "*/15 * * * *" [--json]
```

---

## 2. CLI Subcommands

| Subcommand | Key Options | Primary Function | Output Formats |
| :--- | :--- | :--- | :--- |
| `daemon` | `--name`, `--exec`, `--memory-max`, `--restart`, `--no-harden` | Generates systemd `.service` files with security sandboxing | Raw unit, file, or JSON |
| `status` | `[unit]`, `--json` | Reads live service state from systemctl | Text output or JSON |
| `watchdog` | `--cmd`, `--max-restarts`, `--cooldown`, `--dry-run` | Keeps critical processes alive with automatic crash recovery | Real-time supervisor log |
| `cron` | `<expression>`, `--json` | Parses and explains 5-field cron syntax | Human explanation or JSON |

---

## 3. Sandboxing Directives Included

When running `linux-ops daemon` without `--no-harden`, the generated unit automatically includes:
- `NoNewPrivileges=true`: Blocks privilege escalation.
- `ProtectSystem=full`: Mounts `/usr`, `/boot`, and `/etc` read-only.
- `ProtectHome=read-only`: Guards `/home` from unauthorized writes.
- `PrivateTmp=true`: Isolates `/tmp` namespace.
- `RestrictRealtime=true` & `RestrictSUIDSGID=true`: Disallows real-time scheduling priority and SUID binaries.
