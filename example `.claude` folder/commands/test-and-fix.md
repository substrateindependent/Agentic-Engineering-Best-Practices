Run the lint + test + typecheck loop in `pepper-v2-app/` until all checks pass cleanly.

**Step 0 — Lint (strict):**
Run `pnpm run lint -- --max-warnings 0` in `pepper-v2-app/`.
If lint fails:
1. Fix every lint error/warning.
2. Re-run `pnpm run lint -- --max-warnings 0`.

**Step 1 — Tests:**
Run `pnpm test` in `pepper-v2-app/`.
If any tests fail:
1. Read the failure output carefully.
2. Identify the root cause (not just the symptom).
3. Fix the root cause in the source or test file.
4. Re-run `pnpm test`.
5. Repeat until all tests pass. Cap at 3 fix attempts per failure — if still failing after 3 tries, report the issue and stop.

**Step 2 — Typecheck:**
Run `pnpm run typecheck` in `pepper-v2-app/`.
If any type errors exist:
1. Fix the type errors.
2. Re-run `pnpm run typecheck`.
3. Re-run `pnpm run lint -- --max-warnings 0` and `pnpm test` to confirm fixes didn't introduce drift/regressions.

**Output**: Report final lint/test/typecheck status, including test count (pass/fail/skip).

This command is used inside `/code` after each piece is implemented, and can be run standalone.
