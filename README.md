<p align="center">
  <img src="https://img.shields.io/badge/claude--code-plugin-8A2BE2" alt="Claude Code Plugin" />
  <img src="https://img.shields.io/badge/skills-1-blue" alt="1 Skill" />
  <img src="https://img.shields.io/badge/agents-1-green" alt="1 Agent" />
  <img src="https://img.shields.io/badge/license-MIT-green" alt="MIT License" />
</p>

# Memoriant Performance Test Skill

A Claude Code plugin that generates complete load test plans from service profiles. Describe your service's traffic shape and SLOs — get back steady-state, burst, and soak test configurations with stage durations, SLO-aligned checks, and metrics to watch. Interprets results from previous runs and appends tamper-evident evidence records.

**No servers. No Docker. Just install and use.**

## Install

```bash
/install NathanMaine/memoriant-perf-test-skill
```

## Cross-Platform Support

### Claude Code (Primary)
```bash
/install NathanMaine/memoriant-perf-test-skill
```

### OpenAI Codex CLI
```bash
git clone https://github.com/NathanMaine/memoriant-perf-test-skill.git ~/.codex/skills/perf-test
codex --enable skills
```

### Gemini CLI
```bash
gemini extensions install https://github.com/NathanMaine/memoriant-perf-test-skill.git --consent
```

## Skills

| Skill | Command | What It Does |
|-------|---------|-------------|
| **Performance Test** | `/perf-test` | Generate steady, burst, and soak test plans from a service profile. Interpret metrics results. Log JSONL evidence. |

## Agent

| Agent | Best Model | Specialty |
|-------|-----------|-----------|
| **Performance Test Planner** | Sonnet 4.6 | Load plan generation, SLO validation, metrics interpretation, evidence logging |

## Quick Start

```bash
# Generate a load test plan
/perf-test

# Or trigger directly
"Generate a load test plan for our checkout API — 50 RPS baseline, 200 peak, burst factor 3. SLOs: p95 < 300ms, error rate < 1%."

# Interpret results from a previous run
"Here are my soak test results: p95=480ms, p99=950ms, error_rate=0.008. SLO is p95<300, p99<600."
```

## What You Get

For each service profile, the skill generates three test scenarios:

| Scenario | Purpose | Duration |
|----------|---------|---------|
| **Steady-state** | Validate normal load meets SLOs | ~13 minutes |
| **Burst** | Validate graceful degradation under spikes | ~9 minutes |
| **Soak** | Detect memory leaks and gradual degradation | ~70 minutes |

Each scenario includes:
- Stage-by-stage RPS targets and durations
- SLO-aligned pass/fail checks
- Metrics to watch during the test
- Clear pass criteria

## Service Profile Format

```yaml
service: checkout-api
summary: Handles checkout flows for web and mobile clients.
traffic:
  baseline_rps: 50
  peak_rps: 200
  burst_factor: 3
slo:
  latency_ms:
    p95: 300
    p99: 600
  error_rate: 0.01
endpoints:
  - path: /cart/submit
    method: POST
    critical: true
```

## Evidence Logging

Every plan generation or metrics interpretation appends a record to a local JSONL evidence log:

```json
{"ts":"2025-03-25T14:30:00Z","service":"checkout-api","scenarios":["steady","burst","soak"],"interpretation":false,"outcome":"plan-generated","plan_hash":"sha256:..."}
```

The log is append-only — suitable for compliance and sprint definition-of-done verification.

## Source

Built from [NathanMaine/agent-perf-test-generator](https://github.com/NathanMaine/agent-perf-test-generator).

## License

MIT
