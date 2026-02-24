# Environment Setup — Full Reference

**Author:** Glenn Clayton, Fieldcrest Ventures
**Version:** 1.0 — February 2026
**Reference Document:** Companion to "AI-Assisted Coding: Best Practices Guide"

---

## Overview: Environment as Quality Ceiling

Your development environment determines the ceiling for AI-assisted coding quality. A well-configured environment with clear context artifacts, automated verification gates, and structured project documentation amplifies AI reasoning. A poorly configured one forces the AI to guess at architectural details, discover patterns through reverse-engineering, and second-guess its own decisions.

This is not about tool choice (Cursor vs. Claude Code vs. Aider). It's about what context you provide, how you structure decisions, and what guardrails you build into the workflow.

The sections below cover:
1. **Repo structure** — Where things live and why
2. **Context files** — Project-level instructions that persist across sessions
3. **Git workflow** — Branching, commits, and checkpoints
4. **Hook configuration** — Automated gates before and after AI edits
5. **Build plan formats** — Dual-format task lists for readability and automation
6. **Parallel workflows** — Running multiple agent sessions safely
7. **Getting started checklist** — Setup steps in order

---

## 1. Recommended Repo Structure

A well-organized repo mirrors how AI agents discover information. Group canonical docs, build plans, and project context in predictable locations.

```
your-project/
├── CLAUDE.md                      # Project context for Claude Code
├── AGENTS.md                      # Open standard agent instructions
├── .cursor/
│   └── rules/
│       ├── architecture.md        # Architectural patterns
│       ├── coding-standards.md    # Style, naming, type-checking
│       └── anti-patterns.md       # Common mistakes to avoid
├── docs/
│   ├── canonical/
│   │   ├── _index.md              # Index of all canonical docs
│   │   ├── authentication.md      # Auth system, data model, integration points
│   │   ├── api-layer.md           # REST/GraphQL APIs, contracts
│   │   ├── database-schema.md     # Current schema, migrations, relationships
│   │   ├── component-library.md   # Reusable UI components, patterns
│   │   └── [feature-name].md      # One doc per major feature area
│   ├── build-plans/
│   │   ├── 001-feature-name.md    # [ ] Task 1; [x] Task 2; etc.
│   │   ├── 002-feature-name.md
│   │   └── _progress.md           # Overall feature completion tracker
│   ├── architecture/
│   │   ├── decision-log.md        # ADRs (Architecture Decision Records)
│   │   └── system-design.md       # High-level system overview
│   └── onboarding/
│       ├── development-setup.md   # How to set up the dev environment
│       └── deployment-guide.md    # How to deploy to staging/prod
├── src/
│   ├── [feature-1]/
│   ├── [feature-2]/
│   └── ...
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .git/
│   └── hooks/
│       ├── pre-commit             # Linting, type-checking, fast tests
│       └── post-merge             # Smoke tests, environment validation
├── package.json (or equivalent)
└── .gitignore
```

**Why this structure:**

- **Canonical docs** (`docs/canonical/`) are the single source of truth. Agents read these to understand existing code without parsing files.
- **Build plans** (`docs/build-plans/`) are audit trails. Checkboxes show progress; the markdown is committable and traceable.
- **Cursor Rules and CLAUDE.md** live at the root so tools discover them automatically.
- **Architecture docs** (`docs/architecture/`) capture decisions (ADRs) to prevent repeated discussions.
- **Hooks** (`.git/hooks/`) run deterministically before and after every commit.

---

## 2. Context Files: CLAUDE.md, AGENTS.md, and Cursor Rules

These are your highest-leverage setup investment. Each persists across sessions and shapes every AI interaction.

### CLAUDE.md (Claude Code)

