# Parallel Agent Workflows

**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Externalized State](Externalized-State.md) · [Integration Contracts](Integration-Contracts.md) · [Agentic Workflow Guide](agentic-workflow-guide.md) · [Environment and Repository Setup](Environment-and-Repo-Setup.md)

---

## What Is Parallel Agent Workflows?

Running multiple AI coding sessions concurrently — each on an independent feature, each in its own git worktree — is the single biggest productivity multiplier in AI-assisted development. Boris Cherny calls it "the single biggest productivity unlock." His typical workflow runs 5 Claude Code sessions in parallel in the terminal, plus 5–10 additional sessions on claude.ai, across independent features. The throughput difference is dramatic: instead of one agent blocking while waiting for tests to run or your review, five agents stay productive while bottlenecks resolve themselves.

The key insight isn't multitasking — it's **eliminating idle time**. While Agent A runs tests, Agent B implements, and Agent C waits for your input. Your role shifts from "do the work" to "orchestrate the work" — keeping all sessions unblocked and merging their output in a disciplined way.

---

## The Throughput Insight

A single agent working sequentially on features has this pattern:

```
Feature 1: Research → Plan → Build → Review → Tests → Merge
  (6 hours)
Feature 2: Research → Plan → Build → Review → Tests → Merge
  (6 hours)
Feature 3: Research → Plan → Build → Review → Tests → Merge
  (6 hours)

Total time: 18 hours
```

With five agents working in parallel on independent features:

```
Agent 1: Research ┐
Agent 2:         Research ┐
Agent 3:                 Research ┐
Agent 4:                         Research ┐
Agent 5:                                 Research ┐
                                              (6 hours per phase)

All agents: Research → Plan → Build → Review → Tests → Merge (concurrently)

Total time: ~6–7 hours per phase (agents overlap at boundaries)
          + 10–15% overhead (orchestration, merge discipline)
```

The reality is closer to 3–4x throughput improvement than 5x, because agents need your input, tests fail and need debugging, and merges create synchronization points. But 3–4x is not trivial. On a 6-month project, parallel workflows can compress 4–5 months of sequential work into 6–8 weeks.

The research backing this comes from multiple sources: Boris Cherny's Claude Code workflow, Addy Osmani's experiments with backend/frontend/test agents, Cursor's "Parallel Agents" feature (built entirely on git worktrees), and internal Anthropic studies. The consistent finding: parallel independent work with disciplined merge practices produces dramatic throughput gains without quality loss.

> *See: [Agentic Workflow Guide](agentic-workflow-guide.md) for how merge discipline and the Diagnose → Plan → Implement workflow prevent quality degradation.*

---

## Git Worktrees: The Foundation

Git worktrees let you check out multiple branches simultaneously, each in its own directory, sharing the same `.git` repository. This is the infrastructure that makes parallel workflows practical.

Without worktrees, parallel agents would either conflict over the working directory or require managing separate clones (expensive in disk space and time). With worktrees, each agent gets its own sandbox while sharing git metadata. Merges are unified, and the solution stays coherent.

**Key properties that make worktrees ideal for AI agents:**

- **Separate working directories.** Each agent can modify files without affecting sibling agents. No `.lock` files, no "saving your changes" friction.
- **Shared .git metadata.** All worktrees see the same commit history, branches, and tags. When Agent A merges its feature, Agent B can immediately see the result via `git fetch`.
- **Minimal disk overhead.** A worktree is mostly hard links and references — typically 10–15% additional disk per worktree, not 100% per clone.
- **Automatic cleanup.** When a worktree is no longer needed, `git worktree prune` removes it.

The trade-off: worktrees require discipline. Each agent must work on its own branch in its own worktree. If two agents try to edit the same file, merges can conflict — not because the tool is broken, but because the human(s) gave them overlapping work.

---

## Setting Up Worktrees for AI Coding

### Step 1: Create Worktrees for Independent Features

```bash
# Main repo is in ~/my-project
cd ~/my-project

# Create a feature branch and worktree for Agent A
git worktree add ../my-project-auth feature/auth

# Create a feature branch and worktree for Agent B
git worktree add ../my-project-payments feature/payments

# Create a feature branch and worktree for Agent C
git worktree add ../my-project-tests feature/test-coverage

# Verify
git worktree list
#   /home/user/my-project (branch main)
#   /home/user/my-project-auth (branch feature/auth)
#   /home/user/my-project-payments (branch feature/payments)
#   /home/user/my-project-tests (branch feature/test-coverage)
```

