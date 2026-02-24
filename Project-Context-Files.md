# Project Context Files

**Parent:** [AI-Coding-Best-Practices](AI-Coding-Best-Practices.md)
**Related:** [Context Engineering](Context-Engineering.md) · [Canonical Documentation](Canonical-Documentation.md) · [Externalized State](Externalized-State.md) · [Environment Setup — Full Reference](Environment-Setup-Full-Reference.md)

---

## What Are Project Context Files?

Project context files are persistent, repository-level instruction documents that travel with your code and apply to every AI interaction. They're the single highest-leverage setup investment you can make.

Unlike prompts that live in a chat session (and disappear when the session ends), context files live in your repo. They encode institutional knowledge: your tech stack, architectural patterns, naming conventions, common mistakes, and project-specific constraints. Every time you invoke an AI coding tool — Claude Code, Cursor, Zed with Agents, or any other — these files load automatically.

The impact is dramatic. Teams using well-maintained context files report 30–40% fewer corrections, cleaner integration with existing code, and fewer repeat mistakes. Boris Cherny's team at Anthropic maintains their context files as a shared error log: "Anytime we see Claude do something incorrectly we add it to the CLAUDE.md." Over time, this builds project-specific institutional memory that compounds across every interaction.

---

## The Three Major Formats

Modern AI coding tools support three context file standards. They serve the same purpose but differ in scope and tool compatibility.

### 1. CLAUDE.md (Claude Code)

**Where:** Root of your repo, or per-subdirectory.
**Scope:** Claude Code sessions only.
**Capacity:** ~200K tokens per file; can be split across multiple files.
**Features:** Hierarchical structure, import syntax, scoped rules by file pattern.

CLAUDE.md is Claude Code's native format. It supports:

- **Markdown structure** with wikilinks (`[[link]]`) for navigation between context files.
- **Scoped rules** that apply only to specific file patterns: `rules/**/*.ts` applies only to TypeScript files; `api/**` applies only to files in the api directory.
- **Import syntax** to pull in referenced files: `/// import <path>` includes another file inline.
- **Comments and explanations** because these are human-readable documents, not just data.

The format is production-tested. Anthropic's own projects use CLAUDE.md extensively. The tool is optimized to load and use it efficiently.

### 2. AGENTS.md (Open Standard)

**Where:** Root of your repo.
**Scope:** Any AI agent (Claude Code, Cursor via extension, Zed, Codex, custom agents).
**Format:** YAML frontmatter + markdown sections.
**Portable:** Works across multiple tools by design.

AGENTS.md is the emerging open standard. It was designed to solve the "locked-in to one tool" problem. A single AGENTS.md file works with:

