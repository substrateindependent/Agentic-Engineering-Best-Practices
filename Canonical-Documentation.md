# Canonical Documentation

**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Context Engineering](Context-Engineering.md) · [Externalized State](Externalized-State.md) · [Integration Contracts](Integration-Contracts.md) · [Agentic Workflow Guide](agentic-workflow-guide.md)

---

## What Is Canonical Documentation?

Canonical documentation is a per-feature document that serves as the single source of truth about what has been built, how it works, and what decisions shaped its implementation.

For each major feature or functional area, you create one markdown file that describes: the implementation approach and rationale, the data model it touches, how it integrates with existing code, edge cases and error handling, and explicit decisions about what *not* to include. This document lives in the repository, typically in `docs/canonical/`, and gets updated each time the feature is complete.

The term "canonical" is borrowed from software design, where a canonical source is the authoritative reference — the place you go to understand the current state of something. In AI-assisted development, canonical docs serve a specific purpose: they're the primary context artifact that prevents the AI from reinventing the wheel or making incorrect assumptions about code that has already changed.

---

## Why Canonical Documentation Matters

### The Spec-Driven Development Connection

The Thoughtworks research on Spec-Driven Development (SDD) found that teams using written specifications as primary artifacts experience 70% less rework than teams that don't. The underlying reason is deceptively simple: a written specification forces you to think through edge cases, integration points, and scope boundaries *before* the expensive part starts.

In AI-assisted development, canonical documentation achieves the same effect — but with a second benefit. Canonical docs aren't just for the team; they're context artifacts for the AI. When the next feature needs to integrate with what you've already built, the research phase reads the canonical doc and gets accurate information about the current state of the codebase.

Without canonical docs, every new feature starts from scratch. The AI re-reads the source code to understand how the booking system works, or the authentication flow, or the payment integration. It rebuilds that mental model from fragments scattered across multiple files. With canonical docs, the AI gets a curated, high-signal summary that took the previous feature's implementer 30 minutes to write — saving the next feature's implementer several hours of exploration and reconstruction.