Each directory is now ready for an independent agent. Agent A works entirely in `/my-project-auth`, Agent B in `/my-project-payments`, etc. They never interfere with each other's working directories.

### Step 2: Assign One Agent Per Worktree

Each Claude Code session (or equivalent agent) gets a dedicated worktree:

```bash
# Terminal 1: Agent A on auth feature
cd ~/my-project-auth
claude code  # runs in this worktree

# Terminal 2: Agent B on payments feature
cd ~/my-project-payments
claude code  # runs in this worktree

# Terminal 3: Agent C on test coverage
cd ~/my-project-tests
claude code  # runs in this worktree
```

Each agent sees its own working directory, runs its own tests in isolation, and creates its own commits. No coordination needed during the build phase.

### Step 3: Use the `--worktree` Flag (Claude Code)

Claude Code has a built-in `--worktree` flag that automates the above:

```bash
# Instead of manual setup:
claude code --worktree feature/auth
claude code --worktree feature/payments
claude code --worktree feature/test-coverage
```

Claude Code creates the worktrees automatically, assigns each session a unique directory, and ensures isolation. This is the fastest path to parallel workflows if you're using Claude Code.

### Step 4: Keep Worktrees Lightweight

After each task completes, before moving to the next feature, clean up the worktree:

```bash
# Agent A finished with the auth feature
git worktree remove ../my-project-auth

# Start a new worktree for a different feature
git worktree add ../my-project-logging feature/logging
```

Worktrees are cheap, but disk still matters. On a 2GB codebase, 5 concurrent worktrees consume ~10GB of disk (plus workspace overhead). For long-running or high-concurrency setups, monitor disk usage and prune aggressively.

---

## Guardrails in the Agentic Workflow

The agentic workflow guide includes explicit guardrails against concurrent `/implement` sessions on the same worktree — the workflow rules forbid this practice precisely to avoid the merge conflicts and contamination issues described in this guide. The `/project` meta-orchestrator provides an automated alternative to manual parallel orchestration: it sequences features through diagnose → plan → implement loops on separate worktrees, automating the isolation and merge discipline that manual parallel workflows require.

---

## Guard Rails: Making Parallel Workflows Work

Parallel workflows amplify both productivity and risk. The risk is manageable with guard rails — constraints that prevent the most common failure modes.

### Guard Rail 1: Independent Features Only

Parallelize features that do not depend on each other. If Feature B requires Feature A to be complete and merged before it can be implemented, don't run them in parallel. The Integration Contracts will conflict, merges will deadlock, and you'll lose the concurrency benefit.

**How to identify independent features:**

- **No shared data model changes.** If both features need to modify the same database schema, they're dependent.
- **No shared API endpoints.** If Feature B wraps or extends Feature A's endpoints, they're dependent.
- **No shared component modifications.** If both features modify the same UI component, they're dependent.
- **Explicitly documented in the Integration Contract.** When drafting parallel features, the Integration Contract must note: "This feature does not depend on Feature X and can be parallelized."

When in doubt, don't parallelize. Merge Feature A, then start Feature B. One extra week of sequential work beats a week of blocked, conflicted parallelism.

### Guard Rail 2: One Worktree Per Session

Enforce a strict 1:1 mapping: one agent, one worktree, one branch. If an agent needs to switch to a different feature mid-work, switch worktrees (clean the current one first), don't reuse the same working directory for two features.

This prevents the worst footgun in parallel workflows: two agents accidentally committing to the same branch from different worktrees, resulting in duplicate commits and merge chaos.

**Implementation:**

- Use shell aliases to make worktree creation effortless: `alias newworktree='git worktree add ../repo-$(uuidgen) feature/'`
- Add a pre-commit hook that verifies the current branch matches the intended feature.
- Log which worktree is assigned to which agent in a central AGENTS.md file.

### Guard Rail 3: Notification System

When an agent is blocked (waiting for your code review, waiting for an API to be available, or hitting a decision point), notify yourself immediately. Don't let agents sit idle because you didn't notice they were waiting.

