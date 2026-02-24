# Agent Self-Verification

**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Context Engineering](Context-Engineering.md) · [Integration Contracts](Integration-Contracts.md) · [Testing Strategy](Testing-Strategy.md) · [Externalized State](Externalized-State.md) · [Project Context Files](Project-Context-Files.md)

---

## What Is Self-Verification?

Self-verification is the discipline of giving AI agents feedback loops so they can confirm their own work before you review it. Instead of asking "Did you finish?" and hoping the answer is accurate, the agent runs tests, checks file references, validates type safety, or executes the code it wrote — then reports back with evidence.

Boris Cherny, creator of Claude Code, calls this "probably the most important thing to get great results." His data supports the claim: agents given verification loops produce output that's 2–3x higher quality than agents without them. The reason is straightforward: without verification, the agent declares "done" based on its own assessment. With a feedback loop, it declares "done" based on *evidence*.

The principle applies everywhere in the development loop:

- **During research:** The agent verifies that API documentation is current, that referenced files actually exist, that examples in the docs still match the API's behavior.
- **During planning:** The agent checks that file references in the plan match reality, that the proposed architecture aligns with the codebase structure, that dependencies are declared correctly.
- **During implementation:** The agent runs tests after writing code, checks linting and type safety, validates that the build succeeds.
- **During review:** The agent runs the full test suite to confirm no regressions were introduced.

In every case, the agent doesn't rely on intuition or training data. It checks.

---

## The Core Pattern: Generate → Verify → Fix → Re-Verify

Every self-verification workflow follows this pattern:

1. **Generate:** The agent produces output — a plan, code, documentation, or test.
2. **Verify:** The agent uses tools or test harnesses to confirm the output is correct.
3. **Fix:** If verification fails, the agent analyzes the failure and produces a correction.
4. **Re-verify:** The agent runs the verification again to confirm the fix worked.

This loop can repeat multiple times if needed. OpenObserve's "Healer" agent demonstrates the pattern at scale: a specialized agent that runs tests after every implementation task, diagnoses test failures by reading error messages and examining the code, produces a fix, re-runs the tests, and iterates up to five times automatically. The result is a dramatic reduction in human intervention — most bugs are self-healing.

The critical insight: **the feedback loop compresses the iteration cycle.** Without it, the agent produces output, you review it, identify issues, and ask for fixes — a human-in-the-loop delay per iteration. With it, the agent identifies and fixes most issues *automatically*, within the same session. You only step in when the feedback loop exhausts its ability to correct (usually 4–5 attempts).

---

## Self-Verification Across All Development Phases

### During Research (Phase 1, Step 1)

When researching how to implement a feature, the agent shouldn't rely on training data about APIs, frameworks, or existing code. Instead:

**Verify documentation currency.** Download the actual API docs rather than reciting training knowledge. OpenAI docs change; so do framework APIs. The agent should be able to say "I read the Stripe API docs from February 2026" rather than "I recall that Stripe v3 works like this."

**Verify file existence and content.** Before planning around existing code, the agent should grep for key files, read their structure, and confirm they contain what the plan assumes. For example: "I searched for `src/auth/` files and found three implementation approaches across the codebase. The one we should integrate with is in `/app/auth/jwt.ts`."

**Verify examples work.** If the docs include example code, the agent should attempt to run or parse them to confirm they're correct. This catches docs that are outdated or contain typos.

**Technique:** Include a "Verification Checklist" section in your canonical documents template. Research-phase tasks should conclude with:
- [ ] Downloaded and verified current API docs
- [ ] Confirmed existing implementation files are where the plan references them
- [ ] Checked that examples in docs match current behavior

### During Planning (Phase 1, Steps 2–4)

Planning is where most agent errors occur — invalid file references, misunderstanding of existing code structure, or oversimplifying integration points. Self-verification during planning prevents cascading errors during implementation.

**Verify file references exist.** Before the plan references `src/components/Button.tsx`, confirm that file exists. A simple grep for the filename catches typos and false assumptions.

**Verify schema matches.** If the plan assumes a database table has a `user_id` column, the agent should read the migration file or schema definition to confirm.

**Verify Integration Contract completeness.** The [Integration Contracts](Integration-Contracts.md) pattern requires a section describing how new code connects to existing code. The agent should verify that each connection listed in the Integration Contract actually exists in the codebase.

