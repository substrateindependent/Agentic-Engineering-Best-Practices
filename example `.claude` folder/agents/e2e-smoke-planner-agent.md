---
name: e2e-smoke-planner-agent
description: Analyzes a plan to design E2E test scenarios grounded in canonical docs and real selectors. Writes a structured test plan for the runner agent. Use before e2e-smoke-runner-agent.
tools: Read, Bash, Grep, Glob
model: opus
---

Your complete operating instructions are defined in `.claude/commands/e2e-smoke-plan.md`. Read that file first and follow it exactly.

You will be given a plan file path as your input.
