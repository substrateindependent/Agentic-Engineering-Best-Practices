---
title: AI Agent Best Practices
date: 2026-02-22
source: direct input
tags: [ai-agents, observability, security, architecture, workflows]
---

## Summary

A curated set of architectural, operational, and security best practices for building AI agent systems — particularly relevant to the Guppie platform. Covers agent design patterns, observability tooling, security layering, workflow modeling, and subagent orchestration strategy.

## Agent Architecture

**Specialized vs. general agents** — Use #ai-agents in a specialized configuration when you need segregation of context and capabilities (e.g., isolating a security-sensitive skill from a general-purpose agent). Use a general agent when flexibility and shared context across tasks are more important.

**SKILLS.md open format** — Adopt the SKILLS.md open format as the standard definition format for agent skills. This creates a consistent, portable contract for how skills are declared and invoked.

**Scripts within skills** — Evaluate the appropriate use of scripts inside skills carefully. Scripts increase capability but also introduce attack surface and should be subject to the security layers described below.

**Model Routing Layer** — Implement a model routing layer to direct tasks to the most appropriate underlying model (e.g., fast/cheap vs. powerful/expensive), depending on task complexity and latency requirements.

## MCP Servers

Use MCP (Model Context Protocol) servers appropriately. They are most valuable when the #architecture requires an agent to have real-time access to an external system's current state — for example, querying a live database, checking current infrastructure status, or interacting with a third-party API in real time. Avoid over-applying MCP where static context or pre-fetched data would suffice.

## Observability

> [!tip] Key Principle
> Observability is not optional for autonomous agent systems. A pipeline that deploys code or takes actions without human review requires a full decision audit trail — Sentry and logs alone are not sufficient.

**Error tracking and uptime** — Use Sentry for error tracking and crash reporting across Guppie's platform and customer apps. Use Better Stack for logs and uptime monitoring.

**Agent tracing with OpenTelemetry** — Use #observability tooling centered on OpenTelemetry for agent tracing, piped into a backend that can visualize traces. Backend options include:

- **Jaeger** — Open source, self-hosted
- **Grafana Tempo** — Open source, integrates with Grafana stack
- **Honeycomb** — Managed service, accepts OpenTelemetry natively
- **Datadog** — Managed service, accepts OpenTelemetry natively

The Claude Agent SDK already supports tracing. LangSmith (if LangChain components are in use) also exports OpenTelemetry-compatible traces. This is primarily a configuration exercise, not heavy custom instrumentation.

**OpenTelemetry as a platform service** — Guppie should consider offering an OpenTelemetry-compatible trace backend as a platform-level service. This is not a replacement for Sentry or Better Stack — it is an additional layer that becomes critical specifically because of the autonomous agent architecture. A traditional SaaS app can operate adequately with Sentry and logs; an autonomous agent pipeline that deploys code without human review needs the full decision audit trail that only distributed tracing provides.

## Security

**Six-Layer Skill Defense** — Apply a layered #security model to all skills, particularly those that execute scripts or interact with external systems:

1. **Static analysis** — Lint and scan skill code before it is registered or executed
2. **Sandboxed audit** — Run skills in an isolated environment during the audit phase
3. **Capability-restricted audit agent** — Use a dedicated agent with minimal permissions to review skill behavior
4. **Script sandboxing** — Execute any scripts within skills in a sandboxed runtime
5. **Cross-provider audit** — Validate skill behavior against a second AI provider to reduce single-model blind spots
6. **Risk-tiered review** — Apply progressively stricter human review thresholds based on assessed risk level

This defense-in-depth model is essential to prevent bad actors from using the Guppie platform to build or deploy malicious applications.

## Workflow Modeling

**State machines for structured workflows** — Model the discovery conversation and the build pipeline as formal state machines. This makes transitions explicit, prevents invalid states, and simplifies debugging. Key workflows to model this way include the initial user onboarding/discovery conversation and the app build pipeline.

**To-do lists as persistent state objects** — Use to-do lists as persistent state objects rather than ephemeral instructions. This supports planning, progress tracking, and failure recovery — the agent can resume from a known state after an error without replaying the entire workflow from scratch.

## Subagent Spawning Strategy

Use a hybrid approach to subagent spawning:

- **Infrastructure decides when** — The platform determines the conditions under which subagents are spawned (e.g., task complexity thresholds, parallelism needs)
- **The agent decides how** — The orchestrating agent determines the approach and coordinates subagent behavior at runtime
- **Developers define what's available** — The set of available subagent types and their capabilities is defined by the developer at build time, not dynamically generated

This separation of concerns keeps spawning predictable and auditable while preserving flexibility.

## Related Notes

- [[claude-code-ai-agent-building-best-practices]] — Curated best practices and ideas from bcherny on building AI agents using Claude Code