**Technique:** Build a pre-flight check tool that validates the plan before implementation starts (Step 7 in the Development Loop). The check should be deterministic — no LLM reasoning, just binary pass/fail tests:

```
Pre-Flight Check:
- [ ] All referenced files exist
- [ ] All referenced database tables exist in schema
- [ ] All API endpoints in Integration Contract exist
- [ ] All dependencies listed are in package.json
- [ ] Build succeeds (smoke test)
```

Make this check non-optional. If it fails, planning wasn't complete enough to proceed safely.

### During Implementation (Phase 2, Steps 8–10)

This is where self-verification has the highest impact. As the agent writes code, it should test that code before declaring the task complete.

**Run unit tests after writing.** After implementing a function or module, the agent should write tests that exercise that code, then run them. If tests fail, the agent debugs and fixes the code, then re-runs.

**Run linting and type checking.** After writing code, run your project's linter and type checker (ESLint, TypeScript, Mypy, whatever applies). Treat linting failures as blocking — the code isn't done until linting passes.

**Run component tests.** For UI code, the agent should write tests that confirm the component renders correctly and responds to user interaction as expected.

**Technique:** In your [CLAUDE.md or Cursor Rules](Project-Context-Files.md), include a section titled "Self-Verification Gates" that lists the exact commands to run after each implementation task:

```markdown
## Self-Verification Gates

After implementing any feature or fix, run:

1. `npm run type-check` (TypeScript type safety)
2. `npm run lint` (linting and formatting)
3. `npm run test -- --testPathPattern=<feature>` (unit tests)
4. `npm run build` (does the build succeed?)

Do not declare a task complete until all gates pass. If a gate fails, debug, fix, and re-run until it passes.
```

This externalized instruction is critical. Boris Cherny's team embeds verification instructions in CLAUDE.md specifically because agents follow explicit instructions more reliably than inferring what verification should happen.

### During Code Review (Phase 2, Step 9)

Code review is where surface-level issues (style, obvious bugs, anti-patterns) are caught. The agent performing the review should verify that the code adheres to stated style and safety criteria.

**Technique:** Provide the review agent with explicit criteria:

```markdown
## Code Review Criteria

Check each piece of code for:
1. [ ] Follows naming conventions (camelCase for vars, PascalCase for classes)
2. [ ] No console.log() or debug statements left in
3. [ ] Error handling: all promise rejections are caught
4. [ ] No N+1 queries in database code
5. [ ] Type safety: no `any` types without justification
6. [ ] Comments explain "why", not "what"

If a check fails, note it as an issue for remediation.
```

Without explicit criteria, review is unreliable. With them, a mid-tier model can consistently identify surface issues.

### During Integration Audit (Phase 2, Step 10)

This is the deep audit phase where system-level correctness is verified. The agent should run the full test suite and validate against the Integration Contract.

**Run full test suite.** Not just tests for the new feature — the entire test suite. This catches regressions where new code breaks previously-working functionality. Anthropic's guidance: "If your test suite takes more than 5 minutes, parallelize it or stratify it (fast tests first, slow tests after verification)."

**Verify Integration Contract.** Re-read the Integration Contract from the planning phase and confirm that every connection it describes actually exists in the code:
- New routes actually register in the router
- Database migrations actually run
- Component exports are actually used in the parent layout
- API responses match the documented shape

**Technique:** Create an Integration Contract verification checklist as part of the audit:

```markdown
## Integration Contract Verification

For each connection in the Integration Contract, confirm:
- [ ] File exists at path `<path>`
- [ ] Function/component is exported correctly
- [ ] Is actually called/imported by the code that depends on it
- [ ] API response shape matches documentation
- [ ] Database references match schema
```

The agent should run this checklist item-by-item, pointing to specific code locations where connections are verified.

---

## The Feedback Loop Pattern in Practice

A concrete example: implementing a new API endpoint.

**Generate (Code):** The agent writes the endpoint handler, validation schema, and tests.

**Verify (Test):** The agent runs the unit tests. Two fail:
- Test expecting `userId` in response gets `user_id` instead.
- Test for invalid input validation fails because validation middleware isn't applied.

