# Verification, Review, and Audit

**Author:** Glenn Clayton, Fieldcrest Ventures
**Version:** 2.0 — March 2026
**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Agentic Workflow Guide](agentic-workflow-guide.md) · [Testing Strategy](Testing-Strategy.md) · [Integration Contracts](Integration-Contracts.md) · [Model Routing](Model-Routing.md) · [Deterministic Sandwich](Deterministic-Sandwich.md)

---

## Overview

Verification, validation, and review form a pipeline that runs throughout the agentic development lifecycle. The goal is catching issues early, with escalating severity and expertise at each gate.

The pipeline has three layers:

1. **Self-Verification:** The agent generating output tests it immediately (within the same session). Catches obvious bugs before human review.
2. **Validation Passes:** Fresh-context review of specifications and integration. Run in separate sessions to avoid confirmation bias.
3. **Review and Audit:** Surface-level and system-level correctness checks post-implementation.

Together, these reduce the defect rate of AI-generated code by 2–3x (Boris Cherny, Claude Code) and improve validation accuracy by 23–31% (Anthropic research).

---

## Part 1: Self-Verification (Generate → Verify → Fix → Re-Verify)

### What Is Self-Verification?

Self-verification is the discipline of giving AI agents feedback loops so they can confirm their own work before you review it. Instead of asking "Did you finish?" and hoping the answer is accurate, the agent runs tests, checks file references, validates type safety, or executes the code it wrote — then reports back with evidence.

Boris Cherny's research supports the impact: agents given verification loops produce output that's 2–3x higher quality than agents without them. The reason is straightforward: without verification, the agent declares "done" based on its own assessment. With a feedback loop, it declares "done" based on *evidence*.

### The Core Pattern

Every self-verification workflow follows this loop:

1. **Generate:** The agent produces output — a plan, code, documentation, or test.
2. **Verify:** The agent uses tools or test harnesses to confirm the output is correct.
3. **Fix:** If verification fails, the agent analyzes the failure and produces a correction.
4. **Re-verify:** The agent runs the verification again to confirm the fix worked.

This loop can repeat multiple times. OpenObserve's "Healer" agent demonstrates the pattern at scale: a specialized agent that runs tests after every implementation task, diagnoses test failures by reading error messages and examining the code, produces a fix, re-runs the tests, and iterates up to five times automatically. The result is a dramatic reduction in human intervention — most bugs are self-healing.

**The critical insight:** The feedback loop compresses the iteration cycle. Without it, the agent produces output, you review it, identify issues, and ask for fixes — a human-in-the-loop delay per iteration. With it, the agent identifies and fixes most issues *automatically*, within the same session. You only step in when the feedback loop exhausts its ability to correct (usually 4–5 attempts).

### Self-Verification Across Development Phases

#### During Research

When researching how to implement a feature, the agent shouldn't rely on training data. Instead:

- **Verify documentation currency:** Download the actual API docs rather than reciting training knowledge. The agent should be able to say "I read the Stripe API docs from March 2026" rather than "I recall that Stripe v3 works like this."
- **Verify file existence and content:** Before planning around existing code, grep for key files, read their structure, and confirm they contain what the plan assumes.
- **Verify examples work:** If the docs include example code, attempt to run or parse them to confirm they're correct. This catches outdated docs.

Include a "Verification Checklist" in canonical documents:
```
Research Verification Checklist:
- [ ] Downloaded and verified current API docs
- [ ] Confirmed existing implementation files are where the plan references them
- [ ] Checked that examples in docs match current behavior
```

#### During Planning

Planning is where most agent errors occur — invalid file references, misunderstanding of existing code structure, or oversimplifying integration points. Self-verification during planning prevents cascading errors during implementation.

- **Verify file references exist:** Before the plan references `src/components/Button.tsx`, confirm that file exists.
- **Verify schema matches:** If the plan assumes a database table has a `user_id` column, read the migration file to confirm.
- **Verify Integration Contract completeness:** Check that each connection listed in the [Integration Contracts](Integration-Contracts.md) actually exists in the codebase.

