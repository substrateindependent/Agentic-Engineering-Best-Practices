# Development Loop — Full Reference

**The Complete 15-Step Pipeline with Model Assignments, Failure Recovery, and State Machine Formalization**

**Author:** Glenn Clayton, Fieldcrest Ventures
**Version:** 3.0 — February 2026
**Derived from:** AI-Coding-Best-Practices.md (Development Loop summary) + NEW Development Process Guide.md (per-epic pipeline)
**Purpose:** The authoritative deep reference for every step in the AI-assisted development pipeline, with model routing, inputs, outputs, failure recovery, and core principle connections.

---

## Overview and How to Use This Reference

This document is the **most detailed reference in the vault** for the development loop. It expands the 15-step pipeline (collapsed into 6 phases in the best-practices guide) into full architectural documentation.

### When to Use This Document

- **During setup:** Choose model routing and checkpoint strategy for your pipeline.
- **During execution:** Understand what each step requires and what can go wrong.
- **During failure:** Jump to the failure recovery section for a step, or use the state machine to understand where to resume.
- **During optimization:** Use the cost profile to identify bottlenecks and expensive steps.

### Structure of Each Step Entry

For every step (1–15), you'll find:

1. **Step name and number** — E.g., "Step 5: Generate Build Plan"
2. **What happens** — Detailed description of the work performed
3. **Model tier assignment** — Frontier, Mid-tier, or Automated; with reasoning
4. **Key inputs** — What the step requires to run
5. **Key outputs** — What the step produces for downstream steps
6. **Failure recovery** — What to do if this step fails
7. **Connection to core principles** — Which principles from the best-practices guide apply

### Core Concepts Used Throughout

- **Frontier model:** Reasoning-intensive work (architecture, validation, audit). Currently Claude Opus 4.6 with adaptive thinking. Cost: highest per token, but fewest tokens needed.
- **Mid-tier model:** Pattern-following execution (code generation, review, remediation). Cost: 40–50% lower than frontier.
- **Automated (no LLM):** Deterministic checks (file existence, dependency verification, binary pass/fail). Cost: near zero.
- **Fresh context:** A separate invocation starting with clean session state, preventing confirmation bias.
- **Checkpoint:** A git commit marking the completion of a discrete unit of work. Always atomic and restorable.

---

## State Machine Diagram: The Full Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PHASE 1: PLAN (Steps 1-6)                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STEP 1 (Frontier)           STEP 2 (Frontier)        STEP 3 (Frontier) │
│  Research the feature        Draft feature plan       Validate           │
│  (canonical document)        + Integration Contract   requirements       │
│           │                          │                (fresh context)    │
│           └──────────────────────────┴─────────────────┬────────────────┘
│                                                        │
│                                                        │ Pass? ───┐
│                                                        │          │ Fail
│                                                        ▼          │
│  STEP 4 (Frontier, fresh context)   STEP 6 (Frontier)│          │
│  Validate integration                Double-check     │          │
│  + architecture                      build plan       │          │
│           │                          (fresh context)  │          │
│           └──────────────────────┬───────────┬────────┘          │
│                                  │ Pass?     │                   │
│                                  │    ┌──────┼──────┐            │
│                                  │    │  Pass │ Fail │ ◄─────────┘
│                                  ▼    ▼      └─────►REVISE (step 2)
│                                                                          │
│                        STEP 5 (Mid-tier)                                │
│                   Generate build plan (dual-format)                     │
│                                                                          │
└──────────────────────────────────────┬─────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼─────────────────────────────────┐
│                      PHASE 2: BUILD (Steps 7-11)                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STEP 7 (Automated)       STEP 8 (Mid-tier)                             │
│  Pre-flight check         Implement + commit                            │
│  - File existence         (per-task unit of work)                       │
│  - Dependencies           - Unit tests                                  │
│  - Schema validation      - Component tests                             │
│           │               - Git commit per task                         │
│           │                       │                                     │
│        Pass?                       ▼                                     │
│        │                                                                │
│      Fail ──► HALT       STEP 9 (Mid-tier)       STEP 10 (Frontier)    │
│        │                 Code review             Deep audit             │
│      Continue             - Style, bugs          (fresh context)        │
│        │                 - Type safety           - Integration          │
│        ▼                 - Error handling        - Cross-file           │
│                                │                - Security             │
│                                ▼                - Performance          │
│                       ┌─────────────────┐               │               │
│                       │ Issues found?   │               │               │
│                       └────────┬────────┘               │               │
│                                │                       │               │
│                        Yes ◄────┴──────► No            │               │
│                        │                │              │               │
│                        ▼                │              ▼               │
│                   STEP 11 (Mid-tier)    │        Proceed             │
│                   Remediation           │                             │
│                   (fix + re-test)       │                             │
│                        │                │                             │
│                        └────────┬───────┘                             │
│                                 ▼                                      │
│                        Continue to Phase 3                             │
│                                                                          │
└──────────────────────────────────────┬─────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼─────────────────────────────────┐
│                   PHASE 3: VERIFY & SHIP (Steps 12-15)                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STEP 12 (Mid-tier)        STEP 13 (Mid-tier)     STEP 14 (Automated   │
│  Full E2E test suite       Final remediation      + Mid-tier)           │
│  (entire test suite)       (fix E2E failures)     Commit + update docs  │
│           │                       │                       │             │
│           ▼                       ▼                       ▼             │
│  ┌────────────────┐      ┌────────────────┐    All tests pass?         │
│  │ Tests passing? │      │ Tests passing? │                            │
│  └────────┬───────┘      └────────┬───────┘         │   │              │
│           │                       │            Yes ◄┴───┘              │
│        Fail ──► YES ──► Remediate │                 │                  │
│           │       Loop back       │                 ▼                  │
│           │ to Step 13            │                                    │
│           │                       │          STEP 15 (Automated)       │
│        Pass?                   Pass?          Smoke test               │
│           │                       │      - App starts?                 │
│           ▼                       ▼      - All tests pass?             │
│        Proceed                 Proceed  - Environment healthy?         │
│                                             │                         │
│                                             ▼                         │
│                                        Pass/Fail → Done or Rollback  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### State Machine Transitions

