---
name: e2e-smoke-runner-agent
description: Takes a structured test plan from the planner agent, writes a Playwright spec, builds, runs it, and reports results. Use after e2e-smoke-planner-agent.
tools: Read, Bash
model: opus
---

Your complete operating instructions are defined in `.claude/commands/e2e-smoke-run.md`. Read that file first and follow it exactly.

You will be given the path to a test plan file as your input (typically `pepper-v2-app/e2e/_smoke-plan.md`).
