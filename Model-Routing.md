# Model Routing

**Parent:** [[AI-Coding-Best-Practices]]
**Related:** [[Validation Passes]] · [[Development Loop — Full Reference]] · [[Context Engineering]] · [[Deterministic Sandwich]] · [[Review and Audit]]

---

## What Is Model Routing?

Model routing is the discipline of assigning the right model tier to each development task based on the cognitive complexity of that task and the cost-quality tradeoff that makes sense for your workflow.

In 2026, the frontier model (Claude Opus 4.6) costs $30–60 per million tokens. A mid-tier model (Claude Sonnet 4.6) costs $10–15/M. A lightweight model costs $0.50–2/M. The cost difference is 60–300x. The quality difference is real but selective — most production workloads perform identically on mid-tier for their specific tasks.

The question isn't "which model is best?" It's "which model is necessary for this specific task, given the cost and the failure modes if something goes wrong?"

---

## The Three Tiers: A Quick Reference

### Frontier Tier: Claude Opus 4.6
**When to use:** Reasoning, architectural decisions, deep analysis, validation, complex problem-solving

**Cost:** $30–60/M tokens
**What it's best at:** Novel problems with no established pattern. Multi-step reasoning where a missed subtlety compounds into expensive rework. Architectural decisions that affect future features. Validation passes where confirmation bias is the real failure mode.

**In the Development Loop:** Steps 1, 2, 3, 4, 6, 10 (research, drafting, validation, deep audit)

---

### Mid-Tier: Claude Sonnet 4.6
**When to use:** Code generation, code review, build plans, remediation

**Cost:** $10–15/M tokens
**What it's best at:** Pattern-following execution. Writing code to a detailed spec. Surface-level review. Remediating known issues. Building from validated plans.

**What changed in 2026:** Sonnet 4.6 now beats prior-generation frontier models (Opus 4.0) in many agentic benchmarks. The gap for real-world coding tasks is so small that the decision to use mid-tier is rarely about capability — it's about cost.

**In the Development Loop:** Steps 5, 8, 9, 12, 13 (build planning, implementation, code review, E2E testing, remediation)

---

### Automated Tier: No LLM
**When to use:** Deterministic checks, validation, linting, testing

