---
name: agent-eval
description: Use when evaluating AI agent tool-calling accuracy, calculating token cost across models, or detecting accuracy regressions across prompt versions.
---

# AI Agent Evaluation & Benchmarks (`agent-eval`)

## Overview

The `agent-eval` skill provides a zero-dependency benchmarking and evaluation framework for AI coding agents. It measures whether prompt updates, model tier switches, or tool refactors improve or degrade agent accuracy, parameters, and costs.

---

## 1. Quick Start

All capabilities are available via the standalone CLI engine at:
`.agents/skills/agent-eval/scripts/agent-eval`

```bash
# 1. Grade an agent tool-call execution run against ground truth
python3 .agents/skills/agent-eval/scripts/agent-eval grade ground_truth.json results.json [--min-pass-rate 90] [--json]

# 2. Calculate exact token costs across Gemini, Claude, and OpenAI
python3 .agents/skills/agent-eval/scripts/agent-eval cost -m gemini-2.5-flash -p 120000 -c 4500 [--cache-tokens 50000] [--json]
python3 .agents/skills/agent-eval/scripts/agent-eval cost -m claude-3-7-sonnet -p 80000 -c 2000 [--json]

# 3. Detect accuracy regressions between two test runs
python3 .agents/skills/agent-eval/scripts/agent-eval regression baseline.json current.json [--json]
```

---

## 2. CLI Subcommands

| Subcommand | Arguments / Options | Primary Checks | Output Formats |
| :--- | :--- | :--- | :--- |
| `grade` | `ground_truth.json`, `results.json`, `--min-pass-rate` | Tool match %, parameter alignment, extra hallucinated arguments | Text report or JSON |
| `cost` | `-m/--model`, `-p/--prompt-tokens`, `-c/--completion-tokens`, `--cache-tokens` | Pricing across Gemini, Claude, and OpenAI | Formatted USD cost or JSON |
| `regression` | `baseline.json`, `current.json` | Identifies cases that passed in baseline but failed in current | Regression list or JSON |

---

## 3. Dataset Format

### Ground Truth Dataset (`ground_truth.json`)
```json
[
  {
    "id": "find-ts-files",
    "prompt": "Find all typescript files in src",
    "expected_tool": "run_command",
    "expected_args": {
      "CommandLine": "find src -name '*.ts'"
    }
  }
]
```

### Actual Agent Results (`results.json`)
```json
[
  {
    "id": "find-ts-files",
    "tool": "run_command",
    "args": {
      "CommandLine": "find src -name '*.ts'"
    },
    "latency_ms": 420
  }
]
```
