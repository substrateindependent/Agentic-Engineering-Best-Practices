# Externalized State

**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Context Engineering](Context-Engineering.md) · [Integration Contracts](Integration-Contracts.md) · [Canonical Documentation](Canonical-Documentation.md) · [Deterministic Sandwich](Deterministic-Sandwich.md)

---

## What Is Externalized State?

Externalized state is the practice of persisting execution progress, decisions, and context outside of the agent's working memory — in durable storage like files, databases, or structured documents that survive session resets, crashes, and context window exhaustion.

The term "externalized" matters. In traditional software, a process maintains state in RAM. If the process crashes, the state is lost unless it was explicitly written to disk. In agentic systems, the equivalent mistake is treating the agent's context window as sufficient long-term storage. But agents crash, sessions timeout, context windows fill up, and models drift. The state that matters must survive all of these events.

The pattern emerged independently across every production agent framework in 2025–2026: Anthropic's Claude Code, Cursor, Devin, Windsurf, and the frameworks backing them (LangGraph, Temporal, Microsoft Agent Framework) all implement some form of externalized state. The convergence is so strong that Manus AI's team calls it "the single architectural decision that determines whether an agent system works in production or fails."

---

## Why Externalized State Matters

### Agents Crash

AI sessions encounter errors, network issues, rate limits, or deliberate cancellations. Even the most robust setup will have sessions that terminate unexpectedly. When that happens, a system without externalized state must start over. A system with externalized state reads the last checkpoint and resumes.

The cost difference is dramatic. A 2-hour build plan interrupted halfway through: without checkpoints, you lose 1 hour of work; with checkpoints, you lose at most 10 minutes (the duration of the last task).

### Sessions Timeout

Cloud-based agent platforms (Claude, Cursor, etc.) have session limits — often 30–60 minutes of inactivity before the session terminates. Long-running features exceed this window. Without externalized state, context resets lose progress. With checkpoints, the next session loads the last task and continues.

### Context Windows Fill Up

Even with excellent context engineering, long-running tasks accumulate conversation history. The context window approaches saturation. As [Context Engineering](Context-Engineering.md) documents, performance degrades in the "smart zone" above 60% utilization. With externalized state, the agent can safely summarize and reset context mid-task, knowing that checkpoints and durable artifacts will let it recover the critical context later.

### Models Drift

This is less obvious but equally important. As a long session progresses, subtle changes in model behavior accumulate. The reasoning from the 50th turn differs from the reasoning from the 5th turn, not because the model "got tired" but because attention has degraded. Externalized state gives you the ability to reset context and model state at natural boundaries (task completion) rather than pushing the agent to produce low-quality work near the context limit.

### Observability Requires Durability

If your only record of what happened is in the agent's context window, you can't debug failures. You can't see what plan was created, whether it was validated, what went wrong during implementation, or why a task failed. Externalized state — build plans, commit messages, test results — becomes your observability log.

---

## The Dual-Format Build Plan Pattern

The most practical expression of externalized state in coding workflows is the **dual-format build plan**: human-readable markdown alongside structured data that agents can parse and consume.

### The Readable Format

A markdown file with task checkboxes:

```markdown
# Feature: Authentication System

## Status: In Progress

- [x] Task 1: Create User model
- [x] Task 2: Implement password hashing
- [ ] Task 3: Build login endpoint
- [ ] Task 4: Add session management
- [ ] Task 5: Write integration tests
- [ ] Task 6: Update docs

## Current Task: Task 3 — Build login endpoint

**Acceptance Criteria:**
- POST /auth/login accepts username and password
- Returns JWT token on success
- Returns 401 on invalid credentials
- Validates input (non-empty, reasonable length)

**Files to modify:**
- src/routes/auth.ts
- tests/auth.test.ts

**Dependencies:**
- Task 2 (password hashing) must be complete
- User model from Task 1

**Estimated effort:** 45 minutes
```

This format is immediately readable by humans. You glance at it and know: we've completed 2 tasks, we're currently on task 3, and there are 3 tasks left. You can track progress without parsing code or running commands.

### The Structured Format

The same information as JSON or YAML for programmatic consumption:

