---
name: managing-upstream-syncs
description: Use when you need to sync, fetch, update, or pull code from the upstream Pi repository.
---

# Managing Upstream Syncs (Sovereign Aery Protocol)

You are operating within the Aery architecture. Aery is a hard fork of the Pi (pi-mono) repository. It has its own branding, packages (`@eminent337/aery`), and custom features (Graph Engine, Swarm Coordinator).

## CRITICAL RULES
1. **NEVER AUTO-MERGE**: You are strictly forbidden from running `git merge upstream/main`, `git pull upstream`, or `git cherry-pick`. 
2. **MANUAL PORTING ONLY**: Because Aery is a sovereign fork with heavily rebranded namespaces (e.g., changing 'kimchi' and 'pi' to 'aery'), automated merges will cause catastrophic conflicts. You must deeply understand both codebases, extract the raw diffs from the upstream, and manually rewrite the fix from scratch to perfectly align with Aery's architecture.

## The Aery Integration Protocol (The 5 Pillars)

When dealing with the `upstream` remote, you MUST follow these 5 strictly enforced steps:

1. **Retire the Bulk Scripts**: You are forbidden from using automated `replaceAll` scripts or automated Git merges (`git merge`, `git cherry-pick`). Aery is a sovereign, distinct codebase. Blind automated merges will corrupt our rebranded architecture.
2. **Keep the Upstream Read-Only**: The `upstream` remote is purely for reference. You may only run `git fetch upstream` to view what the Pi developers are building. You will never push to it or merge from it.
3. **The "Surgical Agent" Workflow**: Whenever the user requests an upstream feature to be ported, you will extract the raw diff from upstream (`git show`), manually read the target file in the Aery repository, and perform a surgical, manual rewrite.
4. **Intelligent Rewrite (STRICT REBRANDING)**: You must actively translate the upstream code into the Aery ecosystem. **CRITICAL: You must meticulously rebrand the logic. You are explicitly forbidden from introducing ANY references, variable names, or strings from the upstream (e.g., 'pi', 'kimchi', 'pi-mono', 'oh my pi') into Aery.** Every feature must be built natively for Aery.
5. **Zero-Error Guarantee**: After injecting any code, you are absolutely required to run `bun run check:ts` and `bun run test` locally. You may not claim the task is complete until both the compiler and test suite return 0 errors.