Build a pre-flight check tool that validates the plan before implementation starts:

```
Pre-Flight Check:
- [ ] All referenced files exist
- [ ] All referenced database tables exist in schema
- [ ] All API endpoints in Integration Contract exist
- [ ] All dependencies listed are in package.json
- [ ] Build succeeds (smoke test)
```

Make this check non-optional. If it fails, planning wasn't complete enough to proceed.

#### During Implementation

This is where self-verification has the highest impact.

- **Run unit tests after writing:** After implementing a function or module, write tests that exercise that code, then run them. If tests fail, debug and fix, then re-run.
- **Run linting and type checking:** After writing code, run your project's linter and type checker (ESLint, TypeScript, Mypy, etc.). Treat linting failures as blocking.
- **Run component tests:** For UI code, write tests that confirm the component renders correctly and responds to user interaction.

In your [CLAUDE.md](agentic-workflow-guide.md), include a section titled "Self-Verification Gates":

```markdown
## Self-Verification Gates

After implementing any feature or fix, run:

1. `npm run type-check` (TypeScript type safety)
2. `npm run lint` (linting and formatting)
3. `npm run test -- --testPathPattern=<feature>` (unit tests)
4. `npm run build` (does the build succeed?)

Do not declare a task complete until all gates pass.
```

#### During Code Review

Code review is where surface-level issues (style, obvious bugs, anti-patterns) are caught. Provide the review agent with explicit criteria:

```markdown
## Code Review Criteria

Check each piece of code for:
1. [ ] Follows naming conventions (camelCase for vars, PascalCase for classes)
2. [ ] No console.log() or debug statements left in
3. [ ] Error handling: all promise rejections are caught
4. [ ] No N+1 queries in database code
5. [ ] Type safety: no `any` types without justification
6. [ ] Comments explain "why", not "what"
```

Without explicit criteria, review is unreliable.

#### During Integration Audit

This is the deep audit phase where system-level correctness is verified.

- **Run full test suite:** Not just tests for the new feature — the entire test suite. This catches regressions.
- **Verify Integration Contract:** Re-read the Integration Contract from planning and confirm every connection exists in the code.

Create an Integration Contract verification checklist:

```markdown
## Integration Contract Verification

For each connection in the Integration Contract, confirm:
- [ ] File exists at path `<path>`
- [ ] Function/component is exported correctly
- [ ] Is actually called/imported by code that depends on it
- [ ] API response shape matches documentation
- [ ] Database references match schema
```

### Test-Driven Self-Verification

One powerful pattern inverts typical development: write tests first, then implement until tests pass.

1. The agent reads the task specification and creates tests that would verify correct implementation.
2. The agent runs the tests (they fail — no implementation yet).
3. The agent writes code to make the tests pass, re-running tests after each change.
4. When all tests pass, implementation is done by definition.

This is self-verification at its most powerful because the feedback loop is built into implementation itself.

### Anti-Patterns

- **Declaring "done" without running tests.** Make tests non-optional.
- **Verifying against training data instead of actual docs.** Explicitly instruct the agent to download docs and verify examples.
- **Insufficient failure analysis.** Instruct the agent to iterate up to 5 times before declaring failure.
- **Trusting the agent's interpretation of test output.** Make error messages explicit with `__EXPECTED__` and `__ACTUAL__` sections.
- **Running only new feature tests.** Insist on full test suite runs.
- **Verification loops that loop infinitely.** After 5 failed iterations, surface for human review.

---

## Part 2: Validation Passes (Fresh-Context Review)

### Why Fresh Context Matters

When an AI agent drafts a feature plan, it accumulates reasoning: why certain architectural choices were made, what constraints were considered, what alternatives were rejected. This reasoning lives in the context window. When asked "does this plan work?", the agent can't fully step back — it inherits all that prior context, which acts as a lens through the entire validation.

**The solution:** Run validation in a separate session or invocation with minimal context history. The validator reads the specification and the output. That's all. Clean slate.