```markdown
# Project Context — [Project Name]

## Quick Facts
- **Tech Stack:** TypeScript + React 18 + Node.js 18 + PostgreSQL
- **Package Manager:** npm (monorepo: pnpm is used)
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

## Common Mistakes We've Made (Living Error Log)
1. **Mistake:** Forgetting to add `NOT NULL` constraints on new database columns.
   **Fix:** Always specify `nullable: false` in migrations. AI often misses this.
2. **Mistake:** Building UI features without updating the Redux schema.
   **Fix:** Update Redux models first, then build the UI.
3. **Mistake:** Using `setTimeout` for async operations instead of proper async/await.
   **Fix:** Use async/await everywhere. No setTimeout for actual async work.
4. [Add more as you discover them.]

## Project Constraints
- **Bundle size:** Keep <500KB gzipped. Check with `npm run build && gzip -c dist/main.js`.
- **Accessibility:** All interactive elements must have keyboard nav + ARIA labels.
- **Mobile:** Responsive design required. Test on 375px width minimum.
- **Performance:** Lighthouse score >80 on desktop, >75 on mobile.

## How to Ask for Help
When stuck, provide: the current file, what you've tried, what's not working. If an error occurs, include the full stack trace.
```

**Keep it to <500 lines.** Longer documents don't get read. Use per-folder `.claude.md` files for topic-specific details.

### AGENTS.md (Open Standard)

The AGENTS.md file is an emerging open standard supported by Claude Code, Cursor, and Devin. It's simpler than CLAUDE.md but more portable.

```markdown
# Agent Instructions — [Project Name]

## Project Summary
[One sentence describing what the project does.]

## Critical Rules
1. Never mutate existing objects. Always create new ones.
2. Run the test suite before committing: `npm test`.
3. Write tests for all new functions and components.
4. Check type safety: `npm run type-check` must pass.

## Tech Stack
- **Language:** TypeScript
- **Frontend:** React 18
- **Backend:** Node.js + Express
- **Database:** PostgreSQL
- **Testing:** Jest + React Testing Library

## Coding Style
- 2-space indentation
- 100-character line limit
- Use Prettier for formatting
- Use ESLint for linting

## Key Files
- `src/app.tsx` — Main application entry point
- `src/api/` — API client and endpoints
- `src/components/` — UI components
- `src/hooks/` — Custom React hooks
- `db/schema.sql` — Database schema

## When Adding Features
1. **Plan:** Create a feature plan with acceptance criteria.
2. **Implement:** Update database schema first (if needed), then implement.
3. **Test:** Write tests. Run full suite.
4. **Document:** Update `docs/canonical/` with integration points and changes.
5. **Commit:** One commit per task. Descriptive message.
```

### Cursor Rules (`.cursor/rules/`)

Cursor Rules are automatically loaded by Cursor for every AI interaction. Keep them focused — one file per topic.

**`.cursor/rules/architecture.md`:**
```markdown
# Architecture Rules

## Data Flow
- Components dispatch Redux actions
- Actions trigger thunks (async operations)
- Thunks call API services
- API services use axios with auth interceptor
- Responses are normalized and stored in Redux
- Components subscribe to Redux selectors

## File Organization
- Every feature gets its own folder: `src/features/{feature-name}/`
- Within that folder: `components/`, `hooks/`, `types.ts`, `index.ts`
- Shared code goes in `src/shared/`

## Component Composition
- All components are functional
- Use hooks for state management (local state)
- Use Redux for global state
- Wrap async operations in React.Suspense (where applicable)
```

**`.cursor/rules/coding-standards.md`:**
```markdown
# Coding Standards

## TypeScript
- No `any` types. Use `unknown` if truly unknown, then narrow.
- All function parameters and returns must be typed.
- Use generics for reusable components.

## React
- All components receive props as a typed object: `interface Props { ... }`
- Use `React.FC<Props>` or functional signature.
- Memoize expensive components: `React.memo(Component)`

## Testing
- Test file lives next to source: `Component.tsx` + `Component.test.tsx`
- Test behavior, not implementation. Query by role, not by ID.
- Example: `screen.getByRole('button', { name: /submit/i })`

## Async Operations
- Always use try/catch for async code
- Always dispatch error actions on failure
- Always show user feedback (toast, modal, inline message)
```

**`.cursor/rules/anti-patterns.md`:**
```markdown
# Anti-Patterns (What NOT to Do)

1. **Direct Redux mutation.** Redux reducers must create new state, not mutate.
2. **Uncaught promises.** Every `.then()` needs error handling or a `.catch()`.
3. **Prop drilling.** If props go deeper than 2 levels, use Redux or Context instead.
4. **No loading states.** Every async operation needs a loading state indicator.
5. **Testing implementation details.** Don't test internals; test what the user sees.
6. **Hardcoded strings.** Use constants or i18n. No magic strings.
```

