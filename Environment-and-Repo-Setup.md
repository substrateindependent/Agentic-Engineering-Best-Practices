# Environment and Repository Setup

**Author:** Glenn Clayton, Fieldcrest Ventures
**Version:** 2.0 — March 2026
**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Agentic Workflow Guide](agentic-workflow-guide.md) · [Context Engineering](Context-Engineering.md) · [Integration Contracts](Integration-Contracts.md) · [Parallel Agent Workflows](Parallel-Agent-Workflows.md) · [Deterministic Sandwich](Deterministic-Sandwich.md) · [Externalized State](Externalized-State.md)

---

## Overview: Environment as Quality Ceiling

Your development environment determines the ceiling for AI-assisted coding quality. A well-configured environment with clear context artifacts, automated verification gates, and structured project documentation amplifies AI reasoning. A poorly configured one forces the AI to guess at architectural details, discover patterns through reverse-engineering, and second-guess its own decisions.

This is not about tool choice (Cursor vs. Claude Code vs. Aider). It's about what context you provide, how you structure decisions, what guardrails you build into the workflow, and how you organize the repository so that future AI sessions can pick up where previous ones left off without losing institutional memory.

This consolidated guide covers:
1. **Structural principles** — why the repo is shaped the way it is
2. **Directory structure** — where things live and why
3. **Context file architecture** — project-level instructions that persist across sessions
4. **Git workflow** — branching, commits, and checkpoints
5. **Hook configuration** — automated gates before and after AI edits
6. **Plan files and documentation** — artifacts from the agentic workflow
7. **Quality gates and deterministic enforcement** — pre-flight checks and post-processing validation
8. **Parallel workflows** — running multiple agent sessions safely
9. **Getting started checklist** — setup steps in order

---

## Part 1: Structural Principles

### The Repo Is the Communication Layer

In traditional development, the repository stores code. In AI-agent-built applications, the repository is the primary communication layer — not between humans and AI, but between AI sessions across time. Every future agent session starts from cold. It has no memory of what the last session did, what decisions were made, or what traps to avoid. The repo is the only thing that persists.

This shifts what belongs in a repo. Code is necessary but not sufficient. The repo must also contain the institutional memory that lets any agent session pick up where the last one left off: what was built (canonical documentation), how things connect (integration contracts), what was decided and why (architecture decision records), what went wrong and how to avoid it (living error logs), and what comes next (plan files).

The Thoughtworks specification-driven development research found that teams using written specifications as primary artifacts experience 70% less rework. In agent-built applications, this compounds — each feature builds on documented understanding from previous sessions. Without durable documentation, every session starts from scratch, and the 70% rework penalty applies to every feature.

### Vertical Slices Over Horizontal Layers

Code should be organized by domain feature, not by technical layer. A "vertical slice" architecture groups everything needed to understand and modify a domain — business logic, types, database queries, API handlers, background workers, AI prompts, and UI components — into one directory tree.

This matters for agents because it minimizes the context they need to load. An agent working on email classification loads `features/email/` and gets everything relevant. In a horizontal-layer architecture (`components/`, `lib/`, `api/`, `db/`), the same task requires reading across four directories, loading irrelevant code from other domains at each level. The token cost is higher, the signal-to-noise ratio is worse, and cross-feature contamination is more likely.

The vertical slice pattern also makes cross-domain dependencies explicit. Each feature exports a public API via `index.ts`. Imports between features go through this file — never into internal subdirectories. When an agent sees `import { getEmailById } from '@/features/email'`, the dependency is visible and auditable.

### Thin Shared Layer

The `shared/` directory contains only code that is genuinely used across all or most features — database client singletons, authentication, API client configuration, error handling utilities, and shared type definitions. If something is only used by two features, it stays in those features (one imports from the other via its public API). The shared layer must stay small and stable because every feature depends on it, and changes to shared code create a blast radius across the entire application.

### Explicit Public APIs

Each feature directory exports a public API via `index.ts`. Cross-feature imports go through this file — never into a feature's internal subdirectories. This gives AI agents a clear contract to work against:

```typescript
// features/email/index.ts — Public API
export type { ClassificationResult, EmailSummary } from './types';
export { getEmailById, getRecentThreadsForContact } from './db/emails';
export { getClassificationForEmail } from './db/classifications';
export { CONFIDENCE } from './constants';

// ✅ CORRECT — import through public API
import { getRecentThreadsForContact } from '@/features/email';

// ❌ WRONG — reaching into internal implementation
import { getRecentThreadsForContact } from '@/features/email/db/emails';
```

### Hierarchical AI Context

Context files (CLAUDE.md, AGENTS.md, and tool-specific rule files) are placed at the root and within each feature directory. AI tools automatically load the relevant context files based on which directory they're working in. This provides focused, domain-specific context without loading the entire codebase's conventions into every session.