Anthropic's testing showed a **23–31% improvement in validation accuracy** when using fresh-context passes compared to in-session validation.

### Two Types of Validation

#### Requirements Fidelity

**Question:** Does the plan satisfy the specification?

Requirements fidelity validation checks:
- Completeness: Does the plan address every requirement in the spec?
- Correctness of interpretation: Are ambiguous spec requirements interpreted correctly?
- Scope boundaries: Does the plan include anything not requested? (Scope creep is common.)
- Acceptance criteria: Are success criteria clear and measurable?
- Edge cases: Are edge cases mentioned in the spec explicitly handled?

**Who should run this pass:** Anyone who understands the original specification — ideally the person who wrote it, or a fresh agent with the spec in context. This is less about technical judgment and more about intent alignment.

#### Architectural Coherence

**Question:** Does the plan integrate correctly with the existing codebase?

Architectural coherence validation checks:
- Pattern alignment: Does the plan follow established architectural patterns?
- Integration points: Are all connections to existing code explicitly specified? (Integration Contracts pattern.)
- No orphaned code: Will the new code be connected to the rest of the system?
- Dependency direction: Does new code depend on existing code, not the reverse?
- Design consistency: Does the plan introduce patterns that contradict established ones?

**Who should run this pass:** A senior engineer familiar with the existing codebase, or a fresh agent loaded with architectural context. This requires deep knowledge of how the system works.

### The LLM-as-Judge Pattern

For subjective criteria — code style, readability, adherence to architectural patterns, design coherence — the **LLM-as-Judge** pattern has emerged as the dominant approach.

Instead of writing explicit rules (which always fail on edge cases), use a second LLM to evaluate the output against stated criteria. The judge receives the specification, code or plan, and explicit evaluation rubric, then scores the output.

#### How to Structure a Validation as LLM-as-Judge

**1. Define explicit evaluation criteria:**

```
Evaluate the plan against these criteria:
- Completeness (0-5): Does the plan address all requirements?
- Clarity (0-5): Are acceptance criteria unambiguous?
- Integration (0-5): Are connection points to existing code explicit?
- Scope (0-5): Does the plan include only requested functionality?
- Architecture (0-5): Does the plan follow established patterns?
```

**2. Frame the judge as a senior reviewer:** "You are a staff engineer reviewing a junior's pull request." This primes appropriate rigor.

**3. Provide both the spec and the output:** The judge needs both as baseline and assessment.

**4. Ask for scores and specific feedback:**
- Score on each dimension
- Specific issues or concerns
- Severity: production problems or polish?

**5. Optional: include passing examples** to help calibrate standards.

#### Effectiveness and Cost

LLM-as-Judge benchmarks:
- **Alignment with human preferences:** 78–82% agreement with human expert reviews
- **Cost vs. human review:** 500–5,000x cheaper than a human staff engineer for 30 minutes
- **Model performance:**
  - Frontier models (Opus, Sonnet): 78–82% human agreement
  - Reasoning models (o1, QwQ): 81–86% human agreement — drastically outperform
  - Fine-tuned judges (Prometheus, AceCodeRM): Performed poorly; specialization hurt generalization

For production use: use a reasoning model if cost allows. If not, use frontier models, or use LLM-as-Judge for only 5–10% of critical requests.

### Multi-Pass vs. Single-Pass Validation

A common mistake: running one "validation pass" that tries to check everything — requirements, architecture, code quality, integration.

**Single-pass validation (poor):**
```
Evaluate this plan:
- Requirements coverage
- Architecture alignment
- Integration completeness
- Code quality
- Edge case handling
```

The judge's attention is divided. Issues get missed because of context-switching.

**Multi-pass validation (better):**
```
Pass 1: Does this plan satisfy the specification?
Pass 2: Does this plan integrate with the existing codebase?
Pass 3: Is the code quality sufficient?
```