- Cursor (with the AGENTS.md extension)
- Claude Code
- Zed (with agentic support)
- Custom-built agents
- Future tools (because it's standardized)

AGENTS.md typically includes:

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

If you're building for multiple tools or want long-term tool flexibility, AGENTS.md is the way forward.

### 3. Cursor Rules (.cursor/rules/)

**Where:** `.cursor/rules/` directory.
**Scope:** Cursor IDE only.
**Hierarchy:** Global (`global.md`), Project (`.cursor/rules/`), User (`~/.cursor/rules/`).
**Format:** Markdown files, one rule per file or one per category.

Cursor Rules have three levels:

- **Global rules** (Anthropic-maintained) apply to all Cursor projects.
- **Project rules** (`.cursor/rules/`) apply to the current repo.
- **User rules** (`~/.cursor/rules/`) apply to all your projects.

Project-level rules override user-level, which override global. This hierarchy lets teams enforce standards while respecting personal preferences.

Cursor Rules are typically organized by category:

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

---

## File Placement Hierarchy

If you use only one tool, pick the format that matches it. If you use multiple tools or want future flexibility, use a hybrid approach:

```
repo/
  CLAUDE.md              # Primary context for Claude Code (200–300 lines)
  AGENTS.md              # Portable context (200–300 lines, overlaps with CLAUDE.md)
  CLAUDE.local.md        # Personal preferences (.gitignore this)
  .cursor/rules/
    architecture.md
    testing.md
    style.md
  docs/
    canonical/
      _index.md
      feature-1.md
      feature-2.md
```

The symlink strategy works too: `AGENTS.md → CLAUDE.md` (via git symlink) means you maintain one file and both tools see it. This only works if you're not using tool-specific syntax.

For personal preferences, use `CLAUDE.local.md`. Add it to `.gitignore` so it doesn't clutter the team's repo.

---

## Three-Layer Structure: Global, Project, Local

Well-designed context files follow a three-layer pattern:

**Layer 1: Global (Universal)**
Rules that apply to all projects you work on. Examples: your preferred git commit message format, how you structure feature branches, your approach to error handling, code organization philosophy.

These live in `~/.cursorrules` or a personal `CLAUDE.local.md` template you copy into new projects.

**Layer 2: Project (Team Conventions)**
Rules specific to this codebase. Examples: the tech stack, architectural decisions, integration points, testing patterns, deployment process, common gotchas.

These live in `CLAUDE.md` or `.cursor/rules/` and are committed to the repo.

**Layer 3: Local (Personal Quirks)**
Your personal preferences that don't affect the team. Examples: your editor settings, tools you use, how you organize your notes.

These live in `CLAUDE.local.md` or `~/.cursor/rules/` and don't get committed.

The practical benefit: new team members copy the project-level rules and inherit institutional knowledge without needing a 30-minute onboarding chat. Senior team members can layer in personal preferences without diluting the shared standard.

---

## What to Include in Project Context Files

### Core Sections

**1. Project Context (One-liner)**

```
One sentence that captures what this project does and its primary constraint.

Example:
"A React Native mobile app for event booking. Constraint: offline-first, <2MB bundle."
```

The one-liner keeps the AI grounded. Without it, the AI makes assumptions. With it, every decision is filtered through that lens.

**2. Tech Stack & Key Dependencies**

List the technologies, versions, and critical libraries:

```
- Backend: Node.js 18+, Express
- Database: PostgreSQL 14+
- Frontend: React 18, TypeScript 5
- Testing: Jest, React Testing Library, Playwright
- CI/CD: GitHub Actions
```

Include version constraints. AI makes different recommendations for different versions (React 16 vs. 18 are *very* different).

**3. Architecture & Project Structure**

Describe how the code is organized:

```
src/
  components/      # Functional React components, colocated styles
  services/        # External API clients, business logic
  hooks/           # Custom React hooks, reusable logic
  types/           # Shared TypeScript types
  utils/           # Utility functions, helpers
  tests/           # Test files (*.test.ts, *.test.tsx)
  styles/          # Global styles, design tokens
```

Include decision rationale: "Why do we colocate styles with components?" helps the AI understand the philosophy, not just the pattern.

**4. Code Style & Naming Conventions**

Examples are better than descriptions:

```
✓ DO: const getTotalPrice = () => { }
✗ DON'T: const get_total_price = () => { }

✓ DO: interface UserProfile { firstName: string; }
✗ DON'T: interface IUserProfile { }

✓ DO: const user = await fetchUser(id);
✗ DON'T: const user = await get_user(id);
```

A dozen concrete examples replace a hundred words of rules.

**5. Testing Requirements & Patterns**

Specify what gets tested and how:

```
- Unit tests for business logic (services/, utils/)
- Component tests for interactive UI (React Testing Library)
- E2E tests for critical user flows (Playwright)
- Minimum coverage: 80% on services/, 60% on components/
- Test location: adjacent to source (feature.ts → feature.test.ts)
```

Include examples of a good test and a bad test for the same code.

**6. Common Patterns You Follow**

Show, don't tell:

```
✓ API error handling pattern:
try {
  const data = await api.fetchUser(id);
  return { ok: true, data };
} catch (err) {
  logger.error('fetchUser failed', { id, err });
  return { ok: false, error: err.message };
}

✓ React component pattern:
export const UserCard = ({ userId }: Props) => {
  const user = useUser(userId);
  if (user.loading) return <Skeleton />;
  return <div>{user.data.name}</div>;
};
```

Patterns are templates. If the AI knows your template, it will follow it.

**7. Project-Specific Constraints**

What's special or tricky about *this* codebase?

```
- No async/await in middleware (use .then chains)
- All API endpoints must support ETags
- Database migrations are immutable (never edit, only create new)
- CSS variables must be prefixed with --app- (no --component-)
- SVG imports must use react-svg-loader
```

These are gotchas. Without them, the AI will write code that conflicts with your constraints and you'll have to fix it.

---

## What NOT to Include

### Avoid These Anti-Patterns

**Formatting Rules**

Don't list formatting rules in context files. Use linters.

```
✗ DON'T:
"Always use 2 spaces for indentation"
"Always add a blank line between imports and code"
```

✓ DO: Configure Prettier + ESLint. The linter enforces it automatically. The AI will follow whatever the linter does.

**Rules You Don't Enforce**

Every rule in your context file should be something you actually enforce. If you don't catch violations during code review, don't document them.

```
✗ DON'T:
"Always write JSDoc comments above functions"
(unless you actually enforce this)

✓ DO:
"JSDoc is encouraged but not required"
(honest about the standard you maintain)
```

**Overly Specific Instructions**

Context files should capture principles, not micro-manage implementation.

```
✗ DON'T:
"When fetching user data, call getUserById() with the userId parameter,
then check the response status, then log the result..."

✓ DO:
"Use the getUserById() helper from services/. It handles logging internally."
```

**Duplicate Rules**

If you have CLAUDE.md and AGENTS.md and .cursor/rules, keep them in sync. Conflicting rules confuse the AI.

Best practice: maintain one primary file (say, CLAUDE.md) and symlink or import the others.

---

## The Living Error Log Pattern

Boris Cherny's team at Anthropic treats their CLAUDE.md as a shared mistake journal. Every time the AI makes a mistake, they add it to CLAUDE.md so it won't repeat.

**Example Workflow:**

1. AI writes code that violates a pattern.
2. During code review, you catch it.
3. You tag the error: "This is a *pattern violation* — we call the logger differently."
4. You add a two-line example to CLAUDE.md showing the right pattern.
5. Next time the AI writes similar code, it includes the correct pattern.

Over 10–20 features, this builds substantial institutional memory. The time investment is minimal (1–2 minutes per error), and the compounding benefit is dramatic.

**How to structure it:**

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

The "(Learned: PR #XX)" tag documents where the lesson came from. Over time, you have a paper trail of lessons.

---

## Keeping It Concise

The sweet spot is **100–300 lines**. Under 100 lines feels insufficient. Over 500 lines approaches "manual to memorize" territory, and the AI's attention starts to degrade.

**How to keep it concise:**

- One example per pattern, not five.
- Link to longer docs rather than inlining them.
- Remove outdated rules aggressively.
- Use per-folder context files for topic-specific instructions.

Per-folder context files are underused. Instead of a 500-line root CLAUDE.md, you could have:

```
CLAUDE.md                    # Global rules (100 lines)
src/backend/CLAUDE.md        # Backend-specific (80 lines)
src/frontend/CLAUDE.md       # Frontend-specific (80 lines)
docs/CLAUDE.md               # Documentation rules (40 lines)
```

Each file is focused and scannable. Claude Code loads the relevant file when working in that directory.

---

## Progressive Disclosure: Scoped Rules

CLAUDE.md and Cursor Rules support file-path scoping. Use it.

```markdown
## TypeScript Rules (*.ts, *.tsx)

Strict mode: all values must be typed, no `any`.

✓ const x: number = 5;
✗ const x: any = 5;

## JavaScript Rules (*.js, *.mjs)

Looser standards (e.g., dynamic typing allowed) for scripts and config files.

## SQL Rules (**/*.sql)

All queries use parameterized statements, never string interpolation.
```

When the AI is working on a TypeScript file, it sees the TS rules. When it's in a `.sql` file, it sees the SQL rules. No context waste.

Cursor Rules make this explicit with file paths:

```
.cursor/rules/
  typescript.md      # Applies to *.ts, *.tsx
  sql.md             # Applies to *.sql
  scripts.md         # Applies to *.js, *.mjs in scripts/
```

---

## Unifying Across Tools

If you work across Claude Code, Cursor, and custom agents, you need a strategy for keeping context files in sync.

**Option 1: Single Source (Symlink)**

```bash
# AGENTS.md is the source of truth
ln -s AGENTS.md CLAUDE.md
```

Both Claude Code and the AGENTS.md extension read the same file. One edit, two tools see it.

Limitation: you can't use tool-specific syntax. But most context is tool-agnostic, so this works.

**Option 2: Multiple Files (Import Syntax)**

```
CLAUDE.md
AGENTS.md
.cursor/rules/architecture.md

# In CLAUDE.md:
/// import AGENTS.md
/// import .cursor/rules/architecture.md
```

CLAUDE.md includes the content of other files. Single source of truth, one file to edit.

Limitation: imports only work in CLAUDE.md, not Cursor Rules.

**Option 3: Separate Maintenance**

Keep each tool's context file separate. Use a checklist to ensure they stay in sync:

```
## Context File Sync Checklist
- [ ] Updated CLAUDE.md
- [ ] Updated AGENTS.md with same info
- [ ] Updated .cursor/rules/
- [ ] Tested in Claude Code
- [ ] Tested in Cursor
```

More manual, but if you need tool-specific syntax, it's necessary.

---

## The # Shortcut in Claude Code

Claude Code has a shortcut for capturing patterns during sessions. When you see code that embodies a pattern worth remembering, use `#` to flag it:

```
# This is a good pattern for handling errors in async operations
```

At the end of the session, you can grep for all the `#` comments and decide what should go into CLAUDE.md.

This is lightweight note-taking for patterns you discover mid-session.

---

## Iterative Maintenance

Context files aren't write-once artifacts. They evolve as your codebase evolves.

**After every code review:**

- Did the AI violate a pattern you've documented? Add it to the error log.
- Did you invent a new pattern? Document it so the next AI session knows.
- Are there outdated rules? Remove them.

**After every feature completion:**

- Update canonical documentation (see [Canonical Documentation](Canonical-Documentation.md)).
- Capture any architectural decisions that affect future work.
- Note any gotchas discovered during implementation.

**Monthly (or quarterly):**

- Review your context files. Remove rules that aren't enforced.
- Consolidate redundant entries.
- Add index links between related documents.

Teams that treat this as a living process report that within 3–4 features, the context files become so effective that AI code requires minimal review.

---

## Templates & Resources

### Starter Templates

- **abhishekray07/claude-md-templates** — Ready-made CLAUDE.md templates for popular stacks (React, Node, Python, Rust).
- **Anthropic Official Guide** — [Project Context Files](Project-Context-Files.md) from Anthropic's documentation.

### From Production

Real examples from working projects:

- **Claude Code team** — Anthropic's internal CLAUDE.md (sanitized for public use).
- **Cursor templates** — Community Cursor Rules for common stacks.

### Creating Your Own

Start with the structure in this document. Pick three rules that matter most to your project. Write examples. Add the others over time.

---

## Anti-Patterns to Avoid

**The 1,000-Line Monster File**

A context file that spans 1,000 lines becomes a burden. It's too large to review. The AI can't focus on what matters. Split it into multiple files.

**Tool-Specific Lock-In**

Writing context files in tool-specific syntax (e.g., only Cursor Rules syntax) locks you into one tool. Use AGENTS.md for portability, tool-specific formats only for advanced features you actually need.

**Set It and Forget It**

Updating context files once at project setup, then never again. Context rot happens. Rules become outdated. As your project evolves, your context files must evolve.

**Conflicting Rules Across Tools**

Having one pattern documented in CLAUDE.md and a different pattern in Cursor Rules. The AI gets confused. Contradiction is worse than missing rules.

**Rules You Don't Enforce**

If you don't catch violations during code review, don't document them. Fake rules train the AI to ignore documentation.

---

## Getting Started: Five Steps

1. **Write your one-liner.** One sentence describing the project and its primary constraint.

2. **List your tech stack.** What technologies, versions, and critical libraries? This takes 10 minutes.

3. **Document three core patterns.** Show one example of each: how you handle errors, how you organize components, how you structure database queries.

4. **Capture three anti-patterns.** What should the AI *never* do? What do you always have to fix?

5. **Add to version control.** Commit CLAUDE.md or AGENTS.md to your repo. Update it in the next code review cycle.

That's it. A working context file in 30 minutes. Add to it as you learn.

---

## Sources

- Boris Cherny (Head of Claude Code, Anthropic) — CLAUDE.md workflow, "living error log" pattern.
- Anthropic, [Project Context Files](Project-Context-Files.md) (official guide).
- HumanLayer, "Writing a good CLAUDE.md" (2025).
- Cursor, official Cursor Rules documentation.
- AGENTS.md specification (open standard).
- abhishekray07, "claude-md-templates" (GitHub).