The key insight from HumanLayer and GitHub's analysis of 2,500+ repositories: keep context files lean. The root CLAUDE.md should be a map, not a manual — under 300 lines. Feature-level context files should stay under 200 lines. Deep reference content belongs in dedicated documentation files that context files point to but don't duplicate. This is the **progressive disclosure pattern**: the agent loads what it needs for the current scope, with pointers to deeper content if it needs to go further.

A critical anti-pattern: don't document file paths in context files. Paths change constantly; stale paths poison the agent's understanding. Instead, document capabilities and domain concepts. Let the agent discover current file paths through the file system.

### Aligned with the Agentic Workflow

The directory structure must reinforce the Diagnose → Plan → Implement workflow. Every artifact the agentic workflow produces — research docs, plan files, implementation code, post-build documentation, audit reports, context file updates — needs a defined home in the repo. If an artifact doesn't have a home, it gets lost. If it gets lost, the next session can't find it, and institutional memory degrades.

---

## Part 2: Recommended Directory Structure

A well-organized repo mirrors how AI agents discover information. Group canonical docs, plan files, and project context in predictable locations.

```
your-project/
├── CLAUDE.md                      # Project context for Claude Code (≤300 lines)
├── AGENTS.md                      # Portable context for other AI tools
├── .cursor/
│   └── rules/
│       ├── architecture.md        # Architectural patterns
│       ├── coding-standards.md    # Style, naming, type-checking
│       ├── testing.md             # Testing patterns
│       └── anti-patterns.md       # Common mistakes to avoid
├── docs/
│   ├── canonical/
│   │   ├── _index.md              # Index of all canonical docs
│   │   ├── authentication.md      # Auth system, data model, integration points
│   │   ├── api-layer.md           # REST/GraphQL APIs, contracts
│   │   ├── database-schema.md     # Current schema, migrations, relationships
│   │   ├── component-library.md   # Reusable UI components, patterns
│   │   └── [feature-name].md      # One doc per major feature area
│   ├── plans/
│   │   ├── _template-plan.md      # Plan file template
│   │   ├── 001-feature-name.md    # Completed plan file
│   │   ├── 002-feature-name.md
│   │   └── archive/               # Completed plans, moved here
│   ├── architecture/
│   │   ├── decision-log.md        # ADRs (Architecture Decision Records)
│   │   └── system-design.md       # High-level system overview
│   ├── research/
│   │   └── [topic].md             # Pre-build research and analysis
│   └── onboarding/
│       ├── development-setup.md   # How to set up the dev environment
│       └── deployment-guide.md    # How to deploy to staging/prod
├── src/
│   ├── features/
│   │   ├── [feature-1]/
│   │   │   ├── CLAUDE.md          # Feature-level context (≤200 lines)
│   │   │   ├── INTEGRATION.md     # Cross-feature dependencies
│   │   │   ├── index.ts           # Public API
│   │   │   ├── types.ts
│   │   │   ├── constants.ts
│   │   │   ├── db/
│   │   │   ├── components/
│   │   │   └── ...
│   │   ├── [feature-2]/
│   │   └── ...
│   ├── shared/
│   │   ├── db/                    # Database client singleton
│   │   ├── auth/                  # Authentication
│   │   ├── api-helpers.ts
│   │   ├── errors.ts
│   │   └── ...
│   └── ui/                        # Shared UI components
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .git/
│   └── hooks/
│       ├── pre-commit             # Linting, type-checking, fast tests
│       ├── post-merge             # Smoke tests, environment validation
│       └── PostToolUse            # Auto-formatting after AI edits
├── package.json (or equivalent)
└── .gitignore
```

**Why this structure:**

- **Canonical docs** (`docs/canonical/`) are the single source of truth. Agents read these to understand existing code without parsing files.
- **Plan files** (`docs/plans/`) are audit trails of the agentic workflow. Checkboxes show progress; the markdown is committable and traceable.
- **Research docs** capture pre-implementation analysis and decisions.
- **Cursor Rules and CLAUDE.md** live at the root so tools discover them automatically.
- **Architecture docs** (`docs/architecture/`) capture decisions (ADRs) to prevent repeated discussions.
- **Hooks** (`.git/hooks/`) run deterministically before and after every commit.
- **Features** are vertical slices with consistent internal structure.

---

## Part 3: Context File Architecture

Context files are the operating system for AI-agent-built applications. They determine what the agent knows before it reads a single line of code. Getting them right is the highest-leverage investment in the entire repo setup.

### Three Major Formats

Modern AI coding tools support three context file standards serving the same purpose but differing in scope and compatibility.

#### CLAUDE.md (Claude Code)