**Notification patterns:**

- **Terminal notifications.** Use `notify-send` (Linux) or native OS notifications to alert you when a session needs input.
- **Browser tabs.** Assign each agent to a separate browser tab. Flashing or highlighting tabs signals which agents need attention.
- **Slack/Discord integration.** Route notifications to a Slack channel so you stay alert even when away from the terminal.
- **Dashboard.** Build or use a tool that shows the status of all active agents (see Tools section below).

The pattern: every time an agent needs human input, it sends a notification. Your job is to check notifications every 10–15 minutes and unblock waiting agents.

### Guard Rail 4: Merge Discipline

When features complete, merge them one at a time through the full Verify & Ship phase (Steps 12–15 from the Development Loop), not in parallel. This is where the orchestrator mental model becomes critical.

**Merge sequence:**

```
Agent A completes feature/auth
  ↓ (Steps 12–15: Full tests, remediation, smoke test)
Merge to main
  ↓ (All agents fetch the updated main)
Agent B completes feature/payments
  ↓ (Steps 12–15)
Merge to main
  ↓
Agent C completes feature/test-coverage
  ↓ (Steps 12–15)
Merge to main
```

**Why merge one at a time:**

1. **Isolation of regressions.** If a merge causes a regression, you know exactly which agent introduced it.
2. **Verification per feature.** Each feature's full test suite runs independently. If tests fail, the failure is localized to that feature.
3. **Clear merge commits.** The main branch history shows one merge per feature, making auditing and bisecting clean.
4. **Integration discovery.** Sometimes Feature A and Feature B don't conflict at the file level but do conflict semantically (e.g., they modify different endpoints but the same database). Merging one at a time gives you a chance to catch these conflicts before they compound.

The alternative — batch-merging multiple features at once — defeats the parallelism advantage and introduces risk: one bad merge breaks everything, and you can't isolate which agent caused the issue.

---

## The Orchestrator Mental Model

This is the psychological frame that makes parallel workflows sustainable.

You are not multitasking. You are **orchestrating a team of AI agents**. Your job:

1. **Assign work clearly.** Each agent gets a spec, a feature branch, and a worktree. No ambiguity.
2. **Provide context.** Load the canonical documentation, project context files, and Integration Contracts so each agent has what it needs.
3. **Keep agents unblocked.** Check notifications. When an agent is stuck, provide input immediately — don't let it spin.
4. **Review and merge carefully.** When an agent completes a feature, run through Steps 12–15 (tests, remediation, smoke test) before merging.
5. **Document what was built.** Update canonical documentation after each merge so the next agent's research phase has current information.

This is fundamentally different from "I'll just run 5 agents and hope they work out." It requires discipline and attention, but the throughput payoff is real.

---

## Parallel Competitive Implementations

An advanced pattern: run the *same spec* through multiple agents simultaneously and pick the best result. This is expensive (5x the compute cost) but produces higher-quality output in research-heavy or architecturally-critical features.

**How it works:**

1. Write a detailed spec for a critical feature (e.g., authentication system, data pipeline).
2. Spawn 3–5 agents, each with the same spec, each in its own worktree.
3. All agents research and plan independently.
4. You review all 5 plans. Pick the best one.
5. Assign one agent to implement the chosen plan. The other 4 worktrees are discarded.

**When to use this:**

- High-impact features where architecture mistakes are expensive to fix later.
- Features with multiple valid implementation strategies.
- Security-critical or performance-critical features where you want the best of multiple approaches.
- Research-heavy tasks where different agents might uncover different insights.

**Trade-offs:**