Specialized judges perform better and are cheaper in total cost per dimension. Andrej Karpathy and the Cursor team advocate for this approach. The ~20% quality improvement outweighs modest cost increase.

### LoopAgent Pattern

Some teams have extended multi-pass into a full loop: a Researcher produces a plan, a Judge evaluates it, and if the plan doesn't pass, the Researcher revises and resubmits.

```
Loop:
  1. Researcher: Draft plan
  2. Judge: Score against criteria
  3. If score < threshold:
     a. Researcher: Revise based on feedback
     b. Go to step 2
  4. If score >= threshold: Exit loop
```

Works well for high-stakes decisions (architectural choices, security-critical reviews). It's overkill for routine code generation. Most plans pass on 1–2 iterations.

### Agent-as-Judge vs. LLM-as-Judge

**LLM-as-Judge:** A single LLM invocation that reads the spec and output and returns a score.

**Agent-as-Judge:** An agent that systematically investigates the plan — checking file references, exploring patterns, running code snippets — then returns a detailed audit.

Agent-as-Judge is more expensive and slower, but catches subtle integration issues. Pattern: use LLM-as-Judge for quick screening, escalate to Agent-as-Judge for failing or borderline plans.

### Targeted Validation (AgentAuditor Pattern)

Research revealed that auditing every decision is less effective than auditing only decision-critical divergences.

Rather than scoring all dimensions equally, identify the 2–3 highest-risk decisions (the ones most expensive to fix later) and focus audit on those.

**Example:** For a payment system feature, the critical decision is "how do we handle failed transactions?" Audit that rigorously. Less critical: "should we use a Timeout struct or an integer?" Don't audit that.

**Results:** Targeted approach outperformed generic judging by ~9 points. The reason: focused attention, no noise, shorter context window.

**How to apply:** Mark criteria as decision-critical or polish. Instruct the judge: "Audit these three dimensions thoroughly. Skim the rest."

---

## Part 3: Code Review (Step 9 — Surface-Level Correctness)

### What to Check

- **Style and formatting:** Is code consistent with project conventions? Naming, indentation, line lengths?
- **Obvious bugs:** Off-by-one errors, null pointer dereferences, unreachable code, logic inversions?
- **Anti-patterns:** Forbidden patterns documented in your project context?
- **Type safety:** Are types declared? Are casts justified? Are unchecked type assertions present?
- **Error handling:** Are errors caught? Are edge cases handled? Are error messages clear?
- **Documentation:** Are public functions documented? Are complex blocks commented?

### Model Tier

**Mid-tier.** Code review is pattern matching — comparing code against a checklist of known issues. A mid-tier model (Claude 3.5 Sonnet or equivalent) has sufficient pattern-matching capability and costs 40–50% less than frontier. Reserve frontier models for deep reasoning tasks.

### Prompt Template Structure

```
You are a code reviewer for [PROJECT].

CODE STANDARDS (from CLAUDE.md):
[Copy relevant sections from your project context]

ANTI-PATTERNS TO FLAG:
[List forbidden patterns, architectural violations, documented mistakes]

RECENT CODE REVIEW FEEDBACK:
[Last 3–5 issues found in this codebase, so the reviewer knows the quality bar]

---

Review the following code for surface-level issues:
- Style and naming consistency
- Obvious bugs and logic errors
- Anti-pattern violations
- Type safety issues
- Error handling gaps
- Missing or unclear documentation

CODE:
[Code to review]

For each issue, provide:
1. Line number(s)
2. Category (style, bug, anti-pattern, type, error handling, docs)
3. Severity (critical, major, minor)
4. Description and suggested fix

Format as a structured issue report.
```

### Interpreting Results

- **Critical issues:** Prevent code from working or expose major bugs. Must be fixed before audit.
- **Major issues:** Violate architectural patterns or introduce subtle bugs. Should be fixed before audit.
- **Minor issues:** Style inconsistencies or incomplete documentation. Can be addressed concurrently with audit.

---

## Part 4: Deep Audit (Step 10 — System-Level Correctness)