**Where:** Root of your repo, or per-subdirectory.
**Scope:** Claude Code sessions only.
**Capacity:** ~200K tokens per file; can be split across multiple files.

CLAUDE.md supports:

- Markdown structure with wikilinks (`[[link]]`) for navigation between context files.
- Scoped rules that apply only to specific file patterns: `rules/**/*.ts` applies only to TypeScript files.
- Import syntax to pull in referenced files: `/// import <path>` includes another file inline.
- Comments and explanations because these are human-readable documents, not just data.

The format is production-tested. Anthropic's own projects use CLAUDE.md extensively.

#### AGENTS.md (Open Standard)

**Where:** Root of your repo.
**Scope:** Any AI agent (Claude Code, Cursor via extension, Zed, OpenCode, GitHub Copilot).
**Format:** YAML frontmatter + markdown sections.

AGENTS.md is the emerging open standard designed to solve the "locked-in to one tool" problem. A single AGENTS.md file works with multiple tools without sacrifice. Content should be functionally identical to the root CLAUDE.md. When one is updated, update the other.

```yaml
---
name: "Project Name"
version: "1.0"
description: "One-line project summary"
tech_stack: ["TypeScript", "React", "Node.js"]
---

## Architecture

## Tech Stack & Dependencies

## Coding Standards

## Common Patterns

## Anti-Patterns

## Testing Strategy

## Known Issues & Gotchas
```

#### Cursor Rules (.cursor/rules/)

**Where:** `.cursor/rules/` directory.
**Scope:** Cursor IDE only.
**Hierarchy:** Global, Project, User.

Cursor Rules have three levels. Project-level rules override user-level, which override global. This hierarchy lets teams enforce standards while respecting personal preferences.

Project-level rules are typically organized by category:

```
.cursor/rules/
  architecture.md       # Architectural patterns, design system, file structure
  testing.md           # How to write tests, test locations, coverage expectations
  style.md             # Code style, naming conventions, formatting
  backend.md           # Backend-specific (API patterns, database conventions)
  frontend.md          # Frontend-specific (component patterns, state management)
  gotchas.md           # Common mistakes, known issues, what not to do
```

Each file is 50–150 lines, focused and scannable.

### Root CLAUDE.md (≤300 Lines)

The root CLAUDE.md is the project constitution. Every agent session that touches the repo reads this file. It should contain:

- **Architecture overview** — what kind of project this is, how domains relate, the organizing principle (vertical slices, monorepo, etc.)
- **Tech stack** — frameworks, database, queue system, AI provider, CSS approach. Version numbers matter.
- **Global conventions** — naming, file organization, import path rules, error handling patterns.
- **Directory map** — brief description of each top-level directory's purpose. Describe capabilities, not file paths.
- **Cross-feature rules** — "Import through `index.ts` only. Never reach into another feature's internals."
- **Testing conventions** — test runner, test file locations, how to run tests.
- **Pointer to detailed rules** — "See `.cursor/rules/` for detailed coding conventions" rather than duplicating.
- **Living error log** — mistakes the agent has made in past sessions, with the fix.

**Template:**

```markdown
# Project Context — [Project Name]

## Quick Facts
- **Tech Stack:** TypeScript + React 18 + Node.js 18 + PostgreSQL
- **Package Manager:** npm (or pnpm)
- **Testing:** Jest + React Testing Library + Cypress E2E
- **CI/CD:** GitHub Actions (tests on every PR, deploy on merge to main)

## Architecture Overview
[2-3 sentences on the core architecture: what the system does, major components, data flow.]

## Key Patterns You Follow
1. **Component Structure:** Functional components + hooks. All components in `src/components/` with co-located tests.
2. **State Management:** Redux Toolkit for global state. Use selectors, not direct state access.
3. **API Layer:** Axios with a custom request interceptor. All API calls in `src/api/`. Type every response with TypeScript.
4. **Error Handling:** Wrap async operations in try-catch. Log errors to Sentry. User-facing errors go to toast notifications.
5. **Database:** PostgreSQL migrations live in `db/migrations/` with timestamps. Write queries with Knex.

## Anti-Patterns (What NOT to Do)
1. **Don't use `any` types.** Spend 30 seconds writing the type correctly.
2. **Don't make synchronous API calls.** Every external request is async.
3. **Don't test implementation details.** Test behavior, not internal state. Test user interactions, not component methods.
4. **Don't mutate Redux state directly.** Always create new objects.
5. **Don't skip error handling.** Every async operation should have try-catch and user feedback.

## Naming Conventions
- **Files:** kebab-case (`user-profile.tsx`, `auth-service.ts`)
- **Variables/Functions:** camelCase
- **Components:** PascalCase
- **Constants:** UPPER_SNAKE_CASE
- **Database tables:** plural, snake_case (`users`, `booking_sessions`)
- **API routes:** `/api/v1/{resource}/{action}` (RESTful)

## Testing Requirements
- Unit tests for all utils and custom hooks (>80% coverage target)
- Component tests for all UI components (focus on user interactions)
- E2E tests for critical user journeys (login, main feature flow, checkout if applicable)
- Run full test suite before merge: `npm test && npm run test:e2e`

## Common Mistakes (Living Error Log)
1. **Mistake:** Forgetting to add `NOT NULL` constraints on new database columns.
   **Fix:** Always specify `nullable: false` in migrations.
2. **Mistake:** Building UI features without updating the Redux schema.
   **Fix:** Update Redux models first, then build the UI.
3. **Mistake:** Using `setTimeout` for async operations instead of proper async/await.
   **Fix:** Use async/await everywhere. No setTimeout for actual async work.
4. [Add more as you discover them.]

## Project Constraints
- **Bundle size:** Keep <500KB gzipped.
- **Accessibility:** All interactive elements must have keyboard nav + ARIA labels.
- **Mobile:** Responsive design required. Test on 375px width minimum.
- **Performance:** Lighthouse score >80 on desktop, >75 on mobile.

## How to Ask for Help
When stuck, provide: the current file, what you've tried, what's not working. If an error occurs, include the full stack trace.
```