**Fix:** The agent reads the test failures, identifies that the response field is incorrectly named and validation is missing. It:
1. Renames the response field to match the test expectation.
2. Adds the validation middleware to the route definition.

**Re-verify:** The agent runs tests again. All pass. It also runs linting (passes) and the build (succeeds).

**Done:** The agent moves to the next task and commits this code.

This entire loop happens within one session, without you intervening. The feedback loop lets the agent self-correct. Without it, the agent produces plausible-looking code that might not work, and you only discover the issues during your review.

---

## Test-Driven Self-Verification

One powerful pattern is inverting the typical development flow: write tests first, then implement until tests pass.

**Process:**
1. The agent reads the task specification and creates tests that would verify correct implementation.
2. The agent runs the tests (they fail — no implementation yet).
3. The agent writes code to make the tests pass, re-running tests after each change.
4. When all tests pass, the implementation is done by definition.

This is self-verification at its most powerful because the feedback loop is built into the implementation process itself. The agent can't declare "done" without passing tests — the tests are the ground truth.

**When to use:** For well-specified features where the acceptance criteria are clear. Not for research-heavy work or tasks requiring significant architectural exploration.

**Technique:** In your planning phase (Step 2), include a "Test Plan" section that lists the acceptance tests that will verify the feature works. Then instruct the agent: "Write tests first, implement until all tests pass."

---

## ReAct: The Verification Loop Pattern

The ReAct pattern (Reason + Act) is fundamentally a verification loop:

1. **Reason:** The agent reasons about the current state and what to do next.
2. **Act:** The agent takes an action (write code, run tests, grep for a file, etc.).
3. **Observe:** The agent observes the result of the action.
4. **Reason:** The agent reasons about what the observation means.

This loop repeats until the agent reaches its goal. The "Observe" phase is the verification step — the agent checks whether the action had the intended effect.

Claude Code and other modern agents implicitly implement ReAct: they generate code, run it, observe the test output or error messages, reason about what went wrong, and modify the code. The loop is the verification mechanism.

**To make ReAct work well:** Ensure the agent's tools provide clear, actionable feedback. When a test fails, the error message should clearly explain what went wrong. When a type check fails, the error should pinpoint the issue. Ambiguous feedback breaks the loop — the agent can't reason clearly if it doesn't understand what went wrong.

---

## Emerging Research: Vericoding and Formal Verification

MIT's research into "vericoding" — formal verification generated from natural language specifications — is beginning to influence how agents verify their own work. The long-term vision is agents that produce not just code but mathematical proofs that the code is correct.

**Current state (February 2026):** Still research. Not yet practical for production coding.

**Near-term impact:** Formal specifications (expressed in a specification language like TLA+ or Alloy) for critical components. Agents can verify their implementations against these specs automatically. This is beyond testing (which samples behavior) and approaches formal correctness.

**If your team cares deeply about correctness:** Invest in learning formal specification languages. Instruct agents to generate code that conforms to a formal spec. The gap between "code passes tests" and "code is provably correct" is significant for critical systems.

---

## Anti-Patterns in Self-Verification

**Declaring "done" without running tests.** The agent finishes implementation and says "complete" without running the test suite. This moves the verification burden entirely to you. Avoid by making tests non-optional.

**Verifying against training data instead of actual docs.** The agent says "I know the React API works like this" instead of reading the actual documentation. Avoid by explicitly instructing the agent to download docs and verify examples.

**Insufficient failure analysis.** A test fails, the agent fixes the code once, re-runs tests, and if they still fail, it gives up. Insufficient — real debugging sometimes requires multiple attempts. Instruct the agent to iterate up to 5 times before declaring failure.

**Trusting the agent's interpretation of test output.** Agents sometimes misread error messages or draw incorrect conclusions from test failures. Make error messages as explicit as possible. Include `__EXPECTED__` and `__ACTUAL__` sections in test failures so interpretation is unambiguous.

**Running only new feature tests.** The agent implements a feature, runs tests for that feature, and declares done without running the full test suite. This misses regressions. Insist on full test suite runs before feature completion.

**Verification loops that loop infinitely.** If the agent reaches 5 failed iterations, declare the task incomplete and surface it for human review rather than letting it loop forever.

---

## Practical: Writing Self-Verification into CLAUDE.md