Deep audit is run *after* code review issues are remediated. It answers: "Does the code work correctly as a system?"

### What to Check

- **Cross-file integration:** Do new components connect to existing code correctly? Are all connections bidirectional and complete?
- **Async and concurrency bugs:** Are promises awaited? Are race conditions prevented? Are locks held as expected?
- **Resource management:** Are file handles, connections, and memory properly released? Are cleanup operations guaranteed?
- **Security:** Are inputs validated? Are secrets properly handled? Are dependencies audited for known vulnerabilities?
- **Performance:** Are expensive operations cached? Are database queries optimized? Are n+1 queries prevented?
- **Integration Contract verification:** Does the implementation match the contract specified in the plan? Are all promised connections present and correct?

### Model Tier

**Frontier.** Deep audit requires understanding cross-file behavior, async semantics, and system-level correctness — tasks demanding deep reasoning. Use Claude Opus 4.6 or equivalent. This is where a more expensive model provides genuine value.

### Mandatory: Integration Contract Verification

Every audit must include a dedicated Integration Contract section. This is non-negotiable.

The Integration Contract verification checks:
- Files modified: Do the plan and implementation agree on which files change?
- Files created: Are all promised new files present and connected?
- API contracts: Do new/modified endpoints match specified signatures?
- Data model changes: Are migrations written? Is the schema what was promised?
- Component interfaces: Do new UI components connect to navigation, state, and parent layouts?
- Dependencies: Does implementation depend only on what was planned? Are there unexpected coupling points?

### Prompt Template Structure

```
You are a deep auditor for [PROJECT]. Your job is system-level correctness.

PROJECT CONTEXT:
[Canonical documentation for affected systems]

ARCHITECTURAL CONSTRAINTS:
[Integration patterns, concurrency models, security requirements from CLAUDE.md]

FEATURE PLAN:
[The feature plan and Integration Contract]

---

Audit the following code for system-level correctness:

CODE:
[Full implementation]

EXISTING CODEBASE CONNECTIONS:
[Paths to files that this code integrates with]

Perform a deep audit across:
1. Cross-file integration
2. Async and concurrency correctness
3. Resource management
4. Security
5. Performance
6. Integration Contract verification (MANDATORY)

For Integration Contract verification, check:
- Files modified: Do implementation changes match the plan?
- Files created: Are all promised new files present and integrated?
- API contracts: Do endpoint signatures match the plan?
- Data model: Does the schema match? Are migrations included?
- Component connections: Do UI components integrate correctly?
- Dependencies: Are unexpected couplings introduced?

For each finding, provide:
1. Category (integration, async, resource, security, performance, contract)
2. Severity (critical, major, minor)
3. Location (file, function)
4. Description and recommendation

Format as a structured issue report. Conclude with a summary of Integration Contract alignment.
```

### Interpreting Results

- **Integration Contract misalignment:** The implementation doesn't match the plan. This is a hard blocker.
- **Critical issues:** Security vulnerabilities, unhandled async bugs, resource leaks, blocking integration problems. Must be fixed before merging.
- **Major issues:** Performance problems, incomplete error paths, subtle concurrency issues. Should be fixed before merging.
- **Minor issues:** Code clarity, non-critical optimizations. Can be deferred if time is critical.

---

## Part 5: Council of Sub Agents

Code review and deep audit catch different failure classes. A single "review everything" pass forces one agent to context-switch between incompatible cognitive modes, degrading performance on both.

Addy Osmani's data illustrates the cost:
- PRs 18% larger with AI-assisted development
- 24% more incidents per PR
- 30% increase in change failure rate

The solution is specialized agents with narrowly scoped prompts:

- **Code Reviewer:** Mid-tier model, focused on style and obvious bugs. Reports all findings.
- **Deep Auditor:** Frontier model, focused on integration and system correctness. Reports blocking issues and Integration Contract alignment.
- **Security Auditor:** Specialized agent with security context (OWASP, CWEs, framework vulnerabilities). Reports security-specific findings.
- **Performance Auditor:** Specialized agent with performance context (benchmarks, bottlenecks, caching patterns). Reports performance regressions or opportunities.