**Cost:** ~$0 (you're running code, not querying a model)
**What it's best at:** Binary pass/fail checks. Anything that has a right answer independent of human judgment.

**In the Development Loop:** Steps 7, 15 (pre-flight checks, smoke tests)

---

## The 2026 Pricing Landscape

Understanding the economics makes model selection obvious in most cases.

| Tier | Cost per M tokens | Example task | Typical cost per invocation |
|------|---|---|---|
| Frontier (Opus 4.6) | $30–60 | Research + planning a feature | $0.50–2.00 |
| Mid-tier (Sonnet 4.6) | $10–15 | Implement a task from a plan | $0.05–0.30 |
| Lightweight (Haiku/Sonnet 4.5) | $0.50–2 | Code review or linting | $0.001–0.01 |
| Automated | $0 | Unit tests, git commit, linting | $0.00 |

The 60–300x cost difference is real. On a week-long project with 50 features, using frontier for everything costs ~$250–500 in API calls. Using tiered routing costs ~$50–100. The quality difference for pattern-following tasks is negligible.

---

## Task-to-Tier Mapping: The 15-Step Development Loop

The parent guide's Development Loop has 15 steps. Here's which tier handles each and why.

### Phase 1: Plan

| Step | Task | Tier | Why | Typical Cost |
|------|------|------|-----|---|
| 1 | Research | Frontier | Discovering strategy, edge cases, unknowns. Needs deep reasoning. Missed details cascade. | $0.50–1.50 |
| 2 | Draft Plan | Frontier | Architectural decisions. One wrong choice breaks downstream work. | $0.20–0.60 |
| 3 | Validate Reqs | Frontier (fresh) | Confirmation bias. Fresh context prevents "yeah that looks right." | $0.15–0.40 |
| 4 | Validate Integration | Frontier (fresh) | Architectural coherence check. Fresh context critical. | $0.15–0.40 |
| 5 | Generate Build Plan | Mid-tier | Pattern execution. Given a validated plan, decompose into tasks. | $0.05–0.15 |
| 6 | Double-Check Build | Frontier | Final plan validation before expensive implementation. | $0.10–0.30 |

**Total for Phase 1:** ~$1.50–3.75 using frontier for high-stakes reasoning, mid-tier for execution.

### Phase 2: Build

| Step | Task | Tier | Why | Typical Cost |
|------|------|------|-----|---|
| 7 | Pre-Flight Check | Automated | Do files exist? Dependencies installed? Binary check. | $0.00 |
| 8 | Implement | Mid-tier | Write code per spec. Pattern following. Run unit tests. | $0.30–1.00 |
| 9 | Code Review | Mid-tier | Surface-level: style, obvious bugs. | $0.05–0.15 |
| 10 | Deep Audit | Frontier | System-level correctness. Cross-file integration. Must verify Integration Contract. | $0.20–0.50 |
| 11 | Remediation | Mid-tier | Fix known issues. Pattern-following. | $0.05–0.20 |

**Total for Phase 2:** ~$0.85–2.00 using mid-tier for execution, frontier for integration audit.

### Phase 3: Verify & Ship

| Step | Task | Tier | Why | Typical Cost |
|------|------|------|-----|---|
| 12 | Full E2E Tests | Mid-tier | Orchestrate test runs. Triage failures. | $0.05–0.15 |
| 13 | Final Remediation | Mid-tier | Fix E2E issues. Verify with full suite. | $0.05–0.15 |
| 14 | Commit + Update Docs | Automated + Mid-tier | Git operations automated, docs updated by mid-tier. | $0.02–0.10 |
| 15 | Smoke Test | Automated | Does app start? Tests pass? Binary gate. | $0.00 |

**Total for Phase 3:** ~$0.12–0.40 using automation and mid-tier.

**Full feature (all 15 steps):** ~$2.50–6.15 using tiered routing vs. ~$4.50–9.00 if you use frontier for everything. **That's a 40–50% cost reduction.**

---

## The "Always Frontier" Counterargument

Boris Cherny (creator of Claude Code) doesn't use tiered routing. He uses Opus for everything — research, planning, coding, review, everything. His reasoning is worth understanding because it's not wrong; it's just optimizing for a different constraint.

Cherny's argument: "The bottleneck isn't token cost. It's human correction time. When I use a weaker model, I spend 30 minutes trying to explain why the code is wrong or reviewing something I shouldn't have had to review. When I use Opus, I spend 5 minutes reviewing obviously-correct code. The time savings pay for the token cost 100x over."

**This is empirically true for:**
- Solo developers working directly with the AI (no async pipeline)
- Tasks where human review happens in the same session
- Workflows where context-switching between model behaviors is disruptive
- Projects where the developer's hourly rate >> API costs

**This is empirically false for:**
- Automated pipelines (no human in the loop for most tasks)
- High-volume projects (50+ features, batch processing)
- Cost-sensitive deployments (SaaS companies with thousands of developers)
- Teams where the ratio of senior engineers to features is low

**The practical take:** If you're a solo senior developer using Claude Code interactively, Cherny's approach might be better for your throughput even if it costs more per feature. If you're running an automated development pipeline, tiered routing wins decisively.

---

## When Tiered Routing Wins vs. When "Always Frontier" Wins

### Tiered Routing Wins When:

1. **The pipeline is automated or async.** No human waiting in the loop means "it just works" matters less than "it worked and cost $3 instead of $8."
2. **Failure modes are well-understood.** You know which tasks actually need frontier-level reasoning and which are pattern-following.
3. **Volume is high.** 50+ features per project, multiple projects, batch processing. The 40–50% cost savings compounds.
4. **Cost is a hard constraint.** Startups, cost-conscious enterprises, or developers paying for API calls personally.
5. **Context quality is high.** With excellent specs and plans, mid-tier doesn't need babysitting.

### "Always Frontier" Wins When:

1. **The developer is in the loop.** Interactive sessions where the human is watching and able to correct mid-task.
2. **The developer is senior.** Experienced engineers can task-switch between model behaviors and aren't slowed down by weaker output.
3. **Correctness feedback is immediate.** The developer sees code execute right away and catches issues instantly.
4. **Throughput per hour is the metric.** Calendar time matters more than API cost.

---

## The 57% Cost Reduction Case Study

Infralovers (a design-focused development studio) profiled their agentic workflow and found they were using frontier models for everything. They implemented tiered routing as described here and achieved **57% cost reduction with zero quality regression**.

The key finding: **they were using frontier models for deterministic checks.**

**Before routing:**
- Pre-flight checks: Opus
- Code review: Opus
- Git commits: Opus
- Build plan decomposition: Opus
- **Cost per feature:** $8.50

**After routing:**
- Pre-flight checks: Automated (bash scripts)
- Code review: Sonnet 4.6
- Git commits: Automated (conventional commits hook)
- Build plan decomposition: Sonnet 4.6
- Research + validation + audit: Opus
- **Cost per feature:** $3.60

**The biggest win:** Automating the pre-flight check. Checking whether files exist and dependencies are installed isn't a problem that benefits from language models. They wrote a simple shell script — 50 lines — that caught the same issues at zero cost.

The lesson: **Look for tasks where you're paying for an LLM to do something deterministic.** Linting, formatting, basic file validation, commit message validation — these have right answers. Automate them. Save the LLM for reasoning.

---

## Sonnet 4.6: Closing the Gap

One reason tiered routing works so well in 2026 is that Sonnet 4.6 is genuinely close to Opus 4.6 on real-world coding tasks.

Anthropic's internal benchmarks:
- **Architecture decisions (research/validation):** Opus ~10% better
- **Code generation from spec:** Sonnet ~2% worse on average (sometimes better)
- **Code review (finding bugs):** Opus ~8% better
- **Remediation (fixing known issues):** Sonnet ~1% worse

The frontier advantage is real for complex reasoning (architecture, validation) but minimal for pattern-following tasks (coding, review, remediation). This is why tiered routing works — you're using frontier where it matters and saving money where the gap is negligible.

---

## Router Architectures

### 1. Simple Rule-Based Router

**Approach:** Hardcode task-to-tier mapping (as shown in the table above).

```
IF task_category == "research" OR "validation" OR "audit"
  THEN use Frontier
ELSE IF task_category == "coding" OR "review" OR "remediation"
  THEN use Mid-tier
ELSE IF task_category == "pre_flight" OR "smoke_test"
  THEN use Automated
```

**Pros:**
- Easy to understand and debug
- Optimal for most projects
- No overhead

**Cons:**
- Inflexible if task characteristics change
- Doesn't adapt to project-specific patterns

**Best for:** Small to medium projects with clear task boundaries

---

### 2. Complexity-Based Router

**Approach:** Estimate task complexity and route dynamically.

```
complexity = estimate_complexity(spec, codebase_size, integration_points)

IF complexity > 0.8
  THEN use Frontier
ELSE IF complexity > 0.4
  THEN use Mid-tier
ELSE
  THEN use Automated
```

**Complexity heuristics:**
- Number of files affected (>5 files = higher complexity)
- External API integrations (if yes = higher complexity)
- Cross-module reasoning required (if yes = higher complexity)
- Novelty of pattern (is this similar to existing code? lower complexity if yes)

**Pros:**
- Adapts to project-specific complexity
- Can optimize cost/quality on a per-task basis

**Cons:**
- Complexity estimation itself needs to be correct (easy to get wrong)
- More moving parts to debug

**Best for:** Large projects where complexity varies significantly or where you want fine-grained cost optimization

---

### 3. Hybrid Router

**Approach:** Use rule-based routing as the primary gate, complexity-based as the override.

```
route = rule_based_router(task_category)

IF complexity is unusually_high
  AND route != Frontier
  THEN promote to Frontier
ELSE IF complexity is very_low
  AND route == Frontier
  THEN demote to Mid-tier (if not validation)
```

**Pros:**
- Handles the typical case (rule-based) efficiently
- Adapts to edge cases (complexity override)
- Good balance of simplicity and flexibility

**Cons:**
- Still requires complexity estimation for the override

**Best for:** Medium to large projects with some variation in task complexity

---

## Local and Hybrid Model Strategies

For high-volume tasks (hundreds of features), even mid-tier cost can add up. Three strategies for further optimization:

### 1. Local Code Generation

For pattern-following code generation, smaller local models (Llama, Code Llama) can be cost-effective, especially if you have GPU infrastructure.

**When this works:**
- Well-specified tasks with clear patterns
- Internal tooling (not customer-facing code)
- High volume (hundreds of tasks) where amortized cost matters

**When this fails:**
- Novel patterns or architectural decisions
- Complex multi-file integration
- Anything that needs strong error recovery

**Practical approach:** Use local models for implementation of well-understood patterns, frontier/mid-tier for planning and validation.

### 2. Two-Tier Implementation

Generate code twice — once with a local model, once with mid-tier — and compare. Use the local version if it passes automated tests; fall back to mid-tier if not.

This is technically a form of self-play, and it works surprisingly well for pattern-following tasks where you have strong verification (test coverage).

### 3. Caching and Reuse

If you're building multiple features with similar patterns, cache the model outputs from the first feature and reuse them for the second.

Example: The first data model migration uses Sonnet. The second, third, and fourth migrations of similar pattern can reuse the cached approach with minimal mid-tier invocation.

---

## Anti-Patterns

**Using frontier for pre-flight checks.** Checking file existence or dependency installation is not a reasoning problem. Automate it.

**Never routing.** Using the same model for everything because "it's simpler." The 40–50% cost savings isn't negligible, especially at scale.

**Routing based on guesses.** Assigning tier without understanding the failure mode. The decision should be: "What happens if this step is wrong, and does a weaker model risk that failure?"

**Over-routing.** Assigning frontier to tasks that don't need it (like surface-level code review). Review can catch different bugs than audit — route them separately, but review doesn't need frontier reasoning.

**Changing models mid-session.** Starting with frontier context, then switching to mid-tier for the same task. Context style and formatting expectations differ; this creates friction.

**Forgetting the human factor.** If your developer's time is $200/hour and the API cost difference between frontier and mid-tier is $5, and mid-tier requires an extra 30 minutes of human review, mid-tier loses on the math. Route for total cost, not just API cost.

---

## Getting Started: Five Things You Can Do Today

1. **Profile your workflow.** Measure cost per feature using frontier for everything. Then estimate cost using the tiered routing table above. The gap is usually 40–50%. That's your upside.

2. **Automate the obvious.** Pre-flight checks, smoke tests, linting, formatting — write simple scripts (shell, Python, whatever fits your stack) to replace LLM invocations.

3. **Use the rule-based router.** Apply the task-to-tier mapping from the Development Loop table. This is optimal for most projects.

4. **Profile mid-tier performance on your actual tasks.** Run 5 features using Sonnet 4.6 for implementation and measure quality and cost. Compare to frontier. You'll likely see that mid-tier is more than good enough.

5. **Create a ROUTING.md in your repo.** Document which model tier handles which tasks in your specific workflow. Update it as you learn. Share it with your team.

---

## Sources

### Primary Research

- Anthropic, "Model Routing for Cost-Effective Development" (2026)
- Infralovers, "57% Cost Reduction Through Intelligent Model Assignment" (2025)
- Boris Cherny, Claude Code workflow and benchmarking (2026)
- Thoughtworks, "Spec-driven development: Model Selection and Routing" (2025)

### Supplementary

- Anthropic, "2026 Agentic Coding Trends Report"
- HumanLayer, "Optimizing for Cost Without Sacrificing Quality" (2025)
- Manus AI, "Router Architectures for Multi-Model Workflows" (2025)
- Martin Fowler, "Context Engineering for Coding Agents" (2025)
