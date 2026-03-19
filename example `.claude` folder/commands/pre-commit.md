Final quality gate before pushing. Run all checks in `pepper-v2-app/` in this exact order:

1. **Lint**: Run `pnpm run lint -- --max-warnings 0`. Fix all errors and warnings before proceeding.
2. **Typecheck**: Run `pnpm run typecheck`. Fix all type errors before proceeding.
3. **Test**: Run `pnpm test`. Fix all failures before proceeding.
4. **Diff review**: Run `git diff` and review for AI slop:
   - Unnecessary comments or over-documentation
   - Over-abstraction (wrappers that add no value)
   - Dead code or unused imports introduced in this diff
   - `console.log` statements that should be removed
   - Files that grew beyond 300 lines
   - Cross-pillar import violations (see `CLAUDE.md` architecture rules)

Fix any issues found in step 4, then re-run lint/typecheck/test to confirm the fixes are clean. Lint is strict and must pass with zero warnings (`pnpm run lint -- --max-warnings 0`).

5. **Dependency audit**: Run `pnpm audit --audit-level=high`. Review the output for known high/critical vulnerabilities in dependencies. **Warn prominently but do not block** — upstream issues are often outside our control. Report any findings in the output summary so the developer is aware.

6. **Secrets scan**: Scan the staged diff (`git diff --cached`) for accidentally committed secrets. **Block on any findings** — do not proceed until resolved.

   Check for these patterns in the staged diff:
   - API key prefixes: `sk-`, `ghp_`, `AKIA`, `xox[bpras]-`
   - Secret-named variable assignments to string literals (e.g., `SECRET = "..."`, `apiKey = '...'`, `password = "..."`), but **not** assignments to `process.env.*`
   - `.env` files being staged (but **not** `.env.example` or `.env.*.example` files)
   - Embedded credential URLs (e.g., `https://user:password@host`, connection strings with inline credentials)

   **Exclusions** — skip matches found in:
   - `*.test.ts` and `*.spec.ts` files (test fixtures may contain fake keys)
   - `*.example` files
   - Lines that are clearly example/placeholder comments (e.g., `// example: sk-your-key-here`, `# replace with your token`)

   If any real secret is detected, report the file, line, and matched pattern, then **stop**. The developer must remove the secret, rotate it if it was ever committed, and re-stage before continuing.

**Output**: Report the final status of each check (pass/fail/warn) and summarize any manual fixes applied.

This command is run manually after `/implement` completes, as the last step before pushing.