This is [writing context](Context-Engineering.md#1-writing-context-offloading) in action. The knowledge about what was built gets offloaded to a file, where it can be selectively retrieved.

### The Integration Failure Prevention

The #1 failure mode in AI-assisted development is orphaned or disconnected code: routes that don't exist, database tables that are never used, components that aren't wired into the navigation. Canonical documentation prevents this by forcing explicit specification of integration points — what existing code this feature touches, what new interfaces it exposes, what will depend on this feature in the future.

This is why [Principle 7](AI-Coding-Best-Practices.md#7-the-integration-contract) of the parent guide emphasizes the Integration Contract as part of every feature plan. Canonical documentation makes that integration explicit and permanent, so future features can learn from it.

---

## The Canonical Document Template

There's no single "correct" structure, but this template covers the essentials:

```markdown
# [Feature Name]

**Status:** Completed | In Progress | Planned
**Last Updated:** YYYY-MM-DD
**Implemented By:** [Names/Agents involved]

## Overview

One paragraph. What does this feature do? Who uses it? Why does it exist?

## Implementation Approach

How was this built? What strategy did you choose, and why?

Include:
- The core algorithm or pattern
- Why this approach over alternatives
- Key architectural decisions
- Any constraints that shaped the design

## Data Model

What data does this touch? Include:

- **New Tables/Collections:** Schema with field descriptions and constraints
- **Modified Tables:** What changed, what migrations were required
- **Key Relationships:** Foreign keys, references, how data flows
- **Indexes:** What's indexed and why

## Integration Points

How does this feature connect to the rest of the codebase?

- **Imports:** What does this feature depend on?
- **Exports:** What interfaces does this feature expose?
- **Files Modified:** List of existing files that were touched
- **Files Created:** List of new files introduced
- **API Contracts:** Request/response shapes for new endpoints
- **Event Hooks:** What events does this feature emit? What does it listen for?

## Edge Cases and Error Handling

What goes wrong, and how do we handle it?

- **Boundary conditions:** Empty inputs, null values, zero counts, limits
- **Race conditions:** Concurrent operations that could interfere
- **External service failures:** API timeouts, network errors, third-party downtime
- **Resource constraints:** What happens when we run out of memory, disk, or connections?
- **User error:** Invalid inputs, malformed requests, missing permissions

Include the error handling strategy for each case.

## Decisions and Trade-Offs

What did you *not* build, and why?

- **Out of scope:** Features that were considered but deferred
- **Simplifications:** Aspects that could be more complex but don't need to be (yet)
- **Performance trade-offs:** Why you chose speed over storage, or vice versa
- **Future work:** What would improve this if we revisit it

This section is as important as what you *did* build. It prevents future work from re-discussing decisions that have already been made.

## Testing Strategy

What tests were written?

- Unit tests: What components are covered?
- Integration tests: How is this feature tested alongside the rest of the system?
- E2E tests: What user workflows are validated?
- Known gaps: What's not tested, and why?

## Performance Characteristics

- **Latency:** How long do typical operations take?
- **Throughput:** How many operations can happen concurrently?
- **Resource usage:** Memory, CPU, network bandwidth for typical operations
- **Scalability:** How does performance degrade at scale?

## Known Issues and Workarounds

If there are known limitations or bugs that haven't been fixed, document them here. Future work can address them deliberately rather than discovering them painfully.

## Related Features

What other features does this interact with? Cross-link to their canonical docs.
```

Not every section is mandatory for every feature. A simple CRUD endpoint might skip "Known Issues." A data pipeline might skip "API Contracts." Use your judgment — the goal is to capture what the next person (or the AI in the next research phase) actually needs to understand.

---

## Keeping Documentation Current with the Agentic Workflow

The agentic workflow guide includes the `/capture-lesson` subagent, which runs after each implementation and captures surprises, mistakes, and insights. These lessons can include suggestions for updating canonical documentation — either to fix stale content or to add details that would help future features. This mechanism ensures that canonical documentation stays current as the codebase evolves, preventing the drift that turns docs into liabilities.

---

## When to Create Them

Canonical documentation should be created during the research phase (**Step 1 of the Development Loop**). This is actually a refinement of what the parent guide suggests: the "canonical document" mentioned in Principle 2 is this artifact.

The flow looks like:

1. **Research (Step 1):** Investigate the feature. Explore how it should integrate with existing code. Document your findings.
2. **Draft the Feature Plan (Step 2):** Turn your research findings into a buildable plan, including an Integration Contract. The research document becomes the foundation for this plan.
3. **After Implementation (Step 14):** The research document you created in Step 1 gets refined and finalized as the permanent canonical document, reflecting what was actually built.

The reason to create it in research, not after: writing a canonical document during research forces you to think through the problem deeply before coding starts. You're not documenting what was built; you're thinking on paper about what *should* be built. This is the SDD principle in action.

---

## When to Update Them

Canonical documentation is a living document. It gets updated whenever the feature changes materially.

The **mandatory update** happens at Step 14 of the Development Loop, after implementation completes. At that point, you finalize the research document to reflect what was actually implemented — noting any deviations from the original plan and updating the data model, integration points, and testing strategy to match reality.

**Additional updates** should happen whenever:

- A bug is fixed that wasn't documented in "Known Issues"
- An integration point is added or changed (another feature now depends on this, or this feature now depends on something new)
- The data model evolves (migration, schema change, new indexes)
- A performance characteristic changes significantly (10x slower, or 10x faster)
- A decision is revisited (something that was out of scope is now in scope)

The key principle: **if future work will benefit from knowing this, document it.** The bar for "worth updating" is lower than you might think. A quick note that "authentication was refactored on [date] to use OAuth instead of sessions" takes 2 minutes to write and saves the next feature from reading through the implementation details to discover that the old approach isn't used anymore.

---

## The Index Pattern

Maintain a single index file that links to all canonical docs. This lives at `docs/canonical/_index.md` and serves as the discovery mechanism.

A simple index looks like:

```markdown
# Canonical Documentation Index

Last updated: YYYY-MM-DD

## Core Features

- [[Authentication]] — How users log in, session management, permissions
- [[Booking System]] — How bookings are created, modified, cancelled
- [[Payments]] — Stripe integration, transaction flow, refunds
- [[Notifications]] — Email, SMS, push notifications

## Data and Infrastructure

- [[Database Schema]] — Table relationships, migrations, constraints
- [[File Storage]] — S3 integration, file lifecycle, cleanup
- [[Caching Layer]] — Redis, cache invalidation strategy

## Background Processes

- [[Job Queue]] — How async jobs are scheduled and executed
- [[Email Delivery]] — Retry logic, bounce handling, unsubscribe management

## External Integrations

- [[Third-Party APIs]] — Which APIs we call, rate limits, fallback behavior
```

The index isn't meant to be exhaustive. It's a map that tells the next person (or the AI in the research phase) "here's what exists" and "here's where to find it."

When you add a new canonical doc, update the index. When you retire a feature, move its doc entry to an archive section. The index stays current — it's the heartbeat of your documentation system.

---

## Canonical Docs as Offloaded Context

In the [Context Engineering](Context-Engineering.md) framework, canonical documentation is a technique for writing context — saving information outside the context window for later retrieval.

The mechanism works like this:

1. **Research Phase (Step 1):** The AI explores the codebase and external requirements. It produces a canonical document summarizing what needs to be built.
2. **Planning Phase (Step 2):** The AI reads *only* the canonical document (not the full research notes) and produces a feature plan. The context window is small because it's focused on one artifact.
3. **Validation and Implementation:** Subsequent steps work from the plan and the canonical doc. Full codebase context is loaded just-in-time via tool calls.
4. **Next Feature (Step 1 of the next cycle):** When a new feature needs to understand existing systems, the AI reads the canonical doc instead of re-exploring the codebase. Hours of exploration are compressed into minutes of reading.

This is why canonical documentation compounds the quality of your context over time. Each completed feature produces an artifact that makes the next feature's context richer and more focused.

---

## Living Document Practices

Canonical documentation isn't a one-time artifact. It evolves as the codebase evolves. Here are practices that keep docs accurate and useful:

### Tag Changes With Dates and Authors

When you update a section, include a note of what changed and when:

```markdown
## Data Model

**Last modified:** 2026-02-20 by [author] — Added `metadata` column

### Tables

- **users:** [schema...]
- **bookings:** [schema...]
```

This gives future readers context about recency. A section updated last week is more trustworthy than one updated six months ago.

### Link to Related PRs

When you update canonical documentation as part of a PR, include a reference:

```markdown
**Last modified:** PR #451 (Authentication Refactor) — Changed session storage from memory to Redis
```

This gives future readers a place to dig deeper if needed.

### Create an Updates Section

Add a changelog at the top of each canonical doc:

```markdown
# [Feature Name]

## Recent Updates

| Date | Change | PR |
|------|--------|-----|
| 2026-02-20 | OAuth added as auth method | #451 |
| 2026-02-10 | Session timeout reduced to 1 hour | #440 |
| 2026-01-30 | Rate limiting added | #425 |
```

This quickly shows what's changed without reading through the document.

### The Error Log Pattern

Borrow the pattern from Boris Cherny's team: treat your canonical docs partly as error logs. When you discover that the docs don't match reality, or when a misunderstanding happened because the docs were incomplete, update them:

```markdown
## Known Gotchas

- The booking system doesn't prevent double-bookings in race conditions.
  This was discovered in production in Feb 2026. Planned fix for Q2.
- The payments API doesn't retry on network timeout. (See PR #425 for context.)
```

These sections are sometimes the most useful parts of the docs. They capture the collective learning of the team.

---

## Anti-Patterns

**Overly Detailed Documentation**

Writing a 50-page manual describing every function signature and every code path. This is encyclopedic documentation, not canonical documentation. Canonical docs are high-level and decision-focused, not implementation-reference-focused. If you're writing "and here's how we parse the JSON," you're too deep. Step back and focus on "why" and "how it integrates."

**Documentation That Doesn't Match Reality**

The fastest way to make canonical docs worse than useless is to let them become stale. A doc that says "we use Redis for caching" when you've switched to Memcached will actively mislead the next feature. This is why Step 14 of the Development Loop includes updating docs as a mandatory part of feature completion.

**Creating Docs Without a Home**

Canonical documentation that lives in a team wiki or Google Drive, not in the repository. Version control docs alongside code, or they'll drift. The canonical doc for a feature should be in the same PR that implements the feature, so they evolve together.

**Missing Integration Contracts**

Canonical docs that don't explain how the feature connects to the rest of the system. This defeats the main purpose. At minimum, every canonical doc should list: what existing code does this depend on? What will depend on this in the future? What new interfaces are exposed?

**Treating Canonical Docs as Final**

Some teams write canonical docs and then treat them as immutable — "we documented it, we're done." But real systems change. New edge cases are discovered. Integrations shift. The living document pattern recognizes that documentation updates are part of maintenance, not a separate activity.

---

## Practical Getting Started

If you don't have canonical documentation yet, here's how to start:

1. **Pick your most recent completed feature.** Not the first feature you ever built — something from the last 3-6 months while it's still somewhat fresh.

2. **Write a canonical doc for it.** Use the template above. Focus on: what does it do? How does it integrate? What data does it touch? What decisions were made? This will take 30-60 minutes.

3. **Create `docs/canonical/_index.md`.** List all your features and link to their canonical docs. Even if you only have one right now, create the index. It's your scaffold for future docs.

4. **For the next feature, create the canonical doc during research.** Write it as you investigate, not after. Let it guide your planning.

5. **At Step 14 of the Development Loop, finalize the doc.** Add any details that changed during implementation, update the testing section with what was actually tested, and commit it to the repo.

6. **Create a routine to review and update docs.** Once a month, spend 15 minutes checking whether any canonical docs need updates. Has a feature changed? Add a note. Did a misunderstanding happen because the docs were incomplete? Update them.

Start small. One good canonical doc is better than five mediocre ones. The goal is to build institutional memory that compounds — each feature makes the next feature easier.

---

## Connecting to the Development Loop

Here's how canonical documentation maps to each phase:

**Step 1 (Research):** *Create* the canonical document. Explore the feature space, identify edge cases, sketch the data model, identify integration points.

**Steps 2-4 (Draft & Validate):** *Reference* the canonical document. The feature plan is derived from it. Validation passes check that the plan satisfies the canonical doc's scope and integration contracts.

**Step 8 (Implement):** *Keep the canonical doc in view.* As implementation diverges from the plan, note where. Document any surprises.

**Step 14 (Update Docs):** *Finalize* the canonical document. Reflect what was actually built, not what was planned. Update the integration contracts if they changed. Add to the index.

**Step 1 of Next Feature (Research):** *Read* the canonical docs of related features. This is your primary context for understanding the existing system.

This creates a virtuous cycle: each feature's documentation becomes scaffolding for the next feature's planning.

---

## Sources

### Primary Research

- Thoughtworks, "[Spec-driven development](https://www.thoughtworks.com/en-us/insights/blog/spec-driven-development): Unpacking one of 2025's key new AI-assisted engineering practices" (2025)
- Addy Osmani, "[How to Write a Good Spec for AI Agents](https://www.addy.dev/blog/how-to-write-a-good-spec-for-ai-agents)" (2025)
- Martin Fowler, "[Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)" (2025)
- GitHub, "[Spec Kit](https://github.com/github/spec-kit)" — Open-source template for spec-driven development
- AWS, "[Kiro IDE: A Spec-Driven Development Environment](https://www.aws.com/blogs/developer/kiro-ide)" (2026)

### Supplementary

- Boris Cherny, Claude Code workflow documentation (2026)
- HumanLayer, "[Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)" (2025)
- Anthropic, "[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)" (2025)
- Manus AI, "[Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)" (2025)
- JetBrains Blog, "[Make the Spec a Living Document](https://blog.jetbrains.com/upsource/2025/11/make-the-spec-a-living-document)" (2025)