Keep it to <500 lines. Longer documents don't get read. Use per-folder `.claude.md` files for topic-specific details.

### Feature-Level CLAUDE.md (≤200 Lines)

Each feature's CLAUDE.md provides domain-specific context for agents working within that feature:

- **What this feature does** — one-paragraph summary of the domain.
- **Key entities** — the database models this feature owns (table names, key fields, relationships).
- **State machine** — if the feature has processing states, document the transitions.
- **API contracts** — key endpoints and their request/response shapes.
- **Cross-feature dependencies** — what this feature imports from other features (via their `index.ts`), and what other features import from this one.
- **Pointer to canonical documentation** — link to the relevant doc in `docs/canonical/`.
- **Post-build update reminder** — "After completing any work in this feature, update the canonical doc and this CLAUDE.md to reflect what was built."

### INTEGRATION.md (Per-Feature)

Each feature's INTEGRATION.md is the durable form of the Integration Contract. It documents the live wiring between features — not what was planned, but what currently exists:

- **Exports** — what this feature makes available via `index.ts` (functions, types, constants).
- **Imports** — what this feature consumes from other features, with the source feature and function name.
- **Shared entities** — database entities this feature reads from or writes to that are owned by other features.
- **Schema dependencies** — ORM model relationships that cross feature boundaries.
- **Worker triggers** — background pipelines in this feature that are triggered by events from other features.

This file is updated after every implementation that modifies cross-feature boundaries.

### Living Error Log Pattern

Boris Cherny's team at Anthropic treats their CLAUDE.md as a shared mistake journal. Every time the AI makes a mistake, they add it to CLAUDE.md so it won't repeat.

**Example workflow:**

1. AI writes code that violates a pattern.
2. During code review, you catch it.
3. You tag the error: "This is a *pattern violation* — we call the logger differently."
4. You add a two-line example to CLAUDE.md showing the right pattern.
5. Next time the AI writes similar code, it includes the correct pattern.

Over 10–20 features, this builds substantial institutional memory. The time investment is minimal (1–2 minutes per error).

```markdown
## Known Patterns We've Learned

### Error Logging (Learned: PR #47)
✗ logger.error('something failed');
✓ logger.error('operation failed', { userId, action, error });
Always include context. Error messages alone are useless in production.

### Database Queries (Learned: PR #52)
✗ const users = await db.query('SELECT * FROM users');
✓ const users = await db.select().from(usersTable).limit(100);
Always use the query builder; always add limits to prevent runaway queries.
```

### Progressive Disclosure: Scoped Rules

CLAUDE.md supports file-path scoping. Use it to load context relevant to the current task:

```markdown
## TypeScript Rules (*.ts, *.tsx)

Strict mode: all values must be typed, no `any`.

✓ const x: number = 5;
✗ const x: any = 5;

## JavaScript Rules (*.js, *.mjs)

Looser standards for scripts and config files.

## SQL Rules (**/*.sql)

All queries use parameterized statements, never string interpolation.
```

---

## Part 4: Git Workflow for AI-Assisted Development

### Branching Strategy

```
main (production-ready, all tests passing)
  ↓
feature/{epic-name}  ← One branch per feature
  ├── [One commit per task]
  ├── [All tests passing]
  └── [tasks complete]
    ↓
  [Run full test suite]
    ↓
  Merge to main via PR
```

