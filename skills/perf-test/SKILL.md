---
name: perf-test
description: Generate load test plans from service profiles. Produces steady, burst, and soak test configurations with SLO-aligned checks, interprets metrics results, and logs tamper-evident evidence records.
---

# Performance & Load Test Plan Generation Skill

## Purpose

This skill generates structured load test plans for any service given a service profile (baseline RPS, peak RPS, burst factor, SLOs, endpoints). It outputs three canonical test scenarios — steady-state, burst, and soak — with stage configurations, SLO-aligned checks, and metrics to watch. It also interprets results from previous test runs and appends evidence records to an append-only JSONL log.

This skill does NOT execute tests. Tools like k6, Locust, JMeter, or Gatling remain separate. This skill plans what to run and why.

## When to Use This Skill

Use this skill when the user wants to:
- Generate a load test plan before writing k6/Locust/JMeter scripts
- Define SLO-aligned pass/fail thresholds for a new or existing service
- Interpret JSON or CSV metrics output from a previous load test run
- Produce evidence that a performance plan/assessment was created (for compliance, for sprint definition of done, for architecture review)
- Compare two test runs to detect regressions

## Required Inputs

### Service Profile (YAML or JSON)

Collect these fields from the user, or prompt them to provide a profile file:

```yaml
service: <name>             # e.g., checkout-api
summary: <description>      # What the service does
traffic:
  baseline_rps: <number>    # Normal steady-state load
  peak_rps: <number>        # Expected peak traffic
  burst_factor: <number>    # Multiplier for burst tests (e.g., 3 = 3x peak)
slo:
  latency_ms:
    p95: <ms>               # 95th percentile latency ceiling
    p99: <ms>               # 99th percentile latency ceiling
  error_rate: <decimal>     # e.g., 0.01 = 1% max error rate
endpoints:
  - path: <path>
    method: <GET|POST|...>
    critical: <true|false>
dependencies:
  - <service-name>          # Downstream services (for context)
data:
  uses_production_data: <bool>
  notes: <string>
```

If the user provides incomplete information, ask for the missing fields before proceeding. At minimum you need: `baseline_rps`, `peak_rps`, `slo.latency_ms.p95`, and `slo.error_rate`.

## Methodology: Three Canonical Scenarios

### 1. Steady-State Test

**Purpose:** Validate baseline behavior under normal load. Confirm the service meets SLOs at expected traffic levels.

**Structure:**
- Ramp up to `baseline_rps` over 2 minutes
- Hold at `baseline_rps` for 10 minutes
- Ramp down over 1 minute
- Total: ~13 minutes

**Checks:**
- p95 latency < `slo.latency_ms.p95`
- p99 latency < `slo.latency_ms.p99`
- error rate < `slo.error_rate`
- All critical endpoints return expected status codes

**Pass Criteria:** 100% of checks pass throughout the hold period.

### 2. Burst Test

**Purpose:** Validate behavior under sudden traffic spikes. Confirm the service degrades gracefully, recovers, and does not cascade failures to dependencies.

**Structure:**
- Ramp up to `baseline_rps` over 1 minute
- Spike to `peak_rps * burst_factor` over 30 seconds
- Hold burst load for 3 minutes
- Return to `baseline_rps` over 1 minute
- Hold recovery for 3 minutes
- Total: ~9 minutes

**Checks:**
- During burst: error rate < `slo.error_rate * 2` (relaxed during spike)
- During recovery: error rate returns to < `slo.error_rate` within 60 seconds
- During recovery: p95 latency returns to < `slo.latency_ms.p95 * 1.5` within 60 seconds
- No circuit breaker trips that don't self-heal

**Pass Criteria:** Service recovers to SLO within 60 seconds of burst ending.

### 3. Soak Test

**Purpose:** Detect memory leaks, connection pool exhaustion, file descriptor leaks, and gradual degradation under sustained load.

**Structure:**
- Ramp up to `peak_rps` over 5 minutes
- Hold at `peak_rps` for 60 minutes (or user-specified duration)
- Ramp down over 5 minutes
- Total: ~70 minutes

**Checks:**
- p95 and p99 latency must not degrade more than 20% between the first 10 minutes and last 10 minutes of the hold
- Error rate must remain < `slo.error_rate` throughout
- Memory usage (if instrumented) must not grow more than 15% between first and last 10 minutes
- Thread/connection pool must not trend toward exhaustion

**Pass Criteria:** No statistically significant degradation across the hold window.

## Output Format

Generate the plan as structured JSON (or present it as YAML if the user prefers). Each scenario must include:

```json
{
  "service": "<name>",
  "generated_at": "<ISO8601 timestamp>",
  "slo_reference": {
    "latency_ms_p95": <number>,
    "latency_ms_p99": <number>,
    "error_rate": <decimal>
  },
  "scenarios": [
    {
      "name": "steady",
      "stages": [
        {"duration": "2m", "target_rps": <baseline>},
        {"duration": "10m", "target_rps": <baseline>},
        {"duration": "1m", "target_rps": 0}
      ],
      "checks": [...],
      "metrics_to_watch": [...],
      "pass_criteria": "..."
    },
    {
      "name": "burst",
      "stages": [...],
      "checks": [...],
      "metrics_to_watch": [...],
      "pass_criteria": "..."
    },
    {
      "name": "soak",
      "stages": [...],
      "checks": [...],
      "metrics_to_watch": [...],
      "pass_criteria": "..."
    }
  ]
}
```

