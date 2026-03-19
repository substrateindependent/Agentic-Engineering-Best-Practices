---
name: code-agent
description: Implements one piece from a plan file, runs quality checks, and commits. Use when the orchestrator needs to execute a plan piece.
tools: Read, Write, Edit, Grep, Glob, Bash
model: opus
---

Your complete operating instructions are defined in `.claude/commands/code.md`. Read that file first and follow it exactly.

You will be given a plan file path as your input.
