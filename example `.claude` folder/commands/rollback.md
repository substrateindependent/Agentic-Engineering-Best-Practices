You are running a rollback to recover from a failed or incorrect implementation piece.

**Ask the user**: "What should I roll back? Options: (1) **last piece** — reverts the most recent commit, or (2) **full reset** — resets to the base commit from before `/implement` started. Which mode?"

Once the user chooses a mode, follow the appropriate procedure below.

---

## Mode 1: Last-Piece Revert

Use this when the most recent piece introduced a problem but earlier pieces are fine.

**Step 1 — Confirm the target.**

Run `git log --oneline -5` to show recent commits. Identify the HEAD commit that will be reverted. Display it to the user:

> "I will revert commit `<SHA>` (`<message>`). This creates a new commit that undoes those changes. Your earlier work is preserved. Proceed? (yes/no)"

Wait for the user to confirm before proceeding.

**Step 2 — Revert.**

Run `git revert HEAD --no-edit`.

**Step 3 — Post-rollback guidance.**

Tell the user:
- The revert commit has been created. Your git history is intact and safe.
- Open the plan file and uncheck the piece that was reverted (change `[x]` back to `[ ]`).
- Add a note in the plan's Current State section describing why the piece was rolled back.
- Re-run `/implement` to retry the failed piece in a fresh session.

---

## Mode 2: Full Reset

Use this when multiple pieces went wrong and you need to return to the clean state before `/implement` started.

**Step 1 — Find the base commit.**

Read the plan file the user is working on. Look for `Base commit:` in the Current State section. Extract the SHA.

If no base commit is recorded, tell the user: "No base commit found in the plan file. I cannot perform a full reset without a known safe point. Please provide the SHA to reset to, or use Mode 1 to revert individual commits."

**Step 2 — Record the escape hatch.**

Run `git rev-parse HEAD` and record the current HEAD SHA. Display it to the user:

> "Before resetting, I am recording your current HEAD as an escape hatch:"
> ```
> git checkout <current-HEAD-SHA>
> ```
> "If you need to recover any work from the current state, use the command above."

**Step 3 — Confirm the reset.**

Show the user what will happen:

> "I will hard-reset to base commit `<base-SHA>`. This will discard all commits made since that point. The escape hatch SHA above can recover the discarded work if needed. Proceed? (yes/no)"

Wait for the user to confirm before proceeding.

**Step 4 — Reset.**

Run `git reset --hard <base-SHA>`.

**Step 5 — Post-rollback guidance.**

Tell the user:
- The repository has been reset to the state before `/implement` started.
- Open the plan file and uncheck all pieces that were rolled back (change `[x]` back to `[ ]`).
- Update the plan's Current State section to note the reset and any lessons learned.
- If the plan itself was flawed, consider running `/diagnose` again before re-running `/plan` and `/implement`.
- If only the implementation was flawed, re-run `/implement` in a fresh session.
- The escape hatch SHA (`<current-HEAD-SHA>`) is available if you need to cherry-pick any good work from the discarded commits.

---

**Important rules:**
- NEVER proceed with a revert or reset without explicit user confirmation.
- NEVER run `git reset --hard` without first recording and displaying the escape hatch SHA.
- This command does not modify the plan file itself — instruct the user to update checkboxes and Current State manually (or in a follow-up session).
- If the user is unsure which mode to use: recommend Mode 1 (last-piece revert) if only the latest piece is problematic, and Mode 2 (full reset) if the implementation has gone off track across multiple pieces.