- **Forward:** Steps progress linearly within each phase.
- **Backward on failure:** Steps 3, 4, 6 can loop back to Step 2 (revise plan). Steps 11, 13 can loop back to Step 8 (remediate). Step 15 failure triggers rollback.
- **Checkpoints:** After Step 8 (each task), Step 11, Step 13, Step 14. Resume from last checkpoint on session failure.

---

## Step-by-Step Detailed Reference

### PHASE 1: PLAN

#### Step 1: Research the Feature (Frontier)

**What Happens:**
The AI conducts deep research into the implementation strategy for the feature. This step produces a canonical document that explores:
- What the feature does (scope, user journey, edge cases)
- How it will be implemented (architectural choices, technology selection)
- Why this approach (trade-offs, why not alternatives)
- What could go wrong (edge cases, error handling, integration pain points)
- What explicitly will NOT be built (scope boundaries)

Research is iterative: the AI writes a draft, stress-tests it against known edge cases and the existing codebase, and revises. This step consumes more frontier model tokens than any other because it's where subtle architectural mistakes cascade downstream.

**Model Tier:** Frontier (Claude Opus 4.6 with adaptive thinking)
**Reasoning:** Architecture and implementation strategy are high-stakes decisions. A mediocre research phase produces a mediocre plan and mediocre code downstream. Frontier model reasoning power is fully justified here.

**Key Inputs:**
- Feature specification (from discovery/requirements phase)
- Access to existing codebase (for understanding current patterns, prior implementation approaches)
- API documentation for third-party integrations (if applicable)
- Database schema documentation
- Architectural decision records (ADRs) and style guide

**Key Outputs:**
- Canonical research document (~2,000–5,000 words)
  - Implementation approach and rationale
  - Data model implications
  - Integration points with existing code
  - Edge cases and error handling strategy
  - Explicit scope boundaries
- File checklist (list of files that will be touched)
- Risk assessment (high/medium/low per risk category)

**Failure Recovery:**
- If the canonical document is incomplete or misses edge cases: Re-run Step 1 with additional context (e.g., "Focus on async error handling" or "Stress-test against the database schema").
- If the document contradicts the specification: Loop back to the specification phase (outside the 15-step loop) before re-running Step 1.
- If the research is sound but late-discovered issues emerge during implementation (Step 8): Update the canonical document immediately and run Step 4 (Validate Integration) again to catch ripple effects.

**Principle Connections:**
- **Core Principle 2 (Research Before Code):** This entire step embodies this principle. The canonical document is the artifact that forces structured thinking.
- **Core Principle 1 (Context Quality):** High-quality context (existing codebase, API docs, ADRs) directly determines canonical document quality.
- **Core Principle 7 (Integration Contract):** The canonical document should include a preliminary Integration Contract section that gets formalized in Step 2.

---

#### Step 2: Draft the Feature Plan + Integration Contract (Frontier)

**What Happens:**
The AI takes the canonical document from Step 1 and decomposes it into a detailed feature plan. This includes:
- Feature overview and scope
- Detailed task breakdown (not yet the build plan task list, but a logical decomposition)
- Acceptance criteria for each task
- Integration Contract: file-level, API-level, component-level, and database-level integration points
- Dependencies (both external libraries and internal dependencies on other features)
- Estimated effort (in complexity points, not time)

The Integration Contract is **mandatory** and gets highlighted for review in Steps 3–4. It should answer:
- Which existing files will this feature modify?
- Which new files will it create?
- Which components/routes/API endpoints does it connect to?
- Which database tables/fields does it touch?
- Which external services does it call?
- How does it handle failures in those external services?

**Model Tier:** Frontier
**Reasoning:** Decomposition and integration planning require architectural judgment. A mid-tier model can draft this, but frontier reasoning catches integration mistakes earlier (cheaper fix) rather than during audit (Step 10).

**Key Inputs:**
- Canonical document (Step 1 output)
- Existing codebase structure and component inventory
- Prior feature plans (for consistency and pattern recognition)
- Integration Contract template

**Key Outputs:**
- Feature plan document (~1,500–3,000 words)
  - Feature overview
  - Logical task breakdown
  - Acceptance criteria per task
  - Integration Contract (filled in completely)
  - Dependencies and effort estimates

**Failure Recovery:**
- If the Integration Contract is incomplete or misses connections: Return to Step 2 with focused context on the missing integration point(s). Do not proceed to Step 3.
- If the plan contradicts the canonical document: Stop, investigate the contradiction, and update the canonical document if the plan's reasoning is sound.
- If the plan's scope has expanded beyond the original specification: Escalate for a scope review before proceeding.

**Principle Connections:**
- **Core Principle 7 (Integration Contract):** The Integration Contract is the centerpiece of this step.
- **Core Principle 4 (Never Let the Author Validate Their Own Work):** This plan will be validated in Steps 3–4 by fresh contexts.
- **Core Principle 2 (Research Before Code):** The feature plan is the actionable output of research.

---

#### Step 3: Validate Requirements (Frontier, Fresh Context)