Here's how to structure self-verification instructions in your project context file:

```markdown
## Self-Verification Protocol

Every implementation task must complete the following gates in order. Do not proceed to the next gate until the previous gate passes.

### Gate 1: Syntax & Type Safety
After writing code, run:
- `npm run type-check` (TypeScript)
- `npm run lint` (ESLint)

If these fail, read the error, fix the code, and re-run until they pass.

### Gate 2: Unit Tests
For the feature you just implemented, run:
- `npm run test -- --testPathPattern=<feature>`

All tests must pass. If tests fail:
1. Read the test failure message
2. Identify what the test expected vs. what it got
3. Modify the code to fix the issue
4. Re-run the test
5. Repeat until all tests pass (maximum 5 iterations)

### Gate 3: Integration & Smoke Tests
Run the full test suite:
- `npm run test` (all tests)
- `npm run build` (does the build succeed?)

Verify no regressions were introduced. If new tests fail, they are likely integration failures — debug by reading error messages and checking how the new code connects to existing code.

### Gate 4: Code Review
Apply self-review using the "Code Review Checklist" below. If you identify issues, fix them and re-run Gate 1–3.

Do not commit or declare a task complete until all four gates pass.

## Code Review Checklist
- [ ] No `console.log()` or `debugger` statements
- [ ] All error cases are handled
- [ ] No `any` types without justification
- [ ] Variable names are clear (no single letters except loop counters)
- [ ] Comments explain "why", not "what"
- [ ] No hardcoded values (use constants instead)
```

This externalized instruction makes verification explicit and repeatable. Every agent session reads it and knows exactly what verification looks like.

---

## Getting Started: Enabling Self-Verification

If you're not doing self-verification yet, start here:

1. **Write a "Self-Verification Gates" section in CLAUDE.md.** List the exact commands to run after implementation. Keep it simple: type-check, lint, test, build.

2. **Enable test-first development.** On your next feature, have the agent write tests first, then implement until tests pass. Experience the difference.

3. **Create a pre-flight check for plans.** Build a simple script that verifies file references, dependencies, and schema assumptions before implementation starts. Make it a mandatory gate.

4. **Run the full test suite, not just new tests.** Instruct the agent to always run the entire test suite, not just tests for the new feature. Catching regressions is worth the extra time.

5. **Give verification feedback loops.** When a test fails, ensure the error message is clear and specific. The agent's ability to self-correct depends on understanding *why* something failed.

6. **Document Integration Contract verification.** For features with explicit Integration Contracts, create a checklist that verifies each connection. Run it during the audit phase.

---

## The ROI of Self-Verification

The 2–3x quality improvement Cherny reports comes from two factors:

**Error catching before human review.** Most bugs are caught by the verification loop and fixed automatically. Fewer issues reach your review, reducing your review burden.

**Iteration compression.** Instead of: generate → you review → you identify issues → you request fixes → agent fixes → you review again, the flow becomes: generate → verify → fix → verify → done → you review once. The iteration count decreases dramatically.

**Research validates this.** Gartner's 2026 report notes that AI agents handling QA workloads (running tests, identifying issues, recommending fixes) are becoming increasingly common. The pattern generalizes: agents are increasingly reliable at catching and fixing errors when given a feedback loop.

The investment is minimal — mostly writing clear verification instructions in your CLAUDE.md. The payoff is substantial: higher-quality output, fewer review rounds, and more reliable code.

---

## Sources

### Primary Research

- Boris Cherny, "Give Claude a way to verify its work" (Claude Code workflow, 2026)
- OpenObserve, "Healer Agent: Autonomous Test Debugging and Remediation" (2025)
- Anthropic, "Agent evaluation guidance: Long-horizon task planning" (2025)
- Anthropic, "Effective harnesses for long-running agents" (2025)

### Supplementary

- Gartner, "AI in QA: Agents, Testing, and Quality Assurance" (2026)
- MIT CSAIL, "Vericoding: Automatic Formal Verification from Natural Language" (2025)
- HumanLayer, "Feedback loops in agentic workflows" (2025)
- Qodo, "State of AI Code Quality in 2025"
- MIT OpenCourseWare, "ReAct: Synergizing Reasoning and Acting in Language Models"
