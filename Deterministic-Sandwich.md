# Deterministic Sandwich

**Parent:** [[AI-Coding-Best-Practices#10. The Deterministic Sandwich|AI-Coding Best Practices]]
**Related:** [[Agent Self-Verification]] · [[Context Engineering]] · [[Integration Contracts]] · [[Externalized State]]

---

## What Is the Deterministic Sandwich?

The Deterministic Sandwich is the foundational architecture for any AI-assisted operation: **deterministic pre-processing → LLM reasoning → deterministic post-processing.**

Input validation, file path resolution, dependency checking, environment verification, code formatting, type-checking, and output validation do not benefit from LLM flexibility — and they actively suffer from LLM non-determinism. The pattern isolates these hard-boundary checks into deterministic code, letting the LLM focus exclusively on genuinely ambiguous decisions: architectural choices, implementation strategy, code generation, and natural-language interpretation.

The Deterministic Sandwich is the practical foundation of [[AI-Coding-Best-Practices#7. The Integration Contract|the Integration Contract]] principle and a concrete implementation of [[Context Engineering]]. It reduces the surface area for LLM unpredictability while preserving reasoning capability where it matters most.

---

## Why It Works

LLMs excel at interpretation and reasoning under ambiguity. They are profoundly poor at deterministic tasks that depend on exact file paths, schema validation, and state verification. The distinction isn't about capability — it's about statistical uncertainty.

When an LLM is asked "Does this file path exist?" it may answer correctly 98% of the time based on training data, but it's generating the answer probabilistically. That 2% failure rate compounds. When an LLM is asked "If this file doesn't exist, create it; if it exists, don't" it's now making a binary decision on a probabilistic foundation. Failures cascade.

The Deterministic Sandwich solves this by delegating boundary checks to code that either succeeds or fails deterministically:

- File path validation: check if files exist (or fail fast)
- Schema verification: parse expected data structure (or reject)
- Dependency checking: verify required packages are installed (or halt)
- Environment checks: confirm working directory, permissions, environment variables (or abort)

These checks happen *before* the LLM is invoked for reasoning. If a pre-flight check fails, the operation stops before the AI wastes time on an impossible task.

Symmetrically, post-processing gates catch mistakes the LLM makes that are easy to fix deterministically:

- Code formatting: run Prettier/ESLint after every edit to enforce consistent style
- Type-checking: run the TypeScript compiler to catch type errors before human review
- Linting: enforce architectural constraints (no imports from certain directories, no circular dependencies)
- Output validation: confirm the generated code matches the expected structure

Boris Cherny's team at Anthropic calls this "the 10% fix." Claude's code output is "usually" well-formatted and correctly typed, but the post-processing hooks catch the last 10% of edge cases — eliminating an entire class of CI failures without relying on the LLM to achieve perfection.

---

## Pre-Processing: Validating Inputs Before Reasoning

Pre-processing gates run *before* the LLM is invoked. They answer the question: "Is it even possible for the AI to complete this task?" If the answer is no, fail loudly and immediately.

### Pre-Flight Checks for Feature Implementation

The Development Loop (Step 7) includes a dedicated pre-flight phase. Before the AI starts implementing a build plan, verify:

1. **File existence:** Do all files referenced in the plan actually exist?
2. **Dependency availability:** Are all required packages installed? Do versions match expectations?
3. **Schema verification:** Do database schemas, API contracts, and data structures match what the plan assumes?
4. **Environment state:** Are we in the right directory? Do we have the expected environment variables? Can we write to the target directories?

This can be implemented as a simple shell script or a lightweight validation tool:

```bash
#!/bin/bash
set -e

# Validate referenced files exist
for file in "$@"; do
  [ -f "$file" ] || { echo "ERROR: Missing $file"; exit 1; }
done

# Verify dependencies
npm list > /dev/null 2>&1 || { echo "ERROR: Dependencies not installed"; exit 1; }

# Check environment
[ -n "$DATABASE_URL" ] || { echo "ERROR: DATABASE_URL not set"; exit 1; }

echo "Pre-flight check passed."
```

The gate is binary: pass or fail. If it fails, the human fixes the environment (install a missing dependency, create a file, set an env var) and retries. The AI never attempts reasoning against an impossible state.

### Schema and Path Validation in Tool Calls

For pre-processing within an AI session, use tool calls that validate state before the LLM reasons about it. Example: before the AI attempts to modify a database schema, a tool validates that the schema file exists and parses correctly:

```python
def validate_schema(schema_path: str) -> dict:
    """Validate schema file exists and is valid JSON/YAML."""
    if not os.path.exists(schema_path):
        raise ValueError(f"Schema file not found: {schema_path}")

    with open(schema_path) as f:
        schema = json.load(f)  # Fails if not valid JSON

    # Validate required fields
    required = ["tables", "relationships"]
    for field in required:
        if field not in schema:
            raise ValueError(f"Schema missing required field: {field}")

    return schema
```

The tool raises an exception if the schema is invalid. The AI reads the error, adjusts its approach, and retries with corrected input.

### File Path Canonicalization

A common failure mode: the AI generates a file path that looks correct but doesn't resolve properly when the code runs. Pre-process by canonicalizing paths:

```python
def resolve_file_path(path: str, base_dir: str = ".") -> str:
    """Resolve and validate file path."""
    resolved = os.path.normpath(os.path.join(base_dir, path))

    # Ensure path stays within base_dir (security check)
    real_base = os.path.realpath(base_dir)
    real_path = os.path.realpath(resolved)
    if not real_path.startswith(real_base):
        raise ValueError(f"Path outside base directory: {path}")

    return real_path
```

Present the resolved path back to the AI. If the AI said "create `src/utils/auth.js`" and the resolved path is `/home/user/project/src/utils/auth.js`, that's a success signal. If the AI said `../../../etc/passwd` and the validator rejects it, that's a failure — the AI knows it made an error before it attempts the operation.

---

## LLM Reasoning: The Middle Layer

With inputs validated and environment confirmed, the LLM can focus on reasoning. This is where the LLM adds value: interpreting specifications, generating code, making architectural decisions, and handling edge cases that require domain understanding.

The LLM operates within known-good state. File paths are valid. Dependencies are installed. Schema is correct. The reasoning can be focused.

### Focused Prompting in the Sandwich Context

When the LLM knows that pre-processing has already validated inputs, you can simplify the prompt. You don't need to say "if the file doesn't exist, create it" because the pre-processing gate already verified the file exists (or the operation never starts).

Example LLM instruction:

```
You are implementing a feature from a validated build plan. Pre-flight checks have confirmed:
- All referenced files exist and are valid
- Dependencies are installed
- The schema matches expectations

Do not spend time checking these preconditions. They have been verified. Focus on implementing the feature according to the build plan.
```

This reduces prompt overhead and lets the LLM's reasoning capacity focus on implementation.

---

## Post-Processing: Deterministic Output Validation and Fixing

Post-processing gates run *after* the LLM completes work. They answer the question: "Does the output meet our quality standards?" If not, fix automatically or raise an issue.

### Auto-Formatting After Every Edit

The most impactful post-processing hook: auto-format code immediately after the LLM writes it.

```bash
# PostToolUse hook: after Claude writes to any file, format it
on_file_written() {
    local file="$1"

    # Only format code files
    case "$file" in
        *.js|*.ts|*.jsx|*.tsx)
            prettier --write "$file" 2>/dev/null || true
            ;;
        *.py)
            black "$file" 2>/dev/null || true
            ;;
    esac
}
```

Why this works: The LLM's formatting output is probabilistic. Even with explicit instructions about indentation and line length, it will occasionally miss edge cases. A deterministic formatter fixes these automatically. Cherny's team reports this single hook eliminates ~10% of issues that would otherwise require human remediation.

The hook is non-invasive — if the file is already correctly formatted, the formatter is a no-op. If it's slightly off, the formatter fixes it silently.

### Type-Checking as a Post-Processing Gate

After code generation, run the type checker before declaring the task complete:

```bash
# PostToolUse hook: type-check TypeScript files
on_typescript_written() {
    local file="$1"
    if [[ "$file" == *.ts || "$file" == *.tsx ]]; then
        if ! tsc --noEmit "$file"; then
            echo "TYPE_CHECK_FAILED: $file has type errors"
            return 1
        fi
    fi
}
```

If the type check fails, the LLM reads the error message and fixes the code. If it passes, the gate signals success.

### Linting and Architectural Constraints

Post-processing can enforce architectural rules that are hard to encode in a prompt but easy to check with code:

```bash
# Verify no imports from forbidden paths
check_imports() {
    local file="$1"
    if grep -q "from 'server/" "$file" && [[ "$file" == src/client/* ]]; then
        echo "LINT_VIOLATION: Client code cannot import from server/"
        return 1
    fi
}
```

These gates enforce policies (separation of concerns, data flow rules, security constraints) deterministically. The AI learns to follow them because violations consistently trigger corrections.

### Output Validation Against Schema

When the LLM generates data structures, validate against schema before accepting the output:

```python
def validate_output(data: dict, schema: dict) -> bool:
    """Validate output against expected schema."""
    required_fields = schema.get("required", [])
    for field in required_fields:
        if field not in data:
            raise ValueError(f"Missing required field: {field}")

    # Type validation
    for field, value in data.items():
        expected_type = schema["properties"][field]["type"]
        if not isinstance(value, type_map[expected_type]):
            raise ValueError(
                f"Field {field}: expected {expected_type}, got {type(value)}"
            )

    return True
```

If the LLM's output doesn't match the schema, the validation fails and the LLM retries.

---

## Claude Code Hooks: A Concrete Implementation

Claude Code (Anthropic's AI-native IDE) implements the Deterministic Sandwich through PreToolUse and PostToolUse hooks:

### PreToolUse: Input Validation Before Actions

PreToolUse hooks run before Claude calls a tool (file write, API call, terminal command). Use this to:

- Scan proposed file writes for PHI (Personally Identifiable Information)
- Require explicit confirmation before modifying files in sensitive directories (payment processing, authentication)
- Validate that a proposed command won't damage the environment (no `rm -rf /`, no write to system directories)

Example:

```javascript
// PreToolUse hook in Claude Code
{
    "hook": "preToolUse",
    "condition": {
        "toolName": "writeFile",
        "pathMatches": "**/payment/**"
    },
    "action": "requireConfirmation",
    "message": "This modifies payment-related code. Confirm that it's safe."
}
```

If the condition matches, Claude pauses and asks for human confirmation before proceeding. This is a safety gate preventing accidental damage.

### PostToolUse: Output Validation After Actions

PostToolUse hooks run after a tool completes successfully. Use this to:

- Auto-format code files after writes
- Run linters and type-checkers on generated code
- Verify file integrity (checksums, schema validation)
- Trigger integration tests for files that were modified

Example:

```javascript
// PostToolUse hook: auto-format after file write
{
    "hook": "postToolUse",
    "condition": {
        "toolName": "writeFile",
        "pathMatches": "src/**/*.{js,ts,tsx}"
    },
    "action": "executeCommand",
    "command": "prettier --write {{filePath}}"
}
```

After Claude writes a TypeScript file, Prettier auto-runs. If the output was already formatted correctly, Prettier is a no-op. If it had minor issues, Prettier fixes them silently. Either way, the output meets quality standards without the LLM having to achieve perfection.

---

## LangChain Middleware: Pre- and Post-Agent Hooks

For agentic systems using LangChain or similar frameworks, implement the Deterministic Sandwich at the middleware layer:

```python
from langchain.agents import AgentExecutor
from langchain.agents import initialize_agent

# Pre-agent hook: validate session state
def before_agent(state: dict) -> dict:
    # Check all referenced files exist
    for file in state.get("files", []):
        if not os.path.exists(file):
            raise ValueError(f"File not found: {file}")

    # Verify environment
    if not shutil.which("python"):
        raise ValueError("Python not in PATH")

    return state

# Post-agent hook: validate output
def after_agent(result: dict) -> dict:
    # Validate generated code
    if "code" in result:
        try:
            compile(result["code"], "<string>", "exec")
        except SyntaxError as e:
            raise ValueError(f"Generated code has syntax error: {e}")

    # Format if successful
    if "files" in result:
        for file in result["files"]:
            subprocess.run(["prettier", "--write", file])

    return result

# Middleware integration
agent = initialize_agent(
    tools=[...],
    llm=model,
    pre_hook=before_agent,
    post_hook=after_agent,
)
```

The hooks run at the session level: before the agent starts, validate environment state. After the agent finishes, validate and clean up the output.

---

## AgentSpec: Customizable Runtime Enforcement

AgentSpec (ICSE '26) formalizes the Deterministic Sandwich as a runtime enforcement system. It allows you to define policies that are checked deterministically before or after agent actions.

Key features:

- **Action termination:** Block certain actions entirely (no writes to `/etc/`, no API calls to external services without approval)
- **User inspection:** Pause execution for human review when policies trigger
- **Corrective invocation:** When a policy violation is detected, invoke a corrective function to fix it

Example AgentSpec policy:

```yaml
policies:
  - name: "NoSystemFileWrites"
    condition: "action.tool == 'writeFile' && action.path.startsWith('/etc')"
    enforcement: "terminate"
    message: "System files cannot be modified."

  - name: "ConfirmPaymentChanges"
    condition: "action.tool == 'writeFile' && action.path.includes('payment')"
    enforcement: "inspect"
    message: "Payment code change. Require human confirmation."

  - name: "AutoFormatCode"
    condition: "action.tool == 'writeFile' && action.path.endsWith('.ts')"
    enforcement: "correct"
    corrective: "prettier --write {{path}}"
```

AgentSpec runs these checks deterministically. Actions that violate policies are blocked, paused for inspection, or automatically corrected — depending on the enforcement mode.

---

## The Guardrails Philosophy

The Deterministic Sandwich is grounded in the guardrails philosophy: **"Agentic AI is not about letting LLMs think more — it's about letting systems act safely, deterministically, within boundaries."**

This inverts the common misconception that more context and fewer constraints improve AI performance. In reality, constraints improve performance because they reduce the surface area for mistakes.

An LLM reasoning about implementation within known-valid state and expected interfaces is far more accurate than an LLM reasoning about whether a file path is correct. Constraints on *action* (what the AI is allowed to do) don't limit *reasoning* (the quality of decisions) — they protect against bad decisions manifesting in production damage.

---

## Policy-Aware Architecture

A mature implementation adds a decision plane separate from the action plane:

```
┌─────────────────────────────────────────┐
│  User Intent                            │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  LLM Reasoning                          │  (generates action proposals)
│  (what should we do?)                   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  Decision Plane                         │  (evaluates against policy)
│  (are we allowed to do it?)             │
└────────────────┬────────────────────────┘
                 │
                 ├─► Blocked (violates policy)
                 ├─► Inspect (requires human approval)
                 └─► Allowed
                     │
        ┌────────────┴────────────────┐
        │                             │
┌───────▼──────────────┐    ┌────────▼────────────┐
│  Action Plane        │    │  Post-Processor     │
│  (do the thing)      │    │  (fix output)       │
└──────────────────────┘    └─────────────────────┘
```

The decision plane is deterministic and policy-aware. It evaluates the LLM's proposed action against constraints *before* execution. The action plane either executes (with deterministic checks) or halts, depending on the decision plane's verdict.

This architecture ensures:

- **Safety:** Policy violations are detected before they cause damage
- **Determinism:** Decisions about what's allowed are made by code, not by LLM likelihood
- **Auditability:** Every decision point has a reason that can be logged and reviewed
- **Flexibility:** Policies can be updated without retraining the model

---

## Anti-Patterns and Pitfalls

### The "Trust the LLM" Anti-Pattern

Never skip pre-processing because "the AI usually gets it right." The 10% failure rate on deterministic tasks compounds quickly. If a file validation step fails 1% of the time and you have 100 such checks per feature, you'd expect ~63% of features to fail from validation alone.

### Over-Processing on the Post Side

Post-processing shouldn't be a tool to mask poor outputs. If the AI is consistently generating code that requires significant fixing, the problem isn't in post-processing — it's in the prompt, context, or model. Use post-processing for edge cases (formatting inconsistencies, minor type issues), not for structural problems.

### Policy Without Feedback

When a policy blocks an action, give the AI clear feedback about why. "Action denied" teaches nothing. "No writes to /etc/ — update docs/ instead" gives the AI a way to recover.

### Skipping Integration Validation

The Integration Contract (linked at the top) pairs with the Deterministic Sandwich: pre-processing validates that files exist and are connected as the contract specifies. Don't validate syntax without validating integration.

---

## Getting Started: Five Implementation Patterns

### 1. Shell Script Pre-Flight Check

Start with a simple script that runs before any AI work:

```bash
#!/bin/bash
set -e
echo "Running pre-flight checks..."

# Verify files
for file in src/main.ts src/utils.ts tests/unit.test.ts; do
    [ -f "$file" ] || { echo "Missing: $file"; exit 1; }
done

# Verify dependencies
npm list > /dev/null || { echo "Dependencies not installed"; exit 1; }

echo "Pre-flight check passed."
```

Run this before starting an AI session. It's deterministic, fast, and catches 80% of setup issues.

### 2. Post-Edit Formatting Hook

Add a post-processing step to format code after every write:

```bash
# In your editor or IDE config:
on_save_handler = "prettier --write {file} && eslint --fix {file}"
```

Or in Claude Code's hooks configuration, register a PostToolUse hook that runs formatters automatically.

### 3. Schema Validation for Generated Data

When the AI generates structured data (JSON, YAML, database records), validate against schema:

```python
from jsonschema import validate, ValidationError

def validate_generated_data(data: dict, schema_path: str):
    with open(schema_path) as f:
        schema = json.load(f)

    try:
        validate(instance=data, schema=schema)
        print("✓ Data valid")
    except ValidationError as e:
        print(f"✗ Validation error: {e.message}")
        raise
```

### 4. Type-Checking as a Gate

Make TypeScript (or mypy for Python) a required gate:

```bash
#!/bin/bash
if ! tsc --noEmit; then
    echo "Type check failed. AI should fix errors."
    exit 1
fi
echo "Type check passed."
```

The AI sees the error and retries.

### 5. Pre-Commit Hook for Architectural Rules

Use a pre-commit hook to enforce architectural constraints:

```bash
#!/bin/bash
# Prevent client code from importing server-side modules
if git diff --cached --name-only | grep -q "^src/client/"; then
    if git diff --cached src/client/ | grep -q "from 'src/server/"; then
        echo "Error: client code cannot import from server/"
        exit 1
    fi
fi
```

These patterns are simple, deterministic, and catch real classes of errors before they cause problems.

---

## Sources

- Boris Cherny, Claude Code workflow documentation and team practices (2026)
- Anthropic, "Claude Code: Agent Design for Reliability" (2026)
- ICSE '26, "AgentSpec: Policy-Aware Enforcement for AI Agents"
- Addy Osmani, "Agentic Engineering: Building Safe Autonomous Systems" (2026)
- Manus AI, "[Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)" (2025)
- HumanLayer, "[Guardrails for Agentic AI](https://www.humanlayer.dev/blog/guardrails-for-agentic-ai)" (2025)