**Naming convention:**
- `feature/{epic-name}` — Main branch for a feature or epic
- Tasks are tracked as individual commits, not separate branches

**Why this structure:**
- One feature branch per epic prevents merge conflicts across independent features.
- Descriptive commit messages (one per task) make it easy to bisect issues.
- All tests pass on main; failures are caught before merge.

### Commit Messages

Format: `[TASK] {task-number}: {action} {what} in {file/component}`

Examples:
```
[TASK] 001: Add login form component in src/components/LoginForm

- Added LoginForm component with email + password fields
- Added Redux action for submitLogin
- Added test suite for user interactions
- Tested: form submission, validation, error handling
```

```
[TASK] 003: Fix password reset email template in src/api/auth

- Updated Sendgrid template to include reset link
- Added expiration validation (1 hour)
- Tested with sendgrid sandbox
- Verified link structure matches reset handler
```

AI can generate these. During the build phase, after each task completes, prompt the AI to commit with a descriptive message.

### One Commit Per Task

During implementation, commit after each task completes. This is your checkpoint system.

```bash
# Task 1: Add database schema
git add db/migrations/001_add_users_table.sql
git commit -m "[TASK] 001: Add users table schema in db/migrations"

# Task 2: Implement auth service
git add src/api/auth-service.ts src/api/auth-service.test.ts
git commit -m "[TASK] 002: Implement login and signup in src/api/auth-service"

# Task 3: Add component
git add src/components/LoginForm.tsx src/components/LoginForm.test.tsx
git commit -m "[TASK] 003: Add LoginForm component in src/components"
```

If a session crashes, you resume from the last completed task, not from the beginning.

---

## Part 5: Hook Configuration: Deterministic Gates

Git hooks run automatically before and after commits. Use them to enforce deterministic checks that don't depend on the AI.

### Pre-Commit Hook (`.git/hooks/pre-commit`)

Runs before every commit. Catches issues before they enter the repo.

```bash
#!/bin/bash
set -e

echo "Running pre-commit checks..."

# 1. Linting
echo "Checking ESLint..."
npm run lint -- --fix  # Auto-fix what's fixable
if ! npm run lint; then
  echo "ESLint failed. Fix issues above."
  exit 1
fi

# 2. Type checking
echo "Checking TypeScript..."
if ! npm run type-check; then
  echo "TypeScript errors found. Fix above."
  exit 1
fi

# 3. Unit tests (fast subset)
echo "Running unit tests..."
if ! npm test -- --testPathPattern="unit" --bail; then
  echo "Unit tests failed. Fix above."
  exit 1
fi

echo "Pre-commit checks passed!"
```

**Why this works:**
- Linting auto-fixes formatting so the AI doesn't have to.
- Type-checking catches errors immediately.
- Fast unit tests give quick feedback (< 1 minute).
- If any check fails, the commit is blocked. The agent fixes and tries again.

### Post-Merge Hook (`.git/hooks/post-merge`)

Runs after a successful merge to main. Validates the final state.

```bash
#!/bin/bash
echo "Running post-merge validation..."

# 1. Verify dependencies
if git diff HEAD~1 package.json > /dev/null; then
  echo "Package.json changed. Running npm install..."
  npm install
fi

# 2. Run full test suite
echo "Running full test suite..."
if ! npm test; then
  echo "Full test suite failed after merge!"
  echo "Rolling back might be necessary. Review the failures above."
  exit 1
fi

# 3. Smoke test
echo "Running smoke tests..."
npm run test:e2e --testNamePattern="smoke"

echo "Post-merge validation passed! ✓"
```

### PostToolUse Hook (Auto-Formatting Pattern)

Some tools (like Claude Code) support PostToolUse hooks that run after the AI edits files. Use this to auto-format:

```bash
#!/bin/bash
# Auto-format files after AI edits
echo "Auto-formatting files..."
npm run format -- --write

# Optionally re-lint
npm run lint -- --fix

echo "Post-edit formatting complete"
```

This prevents the AI from having to worry about formatting—the hook handles it deterministically.

---

## Part 6: Plan Files and Documentation Architecture

The agentic workflow produces plan files as durable artifacts. Store them in `docs/plans/`.

### Plan File Format

**`docs/plans/001-authentication.md`:**

