# AI-Assisted Coding: Best Practices Guide
**Author:** Glenn Clayton, Fieldcrest Ventures
**Version:** 3.0 — March 2026
---
## Why This Guide Exists
In February 2026, Andrej Karpathy marked the one-year anniversary of the term "vibe coding" — and declared it passé. The approach that replaced it doesn't have a catchy name yet, but the pattern is clear: AI does the implementation, the human owns architecture, quality, and correctness. Addy Osmani calls it "agentic engineering." Boris Cherny's Claude Code team simply calls it "how we work."

The shift matters because the alternative — shipping AI-generated code without disciplined verification — is producing what the community has started calling the "slopacolypse": a flood of plausible-looking but subtly broken code across GitHub. Qodo's 2025 research found that AI-generated PRs are 18% larger, incidents per PR are up 24%, and change failure rates are up 30%. The irony is that AI-assisted development rewards strong engineering discipline *more* than traditional coding, not less.

This guide codifies a methodology that has been battle-tested through production projects like Pepper V2, independently converges on the same patterns that the best agentic coding practitioners have arrived at, and goes further with a structured development loop, environment setup guide, and linked deep dives for each major concept. It represents both research validation and real-world experience.

## How to Use This Guide
This is a **Map of Content** — a concise reference to the principles, processes, and environment setup that make AI-assisted coding reliable and repeatable. Each section links to a deeper dive where the concept is explored in full.

The guide is organized in three layers:
1. **Core Principles** — The philosophy. Why certain patterns work and others fail.
2. **The Agentic Workflow** — The step-by-step process for building features with AI.
3. **Environment Setup** — How to configure your repo, tools, and context so the AI can do its best work.

