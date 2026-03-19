# Observability in AI-Assisted Development

**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Externalized State](Externalized-State.md) · [Verification and Review](Verification-and-Review.md) · [Canonical Documentation](Canonical-Documentation.md) · [Agentic Workflow Guide](agentic-workflow-guide.md)

---

## Why Observability Is Non-Negotiable

This is [Principle 13](AI-Coding-Best-Practices.md#13-observability-is-not-optional) of the parent guide. If you can't see what the AI did and why, you can't debug failures, learn from mistakes, or improve your process.

Observability isn't instrumentation — it's visibility into intent vs. outcome at every step. In traditional software development, observability is optional because you wrote the code yourself and know what it should do. In AI-assisted development, the AI made decisions you didn't write. Understanding those decisions — and whether they matched what you asked for — is the only way to spot systematic issues before they compound into bugs.

The teams that scale AI coding successfully have one thing in common: comprehensive observability. They can point to exactly what was planned, what was generated, which tests passed, which model produced which output, and why each decision was made. The teams that fail at scale invariably have insufficient observability — they can't trace back from a failed test to the original specification, or they discover cost overruns without knowing where tokens were spent.

---

## What to Observe

Observability in AI coding means tracking these signals across the entire development loop:

**Planning vs. Reality**
- What was specified in the canonical document
- What was included in the feature plan
- What was actually built
- Divergences and why they occurred

**Test Results Per Stage**
- Unit test results after each task
- Component test results after implementation
- Full E2E suite results after integration
- Which tests are new, which are regressions

**Model and Prompt Per Output**
- Which model generated each artifact (research, plan, code, tests)
- The prompt version used (if you version your system prompts)
- Token counts per step
- Cost per step

**Tool Calls and Decisions**
- Which files were read and when
- Which tools were called and in what order
- Tool results (especially errors and edge cases)
- Why the agent made each architectural decision

**Decision Chains**
- The full reasoning path from problem to solution
- Validation results at each stage
- Issues found and how they were fixed
- The mapping from build plan tasks to git commits

---

## Two Tiers

Observability scales with project complexity. Most teams need one or both.

### Tier 1: Lightweight (Individual Developers)

For solo developers or small teams, lightweight observability uses what you already have:

**Git as audit trail.** Every task completed = one commit with a descriptive message. The message includes what was implemented, what was tested, and any decisions made. Over time, your git log becomes a readable narrative of what the AI built and why.

```
commit abc1234
Author: Claude <claude@ai>
Date:   Feb 23 2026

  Implement authentication with OAuth2 (Task 3)

  - Added UserModel with password hashing
  - Created /auth endpoints (login, logout, callback)
  - Tests: 12/12 passing (auth-unit.test.js)
  - Integration: verified with existing user routes
  - Model: Claude-opus, tokens: 4,200
  - Decisions: Used bcrypt for hashing, not argon2 — faster iteration cycles

commit def5678
Author: Claude <claude@ai>
Date:   Feb 23 2026

  Build tests for authentication (Task 2)

  - Added auth-unit.test.js: oauth flow, token refresh, error cases
  - Coverage: 89% (acceptable for auth; higher in unit phase)
  - Model: Claude-sonnet, tokens: 1,800
```

**Build plan with checkboxes.** Your markdown build plan lives in the repo. As each task completes, check it off. This gives you at-a-glance progress tracking and a clear record of what was completed in what order.

```markdown
## Build Plan: Authentication Feature

- [x] Task 1: Research OAuth2 patterns (research phase)
- [x] Task 2: Build tests (unit phase)
- [x] Task 3: Implement authentication (implementation phase)
- [x] Task 4: Code review (review phase)
- [ ] Task 5: Integration audit (audit phase)
- [ ] Task 6: E2E testing (verify phase)
```

**Markdown notes per session.** Keep a NOTES.md in your docs/ folder that tracks what happened in each session: what was planned, what was executed, what failed, what succeeded. Include token counts, model choices, and any anomalies.

```markdown
## Session Log: Auth Feature Build

**Date:** Feb 23, 2026
**Duration:** 2h 15m
**Model:** Claude-opus (research, audit), Claude-sonnet (implementation)

### What Went Well
- OAuth2 research converged quickly (90 min)
- Implementation of login endpoint was 1st-pass correct
- Test suite comprehensive and passing

### Issues Found & Fixed
- Token refresh endpoint returned wrong format (found in code review)
- Error handling in logout missing try-catch (found in audit)
- Integration Contract gap: forgot to update /api/user endpoint

### Metrics
- Total tokens (research): 5,200
- Total tokens (implementation): 8,900
- Total tokens (review/audit): 3,400
- All tests: 23/23 passing
- Code coverage: 87%

### Next Steps
- E2E tests for full auth flow
- Update canonical docs for auth system
```

This tier requires discipline but no additional tools. It's auditable, repeatable, and sufficient for teams up to ~10 developers.

### Tier 2: Production-Grade (Teams and Automation)

When you scale to multiple parallel agent sessions, continuous integration pipelines, or cost-sensitive automation, lightweight observability hits its limits. You need structured observability infrastructure.

**OpenTelemetry for AI agents.** OpenTelemetry (OTel) is the industry standard for distributed tracing. Each step the agent takes becomes a **span** — a named, timestamped unit of work with metadata.

```
Span hierarchy for a feature build:

[Feature Implementation] (root span, 2h 15m)
├── [Research Phase] (45m)
│   ├── [Read canonical docs]
│   ├── [Explore API docs]
│   └── [Write research doc]
├── [Implementation Phase] (90m)
│   ├── [Task 1: Implement OAuth2]
│   │   ├── [Read existing auth code]
│   │   ├── [Write new auth module]
│   │   ├── [Run unit tests]
│   │   └── [Git commit]
│   ├── [Task 2: Add login endpoint]
│   │   ├── [Write endpoint code]
│   │   ├── [Run integration tests]
│   │   └── [Git commit]
│   └── [Task 3: Error handling]
└── [Review Phase] (30m)
    ├── [Code review pass]
    ├── [Deep audit pass]
    └── [Fix issues found]
```

Each span captures:
- **Attributes:** The task name, file being edited, test results, any errors
- **Events:** Significant milestones (test passed, error found, decision made)
- **Metrics:** Token count, latency, cost for this span
- **Links:** Parent-child relationships and causality

**OTel GenAI Semantic Conventions.** In 2025–2026, OpenTelemetry adopted GenAI semantic conventions — a standard way to record LLM-specific data. This includes:

- **Prompts and completions** (optionally redacted for privacy)
- **Model name and provider** (e.g., claude-opus-4-6, gpt-4-turbo)
- **Token usage** (input tokens, output tokens, cached tokens)
- **Tool and agent calls** (which tool was invoked, with what parameters, what was the result)
- **Finish reasons** (why the model stopped — end-of-response, max-tokens, tool-call)

**Integration with observability backends.** Once you're emitting spans, you can send them to:

- **Braintrust** — Purpose-built for AI agent tracing. 13+ framework integrations. Traces stay with your data.
- **LangSmith** — Step-by-step visibility into agent execution. Good for debugging individual runs.
- **OpenLIT** — One-line instrumentation for OpenTelemetry. Works with any model and framework.
- **Datadog** — Full observability stack with native OTel GenAI convention support.
- **LiteLLM** — Cross-provider logging and cost tracking. Sits between your code and the model API.

For teams building custom AI pipelines, Braintrust and LangSmith are the industry standards. For teams using off-the-shelf tools (Cursor, Claude Code, Windsurf), look for built-in observability — Claude Code can export spans to external systems; Cursor integrates with Braintrust.

---

## Run Logs as Lightweight Observability

The agentic workflow guide produces `.log.md` files after each run — markdown documents capturing what happened during a session: which steps were executed, what succeeded, what failed, and why. These run logs serve as a lightweight observability mechanism for development sessions. They're stored in the repo alongside plan files, making them auditable and queryable. For teams not yet ready for full OpenTelemetry instrumentation, run logs provide the essential observability signals: task completion, test results, and decision chains.

---

## The Observability Stack for Coding Pipelines

Here's the complete stack for AI-assisted feature development:

**Per-step logging.** Every task in the build plan gets logged:
- What was planned (from the build plan spec)
- What was generated (code, tests, docs)
- What tests passed or failed (per task)
- What decision chain led here (why this approach vs. alternatives)

This logging happens automatically if you:
1. Use a structured build plan (task → acceptance criteria → implementation)
2. Run tests after each task and capture results
3. Commit after each task with a descriptive message
4. Use a tracing library (OTel) to capture the agent's reasoning steps

**Cost tracking.** Log token usage per step, broken down by:
- Model (opus, sonnet, haiku)
- Phase (research, plan, implement, review, audit)
- Task (which build plan task)

This lets you see which steps are expensive and spot anomalies (e.g., "why did research use 2x tokens last week?").

**Decision chain tracing.** When the agent makes an architectural choice, log:
- The alternatives considered
- Why each was rejected or chosen
- The model's reasoning (from the LLM response)
- Any constraints that ruled out options

This becomes crucial when debugging later: "The agent chose a monolithic service instead of microservices. Let me look at why..."

**Integration Contract verification.** For each task, verify:
- Did the Implementation Contract specify the right connections?
- Were those connections actually implemented?
- Are integration tests passing?

Log the result as part of the task span: `integration_contract_verified: true` or `integration_contract_verified: false` with a reason.

**Error and issue tracking.** Every issue found during code review or audit becomes a span event with:
- The issue type (style, logic error, security concern, integration problem)
- Severity (blocker, major, minor)
- Whether it was fixed
- Which model fixed it
- Tokens consumed in remediation

---

## Key Signals to Monitor

Once observability is in place, watch these signals:

**Token usage and cost.** The most immediate metric. Track:
- Cost per feature (should be trending down as the team improves)
- Cost per task (which tasks are token-intensive?)
- Cost per phase (research is typically the most expensive)
- Outliers (if a feature costs 3x the average, why?)

**Latency per task.** How long does each task take? Watch for:
- Regression (tasks that used to take 5m now take 15m)
- Bottlenecks (E2E testing slow? Suggests test suite growth)
- Opportunities (if a task consistently takes longer than expected, maybe the spec is unclear)

**Rate limiting and quota issues.** If you're using external APIs (Claude, OpenAI, etc.):
- Are you hitting rate limits?
- How close are you to quota caps?
- Can you shift work across hours of the day to smooth demand?

**Error rates and failure modes.** Track:
- How many tasks fail initially and need remediation?
- Which types of errors are most common (compilation, logic, integration)?
- Are error rates improving over time (suggesting better prompts and context)?

**Finish reasons.** Why does the model stop generating?
- **end_turn:** Normal, expected.
- **max_tokens:** The response was cut off. Suggests the task might need more detailed spec or the model's planning wasn't precise enough.
- **tool_use:** The model invoked a tool. Count how many tools per task — high numbers suggest the agent is uncertain or exploring.

Monitor finish reason distributions. If 30% of implementation tasks end with `max_tokens`, something's wrong with how you're specifying tasks.

---

## Tools Landscape (2026)

**Braintrust.** The most comprehensive AI agent tracing platform. Tracks prompts, model responses, tool calls, and outcomes. 13+ framework integrations (Anthropic, OpenAI, LiteLLM, LangChain, etc.). Offers experiment tracking (run the same task with different prompts or models and compare results). Traces stay with your organization — no data sharing with Anthropic or other model providers.

**LangSmith.** LangChain's observability platform. Step-by-step visibility into agent execution. Strong debugging UI (you can "rewind" to any step and see the exact state). Good for development and small-scale production. Less suitable for high-volume pipelines (pricing scales with traces).

**OpenLIT.** One-line instrumentation for any model and any framework. Emits OpenTelemetry traces. Lightweight, vendor-agnostic. Works great if you already have an observability backend (Datadog, New Relic, Jaeger).

**Datadog.** Full-stack observability platform with native OpenTelemetry GenAI convention support. Integrates logs, metrics, traces, and user session replays. High cost, but if you're already using Datadog for infrastructure, adding AI observability is straightforward.

**LiteLLM.** Proxy layer between your code and LLM APIs. Logs all requests and responses, tracks costs per model/provider, handles retries and rate limiting. Works with Claude, OpenAI, Bedrock, and 50+ other providers. Not a full observability platform, but excellent for cost tracking and cross-provider standardization.

For teams just starting, **OpenLIT + your chosen backend** (or **Braintrust** standalone) is the right balance of simplicity and power. For teams with existing Datadog investment, use **OpenLIT** + **Datadog**. For high-volume automation, **Braintrust** with experiment tracking.

---

## Connecting to Failure Recovery

Observability enables one of the most powerful patterns in AI-assisted development: **checkpoint-based resume.**

When you log every task's completion (git commit + span), you create a deterministic checkpoint. If a session fails (network error, timeout, token limit), the next session can:

1. Read the last completed task from git history or observability logs
2. Load the build plan and jump to the next uncompleted task
3. Continue from there, without losing work

This works because the previous session's output is externalized:
- Completed code is committed to git
- Completed tests are passing (and logged)
- Completed docs are in the repo

The new session doesn't need to reconstruct anything. It just picks up where the old one left off.

This is only possible with observability. Without it, you don't know where the last session actually completed, so you restart from the beginning.

---

## Privacy Considerations

Logging prompts and model responses is powerful for debugging but sensitive for data privacy. Most teams want observability without exposing the full conversation to external systems.

**Recommended approach:** Log metadata, not content.

```json
{
  "task": "Implement authentication",
  "model": "claude-opus-4-6",
  "input_tokens": 8200,
  "output_tokens": 3400,
  "cached_tokens": 2100,
  "cost_usd": 0.42,
  "test_results": "23 passing, 0 failing",
  "finish_reason": "end_turn",
  "latency_seconds": 45,
  "integration_contract_verified": true
}
```

This gives you all the signals (cost, performance, quality) without exposing the actual prompts or code samples the agent saw. Most observability platforms support this "redaction mode" — they capture metrics while discarding sensitive content.

For teams that need full visibility (auditing, compliance, detailed debugging), keep full traces but store them in a secure, role-gated system — not in cloud logging that's accessible to everyone.

---

## Anti-Patterns

**No observability at all.** The worst case. You build a feature, some tests fail, and you can't trace back to why. Did the research phase miss something? Was the spec unclear? Did the implementation diverge? Without logs, you're blind.

**Observing only errors.** Logging failures but not successes. This is like only keeping receipts when you have a problem at a store — you have no baseline for normal operation. You need comprehensive logging to spot patterns and trends.

**Turning off cost tracking.** "It's too detailed." Then you wonder why your feature cost 3x as much this month. Cost tracking is the easiest signal to instrument and the hardest to retrofit.

**Logging raw conversations.** Storing full prompts and responses in shared systems. This works for small teams, but at scale it's a privacy and security nightmare. Extract metadata instead.

**Not compacting logs.** Let logs grow indefinitely without summarization. Useful for detailed debugging of individual runs, but useless for spotting trends across 100 runs. Archive old logs and keep recent ones readily queryable.

**Conflating observability with micro-optimization.** "Our traces show this task takes 5 seconds, let's optimize it." Most of the time, the 5 seconds are fine. Observe broadly, act on patterns, not on individual latencies.

---

## Getting Started: Three Levels

### Level 1: Git-Based (Today, No Setup)

Start here. Costs nothing, requires only discipline.

1. **Commit after every task.** Make the message descriptive: what was done, what tests passed, any decisions.
2. **Maintain a build plan checklist.** Check off tasks as they complete.
3. **Keep a session notes file.** At the end of each session, write a paragraph or two: what worked, what didn't, key metrics.

Your git log becomes your observability system.

### Level 2: File-Based (This Week)

Add structure without external tools.

1. **Create a `docs/observability/` folder.** Store session logs there as markdown files.
2. **Use a template for session logs:**

```markdown
## Session: Feature Name, Date

**Duration:** XX minutes
**Models Used:** Claude-opus (research), Claude-sonnet (implementation)
**Total Tokens:** 12,400
**Estimated Cost:** $0.42

### Tasks Completed
- [x] Task 1: Title (tokens: 2,100, cost: $0.08)
- [x] Task 2: Title (tokens: 3,200, cost: $0.12)
- [ ] Task 3: Title (blocked on X)

### Issues Found & Fixed
- Issue: Returned 500 on invalid input (found in review, fixed in 5 min)
- Issue: Missing error logging (found in audit, fixed in 10 min)

### Integration Contract
- Files modified: auth.ts, handlers/login.ts
- New endpoints: POST /auth/login, POST /auth/logout
- Database changes: None

### Next Session
- Pick up at Task 3
- Run full E2E suite before shipping
```

3. **Create a metrics tracker.** A CSV or JSON file that logs cost, latency, and test results per feature.

This stays in the repo and becomes part of project history.

### Level 3: Full Tracing (Next Month)

Instrument with OpenTelemetry.

1. **Choose a backend.** Braintrust, LangSmith, or OpenLIT + Datadog.
2. **Emit spans.** Wrap each task, tool call, and decision in a span with metadata.
3. **Query and alert.** Set up dashboards for cost, latency, error rates.

This requires engineering effort but gives you the full picture at scale.

---

## Sources

### Primary Research

- Anthropic, "Observability for AI Agents" (2026)
- Braintrust, "Building Observable AI Agents" (2025)
- OpenTelemetry, "GenAI Semantic Conventions" (2025)
- LangSmith Documentation, "Observability and Tracing" (2025)
- OpenLIT Observability Platform, "Getting Started with OpenTelemetry" (2026)

### Tools and Frameworks

- Braintrust — Agent tracing and experiment tracking
- LangSmith — LangChain observability
- OpenLIT — One-line instrumentation for OpenTelemetry
- Datadog — Full-stack observability with GenAI support
- LiteLLM — Cross-provider logging and cost tracking