```markdown
# Plan: Authentication Feature

## Epic Summary
Implement user authentication (signup, login, password reset, session management).

## Acceptance Criteria
- [ ] Users can sign up with email + password
- [ ] Users can log in with email + password
- [ ] Sessions persist across page reloads
- [ ] Users can reset forgotten passwords
- [ ] All credentials are hashed (bcrypt)
- [ ] All E2E tests pass

## Integration Contract
**Files Modified:**
- `src/redux/store.ts` — Add auth slice
- `src/api/` — Add auth endpoints

**Files Created:**
- `src/api/auth-service.ts` — Login, signup, passwordReset functions
- `src/components/LoginForm.tsx` — UI component
- `db/migrations/001_add_users_table.sql` — Create users table

**API Contracts:**
- `POST /api/v1/auth/signup` — { email, password } → { userId, token }
- `POST /api/v1/auth/login` — { email, password } → { userId, token }

**Data Model:**
- New table: `users` (id, email, password_hash, created_at)

---

## Tasks

### Task 1: Create users table
- [ ] Write migration in `db/migrations/001_add_users_table.sql`
- [ ] Migration includes: id (primary key), email (unique), password_hash, created_at
- [ ] Verify with: `npm run db:migrate`

### Task 2: Implement auth service
- [ ] Create `src/api/auth-service.ts`
- [ ] Implement `signup(email, password)` — hash password with bcrypt
- [ ] Implement `login(email, password)` — validate credentials
- [ ] Implement `passwordReset(email)` — send email with reset link
- [ ] Write unit tests (>90% coverage)
- [ ] Verify with: `npm test -- src/api/auth-service.test.ts`

### Task 3: Add Redux auth slice
- [ ] Create `src/redux/authSlice.ts`
- [ ] Add actions: `setUser`, `setLoading`, `setError`, `logout`
- [ ] Add selectors: `selectUser`, `selectIsLoggedIn`
- [ ] Verify with: `npm test -- src/redux/authSlice.test.ts`

### Task 4: Build LoginForm component
- [ ] Create `src/components/LoginForm.tsx`
- [ ] Form fields: email, password, submit button
- [ ] On submit: dispatch login action
- [ ] On success: redirect to dashboard
- [ ] On error: show toast notification
- [ ] Write component tests
- [ ] Verify with: `npm test -- src/components/LoginForm.test.tsx`

### Task 5: Add login route
- [ ] Create `/login` route in `src/pages/LoginPage.tsx`
- [ ] Render LoginForm component
- [ ] Redirect authenticated users away from login page
- [ ] Verify with E2E test

### Task 6: Test full flow
- [ ] Write E2E test: user signup → login → dashboard
- [ ] Run: `npm run test:e2e -- authentication`
- [ ] All tests pass

---

## Success Criteria
- [ ] All tasks complete
- [ ] All unit tests passing (>90% coverage)
- [ ] All E2E tests passing
- [ ] No TypeScript errors
- [ ] No ESLint errors
- [ ] Code review passed
- [ ] Merged to main
```

**Update the plan as work progresses.** Check off tasks with `[x]`. This becomes your audit trail and provides checkpoint recovery if a session crashes.

### Documentation Locations

- **`docs/canonical/`** — Single source of truth. Updated after each implementation.
- **`docs/plans/`** — Plan files from the agentic workflow. Archived to `plans/archive/` when complete.
- **`docs/research/`** — Pre-build research and analysis. Referenced by plan files.
- **`docs/architecture/`** — Architecture Decision Records (ADRs) and system design.
- **`docs/onboarding/`** — Development setup and deployment guides.

---

## Part 7: Quality Gates and Deterministic Enforcement

### Pre-Flight Checks

Before an agent starts implementation, verify:

- [ ] All canonical docs updated from previous session
- [ ] Integration contract clearly specified in the plan
- [ ] Feature-level CLAUDE.md exists and is current
- [ ] No uncommitted changes in the working tree
- [ ] All tests passing on the base branch

### Post-Processing Gates

After the agent completes implementation, before merging:

1. **Full Test Suite** — `npm test && npm run test:e2e`
2. **Type Safety** — `npm run type-check`
3. **Linting** — `npm run lint`
4. **Integration Verification** — INTEGRATION.md updated and accurate
5. **Canonical Docs** — `docs/canonical/` reflects what was built
6. **Context Files** — CLAUDE.md and living error log updated
7. **Code Review** — Peer review or deep audit before merge

### Living Error Log as Quality Gate

Before shipping, scan the code for patterns in the living error log. If the log says "always specify `nullable: false` in migrations," scan all migrations added in this feature. If violations exist, add them to the log's update and require the agent to fix them.

---

## Part 8: Parallel Agent Workflows

Run multiple AI sessions concurrently on independent features. Each session gets its own git worktree.

### Setup

```bash
# Main worktree (main branch)
cd your-project

# Create a new worktree for feature 1
git worktree add ../feature-1 -b feature/auth

# Create another for feature 2
git worktree add ../feature-2 -b feature/payments

# View all worktrees
git worktree list
```

Each worktree is an independent working directory with its own branch. No merge conflicts during implementation.

### Orchestration Pattern

You manage multiple sessions. Each has its own terminal window (or tmux pane).