- Costs 5x as much compute (but only for the research/planning phase).
- Takes longer wall-clock time (agents plan in parallel, but you review serially).
- Produces better output (you're selecting the best of 5 independent approaches).

Competitive implementations are a pattern from Addy Osmani's workflow and from Cursor's research phase. It's not the default, but it's available when the stakes justify the cost.

---

## Tools for Managing Parallel Workflows

### Maestro

A specialized tool for orchestrating AI agents across multiple worktrees. Maestro can:

- Spawn 6+ concurrent Claude Code sessions, each with its own worktree.
- Route notifications when agents are blocked.
- Manage merge scheduling and coordinate merges.
- Maintain an agent status dashboard.

Maestro is opinionated (assumes you're using Claude Code and git) but handles much of the orchestration overhead.

### ccswarm

A lighter-weight tool focused on managing "pools" of agents for specific task types. You can configure a pool of 3 agents for "testing," 2 for "refactoring," etc., and work is routed to available agents. Useful for organizations running many parallel features.

### Crystal

A visual management tool that shows the status of all active agents (which feature, which step in the Development Loop, blocked or running). Integrates with git and common CI/CD systems. Primarily useful when managing 5+ concurrent agents.

For most practitioners, the tools don't matter more than discipline. Five independent Terminal tabs running `claude code --worktree` with a Slack notification hook is sufficient. The tools become valuable above 5–10 concurrent agents or in team settings.

---

## Resource Considerations

Each worktree + agent session consumes resources. Budget accordingly:

### Disk Space

- **Base codebase:** 100MB–5GB (highly variable by project).
- **Per worktree overhead:** 10–15% of base codebase size (hard links, branch metadata).
- **Per agent session:** 100MB–1GB (Claude Code session artifacts, node_modules duplication if not shared, etc.).

**Rule of thumb for a 2GB codebase:**

- 2 concurrent agents: 4–5GB total disk.
- 5 concurrent agents: 10–15GB total disk.
- 10 concurrent agents: 25–30GB total disk.

Researchers have measured disk consumption as high as 9.82GB in 20 minutes for a 2GB codebase with 5 concurrent agents (likely due to dependency installation duplication). Monitor disk closely on constrained systems.

### RAM

Each agent session needs RAM for its process, package managers, test runners, and build tools:

- **Per agent session:** 2–4GB (typically 3GB average).
- **Plus base system overhead:** 2–4GB.

**Realistic capacity:**

- On a 16GB machine: 3–4 concurrent agents.
- On a 32GB machine: 5–6 concurrent agents.
- On a 64GB machine: 10+ concurrent agents.

These are soft limits — you can often squeeze more, but performance degrades. Go by active agent count, not theoretical maximum.

### Network

Parallel agents pulling the same large dependencies (npm install, pip install, etc.) can saturate bandwidth on weak connections. If you're on a 10 Mbps connection, installing node_modules across 5 agents in parallel will thrash your network.

Mitigations:

- Use a local cache (npm proxy, pip cache, Docker layer caching).
- Stagger agent start times (don't spawn all 5 simultaneously).
- Parallelize only when agents are working on different dependency sets.

---

## Known Limitations

### Shared Services and Databases

If your application uses a single database or message queue, parallel agents can conflict during testing. Agent A's tests write data that confuses Agent B's tests.

**Solutions:**

- **Per-agent test databases.** Spin up a containerized database per agent (Docker Compose per worktree).
- **Deterministic test isolation.** Ensure each agent's tests clear their state before and after running.
- **Test namespacing.** Prefix test data with the agent's ID so Agent A's tests don't see Agent B's test data.
- **Mock external services.** In most feature-level parallel work, you can mock the database and message queue entirely, eliminating the conflict.

The simplest approach: mock everything during agent development. Once all features merge, run full E2E integration tests against a real shared database (serially, not in parallel).

### Merge Conflicts

If two agents edit the same file, git will require manual merge resolution. This isn't a tool failure — it's a constraint of parallel work on shared code.

**Prevention (primary):**

- Follow the "independent features" guard rail. If features don't overlap files, merges are automatic.

**Handling (when it happens):**

- You resolve the conflict (not the agent — agents often misinterpret merge conflicts).
- Verify the merged code works with both agents' changes.
- Re-run the full test suite after resolving.

GitButler and other tools warn about this: parallel agents increase merge complexity. The risk is real, but manageable with discipline.

### Disk Space Explosion

As noted above, concurrent agents can consume surprising amounts of disk. Manus AI reports seeing 9.82GB+ consumed in 20 minutes for a 2GB codebase with 5 agents. This is typically from dependency duplication and build artifacts.

**Mitigation:**

- Monitor disk usage actively.
- Set up automatic worktree cleanup: `git worktree prune` weekly.
- Share dependency caches across worktrees (e.g., symlink node_modules or use npm's global cache).
- For very large codebases, consider monorepo tools (Nx, Turborepo) that cache builds across worktrees.

---

## Anti-Patterns in Parallel Workflows

### Anti-Pattern 1: Parallelizing Dependent Features

If Feature B requires Feature A to be complete, don't run them in parallel. One agent will block, communication overhead increases, and merge discipline breaks down. Sequential work is often faster than parallel work with dependencies.

### Anti-Pattern 2: No Notification System

Spawning 5 agents and checking on them once an hour. Agents sit idle for 50+ minutes waiting for your input. Defeat the purpose of parallelism. A notification system (however simple) is non-negotiable.

### Anti-Pattern 3: Batch Merging

Letting all 5 agents complete their work, then merging all 5 at once. If any merge introduces a regression, you can't isolate which agent caused it. You lose the localization benefit of serial merging. Merge one at a time, always.

### Anti-Pattern 4: Shared Worktree Between Agents

Two agents sharing the same worktree, each working on different features. Causes merge conflicts, duplicate commits, and chaos. One worktree per agent, enforced.

### Anti-Pattern 5: No Integration Contract Verification

Agents implement features in isolation without verifying that their Integration Contracts are compatible. Two agents unknowingly modify the same database schema or endpoint. Merges fail or produce silent integration bugs.

Check Integration Contracts during planning, before agents start building.

---

## Getting Started: Parallel Workflows Step by Step

### Step 1: Start with Two Independent Features

Don't jump to 5 agents. Pick two features that don't depend on each other:

```bash
# Feature 1: Logging system (independent infrastructure)
git worktree add ../repo-logging feature/logging

# Feature 2: Payment validation (depends only on existing user schema)
git worktree add ../repo-payments feature/payments

# Start two Claude Code sessions
cd ../repo-logging && claude code --worktree feature/logging &
cd ../repo-payments && claude code --worktree feature/payments &
```

### Step 2: Add a Simple Notification Hook

```bash
# When an agent needs your input, it outputs a specific marker
# Hook into that marker with a notification

# Add to your shell rc:
watch-agents() {
  while true; do
    git worktree list | grep '(detached)' && notify-send "Agent needs input"
    sleep 60
  done
}

watch-agents &
```

(This is simplified; see notification patterns above for production setups.)

### Step 3: Merge with Discipline

Feature A completes first. Run Steps 12–15 before merging:

```bash
cd ../repo-logging
# Step 12: Full test suite
npm test
# Step 13: Fix failures
# Step 14: Merge
git checkout main && git merge feature/logging
# Step 15: Smoke test
npm run smoke-test
```

Only then does Feature B start the merge process.

### Step 4: Update Canonical Documentation

After merging Feature A:

```bash
# docs/canonical/logging.md
# Describe what was built, how it works, what it touches.
```

Feature B's research phase reads this. The context quality compounds.

### Step 5: Repeat and Scale

After 2 features, add a third. After 3, add a fourth. Watch for resource constraints and merge conflicts. Adjust parallelism based on what you observe.

---

## Sources

**Primary Research on Parallel Workflows:**

- Boris Cherny, Claude Code workflow documentation and Lenny's Newsletter interview (2026).
- Addy Osmani, "The Orchestrator Model: Multi-Agent AI Development" (2026).
- Cursor Developer Documentation, "Parallel Agents" feature (built on git worktrees).
- GitButler, "Multi-branch Development and Merge Conflict Warnings" (2025).

**Git Worktree References:**

- Official Git documentation, "git-worktree(1)."
- Upsun Developer Center, "Using Git Worktrees for Parallel Development" (2025).
- Nx Blog, "Monorepos and Worktrees: Scaling Parallel Development" (2025).
- Steve Kinney course materials, "Advanced Git Workflows" (2025).

**Scale and Resource Analysis:**

- Manus AI, "Resource Planning for Multi-Agent Workflows" (2025).
- HumanLayer, "Orchestrating Agentic Teams" (2025).
- Anthropic, "2026 Agentic Coding Trends Report."

**Related Methodologies:**

- Thoughtworks, "Spec-driven development" (2025).
- Microsoft, "Taxonomy of failure modes in AI agents."
- Concentrix, "12 Failure Patterns of Agentic AI Systems."
