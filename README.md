# Aery Skills Hub (`aery-skills`)

Official on-demand skills repository for the [Aery](https://github.com/eminent337/aery) AI coding agent.

## Overview

The Aery Skills Hub allows Aery users to discover and install specialized engineering and development skills on demand without bundling them into the core Aery binary.

## Installing Skills in Aery

Inside any Aery session:

```bash
# Discover all skills (installed and available)
/skill

# Install a skill on demand
/skill install ai-humanizer
/skill install leanai
/skill install cybersec

# Uninstall a skill
/skill uninstall ai-humanizer
```

## Available Skills (21 Total)

| Skill | Category | Description |
| :--- | :--- | :--- |
| [`agent-eval`](skills/agent-eval/SKILL.md) | **AI Agent Evaluation** | Use when evaluating AI agent tool-calling accuracy, calculating token cost across models, or detecting accuracy regressions across prompt versions. |
| [`ai-humanizer`](skills/ai-humanizer/SKILL.md) | **Writing & Forensics** | Use when analyzing text for AI detection markers, evaluating burstiness or perplexity scores, or rewriting content to evade AI detectors like Scribbr, Turnitin, or QuillBot. |
| [`brainstorming`](skills/brainstorming/SKILL.md) | **Agent Development Workflow** | "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements and design before implementation." |
| [`cybersec`](skills/cybersec/SKILL.md) | **Security & Vulnerability** | Use when inspecting codebases for vulnerabilities, auditing host posture, triaging web/auth logs, auditing supply chain dependencies, or applying defensive security hardening patch recipes. |
| [`dispatching-parallel-agents`](skills/dispatching-parallel-agents/SKILL.md) | **Agent Development Workflow** | Use when facing 2+ independent tasks that can be worked on without shared state or sequential dependencies |
| [`executing-plans`](skills/executing-plans/SKILL.md) | **Agent Development Workflow** | Use when you have a written implementation plan to execute in a separate session with review checkpoints |
| [`finishing-a-development-branch`](skills/finishing-a-development-branch/SKILL.md) | **Agent Development Workflow** | Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup |
| [`leanai`](skills/leanai/SKILL.md) | **Architecture & Reverse Engineering** | Use when inspecting application architectures, decoding binary formats and protocols, analyzing Electron/Go/native runtimes, or extracting type schemas and API contracts for interoperability. |
| [`linux-ops`](skills/linux-ops/SKILL.md) | **System & Infrastructure** | Use when generating systemd background services, setting up process watchdogs, validating cron schedules, or inspecting Linux service health. |
| [`managing-upstream-syncs`](skills/managing-upstream-syncs/SKILL.md) | **Git & Repository** | Use when you need to sync, fetch, update, or pull code from the upstream Pi repository. |
| [`perf-opt`](skills/perf-opt/SKILL.md) | **Performance & Profiling** | Use when profiling process CPU or memory usage, diagnosing memory leaks, measuring event loop lag, or benchmarking execution throughput. |
| [`receiving-code-review`](skills/receiving-code-review/SKILL.md) | **Agent Development Workflow** | Use when receiving code review feedback, before implementing suggestions, especially if feedback seems unclear or technically questionable - requires technical rigor and verification, not performative agreement or blind implementation |
| [`requesting-code-review`](skills/requesting-code-review/SKILL.md) | **Agent Development Workflow** | Use when completing tasks, implementing major features, or before merging to verify work meets requirements |
| [`subagent-driven-development`](skills/subagent-driven-development/SKILL.md) | **Agent Development Workflow** | Use when executing implementation plans with independent tasks in the current session |
| [`systematic-debugging`](skills/systematic-debugging/SKILL.md) | **Agent Development Workflow** | Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes |
| [`test-driven-development`](skills/test-driven-development/SKILL.md) | **Agent Development Workflow** | Use when implementing any feature or bugfix, before writing implementation code |
| [`using-git-worktrees`](skills/using-git-worktrees/SKILL.md) | **Agent Development Workflow** | Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback |
| [`using-superpowers`](skills/using-superpowers/SKILL.md) | **Agent Development Workflow** | Use when starting any conversation - establishes how to find and use skills, requiring Skill tool invocation before ANY response including clarifying questions |
| [`verification-before-completion`](skills/verification-before-completion/SKILL.md) | **Agent Development Workflow** | Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output before making any success claims; evidence before assertions always |
| [`writing-plans`](skills/writing-plans/SKILL.md) | **Agent Development Workflow** | Use when you have a spec or requirements for a multi-step task, before touching code |
| [`writing-skills`](skills/writing-skills/SKILL.md) | **Agent Development Workflow** | Use when creating new skills, editing existing skills, or verifying skills work before deployment |

## Authoring a Skill

1. Create a directory under `skills/<skill-name>/` containing `SKILL.md`.
2. (Optional) Add executable helpers under `skills/<skill-name>/scripts/`.
3. Add the skill entry to `skills.json`.