**What Happens:**
In a fresh session (clean context, no memory of Steps 1–2), the AI validates that the feature plan satisfies the original specification. This is a **requirements fidelity check**. The validator:
- Reads the original feature specification
- Reads the feature plan (Steps 1–2 outputs)
- Checks for completeness: Does the plan cover all requirements?
- Checks for intent alignment: Does the plan achieve what the user asked for, not just what was written?
- Checks for scope creep: Has anything been added that wasn't in the specification?
- Produces a pass/fail decision with a written explanation

**Model Tier:** Frontier (fresh context)
**Reasoning:** Validation requires judgment about alignment, not just pattern matching. Fresh context prevents the validator from inheriting the author's reasoning and inadvertently confirming biases.

**Key Inputs:**
- Original feature specification
- Feature plan (Steps 1–2 outputs)
- Canonical document (Step 1 output, for reference)

**Key Outputs:**
- Validation report (written)
  - Pass or Fail decision
  - If Fail: specific gaps or misalignments, ranked by severity
  - If Pass: signed-off (validator identifies as "validator," not "author")

**Failure Recovery:**
- If the validator finds gaps: Return to Step 2 with the specific gaps identified. Do NOT skip to later steps.
- If the validator finds minor inconsistencies: Return to Step 2 for minor edits (no re-planning needed).
- If validation passes but later implementation (Step 8+) reveals misalignments: Post-mortem the specification phase (outside the 15-step loop) to understand why it wasn't caught here. Update the specification template if necessary.

**Principle Connections:**
- **Core Principle 4 (Never Let the Author Validate Their Own Work):** This step is the textbook application.
- **Core Principle 3 (Design for Agent Self-Verification):** The validator has a clear pass/fail criterion (specification alignment).

---

#### Step 4: Validate Integration (Frontier, Fresh Context)

**What Happens:**
In another fresh session (separate from Step 3), the AI validates that the feature plan integrates correctly with the existing codebase. This is an **architectural coherence check**. The validator:
- Reads the feature plan (Steps 1–2 outputs, particularly the Integration Contract)
- Reviews the existing codebase (file structure, key components, API routes, database schema)
- Reviews prior feature documentation
- Checks for architectural coherence: Does this feature follow existing patterns? Are all integration points viable?
- Checks for conflict: Will this feature conflict with existing code or concurrent features?
- Checks for missing dependencies: Does the plan assume something that doesn't exist?
- Produces a pass/fail decision with a written explanation

This step is distinct from Step 3 because it answers a different question. Step 3 asks, "Does this satisfy the user?" Step 4 asks, "Does this work in our codebase?"

**Model Tier:** Frontier (fresh context)
**Reasoning:** Architectural validation requires deep understanding of the existing codebase and subtle integration patterns. Frontier reasoning catches mistakes that mid-tier models miss.

**Key Inputs:**
- Feature plan with Integration Contract (Steps 1–2 outputs)
- Existing codebase (file structure, components, API routes, database schema)
- Prior feature documentation and implemented features
- Architectural decision records (ADRs)

**Key Outputs:**
- Validation report (written)
  - Pass or Fail decision
  - If Fail: specific architectural issues, ranked by severity (e.g., "Integration Contract references non-existent route")
  - If Pass: signed-off with notes on integration confidence

**Failure Recovery:**
- If the validator finds integration issues: Return to Step 2 for architectural revisions. Do NOT proceed.
- If the validator finds missing dependencies (e.g., "Plan assumes a utility that doesn't exist"): Either add it to the plan (new task) or revise the plan to avoid the dependency.
- If integration validation passes but later code review (Step 9) or audit (Step 10) finds integration bugs: This signals a gap in the validation criteria. Update the validation template and re-run Step 4 with the new criteria.

**Principle Connections:**
- **Core Principle 7 (Integration Contract):** The Integration Contract is the primary artifact being validated.
- **Core Principle 4 (Never Let the Author Validate Their Own Work):** Fresh context again prevents bias.
- **Core Principle 11 ("Works in Isolation" ≠ "Works in Context"):** This step catches context misalignment early.

---

#### Step 5: Generate Build Plan (Mid-Tier)

**What Happens:**
The AI translates the validated feature plan (Steps 1–4) into a concrete, ordered task list. This is the **build plan**: a sequence of discrete, implementable tasks that can be executed in order without backtracking.

Each task includes:
- Task description (what to do)
- Files to create/modify
- Acceptance criteria (how to know when this task is done)
- Dependencies (which prior tasks must be complete)
- Estimated effort (in Fibonacci points: 1, 2, 3, 5, 8, etc.)

The build plan is produced in **dual format**:
1. **Readable format:** Markdown document, human-friendly, suitable for review and discussion
2. **Structured format:** JSON or YAML, machine-readable, suitable for automation and checkpoint tracking

Both formats contain identical information; the dual format enables both human review and programmatic iteration.

**Model Tier:** Mid-Tier
**Reasoning:** Build planning is pattern-following work (decompose a plan into tasks). A mid-tier model can do this reliably if the input plan is well-formed (which it is, after Steps 1–4). Frontier model reasoning is not needed here.

**Key Inputs:**
- Validated feature plan (Steps 1–4 outputs)
- Existing task templates and examples (for consistency)
- Build plan structure (what fields are required)

**Key Outputs:**
- **Dual-format build plan:**
  - Readable format (Markdown): Ordered task list with descriptions, file targets, criteria, dependencies
  - Structured format (JSON): Same tasks in machine-readable form
- Task count (total number of tasks)
- Estimated total effort (sum of all task efforts, in points)

