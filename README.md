# 🎯 Aery Skills Hub (`aery-skills`)

Official on-demand skill registry for [Aery](https://github.com/eminent337/aery).

Aery keeps its core binary lean and lightweight. Skills are distributed and discovered via this external hub, allowing users to browse and install domain-specific capabilities on demand.

---

## 📦 Available Skills

| Skill | Category | Version | Description |
|---|---|---|---|
| [`ai-humanizer`](skills/ai-humanizer) | Writing & Forensics | `1.2.0` | Evade AI detectors (Scribbr, Turnitin, QuillBot) with 0% AI detection; burstiness & perplexity rhythm modulation. |
| [`managing-upstream-syncs`](skills/managing-upstream-syncs) | Upstream Sync | `2.0.0` | Sovereign Aery protocol for inspecting, diffing, and surgically porting upstream commits with automated rebranding audits. |
| [`leanai`](skills/leanai) | Reverse Engineering | `1.0.0` | Forensic binary reverse engineering, electron/protocol analysis, JS deobfuscation, and schema extraction. |
| [`agent-eval`](skills/agent-eval) | Evaluation & Quality | `1.0.0` | Evaluate AI agent tool-calling accuracy, benchmark token costs, and detect regressions across models. |
| [`cybersec`](skills/cybersec) | Security | `1.0.0` | Inspect codebases for vulnerabilities, audit host posture, triage logs, and apply defensive hardening recipes. |
| [`linux-ops`](skills/linux-ops) | Systems & DevOps | `1.0.0` | Generate systemd background services, configure process watchdogs, and inspect Linux service health. |
| [`perf-opt`](skills/perf-opt) | Performance & Profiling | `1.0.0` | Profile CPU/memory usage, diagnose memory leaks, measure event loop lag, and benchmark throughput. |

---

## ⚡ Usage in Aery

Inside any Aery session:

```bash
# 1. View all installed and available hub skills
/skill

# 2. Install a skill on demand
/skill install ai-humanizer
/skill install managing-upstream-syncs

# 3. Uninstall when no longer needed
/skill uninstall ai-humanizer
```

Installed skills are saved to `~/.aery/skills/<skill-name>` and are immediately available to the agent.

---

## 🛠️ Repository Structure

```
aery-skills/
├── skills.json             # Registry manifest & catalog index
└── skills/
    ├── ai-humanizer/       # AI detector evasion & rhythm analysis engine
    ├── managing-upstream-syncs/ # Sovereign upstream sync & rebranding audit CLI
    ├── leanai/             # Forensic reverse-engineering skill
    ├── agent-eval/         # Agent benchmark & accuracy evaluation
    ├── cybersec/           # Security audit & defensive hardening
    ├── linux-ops/          # Linux systems & systemd operations
    └── perf-opt/           # Performance profiling & optimization
```

---

## 📄 License
MIT © [eminent337](https://github.com/eminent337)
