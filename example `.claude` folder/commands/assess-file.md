Analyze the file specified by the user for structural quality issues.

Read the target file, then evaluate it against these criteria:

1. **Size**: Line count vs the 300-line soft / 500-line hard thresholds from `pepper-v2-app/docs/conventions.md`.
2. **Cyclomatic complexity**: Identify functions with deeply nested branches or long conditional chains.
3. **Dead code**: Unreachable branches, unused exports, commented-out blocks.
4. **Duplicate logic**: Repeated patterns that could be extracted into a shared helper.
5. **Inconsistent patterns**: Compare error handling, logging, and import style against neighboring files in the same feature directory.
6. **Decomposition opportunities**: Suggest concrete extractions (function, module, or file-level) with stable interface boundaries.
7. **Lint drift risk**: Flag patterns likely to violate zero-warning lint gates (unused imports/vars, stale eslint-disable comments, debug residue).

**Output format**: A structured report with one section per criterion. Each finding must include the file path, line number(s), and a one-sentence explanation. End with a prioritized list of recommended actions.

This command is typically used during `/diagnose` to scope work, or standalone for ad-hoc file review.
