Review the current git diff for code quality issues.

Run `git diff` (include staged changes with `git diff HEAD` if needed) and check for:

1. **Unused imports**: Any imports added but never referenced.
2. **Debug residue**: `console.log`, `console.debug`, or `debugger` statements that should be removed.
3. **Error handling**: Inconsistent patterns — compare against `pepper-v2-app/docs/conventions.md` section 3 (typed AppError, no silent swallowing).
4. **File size**: Any file that grew beyond 300 lines in this diff.
5. **Cross-pillar violations**: Imports that bypass the `@/features/<pillar>/index.ts` public API boundary.
6. **Dead code**: Functions, variables, or branches added but unreachable or unused.
7. **Style drift**: Logging without structured prefixes, raw `Error` throws instead of `AppError`, or `any`/`require()` introduced.
8. **Lint gate alignment**: Confirm `pnpm -C pepper-v2-app run lint -- --max-warnings 0` passes with zero warnings.

**Output format**: List each finding with the file path, line reference, and a one-sentence description. Group findings by category. End with a pass/fail summary.

This command is used inside `/verify` and can also be run standalone.
