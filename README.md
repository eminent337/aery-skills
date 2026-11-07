# Aery Skills Hub (`aery-skills`)

Official on-demand skills repository for the [Aery](https://github.com/eminent337/aery) AI coding agent.

## Overview

The Aery Skills Hub allows Aery users to discover and install community and official skills on demand without bundling them into the core Aery binary.

## Installing Skills in Aery

Inside any Aery session:

```bash
# Browse the Skill Hub catalog
/skill hub

# Search available skills
/skill search <query>

# Install a skill on demand
/skill install ai-humanizer

# Uninstall a skill
/skill uninstall ai-humanizer
```

## Available Skills

| Name | Version | Category | Description |
| :--- | :--- | :--- | :--- |
| [`ai-humanizer`](skills/ai-humanizer/SKILL.md) | `1.2.0` | Writing & Forensics | Measures burstiness, eliminates AI cliché markers, and humanizes text to pass Scribbr/Turnitin. |

## Authoring a Skill

1. Create a directory under `skills/<skill-name>/` containing `SKILL.md`.
2. (Optional) Add CLI helpers under `skills/<skill-name>/scripts/`.
3. Add the skill entry to `skills.json`.