```
Terminal 1 (Feature A):
$ cd ../feature-1
$ npm run dev
[Claude Code session for auth feature]

Terminal 2 (Feature B):
$ cd ../feature-2
$ npm run dev
[Claude Code session for payments feature]

Terminal 3 (You):
$ cd your-project
[Review PRs, merge completed features]
```

Your job: keep sessions unblocked. When one waits, start another.

### Merge Discipline

Merge features one at a time through the full verification and ship process. Never batch-merge features.

```bash
# Feature A completes and passes E2E tests
$ cd feature-1
$ git push origin feature/auth
$ gh pr create --title "Add authentication" ...
[Wait for PR review]

# While waiting, start Feature B if not already running
# After Feature A PR is approved:
$ git checkout main
$ git pull origin main
$ git merge feature/auth
$ git push origin main
$ git worktree remove ../feature-1  # Clean up
```

See [Parallel Agent Workflows](Parallel-Agent-Workflows.md) for the deep-dive guide.

---

## Part 9: Environment Variables and Secrets

Use `.env.local` for development and `.env.example` for template.

**`.env.example`:**
```
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=[your-jwt-secret]
API_BASE_URL=http://localhost:3000/api
STRIPE_PUBLIC_KEY=[stripe-public-key]
SENDGRID_API_KEY=[sendgrid-api-key]
```

**`.env.local` (never commit):**
```
DATABASE_URL=postgresql://postgres:dev@localhost:5432/mydb_dev
JWT_SECRET=dev-secret-key-change-in-production
API_BASE_URL=http://localhost:3000/api
STRIPE_PUBLIC_KEY=pk_test_...
SENDGRID_API_KEY=SG.test_...
```

**In `.gitignore`:**
```
.env.local
.env.*.local
secrets/
```

**Do NOT ask the AI to enter secrets.** If a plan or task requires an API key, tell the AI: "Use environment variable `STRIPE_KEY` which is set on the system."

---

## Part 10: CI/CD Integration

Configure your CI/CD to run tests on every commit and PR.

**GitHub Actions example (`.github/workflows/test.yml`):**
```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - run: npm install
      - run: npm run lint
      - run: npm run type-check
      - run: npm test
      - run: npm run test:e2e
```

**This enables:**
- Tests run automatically on every PR. Failures block merge.
- The AI can check CI results and iterate.
- You see exactly which commit broke the build.

---

## Part 11: Tool Selection Guide

Different tools have different strengths. Choose based on your workflow:

| Tool | Best For | Notes |
|------|----------|-------|
| **Claude Code** | CLI development, batch operations, planning, deep work | Runs in terminal. Full codebase access. Frontier model. |
| **Cursor** | Interactive real-time editing, UI coding | IDE integration. Automatic Cursor Rules loading. Faster iteration. |
| **Aider** | Git-aware code editing, multi-file refactors | Understands git context. Commits automatically. CLI-based. |
| **Windsurf** | Agentic flows, autonomous sessions | Designed for longer, more autonomous workflows. |

**Our recommendation:** Use Claude Code for planning and architectural decisions (frontier model). Use Cursor or Aider for implementation (faster iteration). Both read your context files automatically.

---

## Getting Started Checklist

Set up a new project for AI-assisted development in this order:

- [ ] **1. Create repo structure**
  - [ ] Create `docs/canonical/`, `docs/plans/`, `docs/architecture/`, `docs/research/`
  - [ ] Create `.cursor/rules/` with architecture, standards, anti-patterns
  - [ ] Create `.git/hooks/` directory
  - [ ] Create `src/features/`, `src/shared/`, `src/ui/`

- [ ] **2. Write context files**
  - [ ] Write `CLAUDE.md` (project context, patterns, anti-patterns, naming conventions)
  - [ ] Write `AGENTS.md` (tech stack, rules, key files — synced with CLAUDE.md)
  - [ ] Write `.cursor/rules/architecture.md`, `coding-standards.md`, `testing.md`, `anti-patterns.md`

- [ ] **3. Configure git workflow**
  - [ ] Set up `pre-commit` hook (linting, type-check, unit tests)
  - [ ] Set up `post-merge` hook (smoke tests)
  - [ ] Commit hook files: `git add .git/hooks/ && git commit -m "Add git hooks"`

- [ ] **4. Set up tests**
  - [ ] Create `__tests__/unit/`, `__tests__/integration/`, `e2e/` directories
  - [ ] Write smoke test E2E test (basic app startup)
  - [ ] Verify `npm test` runs unit tests and `npm run test:e2e` runs E2E tests

- [ ] **5. Initialize canonical docs**
  - [ ] Write `docs/canonical/_index.md` (index of all canonical docs)
  - [ ] Write `docs/canonical/architecture.md` (high-level system design)
  - [ ] Write `docs/canonical/database-schema.md` (current schema)
  - [ ] Write `docs/canonical/api-contracts.md` (if applicable)