**Failure Recovery:**
- If the build plan is incomplete or missing tasks: Re-run Step 5 with a checklist of "things this feature must do" as a reference.
- If task ordering is wrong (dependencies not respected): Re-run Step 5 with emphasis on dependency ordering.
- If tasks are too large (>8 points) or too small (<1 point): Re-run Step 5 to re-decompose into right-sized tasks.
- If implementation (Step 8) discovers missing tasks: Add them to the build plan, update the structured format, and continue. Post-mortem Step 5 to improve decomposition templates.

**Principle Connections:**
- **Core Principle 8 (Externalize State):** The dual-format build plan is the externalized state that drives Step 8 implementation.
- **Core Principle 9 (Checkpoint Every Discrete Unit of Work):** Each task in the build plan is one checkpoint (one git commit).

---

#### Step 6: Double-Check the Build Plan (Frontier, Fresh Context)

**What Happens:**
In a fresh session, the AI reviews the build plan (Step 5 output) for completeness, intent alignment, and internal consistency. Specifically:
- Does the build plan cover all tasks implied by the feature plan?
- Are dependencies correct and complete?
- Are file references accurate (do referenced files exist or will be created)?
- Is task ordering logical?
- Are acceptance criteria measurable?
- Will executing these tasks in order produce the intended feature?

This is a **final gate before implementation**. If this step finds issues, they're caught before expensive implementation time is spent.

**Model Tier:** Frontier (fresh context)
**Reasoning:** The last validation step before expensive implementation. Frontier reasoning catches inconsistencies and missing dependencies. Fresh context prevents the Step 5 author from inadvertently confirming their own assumptions.

**Key Inputs:**
- Build plan (Step 5 outputs, both formats)
- Feature plan (Steps 1–2 outputs)
- Existing codebase (to verify file references)
- Build plan quality checklist

**Key Outputs:**
- Validation report (written)
  - Pass or Fail decision
  - If Fail: specific issues (e.g., "Task 3 references non-existent component"), ranked by severity
  - If Pass: signed-off, build plan approved for implementation

**Failure Recovery:**
- If issues are found: Return to Step 5 for revisions. Do NOT proceed to Step 7.
- If minor issues (typos, clarifications): Return to Step 5 for quick fixes.
- If issues are fundamental (major task reordering): Re-run Step 5 entirely.
- If the build plan is approved but implementation (Step 8) discovers issues: Post-mortem Step 6 to understand why it wasn't caught. Update the double-check criteria.

**Principle Connections:**
- **Core Principle 3 (Design for Agent Self-Verification):** The build plan is a specification that can be verified before expensive work begins.
- **Core Principle 4 (Never Let the Author Validate Their Own Work):** Fresh context again.
- **Core Principle 8 (Externalize State):** The build plan is the externalized state; this step validates it before execution.

---

### PHASE 2: BUILD

#### Step 7: Pre-Flight Check (Automated, No LLM)

**What Happens:**
Before implementation begins, the system performs deterministic checks to verify that the build environment is ready:
- Do all referenced files exist (in the codebase)?
- Are all required dependencies installed (in package.json)?
- Is the database schema what the build plan expects?
- Is the git repository in a clean state (no uncommitted changes)?
- Are branch protection rules configured correctly?

If any check fails, the build halts with a clear error message. If all checks pass, the build proceeds to Step 8.

**Model Tier:** Automated (no LLM)
**Reasoning:** These are binary, deterministic checks. An LLM would be slower, more expensive, and less reliable. A simple script is better.

**Key Inputs:**
- Build plan (Step 5 structured format)
- Current state of the codebase
- Current state of dependencies
- Database schema

**Key Outputs:**
- Pre-flight check report
  - All checks passed (proceed to Step 8)
  - One or more checks failed (halt with error details)

**Failure Recovery:**
- If file checks fail: The files referenced in the build plan don't exist. Either correct the file references in the build plan (return to Step 5) or create the files before proceeding.
- If dependency checks fail: Install missing dependencies, then re-run pre-flight.
- If schema checks fail: Migrate the database schema or update the build plan to match the actual schema. Investigate why the schema differs from expectations.
- If repository is not clean: Commit or stash uncommitted changes. (This is a safety check to prevent data loss during checkpoints.)

**Principle Connections:**
- **Core Principle 10 (Deterministic Sandwich):** Pre-flight is pure deterministic pre-processing. No LLM needed.
- **Core Principle 8 (Externalize State):** Pre-flight verifies that external state matches assumptions.

---

#### Step 8: Implement (Mid-Tier, Checkpoint-Driven)

**What Happens:**
The AI works through the build plan task by task. For each task:
1. Read the task description, file targets, and acceptance criteria
2. Write the code to satisfy the criteria
3. Write unit tests and component tests
4. Run the tests locally and confirm they pass
5. Git commit with a descriptive message (one commit per task)
6. Move to the next task

This is the longest step in the pipeline. It's where the majority of implementation work happens. Checkpointing is critical: if the session fails after task 5 of 12, the next session resumes at task 6 without losing work.

The AI:
- Follows the coding patterns and style guide (from project context files)
- Integrates with existing code according to the Integration Contract
- Includes error handling and edge case handling as specified in the canonical document
- Writes tests that verify the acceptance criteria

**Model Tier:** Mid-Tier
**Reasoning:** Code generation and test writing are pattern-following tasks. A well-specified build plan (from Steps 1–6) enables a mid-tier model to handle this reliably. Frontier model overhead is not justified here.

**Key Inputs:**
- Build plan (Step 5 structured format, with current task pointer)
- Existing codebase (for context and pattern matching)
- Project context files (CLAUDE.md, AGENTS.md, or equivalent)
- Test examples and patterns
- Integration Contract (from Step 2)