---

## 3. Git Workflow for AI-Assisted Development

### Branching Strategy

```
main (production-ready, all tests passing)
  ↓
feature/{epic-name}  ← One branch per feature
  ├── task/001-auth-login     [One commit per task]
  ├── task/002-auth-logout
  ├── task/003-password-reset
  └── [tasks complete]
    ↓
  [Run full test suite]
    ↓
  Merge to main via PR
```

**Naming convention:**
- `feature/{epic-name}` — Main branch for a feature or epic
- `task/{task-number}-description` — Optional: track individual tasks as commits (labels, not branches)

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

**AI can generate these.** During the build phase, after each task completes, ask the AI to commit with a descriptive message. Example prompt:

> Write a git commit message for this task. Include what was implemented, what was tested, and any decisions made. Then commit with: `git commit -m "message"`

### One Commit Per Task

During the build phase (Step 8 in the Development Loop), commit after each task completes. This is your checkpoint system.

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

## 4. Hook Configuration: Deterministic Gates

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

### PostToolUse Hook (Cherny's Pattern)

Some tools (like Claude Code) support PostToolUse hooks that run after the AI edits files. Use this to auto-format.

If your tool supports it:

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

## 5. Build Plan Formats: Dual-Format Task Lists

Store build plans in two formats: readable markdown (for humans) and structured JSON (for automation).

### Markdown Format (for humans to read)

