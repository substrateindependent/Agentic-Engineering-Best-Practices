# Context Engineering

**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Canonical Documentation](Canonical-Documentation.md) · [Project Context Files](Project-Context-Files.md) · [Agent Self-Verification](Agent-Self-Verification.md) · [Externalized State](Externalized-State.md) · [Model Routing](Model-Routing.md)

---

## What Is Context Engineering?

Context engineering is the discipline of designing what information an AI model sees, when it sees it, and how it's structured — across the entire lifecycle of an interaction, not just the initial prompt.

The term gained mainstream traction in mid-2025 when Andrej Karpathy and Shopify CEO Tobi Lütke started using it publicly. But the practice had already been evolving quietly inside teams building production coding agents. Anthropic defines it as "the set of strategies for curating and maintaining the optimal set of tokens during LLM inference." Martin Fowler frames it as the natural successor to prompt engineering — the shift from "what words do I use?" to "what information configuration produces the best outcome?"

The distinction matters for a simple reason: in agentic coding, the model doesn't just respond to one prompt. It operates across dozens of turns, tool calls, file reads, and reasoning steps. Managing that accumulating context is what separates agents that work from agents that drift, hallucinate, or forget what they were doing.

---

## Why Context Matters More Than Model Choice

This is [Principle 1](AI-Coding-Best-Practices.md#1-context-quality-model-quality) of the parent guide, and it's the most validated finding in the agentic coding literature. A mid-tier model with well-curated context consistently outperforms a frontier model drowning in noise.

The reason is architectural. Transformer-based models have a finite attention budget — every token in the context window competes for the model's attention. As the context grows, the model's ability to focus on what actually matters degrades. This isn't a failure of the model; it's a fundamental property of how attention mechanisms work.

Anthropic's internal testing quantifies the impact: combining their Memory Tool with context editing improved agent search performance by 39%. Context editing alone gave a 29% boost. In a 100-turn dialogue, these techniques reduced token consumption by 84% while maintaining task coherence.

The practical implication: spending 30 minutes curating what the model sees will produce better results than spending 30 minutes switching to a more expensive model.

---

## The Core Problem: Context Rot

As an agent works through a multi-step task, its context window fills with tool calls, intermediate results, failed attempts, and accumulated conversation history. This creates **context rot** — the degradation of model accuracy as the context grows noisy.

Context rot doesn't look like a sudden failure. It looks like gradual drift: the agent starts making slightly off decisions, forgets constraints mentioned earlier, repeats work it already did, or loses track of the overall goal. The critical constraint from step three gets buried under the noise from steps four through forty. The agent doesn't forget because it ran out of space — it forgets because signal got drowned by accumulation.

Manus AI (which rebuilt their agent framework four times learning these lessons) reports an average input-to-output token ratio of 100:1 in production agents. That ratio illustrates the problem: for every token the agent *produces*, it's processing a hundred tokens of accumulated context. If most of those hundred tokens are low-signal noise, the one output token suffers.

---

## The Four Operations of Context Engineering

Every context engineering technique maps to one of four operations. Understanding these categories makes it easier to reason about which techniques to apply and when.

### 1. Writing Context (Offloading)

**What:** Saving information *outside* the context window for later retrieval.

This is the most important operation for long-running tasks. Instead of keeping everything in the context window, the agent writes important state to durable storage — files on disk, databases, structured notes — where it can be retrieved when needed.

**Techniques:**

**Structured note-taking (agentic memory).** The agent maintains a running log of its progress outside the context window — goals, decisions made, key findings, and next steps. Claude Code does this naturally with todo lists and NOTES.md files. Anthropic's Claude playing Pokémon demonstrated the power of this approach: the agent maintained precise tallies across thousands of game steps, developed maps, remembered achievements, and continued multi-hour sessions after context resets — all from reading its own notes.

**File-system as memory.** Manus AI treats the file system as unlimited external memory. When an agent processes a large document, it compresses it to a summary plus a file reference. The compression ratio can reach 100:1 while maintaining full information recovery — because the original file is always retrievable. This insight is powerful: compression is lossless when the original is a file read away.

**Canonical documentation.** In the context of AI-assisted coding, this is the [Canonical Documentation](Canonical-Documentation.md) pattern from the parent guide. Each feature gets a document describing what was built, how it works, and what interfaces it exposes. These docs serve as offloaded context that can be selectively loaded during the research phase of the next feature.

**Build plans and checklists.** The [Externalized State](Externalized-State.md) pattern. The agent's task list and progress live in a markdown file, not in its memory. After a context reset, the agent reads the checklist to know where it left off.

### 2. Selecting Context (Retrieval)

**What:** Pulling the right information *into* the context window at the right time.

The goal is just-in-time context — the model gets exactly what it needs for the current step, no more. This is the opposite of the "dump everything in" approach that wastes the attention budget.

**Techniques:**

**Just-in-time file reads.** Rather than pre-loading all potentially relevant code at the start of a session, the agent uses tools like `grep`, `glob`, and file reads to dynamically load what it needs for the current task. Claude Code uses this hybrid model: CLAUDE.md files are loaded upfront (stable, high-signal context), while code is retrieved just-in-time via tool calls.

**Scoped and path-based rules.** Context files (Cursor Rules, CLAUDE.md) can be scoped to specific file types or directories. A rule scoped to `**/*.ts` only enters the context when the agent is working on TypeScript files. This is progressive disclosure applied to project context — the agent sees task-specific instructions only when they're relevant.

**RAG (Retrieval-Augmented Generation).** For large codebases, embedding-based retrieval can surface relevant code snippets without loading entire files. Production code agents like Windsurf combine grep/file search, knowledge graph-based retrieval, and a re-ranking step to find the most relevant context. The cost difference is dramatic: RAG averages ~$0.00008 per query vs. ~$0.10+ for long-context approaches.

**Sub-agent delegation.** Instead of the main agent exploring a large codebase itself (filling its context with exploration artifacts), delegate the research to a sub-agent. The sub-agent does the deep dive, then returns a concise summary. The main agent's context stays clean. As HumanLayer's Dex Horthy puts it: "Sub-agents are not about mirroring human roles like 'frontend agent' or 'QA agent'. They are tools for controlling context."

### 3. Compressing Context (Reduction)

**What:** Retaining essential information while reducing token count.

As context accumulates, older turns become increasingly less relevant. Compression preserves the important bits while discarding noise.

**Techniques:**

**Compaction (summarization).** When the context approaches its limit, summarize older turns and replace the raw history with the summary. Anthropic's context editing feature in Claude Sonnet 4.5 does this automatically — pruning stale tool outputs and interactions while preserving dialogue flow. The art lies in what to keep vs. discard: overly aggressive compaction loses subtle-but-critical details.

**Tool result clearing.** One of the safest, lightest-touch forms of compaction. Once a tool has been called deep in the message history, the raw result is rarely needed again. Clearing old tool results (recently launched as a feature on the Claude Developer Platform) frees significant context space with minimal information loss.

**Selective turn preservation.** Manus keeps the most recent tool calls in full detail (preserving the model's "rhythm" and formatting style) while summarizing older turns into structured JSON. A common pattern: summarize everything older than the last 3 turns, keep the last 3 raw.

**Frequent intentional compaction.** HumanLayer's approach: don't wait until the context is full to compress. Instead, deliberately compact at regular intervals throughout the workflow — after research completes, after a plan is validated, after a feature is implemented. This keeps context utilization in the "smart zone" (40–60% of the window), where the model performs best. Above ~40% utilization, diminishing returns kick in.

### 4. Isolating Context (Separation)

**What:** Splitting context across agents or sessions so each operates with focused, relevant information.

Context isolation prevents pollution — where one task's context degrades another task's reasoning.

**Techniques:**

**Multi-agent context isolation.** Manus applies a principle from Go concurrency: "Share memory by communicating, don't communicate by sharing memory." Each sub-agent gets its own clean context with only what it needs for its specific task. The orchestrator receives concise results, not the full exploration history.

**The Research → Plan → Implement workflow.** HumanLayer's three-phase approach is fundamentally a context isolation strategy. Research produces a compact artifact. A new session loads that artifact and produces a plan. Another new session loads the plan and implements it. Each phase starts with clean context, inheriting only the compressed output of the previous phase — not its full conversation history.

This is directly related to [Principle 4](AI-Coding-Best-Practices.md#4-never-let-the-author-validate-their-own-work) of the parent guide. Fresh-context validation isn't just about preventing confirmation bias — it's about ensuring the validator operates with clean, focused context rather than inheriting the drafter's accumulated noise.

**`/clear` as a tool.** In interactive coding sessions, clearing the context and restarting with a focused summary is one of the most effective (and underused) context engineering techniques. Anthropic and HumanLayer both recommend this as a first step: "Next time your chat gets too long, stop. Ask it to summarize the current state and the next steps. Open a new chat, paste that summary, and continue. You will notice the difference immediately."

---

## RAG, Long Context, or Both?

A common question in 2026: with 200K+ token context windows, do we still need RAG? The answer depends on your constraints.

**Long context windows excel when** the dataset is small (under ~100 docs, <100K tokens), the data is static, cross-file reasoning is needed (debugging multi-file issues, tracing dependencies), or simplicity matters more than cost.

**RAG excels when** the codebase is large (>100K tokens), data changes frequently, cost matters (RAG is ~1,250x cheaper per query), latency matters, or you need access control and auditability (RAG can filter by permissions before anything hits the model).

**The emerging consensus is hybrid.** Most production coding agents in 2026 use both strategically. The architecture looks like: use long context for the current working set (the files you're actively editing, the plan, the spec). Use RAG for retrieval from the broader codebase (finding relevant implementations, locating similar patterns, understanding dependencies). Use file-system memory for anything that exceeds both (full canonical docs, build history, error logs).

The HumanLayer team reports successfully handling 300K LOC Rust codebases with this hybrid approach — shipping a week's worth of senior-engineer work in a day while maintaining code quality that passes expert review.

---

## The Smart Zone: Context Utilization

One of HumanLayer's most actionable findings is the concept of the **smart zone** — a target range for context window utilization.

**Below 20% utilization:** The model lacks sufficient context to make good decisions. It's guessing rather than reasoning from evidence.

**20–40% utilization:** The sweet spot for most tasks. The model has enough context to reason well, with headroom for tool calls and responses.

**40–60% utilization:** Still functional for complex tasks that require more context, but performance starts to degrade. Reserve this range for research-heavy phases.

**Above 60% utilization:** Diminishing returns accelerate. The model's attention is spread too thin. New information competes with accumulated noise. This is where context rot becomes severe.

The practical takeaway: design your workflow so that you compact or reset context *before* you hit 60%. Don't wait for the context window to fill up — by then, quality has already degraded. Frequent intentional compaction keeps you in the smart zone throughout the task.

---

## KV-Cache: The Production Performance Lever

For teams running agents at scale, Manus AI's insight about KV-cache optimization is critical. The KV-cache stores computed attention values for previous tokens — when the cache is hit, the model doesn't need to recompute attention for those tokens, dramatically reducing latency and cost.

Three principles for maximizing KV-cache hit rates:

**Stable prefixes.** System prompts, tool definitions, and project context should be identical across turns. Even a single token difference (like a timestamp) invalidates the entire downstream cache. Move dynamic content to the end of the context, not the beginning.

**Append-only context.** Never modify previous actions or observations retroactively. Ensure deterministic serialization — even JSON key ordering matters. When you need to update state, append a new entry rather than editing an old one.

**Static tool definitions.** Manus discovered that dynamically adding or removing tools mid-session breaks cache coherence. Their solution: keep all tool definitions stable and use logits manipulation to mask unavailable tools, rather than removing them from the context. For most practitioners, the simpler version is: define your tool set once at the start and don't change it mid-session.

This matters less for interactive coding sessions (where latency per turn is acceptable) and more for automated pipelines processing many features — where the cumulative cost and latency savings are substantial.

---

## Applying Context Engineering to the Development Loop

Here's how these principles map to the [Development Loop](AI-Coding-Best-Practices.md#the-development-loop) from the parent guide:

**Step 1 (Research):** *Select* context by reading relevant canonical docs and exploring the codebase via just-in-time file reads. *Write* context by producing a canonical document that compresses your research into a high-signal artifact. At the end of research, you've created a compact document that captures everything the planning phase needs — without requiring the planning phase to repeat the exploration.

**Steps 2–4 (Draft & Validate):** *Isolate* context by running each validation pass in a fresh session. The validator loads only the spec and the plan — not the research session's full conversation history. This is both a confirmation-bias prevention measure and a context quality measure.

**Steps 5–6 (Build Plan):** *Compress* the validated plan into a dual-format build plan (readable markdown + structured data). This artifact is the highest-signal, lowest-noise representation of what needs to be built. It becomes the primary context for implementation.

**Step 7 (Pre-flight):** Pure *selection* — programmatically verify that referenced files and dependencies exist. No LLM involved, no context consumed.

**Step 8 (Implement):** *Select* context per-task from the build plan. Load only the current task's requirements and the relevant source files. *Write* context by committing after each task — the git history becomes offloaded context. If the session resets, the agent reads the build plan checkboxes to know where to resume.

**Steps 9–10 (Review & Audit):** *Isolate* each review pass in fresh context. The reviewer loads the spec, the code, and the Integration Contract — but not the implementation session's accumulated reasoning.

**Step 14 (Update Docs):** *Write* context by updating canonical documentation. This is the step that makes context quality compound across features — the next feature's research phase benefits from accurate, up-to-date documentation of what was built.

**Step 15 (Smoke Test):** Pure *selection* — programmatic post-merge verification. No LLM context consumed. Deterministic checks confirm the build is healthy before moving to the next feature.

---

## Anti-Patterns

**The "dump everything" approach.** Loading the entire codebase into a long context window "because we can." Signal drowns in noise. The model performs worse than with a carefully selected subset.

**Never compacting.** Letting the context fill up over a long session without summarizing or resetting. By the time the window is full, the model has been degrading for thousands of tokens.

**Dynamic timestamps in system prompts.** Including the current time, date, or session ID in system prompts. This seems harmless but invalidates the KV-cache on every turn, increasing latency and cost.

**Treating sub-agents as role-play.** Spawning a "frontend agent" and a "backend agent" that share context. Sub-agents are tools for context isolation, not organizational metaphors. Each should get minimal, focused context and return a concise result.

**Ignoring the middle.** The "lost in the middle" problem is real for many models — information in the middle of the context receives less attention than information at the beginning or end. Place the highest-signal content (current task, key constraints, recent decisions) at the end of the context, where attention is strongest.

**Over-compacting.** Aggressive summarization that discards subtle but critical details. The art of compaction is knowing that some nuances only reveal their importance later. When in doubt, keep the detail and compress something else.

---

## Getting Started: Five Things You Can Do Today

If you're not doing any context engineering, start here:

1. **Clear and restart when your session gets long.** Ask the AI to summarize the current state and next steps. Start a new session with that summary. You'll notice the difference immediately.

2. **Write a project context file.** Create a CLAUDE.md, Cursor Rules file, or AGENTS.md with your project's tech stack, conventions, and common mistakes. Keep it under 300 lines. Update it when you see the AI make an error.

3. **Create canonical docs for completed features.** After each feature, write a short document describing what was built, how it works, and what it touches. Store these in your repo. Load them at the start of related work.

4. **Use sub-agents for research.** When you need to explore a large codebase, delegate the exploration to a sub-agent (a separate session or a Task tool invocation) and bring back only the summary.

5. **Scope your context per task.** When implementing a feature, load only the files and docs relevant to the current task — not everything that might be related. Add more context only when you need it.

---

## Sources

### Primary Research

- Anthropic, "[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)" (2025)
- Martin Fowler / Thoughtworks, "[Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)" (2025)
- Manus AI, "[Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)" (2025)
- HumanLayer, "[Advanced Context Engineering for Coding Agents](https://www.humanlayer.dev/blog/advanced-context-engineering)" (2025)
- LangChain, "[Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/)" (2025)

### Supplementary

- Inkeep, "[Fighting Context Rot](https://inkeep.com/blog/fighting-context-rot)" (2025)
- HumanLayer, "[Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)" (2025)
- jroddev, "[Context Window Management in Agentic Systems](https://blog.jroddev.com/context-window-management-in-agentic-systems/)" (2025)
- Jason Willems, "[Long Context Windows: Capabilities, Costs, and Tradeoffs](https://www.jasonwillems.com/technology/2026/01/26/Long-Context-Windows/)" (2026)
- ByteIota, "[RAG vs Long Context 2026](https://byteiota.com/rag-vs-long-context-2026-retrieval-debate/)" (2026)
- Prompt Engineering Guide, "[Context Engineering Guide](https://www.promptingguide.ai/guides/context-engineering-guide)" (2025)
