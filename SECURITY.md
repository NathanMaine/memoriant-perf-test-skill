# Security Policy

## What This Plugin Does

This plugin consists entirely of markdown instruction files (SKILL.md and agent .md files). It contains:
- No executable code
- No shell scripts
- No network calls
- No file system modifications beyond what Claude Code normally does

All operations (reading service profiles, writing evidence JSONL, interpreting metrics files) are performed by Claude Code itself using its standard tools, not by any code in this plugin.

## Evidence Log Handling

The skill instructs Claude Code to append evidence records to a JSONL file on the local filesystem. This file:
- Is written in append-only mode
- Contains only metadata (timestamps, service names, scenario names, outcomes) — no sensitive service internals
- Is never uploaded or transmitted by this plugin

## Reporting a Vulnerability

If you discover a security issue, please email nathan@memoriant.com (do not open a public issue).

We will respond within 48 hours and provide a fix timeline.

## Auditing This Plugin

This plugin is easy to audit:
1. All files are markdown — readable in any text editor
2. No `node_modules`, no Python packages, no compiled binaries
3. Review `skills/perf-test/SKILL.md` to see exactly what instructions are given to the AI