- [ ] **6. Set up planning infrastructure**
  - [ ] Create `docs/plans/_template-plan.md` (plan file template)
  - [ ] Create `docs/plans/_template-diagnosis.md` (diagnosis template)
  - [ ] Create `docs/plans/archive/` for completed plans

- [ ] **7. Configure feature structure**
  - [ ] Create first feature directory in `src/features/[feature-name]/`
  - [ ] Add `CLAUDE.md`, `INTEGRATION.md`, `index.ts`, `types.ts`, `constants.ts`
  - [ ] Create `db/`, `components/`, and other subdirectories as needed

- [ ] **8. Start your first feature**
  - [ ] Create feature branch: `git checkout -b feature/first-feature`
  - [ ] Create plan in `docs/plans/001-first-feature.md`
  - [ ] Invoke Claude Code with the plan file

---

## Checklist for Each New Feature

When starting a new feature, use this checklist:

- [ ] **Planning Phase**
  - [ ] Read relevant canonical docs to understand existing code
  - [ ] Research the domain and integration points
  - [ ] Write plan file with acceptance criteria and task breakdown
  - [ ] Identify files modified, created, API contracts, data models
  - [ ] Commit plan file: `git add docs/plans/ && git commit`

- [ ] **Implementation Phase**
  - [ ] Create feature branch: `git checkout -b feature/{name}`
  - [ ] Implement tasks one by one
  - [ ] Commit after each task: `git commit -m "[TASK] N: description"`
  - [ ] Run unit tests after each task
  - [ ] Update plan file: check off completed tasks

- [ ] **Verification Phase**
  - [ ] Run full test suite: `npm test && npm run test:e2e`
  - [ ] Fix any failures
  - [ ] Update canonical docs with what was actually built
  - [ ] Update INTEGRATION.md with cross-feature dependencies
  - [ ] Create PR with summary of changes

- [ ] **Ship Phase**
  - [ ] Merge to main (after PR review)
  - [ ] Verify post-merge tests pass
  - [ ] Move completed plan to `docs/plans/archive/`
  - [ ] Update `docs/canonical/_index.md` if new doc created

---

## Summary

A well-configured AI development environment provides:

1. **Clear context** via CLAUDE.md, AGENTS.md, and Cursor Rules
2. **Structured documentation** (canonical docs, plan files) that agents can read
3. **Deterministic gates** (pre-commit hooks, post-merge validation) that catch errors automatically
4. **Granular commits** (one per task) that serve as checkpoints for failure recovery
5. **Parallel capability** (git worktrees) to run multiple features concurrently
6. **Living error logs** that improve over time as you discover patterns
7. **Durable institutional memory** so every session builds on previous work

This environment amplifies AI reasoning and makes AI-assisted development reliable, repeatable, and scalable.

---

## Sources

**Foundational Research:**
- Thoughtworks: "Spec-Driven Development" (2025) — Written specifications prevent 70% of rework.
- Spectro Cloud: "Will AI Turn 2026 Into the Year of the Monorepo?" — Monorepo advantages for AI agent workflows.
- Anthropic: "2026 Agentic Coding Trends Report" — Context editing improved agent performance by 29%; memory + context editing by 39%.

**Context File Patterns:**
- Boris Cherny (Head of Claude Code, Anthropic) — CLAUDE.md workflow, "living error log" pattern.
- HumanLayer: "Writing a Good CLAUDE.md" (2025) — Keep context files lean; never send an LLM to do a linter's job.
- AI Hero: "A Complete Guide to AGENTS.md" — Document capabilities, not file paths; portable open standard.
- GitHub Blog: "How to Write a Great agents.md" — Analysis of 2,500+ repositories; patterns that work.
- Builder.io: "Improve Your AI Code Output with AGENTS.md" — AGENTS.md as portable agent instructions.
- Matthew Groff: "Implementing CLAUDE.md and Agent Skills" — Progressive disclosure pattern.
- PubNub: "Best Practices for Claude Code Subagents" — Specialized agent roles with scoped permissions.

**Workflow Patterns:**
- Agentic Workflow Guide (internal) — Diagnose → Plan → Implement workflow, plan file templates, artifact management.
- Parallel Agent Workflows (internal) — Concurrent feature development with git worktrees.
- Deterministic Sandwich (internal) — Pre-flight and post-processing gates for quality assurance.
- Integration Contracts (internal) — Preventing orphaned code through explicit dependency contracts.

**AI Tool Documentation:**
- Cursor: Official Cursor Rules documentation.
- AGENTS.md specification (open standard).
- abhishekray07: "claude-md-templates" (GitHub).