**Key Outputs:**
- Implemented code (committed to git, one commit per completed task)
- Unit tests and component tests (passing)
- Updated build plan (structured format, with task pointer advanced)

**Failure Recovery:**
- If a task fails to meet acceptance criteria: Revise the code and re-test. Do NOT move to the next task.
- If tests don't pass: Debug the code and tests until all tests pass. Do NOT commit.
- If the AI realizes the build plan was incorrect: Do NOT revise the build plan while implementing. Instead, document the issue and continue with the current plan. (Post-mortem in Step 6 later.)
- If a session fails mid-task: Checkpoint analysis identifies the last completed task. The next session reads the structured build plan, sees the current task pointer, and resumes at that task. No duplicate work.

**Principle Connections:**
- **Core Principle 9 (Checkpoint Every Discrete Unit of Work):** Each task completed = one git commit (checkpoint).
- **Core Principle 8 (Externalize State):** The build plan structured format and git commits are externalized state.
- **Core Principle 11 ("Works in Isolation" ≠ "Works in Context"):** Unit tests verify isolation; integration testing happens in Step 12.
- **Core Principle 5 (Match the Model to the Task):** Mid-tier model is appropriate for pattern-following code generation.

---

#### Step 9: Code Review (Mid-Tier, Fresh Context Recommended)

**What Happens:**
A code review agent (or fresh session of the same agent) reviews the implemented code for surface-level quality issues:
- Style: Does it follow the project style guide?
- Obvious bugs: Are there null pointer dereferences, type mismatches, logic errors?
- Anti-patterns: Does it follow the codebase conventions?
- Type safety: Are all types correct?
- Error handling: Are errors handled appropriately?
- Test coverage: Do the tests cover the main paths?