```json
{
  "feature": "Authentication System",
  "status": "in_progress",
  "tasks": [
    {
      "id": 1,
      "name": "Create User model",
      "completed": true,
      "commit": "abc1234"
    },
    {
      "id": 2,
      "name": "Implement password hashing",
      "completed": true,
      "commit": "def5678"
    },
    {
      "id": 3,
      "name": "Build login endpoint",
      "completed": false,
      "status": "in_progress",
      "acceptance_criteria": [
        "POST /auth/login accepts username and password",
        "Returns JWT token on success",
        "Returns 401 on invalid credentials",
        "Validates input"
      ],
      "files": ["src/routes/auth.ts", "tests/auth.test.ts"],
      "dependencies": ["Task 2", "Task 1"],
      "estimated_effort_minutes": 45
    }
  ],
  "current_task_id": 3
}
```

An agent can programmatically parse this structure. "What's the current task?" Parse `current_task_id` → Task 3. "Which files do I need to modify?" Parse `tasks[2].files`. "What are my acceptance criteria?" Parse `tasks[2].acceptance_criteria`.

### Why Both?

The markdown is for humans (you, your team, code review). The JSON is for agents. By maintaining both, you get:

- **Readability:** No one needs to parse JSON to understand progress.
- **Programmability:** The agent can consume structure without NLP interpretation.
- **Consistency:** A single source of truth. When you update one, the other must match (enforced by tooling, not trust).
- **Debuggability:** When something goes wrong, you have both human-readable documentation and machine-parseable state.

This dual-format approach is standard in systems that demand high reliability. Kubernetes uses YAML (human-readable) + internal structs (machine-parsed). Terraform uses HCL + JSON state files. The pattern works because humans and machines have different information needs.

---

## Checkpoint-Per-Task: Git Workflow

After every discrete task completes, two things happen:

1. **Git commit with a descriptive message.**
2. **Update the build plan checkboxes.**

### The Commit Message Format

A good checkpoint commit message includes three elements: what was built, what was tested, and any decisions made.

```
Task 2: Implement password hashing

- Added bcrypt-based hashing in User model
- Implemented verify() and hash() methods
- Added 1000-iteration cost factor for security/performance tradeoff
- Unit tests passing (5/5)
- No integration tests at this stage (depend on auth routes from Task 3)

Related to: build-plan/001-authentication.md#task-2
```

Why this format matters:

- **What was built:** If you need to understand the feature later, you have a clear statement of scope.
- **What was tested:** You know the confidence level. If tests failed or were skipped, that's documented.
- **Decisions made:** The 1000-iteration cost factor is a deliberate trade-off. Future you (or a team member) reading the commit understands why it was chosen.
- **Plan reference:** The commit links back to the build plan, creating bidirectional traceability.

### The Checkbox Update

In the markdown build plan:

```markdown
- [x] Task 2: Implement password hashing
```

The checkbox state is now your source of truth for what's complete. An agent reading the plan at the start of the next session immediately knows: "I need to skip tasks 1 and 2, start with task 3."

This is a simple pattern, but it's surprisingly powerful. The combination of commit history + checkbox state gives you:

- **Failure recovery:** Next session knows where to resume.
- **Blame tracking:** `git log` shows who did what, when.
- **Auditability:** You have a complete record of every decision and its rationale.
- **Parallelization:** Multiple agents working on independent tasks can each track their own progress.

---

## The Three Benefits: Failure Recovery, Context Efficiency, Observability

### 1. Failure Recovery

A build plan spans a 2-hour coding task. At the 1-hour mark, the session crashes. With externalized state:

1. Next session reads the build plan.
2. It checks the completed tasks: [x] [x] [ ] [ ] [ ]
3. It loads the last commit message to understand what was built.
4. It resumes at Task 3.

**Without externalized state:** Start the entire build plan over.

**With externalized state:** Lose at most the duration of Task 3 (maybe 30 minutes), not the entire feature (2 hours).

This matters even more for teams running multiple concurrent agents. If Agent A crashes on Task 5 of a 10-task plan, Agent B doesn't have to redo all 5 completed tasks — Agent B reads the checkboxes, skips to Task 6, and continues.

### 2. Context Efficiency

A build plan with 10 tasks is too long to keep in context for all 10 tasks. But you don't need to. Each task only needs:

- The current task's requirements.
- The previous task's output (to understand dependencies).
- The list of remaining tasks (for context about what comes next).

With externalized state:

```
Next session:
  Load build plan (JSON)
  Load Task 5 details
  Load output of Task 4 (previous task)
  Load 3 remaining tasks (for lookahead)
  Start implementing Task 5

Result: ~4KB of context instead of ~50KB (the full build plan)
```