Each agent operates independently with clean context, then results are aggregated. The developer sees a unified issue report, but each agent was focused on what it does best.

Qodo's research on 15+ specialized review agents shows this approach catches **23% more issues** than a single "review everything" agent, while maintaining faster review times.

---

## Part 6: Over-Engineering Detection

A critical but often overlooked verification is detecting features that exceed the plan — unnecessary abstractions, premature generalization, gold-plating, or scope creep beyond what was requested.

### What to Check for Over-Engineering

- **Unnecessary abstractions:** Does the code introduce builder patterns, factories, or inheritance hierarchies that serve only one current use case?
- **Premature generalization:** Are parameters or configuration options added "in case we need them later" without current requirements?
- **Gold-plating:** Does the feature include polish, animations, or edge cases not mentioned in the acceptance criteria?
- **Scope creep:** Does the implementation include related features or components not specified in the plan?
- **Over-sophisticated solutions:** Did the agent choose an advanced approach (e.g., recursive algorithms, complex state machines) when a simple approach would suffice?

### Prompt for Over-Engineering Detection

```
You are an auditor checking for over-engineering.

FEATURE PLAN:
[The feature plan]

ACCEPTANCE CRITERIA:
[Explicit list of what "done" means]

CODE:
[Full implementation]

Identify instances where the code goes beyond the plan:
1. Abstractions that serve only one current use case
2. Configuration or parameters added "for future extensibility"
3. Polish or edge cases not in the acceptance criteria
4. Related features included but not requested
5. Unnecessarily sophisticated solutions to simple problems

For each finding:
- Line number(s) or file
- Description
- What the plan actually required
- What was added beyond the plan
- Impact: is this a technical debt risk, or just unnecessary complexity?

Provide recommendations for simplification.
```

The goal isn't perfection — it's matching scope. Over-engineering is technical debt that creates maintenance burden.

---

## Part 7: Mapping to the Agentic Workflow

These verification concepts map directly to the agentic workflow:

### In `/code` (Implementation Subagent)

The `/code` subagent implements self-verification automatically:

1. Write code
2. Run the Self-Verification Gates (type-check, lint, test, build)
3. If any gate fails, fix and re-run
4. When all gates pass, the implementation is ready for review

### In `/verify` (Verification Subagent)

The `/verify` subagent runs after `/implement` completes. It performs:

1. **Code Review:** Mid-tier review pass on style, obvious bugs, anti-patterns
2. **Deep Audit:** Frontier-tier audit on integration and system correctness, including mandatory Integration Contract verification
3. **Over-Engineering Detection:** Check for scope creep or unnecessary abstractions
4. **Report:** Structured issue report with severity and recommendations

### In `/capture-lesson` (Lesson Capture)

After `/verify`, any patterns of issues (e.g., "async bugs consistently missed") become suggestions for CLAUDE.md or `docs/lessons.md`.

### In `/pre-commit` (Quality Gate)

The `/pre-commit` quality gate runs:

1. Lint
2. Type check
3. Full test suite
4. Diff review (surface-level code quality)

This is the final checkpoint before committing to main.

---

## Part 8: Anti-Patterns

### In Self-Verification

- **Declaring "done" without running tests.** Move verification burden to you.
- **Verifying against training data instead of actual docs.** Explicitly instruct the agent to download docs.
- **Insufficient failure analysis.** Instruct agent to iterate up to 5 times before giving up.
- **Trusting the agent's interpretation of test output.** Make error messages explicit with `__EXPECTED__` and `__ACTUAL__`.
- **Running only new feature tests.** Insist on full test suite runs.
- **Infinite verification loops.** After 5 failed iterations, surface for human review.

### In Validation Passes