This is a **focused review**, not a deep audit. It catches bugs that are obvious in code inspection. It does NOT validate system-level integration (that's Step 10).

The reviewer produces an **issue report** with:
- Issue category (style, logic bug, type error, etc.)
- Severity (high/medium/low)
- File and line number
- Description and suggested fix

**Model Tier:** Mid-Tier
**Reasoning:** Code review is pattern-matching work (compare against style guide, look for known anti-patterns). A mid-tier model is sufficient. Fresh context is recommended to prevent the implementer from defending their choices.

**Key Inputs:**
- Implemented code (from Step 8)
- Project style guide and conventions
- Test code and coverage report
- Project context files

**Key Outputs:**
- Issue report (structured, with severity)
  - 0 issues: proceed to Step 10
  - >0 issues: proceed to Step 11 (remediation)

**Failure Recovery:**
- If the reviewer misses obvious issues: This suggests the review prompt or context needs improvement. Update the code review criteria and re-run Step 9 with additional emphasis.
- If the reviewer reports false positives (things that aren't actually issues): Filter them out manually or update the review criteria to be more specific.
- If issues are found: Proceed to Step 11 (remediation), not directly to Step 10.

**Principle Connections:**
- **Core Principle 6 (Separate "Looks Right" from "Works Right"):** Code review is "looks right" — surface-level quality.
- **Core Principle 5 (Match the Model to the Task):** Mid-tier model for pattern-matching review.

---

#### Step 10: Deep Audit (Frontier, Fresh Context, Integration Contract Focus)

**What Happens:**
A deep audit agent (frontier model, fresh context) performs a system-level code audit. This is different from Step 9. The audit asks:
- **Integration:** Does this code integrate correctly with the rest of the system? Are all Integration Contract points satisfied?
- **Async safety:** Are there race conditions, deadlock risks, or async ordering bugs?
- **Resource management:** Are resources (database connections, file handles, memory) managed correctly?
- **Security:** Are there SQL injection risks, authentication/authorization issues, or data exposure?
- **Performance:** Are there algorithmic inefficiencies, N+1 query problems, or memory leaks?
- **Error handling:** Are errors propagated correctly? Are failures handled gracefully?

The audit reviews the same code from Step 9 but with a different lens. Integration bugs, async bugs, and security issues often pass surface-level review but are caught here.

The reviewer produces an **issue report** with:
- Issue category (integration, async, security, performance, etc.)
- Severity (high/medium/low)
- File and line number (if applicable)
- Description and suggested fix
- **Integration Contract verification:** Explicit section confirming all points in the Integration Contract are satisfied

**Model Tier:** Frontier
**Reasoning:** Detecting integration bugs, async bugs, and security issues requires deep reasoning about system behavior. Mid-tier models miss subtle issues that frontier reasoning catches.

**Key Inputs:**
- Implemented code (from Step 8)
- Integration Contract (from Step 2)
- Existing codebase (for integration context)
- Security guidelines and common vulnerability patterns
- Async patterns and concurrency model

**Key Outputs:**
- Deep audit report (structured, with severity)
  - Integration Contract verification (pass/fail)
  - Issues found (by category and severity)

**Failure Recovery:**
- If integration issues are found: These may require code changes (Step 11) or even plan revisions. Evaluate severity:
  - High severity: May require returning to Step 5 (rebuild plan) if the Integration Contract was wrong.
  - Medium/low severity: Proceed to Step 11 (remediation).
- If async bugs are found: Proceed to Step 11. Async bugs are fixable without re-planning.
- If security issues are found: Fix in Step 11. These are typically fixable without re-planning.
- If the audit misses issues that later testing (Step 12) catches: Update the audit criteria and re-run Step 10 with additional emphasis on the missed category.

**Principle Connections:**
- **Core Principle 6 (Separate "Looks Right" from "Works Right"):** Deep audit is "works right" — system-level correctness.
- **Core Principle 7 (Integration Contract):** The Integration Contract is the centerpiece of this audit step.
- **Core Principle 5 (Match the Model to the Task):** Frontier model for deep reasoning about integration and system behavior.

---

#### Step 11: Remediation (Mid-Tier, Iterative)

**What Happens:**
The AI fixes issues identified in Steps 9 and 10. The process is:
1. Sort issues by severity (high → medium → low)
2. For each issue:
   a. Read the issue description and suggested fix
   b. Implement the fix in code
   c. Run unit tests to verify the fix doesn't break other functionality
   d. If tests pass: git commit the fix. Move to the next issue.
   e. If tests fail: Debug and revise the fix until tests pass, then commit.
3. After all issues are fixed, run the full unit test suite one final time to confirm no regressions.

Remediation is iterative: some fixes may have interactions, so testing after each fix is important.

**Model Tier:** Mid-Tier
**Reasoning:** Implementing fixes is pattern-following work. A mid-tier model can handle this reliably.

**Key Inputs:**
- Issue report (from Steps 9–10)
- Implemented code (from Step 8)
- Unit test suite
- Project context files

**Key Outputs:**
- Fixed code (committed to git, one commit per issue fixed)
- Test results (all unit tests passing)
- Updated code

**Failure Recovery:**
- If a fix breaks other tests: The fix likely has a side effect. Revise the fix to be more surgical. Re-test.
- If a fix doesn't resolve the issue: Clarify the issue and revise the fix. Re-test.
- If fixing one issue creates a new issue (e.g., a fix for async race condition introduces a deadlock): This is rare but possible. Document the interaction and devise a fix that resolves both issues.
- If remediation reveals that the build plan was fundamentally wrong: Do NOT revise the build plan mid-remediation. Document the issue and post-mortem later. Continue fixing the current issues with the current plan.
- If all issues are fixed and tests pass: Proceed to Phase 3 (Step 12).

**Principle Connections:**
- **Core Principle 9 (Checkpoint Every Discrete Unit of Work):** Each issue fixed = one or more git commits.
- **Core Principle 11 ("Works in Isolation" ≠ "Works in Context"):** Unit tests verify isolation. Integration issues (if any remain) will be caught in Step 12.

---

### PHASE 3: VERIFY & SHIP

#### Step 12: Full E2E Test Suite (Mid-Tier)

**What Happens:**
The entire end-to-end test suite is run (not just tests for the new feature). This catches regressions: cases where new code breaks previously-working functionality.

The test suite covers:
- All feature-specific tests (from Step 8)
- All existing tests from prior features (regression detection)
- Full application flow tests (user journeys)
- Integration tests (cross-feature interactions)

If all tests pass: Proceed to Step 13. If tests fail: Identify which tests failed and why, then proceed to Step 13 (remediation).

**Model Tier:** Mid-Tier
**Reasoning:** Running tests and analyzing results is deterministic work (tests either pass or fail). A mid-tier model can analyze failures and suggest fixes.

**Key Inputs:**
- Implemented and remediated code (from Step 11)
- Full test suite (all tests, including existing tests)

**Key Outputs:**
- Test results report
  - All tests passed: proceed to Step 13 directly
  - Tests failed: issue report with failed test names, error messages, and suggested fixes

**Failure Recovery:**
- If tests fail: Proceed to Step 13 (remediation). Remediate the failures.
- If a regression is found (new code broke existing functionality): This is expected occasionally. Fix it in Step 13.
- If a test failure is due to test infrastructure (database connection, mock setup): Fix the test infrastructure, not the code.

**Principle Connections:**
- **Core Principle 11 ("Works in Isolation" ≠ "Works in Context"):** Full E2E suite validates system-level correctness, including regressions.

---

#### Step 13: Final Remediation (Mid-Tier)

**What Happens:**
Any test failures from Step 12 are fixed. The process is the same as Step 11:
1. Analyze the failed tests
2. Implement fixes
3. Re-run the full test suite
4. If tests pass: proceed to Step 14. If tests fail: repeat.

This step must not introduce regressions. Each fix is verified to pass all tests (not just the originally failing tests).

**Model Tier:** Mid-Tier
**Reasoning:** Fixing test failures is pattern-following work (similar to Step 11 remediation).

**Key Inputs:**
- Test results report (from Step 12)
- Implemented code
- Full test suite

**Key Outputs:**
- Fixed code (committed to git)
- Test results (all tests passing)

**Failure Recovery:**
- If all tests pass on first attempt: No remediation needed. Skip to Step 14.
- If test failures persist after remediation: Investigate deeply. This may require returning to Step 10 (audit) to understand systemic issues.
- If remediation introduces new test failures: Debug the fix. It may have side effects.

**Principle Connections:**
- **Core Principle 9 (Checkpoint Every Discrete Unit of Work):** Each test fixed = one or more git commits.
- **Core Principle 11 ("Works in Isolation" ≠ "Works in Context"):** This step ensures full system coherence.

---

#### Step 14: Commit + Update Canonical Documentation (Automated + Mid-Tier)

**What Happens:**
Two tasks:
1. **Final commit:** Create a final git commit summarizing the entire feature implementation. Include a reference to the feature plan and build plan in the commit message.
2. **Update canonical documentation:** Update the canonical research document (from Step 1) to reflect what was actually built. Include:
   - What was built (vs. what was planned)
   - Any deviations from the plan (with reasoning)
   - Performance characteristics, if known
   - Known limitations or future improvements
   - Updated integration notes (how other features should integrate with this feature)

The updated canonical documentation becomes the reference for future features that depend on this feature.

**Model Tier:** Automated (commit) + Mid-Tier (documentation update)
**Reasoning:** Git commits are deterministic. Documentation updates benefit from a mid-tier model to ensure clarity and accuracy.

**Key Inputs:**
- Final code state (from Step 13)
- Original canonical document (from Step 1)
- Feature plan and build plan
- Test results
- Implementation notes (any deviations from plan, why)

**Key Outputs:**
- Final commit (to git)
- Updated canonical documentation

**Failure Recovery:**
- If the canonical documentation update is incomplete: The mid-tier model produces a draft; a human reviews and edits before merge.
- If the final commit is incorrect (wrong commit message, wrong files): Amend the commit or create a follow-up commit.

**Principle Connections:**
- **Core Principle 12 (Canonical Docs Are Living Documents):** This step keeps canonical docs up-to-date for future features.
- **Core Principle 8 (Externalize State):** The git commit and documentation are externalized artifacts.

---

#### Step 15: Smoke Test (Automated, No LLM)

**What Happens:**
Post-merge verification. The system checks:
- Does the application start without errors?
- Do all tests pass in the merged environment?
- Is the application environment healthy (no critical errors in logs)?
- Are all expected features operational?

If all checks pass: Feature is shipped. If any check fails: Rollback or hotfix.

**Model Tier:** Automated (no LLM)
**Reasoning:** Smoke tests are binary checks. They either pass or fail. No reasoning needed.

**Key Inputs:**
- Merged code (from Step 14)
- Test suite
- Application environment (deployed or running)

**Key Outputs:**
- Smoke test report
  - All checks passed: feature is shipped
  - One or more checks failed: rollback and investigate

**Failure Recovery:**
- If the app fails to start: Rollback the merge. Investigate in a new session.
- If tests fail in the merged environment but passed in Step 13: This may indicate an environment discrepancy (e.g., database migration didn't apply). Investigate and fix.
- If the environment is unhealthy: This is a separate ops issue, not a code issue. Escalate to ops.

**Principle Connections:**
- **Core Principle 10 (Deterministic Sandwich):** Smoke test is pure deterministic post-processing.

---

## Failure Recovery Patterns

### Pattern 1: Retry with Fresh Context

**When to use:** Validation steps (3, 4, 6) or review steps (9, 10) produce suspicious results.
**How:** Re-run the same step with a fresh session and clarified context. Do not include previous results.
**Example:** Step 10 audit misses an obvious integration issue. Re-run Step 10 with explicit focus on the missed integration point.

### Pattern 2: Return to Earlier Step

**When to use:** A step discovers that an earlier assumption was wrong.
**How:** Return to the step where the assumption was made. Revise it. Re-run all downstream steps.
**Example:** Step 8 implementation discovers that the Integration Contract was incomplete. Return to Step 4 to revise the contract. Re-run Steps 5–6 before continuing.

### Pattern 3: Escalate for Review

**When to use:** An automated step fails, or a frontier-model step finds critical issues.
**How:** Flag for human review. Provide context and the issue report. Wait for guidance before proceeding.
**Example:** Step 7 pre-flight check fails (missing file). Escalate to human: "Build plan references file X, but it doesn't exist. Should I create it or revise the build plan?"

### Pattern 4: Checkpoint and Resume

**When to use:** A session fails mid-step.
**How:** Identify the last completed checkpoint (git commit). Next session resumes from that checkpoint.
**Example:** Session crashes during Step 8, task 7 of 12. The structured build plan records this. Next session reads the plan, sees task 6 was last completed, and continues from task 7.

### Pattern 5: Post-Mortem and Update Criteria

**When to use:** An issue is discovered late (e.g., Step 12 catches a bug that should have been caught in Step 10).
**How:** After the issue is fixed, post-mortem the validation/review step that missed it. Update the criteria or prompts for that step.
**Example:** Step 10 audit missed an N+1 query problem. Post-mortem: add "N+1 query detection" to the audit checklist. Update the audit prompt for the next feature.

---

## The Fresh-Context Pattern: Why Steps 3, 4, and 6 Use Separate Sessions

Three steps in the pipeline (3, 4, 6) explicitly use fresh-context validation:

- **Step 3 (Validate Requirements):** Fresh context prevents the author from inheriting their own reasoning and inadvertently confirming their own plan.
- **Step 4 (Validate Integration):** A different fresh context validates architecture (separate failure mode from requirements).
- **Step 6 (Double-Check Build Plan):** Another fresh context validates the build plan before expensive implementation.

### Why Fresh Context Works

A fresh session starts with no memory of prior reasoning. The validator reads only:
- The specification or artifact being validated
- The output to be validated
- The evaluation criteria

Without the accumulated reasoning that led to the draft, the validator thinks more independently. Confirmation bias is reduced. Errors that the author would naturally excuse are caught.

### Trade-Off: Context Cost

Fresh context has a cost: the validator must re-read context that the author already has. For steps with large codebases, this can add tokens. However:
- The cost is typically <10% overhead per validation step
- The cost buys errors caught early (cheaper fix)
- Frontier models used for these steps are already expensive; the marginal cost of re-reading context is minor

---

## Checkpoint Strategy Across the Loop

### What Is a Checkpoint?

A checkpoint is a git commit marking the completion of a discrete unit of work. It must be:
- **Atomic:** Either entirely applied or entirely reverted
- **Restorable:** Next session can resume from this checkpoint
- **Descriptive:** Commit message explains what was done and why

### Checkpoint Frequency

| Step | Checkpoint Frequency |
|------|---------------------|
| 1 | None (research phase, output is document) |
| 2 | None (planning phase) |
| 3 | None (validation only) |
| 4 | None (validation only) |
| 5 | None (planning phase) |
| 6 | None (validation only) |
| 7 | None (checks only) |
| **8** | **After each task** (high frequency) |
| 9 | None (review only) |
| 10 | None (audit only) |
| 11 | After each issue fixed (medium frequency) |
| 12 | None (testing only) |
| 13 | After each test failure fixed (low-medium frequency) |
| 14 | Final commit |
| 15 | None (smoke test only) |

### Checkpoint Format

```
git commit -m "Task 5: Implement user authentication verification

- Added POST /auth/verify endpoint
- Validates JWT tokens and refreshes if needed
- Added unit tests for token validation
- Updated database schema for token tracking

Feature: [Feature Name]
Build plan task: 5 of 12"
```

### Resuming from Checkpoint

If a session fails at task 7 of 12 in Step 8:
1. Identify the last completed checkpoint (task 6)
2. Next session reads the structured build plan
3. Next session resumes at task 7 (skipping tasks 1–6)
4. No duplicate work; minimal re-reading of context

---

## Cost Profile: Which Steps Are Expensive, Which Are Cheap

### Frontier Model Cost (Claude Opus 4.6, ~$15 per 1M input tokens)

| Step | Typical Tokens | Estimated Cost |
|------|---|---|
| 1 (Research) | 15k–30k | $0.22–$0.45 |
| 2 (Draft Plan) | 10k–20k | $0.15–$0.30 |
| 3 (Validate Reqs, fresh) | 8k–15k | $0.12–$0.22 |
| 4 (Validate Integration, fresh) | 10k–20k | $0.15–$0.30 |
| 6 (Double-Check Plan, fresh) | 8k–15k | $0.12–$0.22 |
| 10 (Deep Audit, fresh) | 15k–30k | $0.22–$0.45 |
| **Phase 1+6 Total** | ~76k–150k | **~$1.14–$2.25** |

### Mid-Tier Model Cost (Claude Sonnet, ~$3 per 1M input tokens)

| Step | Typical Tokens | Estimated Cost |
|------|---|---|
| 5 (Generate Build Plan) | 8k–15k | $0.024–$0.045 |
| 8 (Implement, per task) | 2k–5k per task | $0.006–$0.015 per task |
| 8 (Implement, total 12 tasks) | ~36k–60k | ~$0.108–$0.18 |
| 9 (Code Review) | 8k–15k | $0.024–$0.045 |
| 11 (Remediation) | 5k–10k | $0.015–$0.03 |
| 12 (E2E Testing) | 3k–8k | $0.009–$0.024 |
| 13 (Final Remediation) | 2k–5k | $0.006–$0.015 |
| 14 (Doc Update) | 5k–10k | $0.015–$0.03 |
| **Phase 2+3 Total (Mid-tier)** | ~75k–138k | **~$0.225–$0.414** |

### Automated Steps (No Model Cost)

| Step | Cost |
|------|------|
| 7 (Pre-flight) | ~$0 |
| 15 (Smoke Test) | ~$0 |

### Total Cost Per Feature

| Scenario | Frontier Cost | Mid-Tier Cost | Total | Notes |
|----------|---|---|---|---|
| **Disciplined pipeline (all steps)** | $1.14–$2.25 | $0.225–$0.414 | **$1.37–$2.66** | Optimal. All validation steps included. |
| **Frontier only (no mid-tier)** | $2.50–$5.00 | — | **$2.50–$5.00** | "Always frontier" strategy. Higher cost, fewer corrections needed. |
| **Minimal validation (skip steps 3, 4, 6)** | $0.37–$0.75 | $0.225–$0.414 | **$0.60–$1.16** | Risky. Validation issues caught late (Steps 9–12, more expensive). |

### Where Money Goes

1. **Step 1 (Research):** ~20% of frontier cost. High value; pays for itself by catching architectural issues early.
2. **Step 10 (Deep Audit):** ~20% of frontier cost. Catches integration and security issues; non-negotiable.
3. **Steps 3, 4, 6 (Validation):** ~35% of frontier cost. Prevent rework; good ROI if your features are complex.
4. **Steps 2, 5, 8, 9, 11–14 (Build/Review):** ~45% of total cost. Dominated by implementation (Step 8).

### Cost Optimization

- **Skip Step 3 or 4** if your features are simple and the specification is already validated elsewhere.
- **Combine Steps 3+4** into a single fresh-context validation if context budget is tight.
- **Use mid-tier for Steps 8+9** instead of frontier; catch issues in Step 10 (more expensive but higher-impact).
- **Batch multiple features** to amortize fixed costs (research, planning) across features.

---

## Summary: Reference Quick-Start

| Phase | Steps | Duration | Key Outputs | Failure Patterns |
|-------|-------|----------|------------|---|
| **Plan** | 1–6 | 1–2 hours per feature | Canonical doc, feature plan, build plan | Return to Step 2 or 5; escalate if architecture is wrong |
| **Build** | 7–11 | 4–8 hours per feature (task-dependent) | Implemented code, passing unit tests | Checkpoint and resume if session fails; fix issues in Step 11 |
| **Verify** | 12–15 | 1–2 hours per feature | All tests passing, shipped feature | Fix E2E issues in Step 13; smoke test or rollback |

---

This completes the **Development Loop — Full Reference**. Use this document as your authoritative guide for understanding what happens at each step, why the model assignments work, and how to recover from failures. For questions about core principles, see [[AI-Coding-Best-Practices.md]]. For implementation guidance, see [[Environment Setup — Full Reference]].