Manus AI reports an average compression ratio of 100:1 — for every KB of context the agent loads into the session, there's 100 KB available externally that can be retrieved if needed. This is possible because most of the context (completed tasks, old decisions, prior research) is rarely needed during active implementation.

The practical effect: the agent stays in the 30–50% context utilization "smart zone" throughout a long task, rather than degrading as it approaches the context limit.

### 3. Observability

After the build plan completes, you have:

- A markdown checklist showing what was built, in order.
- A JSON structure with task metadata, timings, and dependencies.
- A git log with one commit per task, each explaining what was implemented.
- Test results from each phase (review, audit, E2E).

You can ask:

- "How long did Task 5 actually take?" Check the git commit timestamps.
- "Why did we choose this architecture for Task 3?" Read the commit message.
- "Did Task 7 have any test failures?" Check the test results in the JSON metadata.
- "Which tasks depend on Task 4?" Parse the dependencies in the JSON.

This level of observability is critical for debugging production failures. If a feature breaks in production months later, you can trace back to the original build plan, see what was implemented, and understand the context in which decisions were made.

---

## Manus AI's Approach: File-System as External Memory

Manus AI rebuilt their agent framework four times learning these lessons. Their final architecture treats the file system as unlimited external memory. The principle:

> Any information that doesn't fit in the context window can be stored as a file and retrieved when needed. The retrieval is lossless — the original file is always available — so compression is "free" in terms of information loss.

### Example: Processing a Large Document

**Naive approach:** Load the entire document into context, ask the agent to process it.

**Manus approach:**

1. Agent reads document: 50 KB, 100 pages.
2. Agent creates `doc_summary.md`: 500 bytes of key points, structure, and findings.
3. Agent creates `doc_index.md`: one-line entry per section (page, title, first 50 words), 2 KB.
4. Agent stores references: `see file:///docs/original_document.pdf`, `summary at ./doc_summary.md`, `index at ./doc_index.md`.
5. Agent's context uses 2.5 KB (summary + index + references).
6. Next session (or later step) reads the summary and index, can retrieve specific sections from the original as needed.

**Compression ratio:** 50 KB input → 2.5 KB context = 20:1. If later steps only read the summary (most common case), no retrieval is needed. If a detail is required, one file read brings in that section (50 bytes instead of the full document).

For the full agent system across 100-turn tasks, the ratio reaches 100:1. The implication: **externalized state is not a compromise. It's strictly better than trying to fit everything in context.**

---

## Structured Note-Taking: Agentic Memory

Beyond build plans, the most practical form of externalized state is **agentic memory** — a running log maintained by the agent outside its context window.

Claude Code naturally does this with NOTES.md files. The pattern:

```markdown
# Session Notes — Feature: Authentication System

## Goals
- Build user authentication system with JWT tokens
- Implement password hashing with bcrypt
- Create login/logout endpoints

## Decisions Made
1. **Password hashing:** bcrypt with cost factor 10 (2025 recommendation)
2. **Token expiry:** 1 hour access token, 7-day refresh token
3. **Storage:** JWT in httpOnly cookies (prevents XSS exposure)

## Key Findings
- User model already exists in `src/models/User.ts`
- Session handling library available: `express-session`
- Test utility for auth: `authTestHelper.ts` in test suite

## Completed
- [x] Research password hashing options
- [x] Design JWT strategy
- [x] Verify existing User model supports password field
- [x] Implement bcrypt integration

## Blockers
- None currently

## Next Steps
1. Build login endpoint
2. Add session management
3. Write integration tests
4. Update API documentation

## Architecture Decisions
- Store JWT in httpOnly cookie (server-accessible, XSS-proof)
- Use refresh token pattern (short-lived access, long-lived refresh)
- Middleware for auth verification at route level
```

This file is *outside the context window* but accessible to the agent. At the start of the next session, the agent reads NOTES.md to understand:

- What was already decided (no need to re-research).
- What blockers exist.
- What the next steps are.
- Key architectural decisions and their rationale.

**Anthropic's Claude playing Pokémon is the canonical example.** The agent maintained precise tallies of caught Pokémon, explored maps, tracked item locations, and sustained multi-hour sessions across context resets — all by reading its own notes. A transcript-only approach would have failed: the context would have filled with game state observations, leaving no room for reasoning. But with notes, the agent could compress thousands of observations into a summary.

