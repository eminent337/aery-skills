---
name: perf-opt
description: Use when profiling process CPU or memory usage, diagnosing memory leaks, measuring event loop lag, or benchmarking execution throughput.
---

# Performance, Memory & Latency Profiling (`perf-opt`)

## Overview

The `perf-opt` skill provides a zero-dependency, high-precision profiling engine for diagnosing performance bottlenecks, tracking memory leaks over time, measuring event-loop scheduling jitter, and benchmarking command execution throughput.

It is purpose-built for CLI engines, native processes (Rust, Bun, Node), long-running background tasks, and AI agent workloads.

---

## 1. Quick Start

All capabilities are available via the standalone CLI engine at:
`.agents/skills/perf-opt/scripts/perf-opt`

```bash
# 1. Profile a spawned command or background PID (CPU, RSS Memory, Leak Detection)
python3 .agents/skills/perf-opt/scripts/perf-opt profile --cmd "bun run build" [--interval 0.05] [--json]
python3 .agents/skills/perf-opt/scripts/perf-opt profile --pid 12345 --duration 10 [--json]

# 2. Benchmark command execution throughput, mean/median, and percentiles
python3 .agents/skills/perf-opt/scripts/perf-opt bench "bun test" --iterations 10 --warmup 2 [--json]

# 3. Measure event-loop latency and scheduling lag spikes
python3 .agents/skills/perf-opt/scripts/perf-opt eventloop --duration 3.0 --interval-ms 10.0 [--json]

# 4. Analyze V8 / JSON heap snapshots and allocation distributions
python3 .agents/skills/perf-opt/scripts/perf-opt heap snapshot.json [--limit 10] [--json]
```

---

## 2. CLI Subcommands

| Subcommand | Key Options | Primary Output | Use Case |
| :--- | :--- | :--- | :--- |
| `profile` | `--cmd`, `--pid`, `--interval`, `--duration`, `--json` | Peak/avg RSS, CPU %, memory growth slope (MB/min), leak warning | Tracking memory leaks during long-running tasks |
| `bench` | `--iterations`, `--warmup`, `--allow-failure`, `--json` | Mean, median, stddev, min/max, P90, P99, ops/sec | Fast comparison between code implementations |
| `eventloop` | `--duration`, `--interval-ms`, `--json` | P95/P99 lag, max spike, jitter analysis | Detecting blocking sync operations on the event loop |
| `heap` | `--limit`, `--json` | Total heap size, top retained allocations by type | Inspecting memory bloat from heap dumps |

---

## 3. Detecting Memory Leaks

When running `perf-opt profile`:
- The engine computes the linear regression slope of RSS memory over time.
- If memory grows continuously at rate $> 5$ MB/min and total memory expands by $> 20\%$, `perf-opt` flags:
  `[WARNING] Potential memory leak detected (monotonic growth)!`
- In JSON mode (`--json`), `leak_detected: true` is returned for programmatic assertions.

---

## 4. Benchmarking Best Practices

1. **Always use warmup runs:** JIT compilers (V8, Bun, PyPy) optimize hot code paths after initial executions. Set `--warmup 2` or `--warmup 5`.
2. **Review P90 and P99 percentiles:** Averages hide latency tail spikes. If `p99` is significantly higher than `mean`, look for GC pauses or I/O locks.
3. **Capture JSON for automated CI gates:**
   ```bash
   perf-opt bench "my-cli --version" --json | jq '.p99_ms'
   ```