- **Validating in the same context.** Run validator in separate session with fresh context.
- **Vague evaluation criteria.** Be specific about dimensions and scoring thresholds.
- **Over-reliance on single-pass review.** Use specialized passes for different concerns.
- **Ignoring judge feedback.** If the judge flags an issue, investigate before dismissing.
- **Validating in isolation.** A plan can satisfy the spec and still fail to integrate.
- **Using frontier models for every judge.** Reserve reasoning models for high-stakes decisions.

### In Code Review and Audit

- **Single "review everything" pass.** One agent trying to do surface and system-level simultaneously.
- **Review without criteria.** Providing standards, anti-patterns, and architectural context is mandatory.
- **Skipping audit for "simple" changes.** Even simple features can have integration bugs.
- **Integration Contract review outside the audit pass.** It must be a mandatory section in every audit report.
- **No context about recent issues.** Recent review findings become part of the next reviewer's context.
- **Remediation without re-review.** After issues are fixed, run the full review-audit cycle again.

---

## Getting Started: Five Things You Can Do Today

1. **Add self-verification gates to CLAUDE.md.** List the exact commands to run after implementation (type-check, lint, test, build). Make them non-optional.

2. **Run one fresh-context validation per feature plan.** After drafting a plan, open a new session. Load the spec and plan. Ask: "Does this satisfy the spec? Anything missing or misinterpreted?" This catches ~25% of issues before implementation.

3. **Define explicit evaluation criteria.** For any validation, write down dimensions that matter (completeness, clarity, integration, scope). Avoid vague criteria like "quality."

4. **Separate code review and deep audit.** Don't conflate surface-level and system-level checking. Run them as separate passes with different model tiers.

5. **Create an Integration Contract verification checklist.** For every feature with an Integration Contract, verify each connection exists in the code. Make this mandatory in the audit report.

---

## Sources

### Primary Research

- Boris Cherny, "Give Claude a way to verify its work" (Claude Code workflow, 2026)
- OpenObserve, "Healer Agent: Autonomous Test Debugging and Remediation" (2025)
- Anthropic, "Effective harnesses for long-running agents" (2025)
- Anthropic, "Agent evaluation guidance: Long-horizon task planning" (2025)
- Anthropic, "2026 Agentic Coding Trends Report" (2026)
- Anthropic, "[Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)" (2025)
- OpenAI, "[LLM-as-Judge: A Survey of LLMs as Evaluators](https://arxiv.org/abs/2309.15025)" (2023)
- MetaEval, "[When LLMs Judge LLMs: Aggregated Opinion and Consensus](https://arxiv.org/abs/2404.08666)" (2024)
- AgentAuditor, "[Targeted Auditing for Agentic Systems](https://arxiv.org/abs/2402.12141)" (2024)
- Google DeepMind, "[Evaluating LLMs for Code Quality](https://research.google/pubs/evaluating-llms-for-code-quality/)" (2025)

### Supplementary

- Gartner, "AI in QA: Agents, Testing, and Quality Assurance" (2026)
- MIT CSAIL, "Vericoding: Automatic Formal Verification from Natural Language" (2025)
- HumanLayer, "Feedback loops in agentic workflows" (2025)
- HumanLayer, "Multi-agent code review patterns" (2025)
- Qodo, "State of AI Code Quality in 2025"
- MIT OpenCourseWare, "ReAct: Synergizing Reasoning and Acting in Language Models"
- Addy Osmani, "[The 80% Problem in Agentic Coding](https://addyosmani.com/blog/the-80-percent-problem/)" (2025)
- Boris Cherny, Claude Code workflow insights and validation strategies (2026)
- Cursor Engineering, "[How Cursor Validates Generated Code](https://docs.cursor.sh/advanced/validation-patterns)" (2025)
- CodeJudgeBench, "[Execution-Free Code Evaluation](https://github.com/codefuse-ai/CodeJudgeBench)" (2024)
- Taxonomy-Guided Fault Localisation, "[Categorizing Errors by Severity](https://arxiv.org/abs/2312.12840)" (2023)
- Snyk DeepCode AI, "Security-focused code review research" (2025–2026)