Also produce a human-readable Markdown summary of the plan.

## Metrics Interpretation

When the user provides metrics output (JSON or CSV) from a completed test run, interpret it against the service profile SLOs.

### Expected Metrics Format (JSON)

```json
{
  "p50_ms": <number>,
  "p90_ms": <number>,
  "p95_ms": <number>,
  "p99_ms": <number>,
  "error_rate": <decimal>,
  "throughput_rps": <number>,
  "cpu_percent": <number>,
  "memory_percent": <number>
}
```

### Interpretation Logic

For each SLO check:

| Check | Pass | Warning | Fail |
|-------|------|---------|------|
| p95 latency | < slo.p95 | < slo.p95 * 1.2 | >= slo.p95 * 1.2 |
| p99 latency | < slo.p99 | < slo.p99 * 1.2 | >= slo.p99 * 1.2 |
| error rate | < slo.error_rate | < slo.error_rate * 2 | >= slo.error_rate * 2 |
| throughput | >= target_rps * 0.95 | >= target_rps * 0.9 | < target_rps * 0.9 |

Report each check as PASS, WARN, or FAIL with the actual value vs. the threshold. Produce a summary verdict of PASS, WARN, or FAIL based on the worst check result.

If resource metrics (cpu_percent, memory_percent) are provided, flag if CPU > 80% or memory > 85% as potential resource pressure warnings.

## Evidence Logging

After generating a plan or interpreting metrics, append a JSONL evidence entry to the user's evidence log (default: `./evidence.jsonl`). Ask the user where they want the log written if not specified.

### Evidence Entry Format

```json
{"ts":"<ISO8601>","service":"<name>","profile":"<path or inline>","scenarios":["steady","burst","soak"],"interpretation":<bool>,"outcome":"<outcome>","plan_hash":"<sha256 of plan JSON>"}
```

**Outcome values:**
- `plan-generated` — A new plan was generated, no metrics provided
- `plan-and-interpretation-generated` — Plan generated and metrics interpreted
- `interpretation-only` — Only metrics interpreted, no new plan generated
- `issues-detected` — Interpretation found FAIL-level results

**The evidence log is append-only.** Never modify or delete existing entries. Each new run appends a new line.

## Step-by-Step Procedure

1. **Collect or parse the service profile.** If the user provides a file path, read it. If inline, parse it. Validate required fields are present.

2. **Confirm the test scope.** Ask: "Should I generate all three scenarios (steady, burst, soak), or a specific subset?" Default to all three.

3. **Generate the plan.** Apply the three-scenario methodology with stage calculations based on the profile values. Output structured JSON + Markdown summary.

4. **Interpret metrics (if provided).** Apply the SLO comparison logic. Report PASS/WARN/FAIL per check and an overall verdict.

5. **Append evidence.** Write a JSONL line to the evidence log. Report the log path and entry to the user.

6. **Optionally produce k6/Locust pseudocode.** If the user asks, generate pseudocode stages for their test runner of choice. Do not write actual executable scripts unless specifically asked.

## Examples

### Example 1: Generate a plan from a profile

**User:** "Generate a load test plan for our payments service. It handles 100 RPS baseline, 400 RPS peak, burst factor 3. SLOs: p95 < 300ms, p99 < 600ms, error rate < 0.5%."

**Claude should:**
1. Construct the service profile from the stated values
2. Generate all 3 scenarios with calculated stages
3. Output the JSON plan and Markdown summary
4. Append an evidence entry
5. Note which endpoints weren't specified (ask if the user wants to add them)

### Example 2: Interpret failing metrics

**User:** "Here are our soak test results: p95=480ms, p99=950ms, error_rate=0.008, throughput=380. SLO is p95<300, p99<600, error<0.01."

**Claude should:**
1. Evaluate: p95=480 vs 300 → FAIL (1.6x), p99=950 vs 600 → FAIL (1.58x), error=0.008 vs 0.01 → PASS, throughput=380 (adequate)
2. Overall verdict: FAIL
3. Suggest investigation areas: latency degradation in soak tests indicates likely memory leak or connection pool exhaustion
4. Append an `issues-detected` evidence entry

### Example 3: k6 stage pseudocode

**User:** "Give me k6 stages for the burst scenario for the checkout-api (50 baseline, 200 peak, burst factor 3)."

**Claude should:**
Generate stage configuration:
- Ramp to 50 RPS over 60s
- Spike to 600 RPS over 30s
- Hold 600 RPS for 3m
- Return to 50 RPS over 60s
- Hold 50 RPS for 3m

## Error Handling

| Situation | Response |
|-----------|----------|
| Missing baseline_rps or peak_rps | Ask user for these values before proceeding |
| No SLO provided | Use conservative defaults (p95 < 500ms, error < 1%) but flag these as assumptions |
| Metrics format unrecognized | Ask user to describe the metrics format or share a sample |
| Evidence log path doesn't exist | Create it (append mode). Note the path to the user. |
| burst_factor produces implausible load (e.g., 10,000 RPS) | Flag the value and confirm with the user |

## Version History

- 1.0.0 — Initial release. Three-scenario methodology, SLO interpretation, JSONL evidence logging.