**`docs/build-plans/001-authentication.md`:**
```markdown
# Build Plan: Authentication Feature

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

### Structured Format (for automation)

**`docs/build-plans/001-authentication.json`:**
```json
{
  "epicName": "authentication",
  "epicId": "001",
  "description": "User authentication (signup, login, password reset)",
  "acceptanceCriteria": [
    "Users can sign up with email + password",
    "Users can log in with email + password",
    "Sessions persist across page reloads",
    "Password reset works end-to-end",
    "All credentials hashed (bcrypt)",
    "All E2E tests pass"
  ],
  "tasks": [
    {
      "id": 1,
      "title": "Create users table",
      "status": "pending",
      "files": {
        "create": ["db/migrations/001_add_users_table.sql"],
        "modify": []
      },
      "testCommand": "npm run db:migrate",
      "completed": false
    },
    {
      "id": 2,
      "title": "Implement auth service",
      "status": "pending",
      "files": {
        "create": ["src/api/auth-service.ts", "src/api/auth-service.test.ts"],
        "modify": []
      },
      "testCommand": "npm test -- src/api/auth-service.test.ts",
      "completed": false
    }
  ],
  "integrationContract": {
    "filesModified": ["src/redux/store.ts", "src/api/"],
    "filesCreated": ["src/api/auth-service.ts", "src/components/LoginForm.tsx"],
    "apiContracts": [
      { "method": "POST", "path": "/api/v1/auth/signup" },
      { "method": "POST", "path": "/api/v1/auth/login" }
    ],
    "dataModels": ["users (id, email, password_hash, created_at)"]
  }
}
```

**Update the markdown as work progresses.** Check off tasks with `[x]`. This becomes your audit trail.

---

## 6. Parallel Agent Workflows (Worktrees)

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

Your job: keep sessions unblocked. When one waits (e.g., waiting for your review), start another.

### Merge Discipline

Merge features one at a time through the full verify-and-ship process (Steps 12–15 in the Development Loop).

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

Never batch-merge features. Each needs independent verification.

---

## 7. Environment Variables and Secrets

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

**Do NOT ask the AI to enter secrets.** If a build plan or task requires an API key, tell the AI: "Use environment variable `STRIPE_KEY` which is set on the system."

---

## 8. CI/CD Integration

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
- You see exactly which commit broke the build (if it does).

---

## 9. Tool Selection Guide

Different tools have different strengths. Choose based on your workflow:

| Tool | Best For | Notes |
|------|----------|-------|
| **Claude Code** | CLI development, batch operations, long research tasks | Runs in terminal. Full codebase access. Good for planning and deep work. |
| **Cursor** | Interactive real-time editing, UI coding | IDE integration. Automatic Cursor Rules loading. Good for immediate feedback. |
| **Aider** | Git-aware code editing, multi-file refactors | Understands git context. Commits automatically. CLI-based. |
| **Windsurf** | Agentic flows, autonomous sessions | Newer tool. Designed for longer, more autonomous workflows. |

**Our recommendation:** Use Claude Code for planning and architectural decisions (frontier model). Use Cursor or Aider for implementation (faster iteration). Both read your context files automatically.

---

## 10. Getting Started Checklist

Set up a new project for AI-assisted development in this order:

- [ ] **1. Create repo structure**
  - [ ] Create `docs/canonical/`, `docs/build-plans/`, `docs/architecture/`
  - [ ] Create `.cursor/rules/` with architecture, standards, anti-patterns
  - [ ] Create `.git/hooks/` directory

- [ ] **2. Write context files**
  - [ ] Write `CLAUDE.md` (project context, patterns, anti-patterns, naming conventions)
  - [ ] Write `AGENTS.md` (tech stack, rules, key files)
  - [ ] Write `.cursor/rules/architecture.md`, `coding-standards.md`, `anti-patterns.md`

- [ ] **3. Configure git workflow**
  - [ ] Set up `pre-commit` hook (linting, type-check, unit tests)
  - [ ] Set up `post-merge` hook (smoke tests)
  - [ ] Commit hook files: `git add .git/hooks/ && git commit -m "Add git hooks"`

- [ ] **4. Set up tests**
  - [ ] Create `tests/unit/`, `tests/integration/`, `tests/e2e/` directories
  - [ ] Write smoke test E2E test (basic app startup)
  - [ ] Verify `npm test` runs unit tests and `npm run test:e2e` runs E2E tests

- [ ] **5. Initialize canonical docs**
  - [ ] Write `docs/canonical/_index.md` (index of all canonical docs)
  - [ ] Write `docs/canonical/architecture.md` (high-level system design)
  - [ ] Write `docs/canonical/database-schema.md` (current schema)

- [ ] **6. Start your first feature**
  - [ ] Create feature branch: `git checkout -b feature/first-feature`
  - [ ] Create build plan in `docs/build-plans/001-first-feature.md`
  - [ ] Invoke Claude Code with the build plan

---

## Checklist for Each New Feature

When starting a new feature, use this checklist:

- [ ] **Research Phase (Frontier Model)**
  - [ ] Read relevant canonical docs to understand existing code
  - [ ] Identify data models, API contracts, integration points
  - [ ] Write research document (canonical doc) with approach

- [ ] **Planning Phase**
  - [ ] Decompose canonical doc into buildable tasks
  - [ ] Create integration contract (files modified, created, API contracts, data model)
  - [ ] Write build plan in markdown and JSON formats
  - [ ] Commit both files: `git add docs/build-plans/ && git commit`

- [ ] **Build Phase**
  - [ ] Create feature branch: `git checkout -b feature/{name}`
  - [ ] Implement tasks one by one
  - [ ] Commit after each task: `git commit -m "[TASK] N: description"`
  - [ ] Run unit tests after each task
  - [ ] Update build plan: check off completed tasks

- [ ] **Verify Phase**
  - [ ] Run full test suite: `npm test && npm run test:e2e`
  - [ ] Fix any failures
  - [ ] Update canonical docs with what was actually built
  - [ ] Create PR with summary of changes

- [ ] **Ship Phase**
  - [ ] Merge to main (after PR review)
  - [ ] Verify post-merge tests pass
  - [ ] Update `docs/canonical/_index.md` if new doc created

---

## Summary

A well-configured AI development environment provides:

1. **Clear context** via CLAUDE.md, AGENTS.md, and Cursor Rules
2. **Structured documentation** (canonical docs, build plans) that agents can read
3. **Deterministic gates** (pre-commit hooks, post-merge validation) that catch errors automatically
4. **Granular commits** (one per task) that serve as checkpoints for failure recovery
5. **Parallel capability** (git worktrees) to run multiple features concurrently
6. **Living error logs** that improve over time as you discover patterns

This environment amplifies AI reasoning and makes AI-assisted development reliable and repeatable.

---

**Next Steps:**
- Read [[Development Loop — Full Reference]] for the step-by-step process
- Read [[Integration Contracts]] for how to prevent orphaned code
- Read [[Parallel Agent Workflows]] for advanced orchestration patterns