If you're already doing AI-assisted coding and want to level up, start with the [Core Principles](#core-principles) and see which ones you're missing. If you're onboarding to a collaborative project, read [The Agentic Workflow](#the-agentic-workflow) to understand the workflow. If you're setting up a new project, start with [Environment Setup](#environment-setup).

---
## Core Principles
These five principles underpin everything else in the guide. They're listed roughly in order of impact — the first few will improve your results immediately; the later ones compound over time.

### ◈ 1. Context Is Everything
*The quality of what goes in determines what comes out.*

The quality of what you feed the AI matters more than which model you use. A mid-tier model with excellent context (clear specification, relevant code samples, architectural constraints) will outperform a frontier model working from a vague prompt. This is the single most validated finding in the agentic coding literature. Invest your time in preparing context, not in chasing the latest model.

Before any code is written — by you or by AI — invest in a dedicated research phase that produces a written artifact. This "canonical document" forces structured thinking about implementation strategy, edge cases, integration points, and scope boundaries *before* the expensive part starts.

The canonical document typically includes: implementation approach and rationale, data model implications, integration points with existing code, edge cases and error handling, and explicit decisions about what NOT to build. This is the pattern the Thoughtworks SDD (Spec-Driven Development) paper found prevents 70% of rework. It's also what separates disciplined AI-assisted development from "vibe coding and hoping for the best."

Canonical documentation is not a one-time artifact. After each feature is complete, update the canonical documentation to reflect what was actually built. When the next feature needs to integrate with what you've already built, the research phase reads these docs to understand the current state of the codebase. This is how context quality compounds across a project. Without it, later features operate on stale information about the codebase, and the AI makes incorrect assumptions about code that has since changed.

> *See: [Context Engineering](Context-Engineering.md) for how to build and maintain high-quality context artifacts.*
> *See: [Canonical Documentation](Canonical-Documentation.md) for templates, examples, update practices, and index management.*

**Sources:** Martin Fowler, "Context Engineering for Coding Agents" (2025); Spotify Engineering, "Background Coding Agents: Context Engineering" (2025); Thoughtworks, "Spec-driven development" (2025).

### ◇ 2. Verify Independently, Verify Often
*Unverified AI output is unreliable AI output.*

Every task you give the AI should include both the work to do *and* a way to confirm it was done correctly. Tests to run, linting to check, a build to trigger, assertions to validate — any feedback loop that lets the agent self-correct before you review.

Boris Cherny (creator of Claude Code) calls this "probably the most important thing to get great results" — and his data backs it up: giving the agent a verification loop improves 2–3x the quality of final output. The reason is simple: without a feedback loop, the agent declares "done" based on its own assessment. With a feedback loop, it declares "done" based on *evidence*.

This applies beyond testing. During plan generation, give the agent access to the existing codebase so it can verify file references. During research, give it access to the actual API docs rather than relying on training data. During implementation, ensure it can run the code it writes. The principle is universal: the agent should never have to guess whether its output is correct when it could *check*.

When AI drafts a plan or writes code, use a **separate, fresh-context invocation** to validate it. The agent (or session) that created the work is naturally inclined to confirm its own output — the same confirmation bias that affects human code reviewers. A fresh context forces independent evaluation. The validator starts with clean context, reads the specification and the drafted work, and assesses whether they align — without the accumulated reasoning that led to the draft in the first place.

This applies at two levels: validating that the plan satisfies the spec (requirements fidelity) and validating that the plan integrates with the existing codebase (architectural coherence). These are different failure modes and benefit from separate passes.

For subjective criteria that are hard to test automatically — code style, readability, adherence to architectural patterns — the **LLM-as-Judge** pattern is especially effective: a second agent receives *both* the original spec and the output, then scores against stated quality criteria. Osmani and the Claude Code team both use this pattern, sometimes framing the reviewer as "a staff engineer reviewing a junior's PR." The key is giving the judge explicit evaluation criteria, not just "review this code."

Code review and code audit catch fundamentally different problems and should be separate steps. **Code review** asks: "Does each piece of code look correct in isolation?" — style, obvious bugs, anti-patterns, type safety, error handling. **Code audit** asks: "Does the code work correctly as a system?" — cross-file integration, async bugs, resource management, security, performance. Research shows that specialized review agents with focused prompts outperform a single "review everything" pass. The same principle applies to human-assisted review: give the AI one job per pass.

Finally, testing during implementation (unit tests, component tests) verifies that new code works correctly on its own. Testing after integration (full E2E suite) verifies that new code works correctly *within the full application* — catching regressions where new code breaks previously-working functionality. These are different test phases that catch different failure modes. Run both, and run the full E2E suite — not just tests for the new feature. The most common bugs that survive unit testing are integration bugs.

> *See: [Verification and Review](Verification-and-Review.md) for patterns, templates, and checklists across all development phases.*
> *See: [Testing Strategy](Testing-Strategy.md) for the three-tier testing pyramid in AI-assisted development.*

**Sources:** Boris Cherny, Claude Code workflow (2026); Addy Osmani, "Agentic Engineering" (2026).

### ◆ 3. Everything Connects or It's Dead Code
*Orphaned, disconnected code is the #1 failure mode in AI-assisted development.*

The AI writes something that works perfectly in isolation but isn't connected to anything — dead routes, unused components, orphaned database tables.

The antidote is an **Integration Contract**: a section in every feature plan that explicitly specifies how new code connects to existing code. File-level connections, API contracts, component interfaces, database relationships. Make it a mandatory section in your plans, and make its verification a mandatory part of every audit.

> *See: [Integration Contracts](Integration-Contracts.md) for the Integration Contract template, verification checklist, and common orphaned-code failure patterns.*

**Sources:** Anthropic, "2026 Agentic Coding Trends Report"; Microsoft, "Taxonomy of failure modes in AI agents."

### ▣ 4. Write It Down or Lose It
*If it's not durable and visible, it doesn't exist.*

Production-quality AI coding never relies on what the agent "remembers" from earlier in the session. Instead, externalize execution state to durable storage — files on disk, structured data in a database, or at minimum a markdown checklist in the repo.

The benefits compound: failure recovery (resume from the last checkpoint instead of restarting), context efficiency (the agent loads only the current task), and observability (you can see what happened and why). The universal pattern from Cursor, Claude Code, Devin, and Windsurf: externalize state as durable checkpoints. For long-running or autonomous work, this is non-negotiable.

After each task in a build plan is complete: git commit with a descriptive message, update your progress tracker, and move to the next task. If a session crashes, the next session can resume from the last completed checkpoint — not from the beginning. This matters even in human-supervised coding. AI sessions can hang, time out, or produce errors that require a fresh start. Frequent checkpoints mean you lose at most one task's worth of work, not an entire feature's worth.

If you can't see what the AI did and why, you can't debug failures or improve your process. At minimum, maintain: a log of what was planned vs. what was built, test results at each stage, and which model/prompt produced which output. In more sophisticated setups, use tracing (OpenTelemetry) to follow the full decision chain across multiple AI invocations. The systems that fail at scale always have insufficient observability. You need to see every tool call, every decision, and the full state at each step.

> *See: [Externalized State](Externalized-State.md) for patterns like dual-format build plans, checkpoint-per-task, and failure recovery.*
> *See: [Observability](Observability.md) for lightweight and production-grade observability setups.*

**Sources:** Cursor, Claude Code, Devin, and Windsurf architecture documentation.

### △ 5. You're the Architect, AI Is the Builder
*AI handles implementation. You handle the decisions that make implementation worth doing.*

Not every step in the development process needs a frontier model. The pattern that works:

- **Frontier / reasoning models** for architectural decisions, research, validation, and deep code audits — tasks where a missed subtlety cascades into expensive rework.
- **Mid-tier models** for code generation, build plan creation, code review, and remediation — tasks that follow established patterns from earlier reasoning steps.
- **No model at all** for pre-flight checks, smoke tests, and deterministic validation — binary pass/fail checks that don't benefit from LLM reasoning.

This typically yields 40–50% cost reduction compared to using the frontier model for everything, with quality impact only on tasks that genuinely don't need deeper reasoning.

**The counterargument is worth knowing:** Boris Cherny uses the frontier model (Opus) for *everything*, arguing that reduced steering and fewer corrections more than compensate for the higher cost. His reasoning: the bottleneck isn't token generation speed — it's human time spent correcting AI mistakes. A smarter model requires less correction. Both strategies are valid. Tiered routing optimizes for cost at scale and in automated pipelines. "Always frontier" optimizes for solo developer throughput and reduced context-switching between model behaviors. The right choice depends on whether a human is in the loop and how cost-sensitive the workflow is.

The default architecture for any AI-assisted operation should be: **deterministic pre-processing → LLM reasoning → deterministic post-processing.** Input validation, file path resolution, dependency checking, and environment verification don't benefit from LLM flexibility — and they actively suffer from LLM non-determinism. Do these with regular code. Let the LLM focus on genuinely ambiguous decisions: architectural choices, implementation strategy, code generation, and natural-language interpretation.

This three-phase pattern reduces the surface area for LLM unpredictability while preserving reasoning capability where it matters. A practical example: Boris Cherny's team runs a PostToolUse hook that auto-formats code after every AI edit. Claude's output is "usually" well-formatted, but the hook fixes the last 10% deterministically — eliminating a class of CI failures without relying on the LLM to get formatting perfect every time.

AI coding tools are force multipliers, not replacements for engineering judgment. Senior developers consistently outperform juniors when using the same AI tools — not because they write better prompts, but because they have decades of pattern recognition that inform what to ask for, when to push back on AI output, and how to spot subtle architectural issues. Boris Cherny's advice: "You still have to learn the craft. You still have to learn to code, learn languages, learn compilers, runtimes, how to build web apps, how to build programs, system design." Addy Osmani puts it differently: the developers thriving in 2026 have reconceptualized their role from *implementer* to *orchestrator* — spending 70% of their time on problem definition and verification, 30% on execution. The ratios are inverted from traditional development, but the total time decreased dramatically. The implication: invest in your engineering fundamentals, not just your prompting skills. The AI handles the implementation; you handle the judgment.

> *See: [Model Routing](Model-Routing.md) for a framework for assigning models to tasks, including when to use the "always frontier" strategy.*
> *See: [Deterministic Sandwich](Deterministic-Sandwich.md) for examples of pre-processing and post-processing gates, including tool hooks.*

**Sources:** Boris Cherny, Claude Code workflow (2026); Addy Osmani, "Agentic Engineering" (2026); Andrej Karpathy, "Vibe coding is passé" (2026).

---
## The Agentic Workflow
The agentic workflow is the canonical process for building features with AI assistance: **Diagnose → Plan → Implement**. It's the core orchestration loop that runs once per feature (or "epic" — a self-contained unit of functionality). For a full project, you run this loop multiple times, grouping features into milestones for review. For multi-phase projects, use the `/project` meta-orchestrator to manage multiple loops through research, planning, and implementation phases.

> *See: [Agentic Workflow Guide](agentic-workflow-guide.md) for the complete workflow reference, slash commands, role definitions, and failure recovery.*

### Phase 1: Diagnose
Start with read-only analysis. Review the specification, stress-test edge cases, and map integration points. This phase produces a structured diagnosis document that becomes the input to planning. The diagnosis answers: What are we building? Why? What might go wrong? What does it connect to?

### Phase 2: Plan
Take the diagnosis and produce a detailed plan: decomposed tasks with acceptance criteria, an Integration Contract specifying how new code connects to existing code, file targets, dependencies between tasks, and explicit acceptance criteria for the feature as a whole. This phase includes validation gates that use fresh context to catch planning errors before implementation begins.

### Phase 3: Implement
Execute the plan task by task. Each task includes code, tests, and a git checkpoint. Use verification loops (linting, testing, type-checking) after each task. When all tasks are complete, run a full integration audit and E2E test suite, remediate any issues, and merge. Finally, update canonical documentation to reflect what was actually built.

Within each phase, you'll find slash commands (like `/code`, `/verify`, `/visual-check`) that orchestrate subagents. The workflow guide describes each command's inputs, expected outputs, and when to use it.

---
## Environment Setup
How you configure your repo, tools, and context artifacts determines the ceiling for AI-assisted coding quality. This section covers the setup that makes the agentic workflow work well.

> *See: [Environment and Repository Setup](Environment-and-Repo-Setup.md) for detailed configuration guides.*

### Project Context Files
Modern AI coding tools support project-level instruction files that persist across sessions. These are your highest-leverage setup investment:
- **Cursor Rules** (`.cursor/rules/`) — Project-specific conventions, tech stack details, coding patterns, and anti-patterns. Loaded automatically for every AI interaction in Cursor.
- **CLAUDE.md** — Equivalent for Claude Code. Project context, architectural decisions, and coding standards.
- **AGENTS.md** — The emerging open standard. Repository-level instructions for any AI agent working on the codebase.

These files should include: your tech stack and key dependencies, architectural patterns you follow (and anti-patterns to avoid), naming conventions and file organization, testing requirements and patterns, and any project-specific constraints.

**Treat these files as living error logs, not static configuration.** Boris Cherny's team at Anthropic maintains their CLAUDE.md as a shared mistake journal: "Anytime we see Claude do something incorrectly we add it to the CLAUDE.md, so Claude knows not to do it next time." Team members tag each other's PRs to surface learnings. Over time, this builds project-specific institutional memory that meaningfully reduces repeat errors. The Anthropic team recommends keeping these files concise — under 500 lines — and using per-folder context files for topic-specific instructions rather than stuffing everything into the root file.

### Canonical Documentation Index
Maintain an index of canonical documents — one per major feature or functional area. Each document describes: what was built, how it works, what data it touches, and what interfaces it exposes. The index lives in the repo and updates after every feature completion.

When a new feature needs to integrate with existing functionality, the research phase (diagnose stage) reads the relevant canonical docs. This is how the AI gets accurate context about your codebase without needing to parse every file.

A simple structure works:
```
docs/
  canonical/
    _index.md              # Links to all canonical docs
    authentication.md      # How auth works, what it touches
    booking-system.md      # How bookings work end-to-end
    payments.md            # Stripe integration, data flow
    ...
```

> *See: [Canonical Documentation](Canonical-Documentation.md) for the canonical document template.*

### Plan Files in the Repo
Commit your plan files (the markdown version) to the repo. They serve as an audit trail and debugging artifact — when something goes wrong, you can trace back to what was planned vs. what was implemented.

A suggested structure:
```
docs/
  plans/
    001-authentication.md      # [ ] / [x] task checkboxes
    002-booking-system.md
    ...
    archive/                   # Completed plans
```

The checkbox format (`[ ]` / `[x]`) gives you at-a-glance progress tracking and a clear record of what was completed in what order.

> *See: [Externalized State](Externalized-State.md) for the dual-format build plan approach.*

### Integration Contract Template
Include an Integration Contract section in every feature plan. At minimum, the contract should specify:
- **Files modified:** Which existing files will be changed, and how.
- **Files created:** What new files are introduced and where they connect.
- **API contracts:** New or modified endpoints, request/response shapes.
- **Data model changes:** New tables, modified schemas, migration requirements.
- **Component interfaces:** How new UI components connect to existing navigation, layouts, and state.
- **Dependencies:** What existing code this feature depends on. What will depend on this feature.

> *See: [Integration Contracts](Integration-Contracts.md) for the full template and verification checklist.*

### Git Workflow for AI-Assisted Development
AI coding benefits from a more granular commit strategy than typical development:
- **One commit per task** during implementation — not one commit per feature. This is your checkpoint system.
- **Descriptive commit messages** that include what was implemented, what was tested, and any decisions made. The AI can generate these.
- **Feature branches per epic/feature** — keep the main branch clean and merge only after the full workflow completes.

This granularity costs nothing but gives you failure recovery, a clear audit trail, and the ability to bisect issues precisely.

### Parallel Agent Sessions
The single biggest throughput multiplier in AI-assisted development isn't a better prompt or a faster model — it's running multiple agent sessions concurrently on independent features.

Boris Cherny runs 5 Claude sessions in parallel in his terminal, each in its own git worktree, plus 5–10 sessions on claude.ai. His team calls this "the single biggest productivity unlock." Addy Osmani experiments with the same "orchestrator" pattern — assigning backend, frontend, and tests to separate agents simultaneously.

The guard rails that make this work:
- **One git worktree per session.** Each agent works on its own branch in its own working directory. No merge conflicts during implementation.
- **Independent features only.** Don't parallelize features that depend on each other — the Integration Contracts will conflict.
- **A notification system.** You need to know when each agent needs input. Terminal notifications, browser tabs, or a dashboard — whatever keeps you from blocking an idle agent.
- **Merge discipline.** When features complete, merge them one at a time through the full workflow. Don't batch-merge — that defeats the purpose of per-feature verification.

This isn't about multitasking. It's about eliminating idle time — while one agent runs tests, another implements, and a third waits for your review. Your job as orchestrator is to keep all sessions unblocked.

> *See: [Parallel Agent Workflows](Parallel-Agent-Workflows.md) for worktree setup, notification patterns, and merge strategies.*

### Tool Hooks and Automation
Use pre- and post-action hooks to handle deterministic cleanup automatically, rather than relying on the AI to get it right every time:
- **Post-edit formatting hooks** — Auto-run Prettier, ESLint, or your formatter after every AI edit. Catches the 10% of formatting issues the AI misses, preventing CI failures.
- **Pre-commit hooks** — Run linting, type-checking, and fast tests before any commit. The agent's checkpoint commits go through the same quality gate as human commits.
- **Custom slash commands** — Build reusable workflows for common operations. Cherny uses a `/commit-push-pr` command "dozens of times daily" that handles the full commit-push-create-PR flow in one keystroke.

These hooks are instances of the [Deterministic Sandwich](Deterministic-Sandwich.md) principle applied at the tool level. They reduce the surface area for AI inconsistency without adding cognitive overhead to your workflow.

> *See: [Environment and Repository Setup](Environment-and-Repo-Setup.md) for hook configuration examples.*

---
## Linked Deep Dives
The following pages expand on concepts referenced throughout this guide. Each can stand alone but links back here and to related pages.

| Page | Covers |
|------|--------|
| [Agentic Workflow Guide](agentic-workflow-guide.md) | The canonical workflow process: Diagnose → Plan → Implement. Slash commands, role definitions, and failure recovery. |
| [Context Engineering](Context-Engineering.md) | Building and maintaining high-quality context artifacts. The "context > model" principle in practice. |
| [Canonical Documentation](Canonical-Documentation.md) | Templates, examples, update practices, and index management for per-feature canonical docs. |
| [Verification and Review](Verification-and-Review.md) | Patterns, templates, and checklists for multi-pass validation, code review, code audit, and the LLM-as-Judge pattern. |
| [Model Routing](Model-Routing.md) | Framework for assigning frontier, mid-tier, and automated tiers to tasks. Includes when to use the "always frontier" strategy. |
| [Integration Contracts](Integration-Contracts.md) | The Integration Contract template, verification checklist, and common orphaned-code failure patterns. |
| [Externalized State](Externalized-State.md) | Dual-format build plans, checkpoint-per-task, and failure recovery patterns. |
| [Deterministic Sandwich](Deterministic-Sandwich.md) | The deterministic → LLM → deterministic pattern with examples of pre/post-processing gates and tool hooks. |
| [Testing Strategy](Testing-Strategy.md) | The three-tier testing pyramid: unit tests, component tests, E2E tests. When to run each. |
| [Parallel Agent Workflows](Parallel-Agent-Workflows.md) | Running multiple agent sessions concurrently. Worktree setup, notification patterns, and merge strategies. |
| [Environment and Repository Setup](Environment-and-Repo-Setup.md) | Detailed repo structure, hook configuration, tool automation, and canonical documentation index setup. |
| [Observability](Observability.md) | Lightweight and production-grade approaches to logging and tracing AI-assisted development. |

---
## Sources
This guide draws on original methodology developed by Glenn Clayton (Fieldcrest Ventures), battle-tested through production projects like Pepper V2, validated against 2025–2026 industry research, and stress-tested against workflows published by leading AI coding practitioners.

### Research & Industry Reports
- Thoughtworks, "Spec-driven development: Unpacking one of 2025's key new AI-assisted engineering practices"
- Martin Fowler, "Context Engineering for Coding Agents" (2025)
- Spotify Engineering, "Background Coding Agents: Context Engineering" (2025)
- Anthropic, "2026 Agentic Coding Trends Report"
- Anthropic, "Effective harnesses for long-running agents"
- Anthropic, "Effective context engineering for AI agents" (2025)
- Microsoft, "Taxonomy of failure modes in AI agents"
- Qodo, "State of AI Code Quality in 2025"
- Concentrix, "12 Failure Patterns of Agentic AI Systems"
- METR, "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity"
- Playwright Official Documentation, "Best Practices"
- Cursor, Claude Code, Devin, and Windsurf architecture documentation
- AGENTS.md open standard specification

### Practitioner Workflows (2026)
- Boris Cherny (Head of Claude Code, Anthropic) — Claude Code workflow threads, team tips, and Lenny's Newsletter interview
- Addy Osmani (Google) — "Agentic Engineering," "The 80% Problem in Agentic Coding," "How to Write a Good Spec for AI Agents," "My LLM Coding Workflow Going into 2026"
- Andrej Karpathy — "Vibe coding is passé" (February 2026, marking one year since coining the term)
