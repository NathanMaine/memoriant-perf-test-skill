# Memoriant Performance Test Skill

AI-powered load test plan generation and metrics interpretation for coding agents.

## Available Skills

### perf-test
Generate load test plans from service profiles. Produces steady, burst, and soak test configurations aligned to SLOs. Interprets metrics results and appends tamper-evident JSONL evidence entries.

Skill file: `skills/perf-test/SKILL.md`

## Available Agents

### perf-test-planner
Performance engineering specialist. Generates complete three-scenario load test plans, interprets result metrics, and produces append-only evidence records.

Agent file: `agents/perf-test-planner.md`

## Usage

### Claude Code
```bash
/install NathanMaine/memoriant-perf-test-skill
/perf-test
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
