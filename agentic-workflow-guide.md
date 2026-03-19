# Agentic Workflow Guide

**Author:** Glenn Clayton, Fieldcrest Ventures
**Version:** 3.0 — March 2026
**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)

**Related:** [Context Engineering](Context-Engineering.md) · [Externalized State](Externalized-State.md) · [Integration Contracts](Integration-Contracts.md) · [Testing Strategy](Testing-Strategy.md) · [Verification and Review](Verification-and-Review.md) · [Parallel Agent Workflows](Parallel-Agent-Workflows.md)

---

This guide defines the exact process for tackling any feature or fix using AI coding agents. Once you understand this flow, you can use it for anything: new features, bug fixes, refactors, integrations.

---

## The Big Picture

Every task follows three human-driven stages, then one agent-driven loop:

```
Diagnose (you) → Plan (you) → Implement (agent orchestrates) → Done
```

**Diagnose** and **Plan** are manual steps where you run a slash command in a fresh session, review the output, and feed it forward. **Implement** is where you hand off to an orchestrator agent that runs the plan to completion using subagents.

Between stages, a **plan file** (`docs/plans/[name].md`) acts as the shared notebook. Each stage reads it, does its work, and updates it so the next stage knows where things stand.

For work that spans multiple phases — where each phase has its own diagnosis, plan, and implementation cycle — the **`/project` meta-orchestrator** sits above this loop and drives phases to completion automatically. See [Multi-Phase Projects](#multi-phase-projects) below.

---

## The Commands

These slash commands live in `.claude/commands/` and automate each stage:

### Primary Workflow Commands

| Command | What it does | When you use it |
|---|---|---|
| `/diagnose` | Reads files, maps what exists, identifies gaps. Produces structured output you can copy-paste directly into `/plan`. | Start of any new task |
| `/plan` | Takes a pasted diagnosis and produces a step-by-step plan file at `docs/plans/`. No additional commentary from you required. | After `/diagnose` |
| `/implement` | Orchestrator agent. Reads the plan, then loops through subagents: `/code` per piece → `/verify` → `/visual-check` → `/e2e-smoke` → `/capture-lesson`. Commits after each piece. Logs all events to a run log. | After you approve the plan |

### Subagent Commands (invoked by `/implement`, not by you directly)

Each subagent has a dedicated agent definition in `.claude/agents/` with scoped tool access and a `model: opus` directive. The orchestrator spawns them by referencing these definitions — it does not paste command file contents inline.

| Command | Agent definition | What it does | Used by |
|---|---|---|---|
| `/code` | `code-agent.md` | Implements one piece from the plan. Runs `/test-and-fix` internally. Updates the plan file. Commits. Bails out after 2 failed attempts on the same piece. | `/implement` orchestrator |
| `/verify` | `verify-agent.md` (read-only — no Write/Edit tools) | Reviews the full diff with adaptive diffing strategy. Runs `/review-diff` + over-engineering checks. Checks against plan acceptance criteria. | `/implement` orchestrator, after all pieces |
| `/visual-check` | `visual-check-agent.md` | Adaptive browser validation — navigates affected pages, takes accessibility tree snapshots and annotated screenshots, interacts per Done-when criteria, produces proof-of-work artifacts. | `/implement` orchestrator, after `/verify` |
| `/e2e-smoke` | `e2e-smoke-agent.md` (read-only — Read + Bash only) | Runs lightweight Playwright smoke tests against UI flows affected by the plan, including downstream flows impacted by backend changes. Catches integration boundary bugs that unit tests miss. Skips only when all plan pieces have `E2E scope: none`. | `/implement` orchestrator, after `/visual-check` |
| `/capture-lesson` | `capture-lesson-agent.md` | Records surprises to `docs/lessons.md`. Appends CLAUDE.md suggestions to `docs/claude-md-suggestions.md` for human review. | `/implement` orchestrator, after `/e2e-smoke` |

### Utility Commands (used within other commands or standalone)

| Command | What it does | Where it's used |
|---|---|---|
| `/test-and-fix` | Runs tests, fixes failures, runs typecheck | Inside `/code` after each piece |
| `/review-diff` | Reviews current diff for unused imports, console.logs, pattern violations | Inside `/verify` |
| `/pre-commit` | Full quality gate: lint + typecheck + test + diff review + dependency audit + secrets scan | Run manually after `/implement` completes, before final push |
| `/assess-file` | Analyzes a single file for complexity and decomposition opportunities | During `/diagnose` when scoping work, or standalone |

---

## Stage 1: Diagnose

**Goal:** Understand what exists, what's broken, and what's missing. No code changes.

**Open a fresh session and run:**

```
/diagnose
```

It will ask you what to investigate. You provide the area of focus and key files:

```
Investigate the Langfuse observability integration. Goal is to ensure proper full implementation of observability as specified in the docs: https://langfuse.com/docs
Key files:
- src/shared/ai/langfuse.ts (core module)
- src/shared/ai/client.ts (Anthropic client wrapper)
- src/agent/runtime/stream.ts (agent runtime)
- src/features/email/workers/invoke-drafting-agent.ts
- src/features/email/workers/invoke-verification-agent.ts
Compare against Langfuse docs for a standard Next.js + Vercel AI SDK setup.
```

**Context-management guardrails:** The diagnose command now reads files strategically rather than exhaustively. If you list more than 8 files, it prioritizes the most critical ones first. Files over 300 lines are read in relevant sections rather than in full. Neighboring files are read only when needed for understanding dependencies — not automatically.

**What you get back:** A structured diagnosis block designed to be copy-pasted directly into `/plan` with no additional commentary needed. The format:

```markdown
## Diagnosis: [Area Name]

### Task
[One-line statement of what needs to be fixed or built]

### Scope
[Key files and modules involved, listed explicitly]

### What's Implemented
- [Bullet list of what exists and works]

### What's Broken or Missing
1. [Numbered list, ordered by severity]
2. [Each item includes: what's wrong, why it matters, and affected files]

### Recommended Priority
1. [Ordered list of what to fix first and why]

### Plan Location
Write plan to: docs/plans/[suggested-name].md
```

**What you do next:** Read the diagnosis. If it looks right, copy the entire block. You'll paste it into `/plan` in the next step.

> **Tip:** This session is now done. Don't keep using it. Close it or start fresh.

---

## Stage 2: Plan

**Goal:** Turn the diagnosis into a concrete, ordered plan with specific files and acceptance criteria.

**Open a fresh session and run:**

```
/plan
```

It will ask what you're planning. Paste the diagnosis output:

```
[paste entire diagnosis block from Stage 1]
```

That's it. The diagnosis block contains everything the plan command needs: what to fix, which files are involved, the priority order, and where to write the plan. No additional commentary required.

**What you get:** A file at the location specified in the diagnosis (e.g., `docs/plans/langfuse-fix.md`) that looks like this:

```markdown
# Plan: Langfuse Observability Fix

## Goal
Complete and correct the Langfuse integration so all AI calls
produce visible, reliable traces in the Langfuse dashboard.

## Pieces (in order)

### Piece 1: Fix flush coverage
- Files: invoke-drafting-agent.ts, invoke-verification-agent.ts,
  + any other worker routes missing flushLangfuse()
- Change: Add flushLangfuse() call before every worker response
- Done when: Every background worker route calls flush before returning
- Parallel group: A

### Piece 2: Add error tracing
- Files: langfuse.ts, client.ts, stream.ts
- Change: Create a traceError() function that sends failed AI calls
  to Langfuse with error metadata
- Done when: A deliberately failed AI call shows up as an error trace
  in the Langfuse dashboard
- Parallel group: A

### Piece 3: Remove unused dependency
- Files: package.json
- Change: Remove langfuse-vercel if we confirm it's truly unused
- Done when: pnpm install succeeds, no import errors

### Piece 4: Add test coverage
- Files: new file __tests__/shared/ai/langfuse.test.ts
- Change: Mock Langfuse SDK, verify trace/flush functions behave correctly
- Done when: Tests pass, cover happy path + error path + missing env vars

## Progress
- [ ] Piece 1: Flush coverage
- [ ] Piece 2: Error tracing
- [ ] Piece 3: Unused dependency
- [ ] Piece 4: Test coverage

## Current State
Not started.
```

**Key plan features to look for:**

- **Characterization test enforcement:** If a piece modifies existing behavior (not just adding new code), the plan must include a characterization-test piece *before* it that captures the current behavior with tests. This ensures refactors don't silently break things.
- **Parallel groups:** Pieces that touch completely independent files with no ordering dependency get the same `Parallel group` letter (A, B, C…). The orchestrator runs them concurrently instead of sequentially.

**What you do next:** Read through the plan. Does it make sense? Would you change the order or scope? Edit it or ask the agent to revise. Once you're happy, move on to `/implement`.

> **Tip:** This session is now done. Fresh session for implementation.

---

## Stage 3: Implement (Agent-Orchestrated)

**Goal:** Hand the approved plan to an orchestrator agent that drives it to completion using subagents. You kick it off and let it run.

**Open a fresh session and run:**

```
/implement
```

Point it to the plan:

```
docs/plans/langfuse-fix.md
```

**What happens:** The orchestrator records the current `git rev-parse HEAD` as the base commit SHA (used later by `/verify` for accurate diffing), initializes a **run log** (a `.log.md` companion file next to the plan — e.g., `langfuse-fix.log.md`), then runs the following loop. The run log captures timestamped entries for every significant event (piece start/pass/fail, verify results, E2E smoke results, lesson capture) and serves as the operational record for the implementation run.

```
┌──────────────────────────────────────────────────────────────┐
│  /implement (orchestrator agent)                             │
│                                                              │
│  Step 0: Record base commit SHA in plan's Current State      │
│          Initialize run log (.log.md companion file)         │
│                                                              │
│  For each incomplete piece (or parallel group):              │
│    1. Spawn /code subagent(s) via .claude/agents/code-agent  │
│       → Sequential by default; concurrent for parallel groups│
│       → Implements the piece                                 │
│       → Runs /test-and-fix internally                        │
│       → Bails out after 2 failed attempts on same piece      │
│       → Updates plan file (marks piece complete)             │
│       → Commits: git add + git commit                        │
│       → Logs each event to the run log                       │
│                                                              │
│    ⏸  Pause after piece 5 for large plans (>5 pieces)       │
│       → Reports progress to user before continuing           │
│                                                              │
│  After all pieces are complete:                              │
│    2. Spawn /verify subagent via .claude/agents/verify-agent │
│       → Adaptive diffing (base SHA → HEAD, or fallbacks)     │
│       → Runs /review-diff + over-engineering checks          │
│       → Checks against plan acceptance criteria              │
│       → Reports issues or gives the all-clear                │
│                                                              │
│    3. Spawn /visual-check via visual-check-agent             │
│       → Adaptive browser validation against affected pages   │
│       → Screenshot artifacts as proof-of-work                │
│       → Fix loop if FAIL (max 2 cycles)                      │
│       → Skips if no direct UI-facing changes (SKIP/BLOCKED/PASS/FAIL)│
│                                                              │
│    4. Spawn /e2e-smoke via .claude/agents/e2e-smoke-agent    │
│       → Skips only when all pieces have E2E scope: none      │
│       → Builds app and runs Playwright against affected flows│
│       → Up to 2 fix-retest cycles on failure                 │
│       → Catches integration boundary bugs unit tests miss    │
│                                                              │
│    5. Spawn /capture-lesson via .claude/agents/capture-lesson│
│       → Records anything surprising to lessons.md            │
│       → Appends CLAUDE.md suggestions to review queue        │
│                                                              │
│  Done. (Run log saved to [plan].log.md)                      │
└──────────────────────────────────────────────────────────────┘
```

**The subagent sequence in detail:**

**`/code` subagent (runs once per piece):**
1. Reads the plan file and identifies the next incomplete piece
2. Reads the relevant source files listed in that piece
3. Makes the code changes
4. Runs `/test-and-fix` (tests + typecheck, fix any failures)
5. Updates the plan file: marks the piece complete, notes what was done
6. Commits with a descriptive message: `fix(langfuse): add missing flushLangfuse calls to worker routes`
7. **Bail-out rule:** If it fails the same piece twice (tests won't pass, dependency missing, Done-when not satisfied), it stops immediately, updates the plan's Current State with a blocker description, leaves the checkbox unchecked, and returns control to the orchestrator.

**`/verify` subagent (runs once, after all pieces):**
1. Reads the plan file to understand acceptance criteria
2. Reviews the full git diff using an adaptive strategy:
   - Prefers `git diff <base-SHA>...HEAD` if a base commit was recorded
   - Falls back to `git diff main...HEAD` on feature branches
   - Switches to file-by-file review for diffs over ~500 lines
   - Last resort: identifies implementation commits from `git log` and diffs across them
3. Runs `/review-diff` checks (unused imports, console.logs, pattern violations, file size)
4. **Over-engineering checks** (new): flags unnecessary abstractions, premature generalization, gold-plating, and scope creep — changes beyond what the plan required
5. Produces a verification report with sections for acceptance criteria, code quality, over-engineering findings, and a verdict
6. If issues are found, the orchestrator can spawn additional `/code` subagents to fix them (capped at 2 fix-verify cycles)

**`/visual-check` subagent (runs once, after verify — UI plans only):**
1. Reads the plan file and maps source routes and components to rendered URLs
2. Skips if no UI-facing changes detected (reports SKIP)
3. Starts dev server, authenticates, then for each affected page: navigates, takes accessibility tree snapshot, takes before/after screenshots, interacts per Done-when criteria, checks console and network errors
4. Screenshots saved to `visual-check-artifacts/`
5. Reports SKIP, BLOCKED, PASS (with artifact paths), or FAIL (with issue list)
6. On FAIL, the orchestrator runs up to 2 fix-recheck cycles (code-agent fix, then re-run visual-check)

> **What visual check catches vs. E2E smoke:** Visual check runs on the dev server (fast, no build step) and validates rendered pages interactively with screenshots as proof-of-work. It skips if no direct UI-facing changes are detected. E2E smoke runs on a production build and catches SSR/build errors, covering both direct UI changes and downstream backend changes that affect user-visible data. E2E smoke skips only when all plan pieces have `E2E scope: none`.

**`/e2e-smoke` subagent (runs once, after visual check):**
1. Reads the plan file to identify affected flows, including the plan's `E2E Verification Plan` section for pre-identified downstream flows
2. Skips only when all plan pieces have `E2E scope: none`, or the E2E Verification Plan explicitly says "No E2E testing warranted" with justification. Backend-only plans with `downstream` or `direct` scoped pieces proceed to E2E testing.
3. Checks prerequisites (`.env.e2e` or auth credentials, Playwright auth state, Playwright installed)
4. Writes a temporary Playwright spec, builds the app, and runs smoke tests against affected flows. For `downstream` pieces, navigates to the downstream page/flow identified in the E2E Verification Plan and verifies the expected user-visible outcome.
5. Cleans up the temporary spec file (never committed)
6. Reports pass/fail with screenshot paths for failures

> **What E2E smoke catches:** Integration boundary bugs — the kind where unit tests pass but the actual page breaks. A renamed API route, a missing provider wrapper, a build-time error in a server component, or a worker/cron that stops producing correct data visible on a dashboard page. These problems live at the seams between units, and E2E smoke catches them whether the change is in UI code directly or in backend code whose output flows downstream to a user-visible surface.

If smoke tests fail, the orchestrator can spawn `/code` to fix the issues, capped at 2 fix-retest cycles (matching the verify fix cap).

**`/capture-lesson` subagent (runs once, after E2E smoke and visual check):**
1. Reviews the plan file and implementation commits
2. Identifies anything surprising: unexpected failures, gotchas, patterns worth codifying
3. Adds entries to `docs/lessons.md`
4. Appends CLAUDE.md suggestions to `docs/claude-md-suggestions.md` for human review (does NOT edit CLAUDE.md directly)

**After `/implement` completes, you run manually:**

```
/pre-commit
```

This is your final quality gate before pushing: lint + typecheck + test + diff review + dependency audit + secrets scan.

**Orchestrator guardrails to be aware of:**

- **Lean prompts:** The orchestrator passes only the plan file path and piece-specific context to subagents. It does not paste large blocks of code into prompts — subagents read files themselves.
- **Pause at piece 5:** For plans with more than 5 pieces, the orchestrator pauses after piece 5 and reports progress to you before continuing. This prevents runaway execution on large plans.
- **Parallel groups:** If consecutive pieces share a `Parallel group` letter, the orchestrator spawns their `/code` subagents concurrently. After all parallel subagents complete, it re-reads the plan to confirm every piece in that group succeeded.
- **Fix-verify cap:** If `/verify` finds issues, the orchestrator can run up to 2 fix-verify cycles. If issues remain after that, it reports them to you instead of looping forever.

---

## Multi-Phase Projects

Some work is too large for a single diagnose-plan-implement cycle. A testing infrastructure overhaul, a major migration, or a new subsystem might need five or more phases, each with its own diagnosis, plan, and implementation. Running those phases by hand means 15+ manual handoff points.

The **`/project` meta-orchestrator** automates this. It reads a **project brief** (the outer-loop source of truth, analogous to a plan file for `/implement`) and drives each phase through the full inner loop: diagnose, plan, implement. You kick it off and it runs until it finishes or hits a human checkpoint.

### How it works

```
┌──────────────────────────────────────────────────────────────────┐
│  /project (meta-orchestrator — outer loop)                       │
│                                                                  │
│  Reads: Project brief (phases, goals, constraints)               │
│  Writes: Project log, phase tracking                             │
│                                                                  │
│  For each phase:                                                 │
│    ┌──────────────────────────────────────────────────────────┐  │
│    │  Inner Loop:                                              │  │
│    │                                                          │  │
│    │  1. /diagnose  → subagent, produces diagnosis block      │  │
│    │  2. /plan      → subagent, produces plan file            │  │
│    │  3. implement  → INLINE (see below)                      │  │
│    │     └── spawns code/verify/visual-check agents directly  │  │
│    │  4. /pre-commit → final quality gate                     │  │
│    └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Between phases:                                                 │
│    - Evaluate phase outcome against phase goals                  │
│    - Decide: proceed to next phase, adapt plan, or escalate      │
│    - Update project tracking                                     │
│    - Optional: human checkpoint gate                             │
│                                                                  │
│  After all phases:                                               │
│    - Cross-phase integration verification                        │
│    - Project completion report                                   │
│    - Archive all artifacts                                       │
└──────────────────────────────────────────────────────────────────┘
```

### When to use `/project`

Use `/project` when you have **predefined multi-phase work** — you already know the phases, their dependencies, and their diagnosis prompts. Examples:

- Building an E2E testing infrastructure across 5 phases
- A multi-step migration (schema, code, data, verification)
- Any project where you would otherwise run diagnose-plan-implement 3+ times manually

For single-phase tasks, stick with the standard `/diagnose` then `/plan` then `/implement` flow.

### The project brief

The project brief lives at `docs/projects/[name].md` and defines:

- **Goal** — what success looks like when all phases are complete
- **Phases** — each with a diagnosis prompt, dependencies, success criteria, and an optional human checkpoint
- **Phase Dependencies (DAG)** — which phases depend on which, enabling the orchestrator to determine execution order
- **Project Log** — timestamped entries tracking phase transitions and outcomes
- **Current State** — overall project status, which phase is active, any blockers

A template is available at `docs/plans/_template-project-brief.md`.

### How phases execute

For each phase in dependency order, `/project`:

1. **Diagnose** — Spawns a `diagnose-agent` subagent with the phase's diagnosis prompt. The agent returns structured diagnosis text. The orchestrator writes it to a file alongside the project brief.
2. **Plan** — Spawns a `plan-agent` subagent with the diagnosis file path. The agent writes a plan file to `docs/plans/`.
3. **Human checkpoint** — If the phase declares `Human checkpoint: yes`, the orchestrator pauses, presents the plan summary, and waits for your approval before proceeding.
4. **Implement (inline)** — The orchestrator runs implementation directly (not as a subagent), spawning `code-agent`, `verify-agent`, `visual-check-agent`, `e2e-smoke-agent`, and `capture-lesson-agent` as its own direct subagents. This is the same logic `/implement` uses, absorbed into `/project` to work within the single-level subagent constraint.
5. **Evaluate** — Reads the run log. If all pieces passed and verification succeeded, the phase is marked complete and its plan is archived. Otherwise the phase is marked blocked and the project stops.

### Human checkpoints

Each phase in the project brief can declare whether it requires human review. Architecture decisions, high-risk changes, and phase boundaries with significant scope changes are natural checkpoint points. The orchestrator respects these gates — it will not proceed past a checkpoint without your approval.

### Resuming a project

The project brief tracks the status of every phase. If `/project` stops (due to a blocked phase, a human checkpoint, or context exhaustion), you can resume by running `/project` again and pointing it to the same brief. It reads the phase statuses and picks up where it left off.

---

## Recovery: When Things Go Wrong

Every commit the orchestrator makes is a save point. If a piece introduces a problem, you don't have to manually undo anything or start from scratch. The `/rollback` command gives you two recovery modes, and the plan file tells the system exactly where to resume.

### Scenario 1: One bad piece

The latest piece broke something — tests fail, wrong approach, or it just doesn't satisfy the acceptance criteria. Earlier pieces are fine.

**What to do:**

```
/rollback
→ Choose: last piece
```

This runs `git revert HEAD`, creating a new commit that undoes the bad piece while preserving everything before it. Your git history stays clean and intact.

**Then:** Uncheck the reverted piece in the plan file, add a note in Current State about what went wrong, and re-run `/implement` in a fresh session. The orchestrator will pick up from the unchecked piece and try again with a clean slate.

### Scenario 2: Multiple pieces gone wrong

The implementation went off the rails across several pieces. Maybe an early architectural decision cascaded into problems, or the pieces built on each other in a way that compounded an issue.

**What to do:**

```
/rollback
→ Choose: full reset
```

This reads the base commit SHA from your plan's Current State (recorded when `/implement` first started), saves your current HEAD as an escape hatch, and hard-resets to the clean state before any pieces were implemented.

**Then:** Uncheck all rolled-back pieces, update Current State, and re-run `/implement`. If some of the discarded commits contained good work, you can cherry-pick them using the escape hatch SHA that `/rollback` displays.

### Scenario 3: The plan itself was wrong

Sometimes the problem isn't the implementation — it's the plan. You realize the pieces are in the wrong order, the scope was wrong, or the diagnosis missed something fundamental.

**What to do:**

```
/rollback
→ Choose: full reset
```

Then, instead of re-running `/implement`, go back to the beginning:

1. Run `/diagnose` with what you've learned
2. Run `/plan` with the updated diagnosis
3. Run `/implement` with the new plan

The old plan file stays in `docs/plans/` as a record of what didn't work.

### Why this works

Because every piece is a separate commit, your git history is a series of discrete save points. You can always back up to any one of them. The plan file tracks which pieces succeeded and which didn't, so the orchestrator never loses its place. And the base commit SHA gives you a guaranteed clean rollback point for the entire implementation run.

---

## Interactive Debugging

Sometimes you need to visually inspect a page without running a full plan. The `/interact` command opens a Playwright MCP browser session and lets you direct the agent interactively.

**Run in any session:**

```
/interact
```

The agent asks which page or URL to inspect, starts the dev server if needed, authenticates, and navigates there. It takes an accessibility tree snapshot and screenshot, checks console errors and network requests, and reports its findings.

From there, you drive: ask it to click elements, fill forms, navigate elsewhere, check a specific component's state. The agent interacts, reports back with screenshots, and waits for your next instruction.

**Key constraints:**
- **Read-only.** The agent observes and reports — it does not modify code. If you find a bug, use `/diagnose` or fix it in a coding session.
- **Screenshots** are saved to `visual-check-artifacts/` (same directory as automated visual checks).
- **No agent definition required.** This is a user-invoked utility command, not a subagent dispatched by the orchestrator.

---

## When to Use Subagents vs. Fresh Sessions

**Fresh sessions** (new terminal / new conversation):
- Each human-driven stage (Diagnose, Plan)
- Kicking off `/implement`
- Any task that needs the full context window
- When you want completely unbiased fresh eyes

**Subagents** (spawned within a session via Task tool):
- Everything inside the `/implement` loop (code, verify, capture-lesson)
- Quick research ("check the Langfuse docs for flush best practices")
- Specialized review of a small diff
- Anything that's a 2-minute errand, not a 20-minute task

**Rule of thumb:** If it takes more than 5 minutes of agent work, give it a fresh session. If it's a quick lookup or focused sub-task, use a subagent.

---

## The Plan File Is Your Source of Truth

The plan file (`docs/plans/[name].md`) is what makes the whole system work across stages. Here's why:

**Without a plan file:** You open a fresh session and have to re-explain everything. The agent has no idea what happened in previous sessions. You waste context re-establishing what's already been done.

**With a plan file:** You open a fresh session, point to the plan, and say "pick up where we left off." The agent reads 30 lines and knows exactly what's done, what's next, and what success looks like.

**Rules for plan files:**
- Keep them short (under 100 lines)
- Use checkboxes for progress tracking
- Include a "Current State" section that's updated after every piece
- Reference specific commits when marking pieces complete
- Store them in `docs/plans/` so they don't clutter the root
- Archive completed plans to `docs/plans/archive/` when the task is done

**Companion artifact — the run log:** When `/implement` runs, it creates a `.log.md` file next to the plan (e.g., `langfuse-fix.log.md`). This captures timestamped events for every piece start, pass, fail, bail-out, verify result, E2E smoke result, and lesson capture. On resumed runs, the log appends a separator with a progress summary. The run log is committed alongside plan updates and provides a detailed operational record of what happened and when.

---

## Putting It All Together: Your Daily Workflow

**Morning:** Pick a task. Open a fresh session. `/diagnose` to understand the problem.

**Scoping:** If you need to understand a specific file's complexity before committing to a plan, use `/assess-file` during or after diagnosis.

**Planning:** Open a fresh session. `/plan`, paste the diagnosis. Review and approve the plan.

**Working:** Open a fresh session. `/implement`, point to the plan. The orchestrator handles the rest: coding each piece, running tests, committing, verifying, and capturing lessons.

**Before you push:** Run `/pre-commit` as the final quality gate.

**Over time:** Your `CLAUDE.md` and `docs/lessons.md` accumulate knowledge. Your slash commands encode your workflow. Every session gets a little smarter because the repo remembers what previous sessions learned. That's the compounding effect.

---

## Quick Reference: Command Cheat Sheet

| I want to... | Run this | Stage |
|---|---|---|
| Understand a new problem | `/diagnose` | Manual |
| Analyze a file's complexity | `/assess-file` | Manual (during diagnose) |
| Turn diagnosis into a plan | `/plan` | Manual |
| Execute the plan end-to-end | `/implement` | Agent-orchestrated |
| Run a multi-phase project end-to-end | `/project` | Agent-orchestrated |
| Debug a page visually in the browser | `/interact` | Manual (standalone) |
| Undo a bad piece or full reset | `/rollback` | Manual (when things go wrong) |
| Final quality gate before push | `/pre-commit` | Manual (after implement) |
