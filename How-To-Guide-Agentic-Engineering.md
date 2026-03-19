# How to Build Software with AI Coding Agents
## A Practical Guide to Agentic Engineering

**Author:** Glenn Clayton, Fieldcrest Ventures
**Synthesized:** March 2026

---

## What This Guide Is

This is a practitioner's playbook for building production software using AI coding agents. It distills the core practices from a comprehensive body of research and real-world experience into actionable steps you can follow today.

The approach described here goes by several names — "agentic engineering" (Addy Osmani), "how we work" (Boris Cherny's Claude Code team at Anthropic), or simply the disciplined alternative to "vibe coding." The central idea is simple: **AI does the implementation; you own the architecture, quality, and correctness.** The methodology that follows makes that division of labor reliable and repeatable.

---

## The Five Principles That Underpin Everything

Before diving into process, internalize these five principles. They are the most validated findings in the agentic coding literature, and everything else in this guide flows from them.

### 1. Context quality determines output quality

A mid-tier model with excellent context (clear specs, relevant code samples, architectural constraints) will outperform a frontier model working from a vague prompt. This is the single most replicated finding across Anthropic, Spotify, Thoughtworks, and independent practitioner research. Invest your time in preparing what the AI sees, not in chasing the latest model.

### 2. Verify independently, verify often

Every task you give the AI should include both the work to do and a way to confirm it was done correctly. Boris Cherny calls this "probably the most important thing to get great results" — giving the agent a verification loop improves output quality 2-3x. The agent should never have to guess whether its output is correct when it could check.

### 3. Everything connects or it's dead code

Orphaned, disconnected code is the number one failure mode in AI-assisted development. The AI writes something that works perfectly in isolation but isn't wired into anything — dead routes, unused components, orphaned database tables. The antidote is an Integration Contract in every feature plan that explicitly specifies how new code connects to existing code.

### 4. Write it down or lose it

Never rely on what the agent "remembers" from earlier in the session. Externalize execution state to durable storage — plan files, git commits, structured checklists. The benefits compound: failure recovery (resume from the last checkpoint), context efficiency (the agent loads only the current task), and observability (you can see what happened and why).

### 5. You are the architect, the AI is the builder

AI handles implementation. You handle the decisions that make implementation worth doing. Senior developers consistently outperform juniors when using the same AI tools — not because they write better prompts, but because they have the engineering judgment to know what to ask for, when to push back, and how to spot subtle architectural issues. The AI handles the code; you handle the judgment.

---

## Step 1: Set Up Your Repository for AI-Assisted Development

Your development environment determines the ceiling for AI coding quality. A well-configured repo amplifies AI reasoning; a poorly configured one forces the AI to guess.

### Create the directory structure

Organize code by domain feature ("vertical slices"), not by technical layer. This minimizes the context an agent needs to load for any given task.

```
your-project/
├── CLAUDE.md                      # Project context for Claude Code (keep under 300 lines)
├── AGENTS.md                      # Portable context for any AI tool
├── .cursor/rules/                 # Cursor IDE rules by category
├── docs/
│   ├── canonical/                 # One doc per major feature describing what was built
│   │   └── _index.md             # Index linking to all canonical docs
│   ├── plans/                     # Plan files from the agentic workflow
│   │   └── archive/              # Completed plans moved here
│   ├── architecture/             # Architecture Decision Records
│   └── lessons.md                # Accumulated learnings across sessions
├── src/
│   ├── features/                  # Vertical slices, one directory per domain
│   │   └── [feature]/
│   │       ├── CLAUDE.md         # Feature-level context (under 200 lines)
│   │       ├── INTEGRATION.md    # Cross-feature dependency map
│   │       ├── index.ts          # Public API — all cross-feature imports go through here
│   │       ├── types.ts
│   │       ├── db/
│   │       └── components/
│   └── shared/                    # Only code genuinely used across all features
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

### Write your project context file

Create a CLAUDE.md (or AGENTS.md) at your repo root. This is the single highest-leverage setup investment. It should contain your tech stack and key dependencies, architectural patterns you follow (and anti-patterns to avoid), naming conventions, testing requirements, and a living error log of mistakes the AI has made in past sessions.

Keep it under 300 lines. Use per-folder context files for domain-specific details. Treat it as a living error log, not static configuration — every time the AI makes a mistake, add a two-line example showing the correct pattern. Over 10-20 features, this builds substantial institutional memory.

### Configure deterministic gates

Set up git hooks that run automatically, catching issues the AI might miss without relying on the AI to be perfect.

**Pre-commit hook:** Run linting (with auto-fix), type-checking, and fast unit tests before every commit. If any check fails, the commit is blocked and the agent fixes the issue automatically.

**Post-edit formatting hook:** Auto-run your formatter (Prettier, Black, etc.) after every AI edit. This catches the last 10% of formatting inconsistencies deterministically, eliminating an entire class of CI failures.

**Pre-commit architectural rules:** Enforce constraints like "client code cannot import from server modules" with simple grep-based checks in the hook.

This is the **Deterministic Sandwich** pattern: deterministic pre-processing, then LLM reasoning, then deterministic post-processing. Input validation, file path resolution, code formatting, and type-checking don't benefit from LLM flexibility — they actively suffer from LLM non-determinism. Do these with regular code. Let the LLM focus on genuinely ambiguous decisions.

---

## Step 2: The Core Workflow — Diagnose, Plan, Implement

Every task follows three stages. The first two are human-driven; the third is agent-orchestrated.

```
Diagnose (you) → Plan (you) → Implement (agent orchestrates) → Done
```

Between stages, a **plan file** acts as the shared notebook. Each stage reads it, does its work, and updates it so the next stage knows where things stand. This is what makes the system work across sessions — the agent reads 30 lines and knows exactly what's done, what's next, and what success looks like.

### Stage 1: Diagnose — Understand the problem

Open a fresh session. Investigate what exists, what's broken, and what's missing. No code changes.

Provide the agent with the area of focus and key files. It reads strategically (not exhaustively), prioritizing the most critical files, reading large files in relevant sections, and exploring neighboring files only when needed for understanding dependencies.

The output is a structured diagnosis: what needs to be fixed or built, key files involved, what's implemented, what's broken or missing (ordered by severity), and a recommended priority order. This block is designed to be copy-pasted directly into the planning stage.

**Then close this session.** Don't keep using it. Fresh context for each stage.

### Stage 2: Plan — Turn diagnosis into a concrete plan

Open a fresh session. Paste the diagnosis output. The agent produces a plan file at `docs/plans/[name].md` containing a goal statement, ordered pieces with specific files and acceptance criteria, an Integration Contract, and progress checkboxes.

Key features to look for in good plans: characterization tests before modifying existing behavior (ensures refactors don't silently break things), parallel groups for pieces that touch independent files (the orchestrator runs them concurrently), and explicit "Done when" criteria for every piece.

Read through the plan. Edit it or ask for revisions. Once you're satisfied, move on.

### Stage 3: Implement — Hand off to the orchestrator

Open a fresh session. Point the agent to the plan file. The orchestrator then runs this loop automatically:

**For each piece in the plan:**
1. A code subagent implements the piece, runs tests and type-checking internally, updates the plan file, and commits with a descriptive message
2. If a piece fails twice (tests won't pass, dependency missing), it stops, logs the blocker, and moves on

**After all pieces complete:**
3. A verify subagent reviews the full diff against acceptance criteria, checks for over-engineering and scope creep
4. A visual-check subagent validates rendered UI pages with screenshots (if applicable)
5. An E2E smoke subagent runs Playwright tests against affected flows
6. A capture-lesson subagent records surprises to your lessons file

After the orchestrator finishes, run a final pre-commit quality gate manually: lint, type-check, full test suite, diff review, dependency audit, and secrets scan.

---

## Step 3: Write Integration Contracts That Prevent Dead Code

Every feature plan must include an Integration Contract section. This is the executable specification for how new code connects to existing code. Without it, the AI will write code that works in isolation but is wired to nothing.

The contract should specify:

**Files modified** — which existing files change, and how (e.g., "app/api/routes.ts — registers POST /bookings endpoint").

**Files created** — each new file, what it depends on, and what uses it. If a new file isn't used by anything, it's orphaned by definition.

**API contracts** — new or modified endpoints with request/response shapes and error codes.

**Data model changes** — new tables, modified schemas, migration requirements, indexes.

**Component interfaces** — how new UI components connect to navigation, layouts, and state management.

**Dependencies** — what existing code this depends on (upstream) and what will depend on this (downstream).

During the audit phase, verify every point in the contract: does each file exist? Is each route registered? Is each component rendered somewhere? Is each table actually queried? If a connection listed in the contract doesn't exist in the code, it's a blocker.

---

## Step 4: Engineer Your Context Deliberately

Context engineering is the discipline of designing what information the AI sees, when it sees it, and how it's structured. It's the practical application of Principle 1.

### The four operations

**Write context (offload).** Save important state outside the context window — plan files, canonical docs, session notes, git commits. The agent can retrieve it later without keeping everything in memory. Manus AI reports 100:1 compression ratios in production agents using this approach.

**Select context (retrieve).** Load only what's needed for the current step, not everything that might be relevant. Use just-in-time file reads instead of pre-loading the entire codebase. Scope context files to specific file types or directories so the agent sees task-specific instructions only when relevant.

**Compress context (reduce).** As context accumulates, summarize older turns and replace raw history with concise summaries. Don't wait until the context is full — compact at regular intervals (after research completes, after a plan is validated, after a feature is implemented). Keep context utilization in the 20-40% "smart zone" where the model performs best.

**Isolate context (separate).** Use fresh sessions for each development phase so accumulated noise from one phase doesn't degrade reasoning in the next. Sub-agents are tools for controlling context, not organizational metaphors — each should get minimal, focused context and return a concise result.

### The smart zone

Below 20% utilization, the model lacks sufficient context. Between 20-40% is the sweet spot. Between 40-60%, performance starts to degrade. Above 60%, diminishing returns accelerate severely. Design your workflow so you compact or reset context before hitting 60%.

### Practical starting points

Clear and restart when your session gets long. Ask the AI to summarize current state and next steps, then start fresh with that summary. Create canonical docs for completed features so future sessions don't have to re-explore. Use sub-agents for research so the main agent's context stays clean. Scope your context per task — load only the files relevant to the current piece, not everything that might be related.

---

## Step 5: Maintain Canonical Documentation

Canonical documentation is a per-feature document describing what was built, how it works, and what decisions shaped it. It lives in `docs/canonical/` and serves as the primary context artifact that prevents the AI from reinventing the wheel or making incorrect assumptions about code that has changed.

### When to create them

During the research phase (Stage 1), before any code is written. Writing a canonical doc during research forces you to think through edge cases, integration points, and scope boundaries before the expensive part starts. Thoughtworks research found this prevents 70% of rework.

### When to update them

After every implementation that changes the feature materially. At minimum, update when: a bug is fixed that wasn't documented, an integration point changes, the data model evolves, a performance characteristic changes significantly, or a prior decision is revisited.

### What to include

An overview (what does it do, who uses it, why does it exist), the implementation approach and rationale, the data model it touches, integration points (imports, exports, APIs, events), edge cases and error handling, explicit decisions about what was not built and why, testing strategy, and known issues or workarounds.

### The index pattern

Maintain a single `docs/canonical/_index.md` that links to all canonical docs. This is the discovery mechanism — the agent reads the index during the research phase to find what documentation exists. Update the index when you add or retire a canonical doc.

The compounding effect is what makes this practice transformative: each completed feature produces an artifact that makes the next feature's context richer and more focused.

---

## Step 6: Externalize State with Checkpoints

Externalized state is the practice of persisting execution progress outside the agent's working memory in durable storage that survives session resets, crashes, and context exhaustion.

### The checkpoint-per-task pattern

After every discrete task completes, two things happen: a git commit with a descriptive message (what was built, what was tested, any decisions made), and an update to the plan file checkboxes. This is your checkpoint system.

If a session crashes at the 1-hour mark of a 2-hour task, you lose at most the duration of the current task — not the entire feature. The next session reads the plan, finds the last checked-off task, and resumes.

### The dual-format build plan

Maintain both human-readable markdown (checkboxes for progress) and structured data (JSON with task IDs, status, file targets, acceptance criteria). The markdown is for you; the JSON is for agents. Together they give you readability, programmability, and debuggability.

### Session notes as agentic memory

Keep a running NOTES.md outside the context window with goals, decisions made, key findings, and next steps. At the start of each session, the agent reads this to understand what was decided (no re-research needed), what blockers exist, and what comes next. This pattern — demonstrated dramatically by Anthropic's Claude playing Pokemon across thousands of game steps — works because it's selective, additive, readable, and survives context resets.

---

## Step 7: Verify at Every Layer

Verification is not a gate at the end — it's a continuous pipeline that runs throughout the development lifecycle.

### Self-verification during implementation

After every task, the agent runs: type-checking, linting, unit tests, and a build check. It does not declare a task complete until all gates pass. If a gate fails, it fixes and re-runs, iterating up to 5 times before surfacing for human review. This generate-verify-fix-reverify loop is what produces the 2-3x quality improvement Boris Cherny reports.

### Fresh-context validation of plans

After drafting a plan, run validation in a separate session. The validator loads only the spec and the plan — not the drafting session's full conversation history. This prevents confirmation bias and produces 23-31% better validation accuracy (Anthropic research). Check two things separately: does the plan satisfy the spec (requirements fidelity), and does the plan integrate with the existing codebase (architectural coherence).

### Separate code review from deep audit

**Code review** (mid-tier model): surface-level correctness — style, obvious bugs, anti-patterns, type safety, error handling. Give the reviewer explicit criteria, not just "review this code."

**Deep audit** (frontier model): system-level correctness — cross-file integration, async bugs, resource management, security, performance, and mandatory Integration Contract verification. This is where a more expensive model provides genuine value.

**Over-engineering detection**: check whether the implementation exceeds the plan — unnecessary abstractions, premature generalization, gold-plating, or scope creep beyond what was requested.

Use specialized agents with narrowly scoped prompts rather than a single "review everything" pass. Research shows specialized review agents catch 23% more issues.

### The LLM-as-Judge pattern

For subjective criteria, use a second LLM to evaluate output against explicit criteria. Frame the judge as "a staff engineer reviewing a junior's PR." Provide both the spec and the output. Ask for scores on specific dimensions (completeness, clarity, integration, scope, architecture). Use multi-pass validation — separate passes for different concerns — rather than one pass trying to check everything.

---

## Step 8: Test at Three Tiers

AI-generated code looks right — correct syntax, proper style, plausible logic. Subtle bugs are invisible without automation. Testing is the mechanism that separates shipped code from code-that-happened-to-work-in-isolation.

### Tier 1: Unit tests (during implementation)

Write alongside each task. Test individual functions in isolation — happy paths, edge cases, error conditions. Run immediately. Commit only when tests pass. These are your checkpoint gate.

### Tier 2: Component/integration tests (after unit tests pass)

Test interactions between modules. Do data contracts hold? Does module A correctly call module B? Are async coordination issues handled? These catch bugs that unit tests miss because they exercise real interactions between pieces.

### Tier 3: End-to-end tests (after all implementation)

Run the **entire** test suite, not just tests for the new feature. You're looking for regressions — new code breaking previously-working functionality. These catch "works alone but breaks together" bugs, the kind where unit tests pass but the actual page or flow breaks.

The critical rule: "works in isolation" does not mean "works in context." A function can pass all unit tests and still break when called by other code with unexpected timing, state, or data.

---

## Step 9: Choose the Right Model for Each Task

Not every step needs a frontier model. The pattern that works:

**Frontier models** (Claude Opus) for architectural decisions, research, validation, deep code audits — tasks where a missed subtlety cascades into expensive rework. Use for diagnosis, planning, plan validation, and integration audits.

**Mid-tier models** (Claude Sonnet) for code generation, build plan creation, code review, remediation — tasks that follow established patterns from earlier reasoning steps. Use for implementation, surface-level review, and fixing known issues.

**No model at all** for pre-flight checks, smoke tests, linting, formatting — binary pass/fail checks that don't benefit from LLM reasoning. Automate these with scripts.

This tiered routing typically yields 40-50% cost reduction compared to using frontier for everything, with quality impact only on tasks that genuinely don't need deeper reasoning.

**The counterargument is valid:** Boris Cherny uses frontier for everything, arguing that reduced human correction time more than compensates for higher cost. If you're a solo senior developer working interactively, "always frontier" may optimize your throughput. If you're running automated pipelines or are cost-sensitive, tiered routing wins decisively.

---

## Step 10: Run Parallel Agent Sessions

The single biggest throughput multiplier isn't a better prompt or faster model — it's running multiple agent sessions concurrently on independent features.

### How it works

Each agent gets its own git worktree (an independent working directory sharing the same .git metadata). Each works on its own branch. No merge conflicts during implementation.

```bash
git worktree add ../project-auth feature/auth
git worktree add ../project-payments feature/payments
git worktree add ../project-tests feature/test-coverage
```

### The guard rails

**Independent features only.** Don't parallelize features that depend on each other. If both features modify the same database schema, endpoints, or components, they're dependent.

**One worktree per session.** Strict 1:1 mapping. Never let two agents commit to the same branch.

**Notification system.** Know when each agent needs input. Terminal notifications, browser tabs, or a dashboard — whatever keeps you from blocking an idle agent.

**Merge one at a time.** When features complete, merge them individually through the full verification pipeline. Never batch-merge. One bad merge in a batch means you can't isolate which agent caused the issue.

### The orchestrator mindset

You are not multitasking. You are orchestrating a team. Assign work clearly, provide context, keep agents unblocked, review and merge carefully, and update canonical documentation after each merge so the next agent's research has current information.

Realistic throughput improvement is 3-4x, not 5x, because agents need your input, tests fail, and merges create synchronization points. But on a 6-month project, that can compress 4-5 months into 6-8 weeks.

---

## Step 11: Build Observability Into Your Workflow

If you can't see what the AI did and why, you can't debug failures or improve your process.

### Level 1: Git-based (start here, costs nothing)

Commit after every task with descriptive messages including what was built, what was tested, and key decisions. Maintain a build plan checklist with checkboxes. Keep session notes. Your git log becomes your observability system.

### Level 2: File-based (add structure without external tools)

Create a `docs/observability/` folder with session logs using a template: duration, models used, total tokens, estimated cost, tasks completed, issues found and fixed, integration contract status, and next steps.

### Level 3: Full tracing (for teams and automation at scale)

Instrument with OpenTelemetry. Each step becomes a span with attributes, events, metrics, and links. Send traces to a backend like Braintrust, LangSmith, or Datadog. Set up dashboards for cost, latency, and error rates.

### Key signals to watch

Token usage and cost per feature (should trend down as you improve). Latency per task (watch for regressions). Error rates by type (are they improving?). Finish reasons (if 30% of tasks end with max_tokens, your task specs need work).

---

## Step 12: Handle Recovery When Things Go Wrong

Every commit the orchestrator makes is a save point. You never need to start from scratch.

### One bad piece

The latest piece broke something. Run `git revert HEAD` to undo it while preserving everything before. Uncheck the reverted piece in the plan, note what went wrong, and re-run implementation in a fresh session. The orchestrator picks up from the unchecked piece.

### Multiple pieces gone wrong

The implementation went off the rails. Reset to the base commit SHA recorded when implementation started (the plan's Current State tracks this). Uncheck all rolled-back pieces, update the plan, and re-run. If some discarded commits had good work, cherry-pick them.

### The plan itself was wrong

Go back to the beginning. Run diagnosis with what you've learned, create a new plan, and implement from scratch. The old plan stays in `docs/plans/` as a record of what didn't work.

---

## Step 13: Build the Compounding Flywheel

The real power of this methodology is that it compounds. Each session makes the next one better.

**After each feature:** Update canonical documentation to reflect what was actually built. Update CLAUDE.md with any new patterns or mistakes discovered. Archive the completed plan. Update the canonical documentation index.

**After each mistake:** Add it to the living error log in CLAUDE.md with a two-line example showing the correct pattern. Tag it with the PR where it was discovered. Over 10-20 features, this builds institutional memory that meaningfully reduces repeat errors.

**After each session:** Review what went well and what didn't. If a verification step consistently catches the same class of issue, add a rule to your context files. If a pattern keeps emerging, codify it.

The result: your repo remembers what previous sessions learned. Your context files encode accumulated wisdom. Your canonical docs provide accurate information about what exists. Your plan files show what was tried and what worked. Every session gets a little smarter because the system has memory that persists beyond any single conversation.

---

## Quick Reference: Your Daily Workflow

**Morning:** Pick a task. Open a fresh session. Diagnose to understand the problem.

**Scoping:** If you need to assess a file's complexity before committing to a plan, do it during diagnosis.

**Planning:** Fresh session. Paste the diagnosis. Review and approve the plan.

**Working:** Fresh session. Point to the plan. The orchestrator handles coding, testing, committing, verifying, and capturing lessons.

**Before you push:** Run the pre-commit quality gate.

**After you merge:** Update canonical docs. Move the plan to archive. Update the index.

**Over time:** Context files accumulate knowledge. Canonical docs compound. Every session starts smarter than the last. That's the flywheel.

---

## Sources

This guide synthesizes methodology developed by Glenn Clayton (Fieldcrest Ventures), validated against 2025-2026 industry research from Anthropic, Thoughtworks, Spotify, Google, Microsoft, Manus AI, HumanLayer, and Qodo, and stress-tested against workflows published by Boris Cherny (Head of Claude Code, Anthropic), Addy Osmani (Google), and Andrej Karpathy. Full source citations are available in the individual deep-dive documents linked from the companion AI-Coding-Best-Practices guide.