The memory system works because:

1. **It's selective.** Not every fact goes into notes, only high-signal decisions and findings.
2. **It's additive.** The agent appends new insights to existing notes; it doesn't try to re-derive everything.
3. **It's readable.** The agent (and you) can quickly scan the notes to understand what happened.
4. **It survives context resets.** Even if the conversation context clears, the notes persist on disk.

---

## Framework-Level Checkpointing

The major agent frameworks implement checkpointing natively:

### LangGraph: Time-Travel Debugging

LangGraph (LangChain's agent orchestration framework) saves the full execution graph at each step. You can:

- **Resume from any checkpoint:** If a task fails at step 7, restart from step 7 instead of step 1.
- **Time-travel debug:** Inspect the exact state at any point in execution.
- **Branch from checkpoints:** Create alternate paths ("what if we chose option B at step 5?") without re-executing all previous steps.

Example from LangGraph documentation:

```python
# Save checkpoint after each step
executor = Executor(graph, checkpoint_fn=save_checkpoint)

# Resume from checkpoint
result = executor.run(start_from_checkpoint=5)

# Time-travel to inspect state
state = executor.get_state_at_step(7)
```

### Temporal: Durable Execution

Temporal (used by OpenAI for Codex) treats externalized state as the primary feature. Every activity (API call, file write, decision point) is logged. Failures are recovered by replaying the log up to the failure point, then retrying from there.

Temporal's model:

```
Activity A → Activity B → Activity C (fails) → retry Activity C
                         ↓
                    All state saved
                    Resume on retry
                    No activity replay needed
```

### Microsoft Agent Framework: Checkpoints as First Class

Microsoft's agentic AI framework (used internally for service deployments) makes checkpoints a first-class concept. Every agent invocation has a checkpoint ID. State is serialized at regular intervals. Recovery is trivial: deserialize from the last checkpoint ID.

---

## Hybrid Recovery: Snapshots + Serialized Checkpoints

Production systems often use a hybrid approach combining two recovery strategies:

### Stateful Snapshots (Fast Recovery)

After each major milestone (plan validation, code review), save a full snapshot of the state:

```json
{
  "checkpoint_id": "review-complete-2026-02-23-14-30",
  "phase": "code_review",
  "completed_tasks": [1, 2, 3],
  "current_task": 4,
  "build_plan": { ... },
  "files_modified": ["src/auth.ts", "src/models/User.ts"],
  "test_results": { "unit": "passing", "integration": "pending" },
  "context_state": {
    "recent_commits": [...],
    "validation_notes": [...]
  }
}
```

If a session crashes, you can restore from this snapshot in seconds, with all context intact. No need to re-run earlier tasks.

### Stateless Serialized Checkpoints (Durability)

Also keep a lightweight serialization of just the critical state:

```json
{
  "checkpoint_id": "task-4-complete-2026-02-23-14-45",
  "type": "task_completion",
  "task_id": 4,
  "commit_hash": "a1b2c3d4",
  "outputs": [
    "src/routes/login.ts",
    "tests/login.test.ts"
  ],
  "test_status": "passing",
  "next_task_id": 5
}
```

This is much smaller and can be stored in a database or data warehouse for long-term durability and audit trails. Even if you lose the snapshot (which gets discarded after recovery), the serialized checkpoints remain, allowing you to reconstruct the full history.

The hybrid approach: **use snapshots for fast recovery within an active session, use serialized checkpoints for long-term durability and auditability.**

---

## The Plan-and-Execute Pattern with Re-Planning Checkpoints

For multi-hour tasks, externalized state enables a sophisticated pattern:

1. **Plan Phase:** Create the build plan, validate it, commit it to the repo.
2. **Implementation:** Execute the plan task by task, checkpointing after each task.
3. **Mid-task re-planning:** At each checkpoint (after tasks 3, 6, 9, etc.), evaluate the plan. Has context changed? Do we still think the remaining tasks are correct? If not, update the plan.
4. **Final validation:** After all tasks complete, review the plan vs. what was actually built.

### Why Mid-Task Re-Planning Works

As you implement, you learn:

- "We thought Task 5 would take 30 minutes, but it actually took 10 minutes."
- "The refactoring in Task 3 revealed that Task 7 won't be needed."
- "An API integration we planned for Task 8 is already partially implemented."

Without externalized state, you'd discover these things mid-task and have to improvise. With externalized state, you update the plan, adjust the remaining tasks, and maintain alignment between what was planned and what was built.

Addy Osmani's "agentic engineering" workflow includes exactly this pattern: plan with checkpoints, re-plan at checkpoints, execute with full confidence that the plan reflects current understanding.

---

## Practical Example: A Complete Build Plan

Here's what a real build plan looks like, combining readable and structured elements:

**File:** `docs/build-plans/004-payment-integration.md`

```markdown
# Build Plan: Payment Integration with Stripe

**Status:** In Progress
**Created:** 2026-02-20
**Current Task:** Task 4

## Overview
Integrate Stripe for payment processing. Customers can create payments,
webhooks validate completion, database records payment state.

## Tasks

### [x] Task 1: Create Payment model
**Completed:** 2026-02-20 08:15
**Commit:** `payment-model-initial`

Acceptance criteria:
- [ ] Payment model with amount, currency, status, stripe_id fields
- [ ] Database migration for payments table
- [x] Unit test for model initialization

**Files:** src/models/Payment.ts, db/migrations/003_create_payments.ts

---

### [x] Task 2: Add Stripe client setup
**Completed:** 2026-02-20 09:45
**Commit:** `stripe-client-setup`

Acceptance criteria:
- [x] Stripe SDK imported and initialized in env
- [x] API key loaded from environment variables
- [x] Test utilities for Stripe mocking (stripe-mock package)

**Files:** src/lib/stripe.ts, tests/fixtures/stripe-mocks.ts, .env.example

**Dependencies:** Task 1 (needs Payment model reference)

---

### [x] Task 3: Create payment endpoint
**Completed:** 2026-02-21 11:20
**Commit:** `payment-endpoint-create`

Acceptance criteria:
- [x] POST /api/payments endpoint accepts amount, currency
- [x] Creates Payment record in database
- [x] Sends create request to Stripe API
- [x] Returns payment intent to client
- [x] Input validation (positive amount, valid currency)
- [x] Unit tests passing (8/8)
- [x] Integration test with Stripe mock (pass)

**Files:** src/routes/payments.ts, tests/payments.test.ts

**Dependencies:** Task 1 (Payment model), Task 2 (Stripe client)

**Notes:** Endpoint created, but client-side component (from frontend team)
not yet available. Tested with curl. Ready for integration.

---

### [ ] Task 4: Implement webhook handler
**Status:** In Progress
**Started:** 2026-02-21 13:00
**Estimated Completion:** 2026-02-21 14:30

Acceptance criteria:
- [ ] POST /api/webhooks/stripe accepts Stripe webhook events
- [ ] Validates webhook signature (prevents spoofing)
- [ ] Updates Payment record status on payment_intent.succeeded
- [ ] Handles payment_intent.payment_failed (sets status to failed)
- [ ] Idempotent (calling webhook twice with same event_id doesn't double-process)
- [ ] Tests passing (create test fixtures for common webhook events)
- [ ] Error handling (invalid signatures rejected with 401)

**Files:** src/routes/webhooks.ts, tests/webhooks.test.ts

**Dependencies:** Task 1 (Payment model), Task 2 (Stripe client)

**Integration:** Stripe account must be configured to send webhooks to
staging URL: https://staging.example.com/api/webhooks/stripe

**Notes:** Need to test webhook signature validation. Stripe provides
test payload in dashboard. Implementation should be defensive.

---

### [ ] Task 5: Payment status queries
**Status:** Pending

Acceptance criteria:
- [ ] GET /api/payments/:id returns payment details
- [ ] Only accessible by payment owner or admin
- [ ] Returns current status from database (not re-queried from Stripe)
- [ ] Tests passing

**Files:** src/routes/payments.ts (add endpoint), tests/payments.test.ts

**Dependencies:** Task 1 (Payment model), Task 3 (create endpoint)

---

### [ ] Task 6: Update docs and finalize
**Status:** Pending

Acceptance criteria:
- [ ] API documentation updated (endpoints, request/response examples)
- [ ] Integration guide for payment flow (create → webhook → status)
- [ ] Troubleshooting section (common Stripe errors)

**Files:** docs/payments.md

**Dependencies:** All previous tasks

---

## Integration Contract

**Files Modified:**
- src/routes/payments.ts (new)
- src/models/Payment.ts (new)
- src/lib/stripe.ts (new)
- db/migrations/003_create_payments.ts (new)

**Files Created:** 4 (listed above)

**API Contracts:**
- POST /api/payments: Create payment intent
- POST /api/webhooks/stripe: Handle payment events
- GET /api/payments/:id: Query payment status

**Database Changes:**
- New table: payments (id, user_id, amount, currency, status, stripe_id, created_at)
- Index on (user_id, created_at) for query performance

**Component Integration:**
- Payments system is independent; depends on User model (already exists)
- Frontend team will integrate UI component (not part of this plan)

**Dependencies:**
- This feature depends on: User model (existing), database (existing)
- Future features depend on: Payment model, payment endpoints

---

## Blockers / Notes

- Stripe sandbox account configured and tested ✓
- Test credit card: 4242 4242 4242 4242 (Stripe default)
- Webhook signature validation requires `raw request body` (not parsed JSON)

---

## Summary

6 tasks total. 3 complete. 2 in progress / pending. 1 blocked.

Current velocity: ~1.5 hours per task on average.
Estimated completion: 2026-02-21 16:00 (4 more hours).

All planned tasks have clear acceptance criteria and file targets.
Integration with existing User and database systems verified.
```

**Companion JSON file:** `docs/build-plans/004-payment-integration.json`

```json
{
  "plan_id": "004-payment-integration",
  "feature_name": "Payment Integration with Stripe",
  "status": "in_progress",
  "created_at": "2026-02-20T08:00:00Z",
  "current_task_id": 4,
  "tasks": [
    {
      "id": 1,
      "name": "Create Payment model",
      "status": "completed",
      "completed_at": "2026-02-20T08:15:00Z",
      "commit": "payment-model-initial",
      "files": ["src/models/Payment.ts", "db/migrations/003_create_payments.ts"],
      "acceptance_criteria": [
        "Payment model with amount, currency, status, stripe_id fields",
        "Database migration for payments table",
        "Unit test for model initialization"
      ]
    },
    {
      "id": 2,
      "name": "Add Stripe client setup",
      "status": "completed",
      "completed_at": "2026-02-20T09:45:00Z",
      "commit": "stripe-client-setup",
      "files": ["src/lib/stripe.ts", "tests/fixtures/stripe-mocks.ts"],
      "acceptance_criteria": [
        "Stripe SDK imported and initialized",
        "API key loaded from environment",
        "Test utilities for Stripe mocking"
      ]
    },
    {
      "id": 3,
      "name": "Create payment endpoint",
      "status": "completed",
      "completed_at": "2026-02-21T11:20:00Z",
      "commit": "payment-endpoint-create",
      "files": ["src/routes/payments.ts", "tests/payments.test.ts"],
      "acceptance_criteria": [
        "POST /api/payments endpoint",
        "Creates Payment record",
        "Sends request to Stripe API",
        "Returns payment intent",
        "Input validation",
        "Unit tests passing"
      ],
      "dependencies": [1, 2],
      "test_results": {
        "unit": "passing",
        "integration": "passing"
      }
    },
    {
      "id": 4,
      "name": "Implement webhook handler",
      "status": "in_progress",
      "started_at": "2026-02-21T13:00:00Z",
      "estimated_completion": "2026-02-21T14:30:00Z",
      "files": ["src/routes/webhooks.ts", "tests/webhooks.test.ts"],
      "acceptance_criteria": [
        "POST /api/webhooks/stripe endpoint",
        "Validates webhook signature",
        "Updates Payment status on success",
        "Handles payment failures",
        "Idempotent processing",
        "Error handling"
      ],
      "dependencies": [1, 2]
    },
    {
      "id": 5,
      "name": "Payment status queries",
      "status": "pending",
      "files": ["src/routes/payments.ts"],
      "acceptance_criteria": [
        "GET /api/payments/:id endpoint",
        "Authorization checks",
        "Returns status from database",
        "Tests passing"
      ],
      "dependencies": [1, 3]
    },
    {
      "id": 6,
      "name": "Update docs and finalize",
      "status": "pending",
      "files": ["docs/payments.md"],
      "acceptance_criteria": [
        "API documentation",
        "Integration guide",
        "Troubleshooting section"
      ],
      "dependencies": [1, 2, 3, 4, 5]
    }
  ],
  "integration_contract": {
    "files_modified": ["src/routes/payments.ts", "src/models/Payment.ts"],
    "files_created": ["src/lib/stripe.ts", "db/migrations/003_create_payments.ts"],
    "api_contracts": [
      {
        "method": "POST",
        "path": "/api/payments",
        "description": "Create payment intent"
      }
    ],
    "database_changes": [
      {
        "type": "create_table",
        "table": "payments",
        "columns": ["id", "user_id", "amount", "currency", "status", "stripe_id"]
      }
    ]
  }
}
```

This dual format lets humans track progress at a glance (markdown) and agents query structure programmatically (JSON).

---

## Anti-Patterns and Pitfalls

### Externalized State Without Discipline

Maintaining external state is only valuable if you actually use it. If you create a build plan and never update it, it becomes worse than useless — it's a lie. Adopt the checkpoint-per-task discipline or don't adopt externalized state at all.

### Stale State

A build plan that reflects intentions, not reality, misleads future sessions. The cost is higher than having no plan at all. Update the plan immediately after any significant change. Better: update it within 5 minutes of a task completion.

### Over-Granularity

Not every micro-decision needs to be externalized. A task is typically 30 minutes–2 hours of work. If your tasks are "fix one line," you have too many tasks. If your tasks are "build the entire backend," you have too few. Find the Goldilocks zone where tasks are self-contained but meaningful.

### Context Without Checkpoints

Maintaining great context engineering (per [Context Engineering](Context-Engineering.md)) but never committing or checkpointing your work defeats the purpose. The checkpoint *is* what makes context engineering work. It lets you reset and reload from a known, durable state.

### Loss of Observability

If you don't keep records (commit messages, test results, decision logs), externalized state becomes pointless. The state survives, but you don't know what it means. Invest in observability at the same time as you invest in checkpointing.

---

## Getting Started: Five Concrete Steps

If you're not using externalized state yet, start here:

1. **Create a build plan for your next feature in markdown with checkboxes.**
   - One task per 30–90 minutes of work.
   - Explicit acceptance criteria per task.
   - Checkbox format: `[ ]` for pending, `[x]` for complete.
   - Commit this file to the repo before you start coding.

2. **Git commit after every completed task.**
   - Commit message: "Task N: [name] — [what was built, what was tested, decisions made]"
   - Update the checkbox in the build plan immediately after (or as part of the same commit).

3. **Create a NOTES.md file in your project.**
   - Keep it updated with key decisions and findings.
   - At the start of each session, read NOTES.md to understand the context.
   - Add new findings as you work.

4. **Add a JSON companion to your build plan.**
   - Convert the markdown task list to JSON.
   - Include task IDs, completion status, file targets, and acceptance criteria.
   - Keep both in sync (manually, or with a simple script).

5. **When a session crashes, use the checkpoint to resume.**
   - Read the build plan.
   - Find the last completed task.
   - Start the next session with just that task's context.
   - Verify you saved ~50% of the work by resuming from a checkpoint.

---

## Sources

### Framework-Level Checkpointing

- LangGraph, "[Time-Travel Debugging](https://langchain-ai.github.io/langgraph/concepts/agentic_loop/#time-travel-debugging)" (2025)
- Temporal, "[Durable Execution](https://temporal.io/durable-execution)" (official documentation)
- Microsoft, "Agent Framework Checkpointing" (internal documentation, 2026)

### Agent Architecture and State Management

- Manus AI, "[Building Resilient AI Agents: Lessons from Four Framework Rewrites](https://manus.im/blog/Building-Resilient-AI-Agents)" (2025)
- Nanonets, "[Why AI Agents Fail in Production: State Management Failures](https://nanonets.com/blog/ai-agent-failures/)" (2025)
- HumanLayer, "[Externalized State for Agent Resilience](https://www.humanlayer.dev/blog/externalized-state)" (2025)

### Production Agent Systems

- Anthropic, "Effective harnesses for long-running agents" (2025)
- Redis Blog, "[Agent Architecture Patterns](https://redis.com/blog/agent-architecture/)" (2025)
- OpenAI, "Codex Agent Architecture" (inferred from published design, 2025)

### Agentic Coding Practices

- Boris Cherny, Claude Code workflows and checkpoint patterns (2026)
- Addy Osmani, "Plan-and-Execute with Checkpoints" (2026)
- Thoughtworks, "Spec-driven development with externalized state" (2025)
