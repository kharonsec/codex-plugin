---
name: codex
description: >
  Delegate coding tasks to OpenAI Codex CLI for cost-efficient execution. Use this skill
  whenever a task is straightforward, well-defined, and does not require Claude's full
  reasoning capabilities — simple refactors, boilerplate generation, test scaffolding,
  documentation updates, file renaming, linting fixes, dependency bumps, small bug fixes,
  formatting, or mechanical code changes. Also use when the user explicitly asks to use
  Codex or OpenAI for a task. Do NOT use for complex architectural decisions, multi-step
  debugging requiring deep reasoning, or security-sensitive code review.
---

# OpenAI Codex CLI Delegation Skill

Delegate simple, well-defined coding tasks to OpenAI Codex CLI to save Claude tokens for
harder problems. Codex runs non-interactively via `--approval-mode full-auto` and returns
results that Claude summarizes for the user.

## Prerequisites

- `codex` CLI installed and on `PATH` (install: `npm install -g @openai/codex`)
- OpenAI API key configured (`OPENAI_API_KEY` env var or `~/.config/openai/api_key`)
- Confirm installation: `codex --version`; resolve any errors before using this skill

If `codex` is not installed or not configured, inform the user and provide the installation
instructions above. Do NOT attempt to run codex commands if the prerequisites are not met.

## How It Works

1. Claude evaluates the user's request
2. If the task is simple/mechanical, Claude delegates to Codex via a shell command
3. Codex runs non-interactively with `--approval-mode full-auto`, auto-approving all tool executions
4. Claude reads the output and summarizes results back to the user
5. If Codex's output is insufficient or incorrect, Claude can either retry with a refined
   prompt or fall back to handling the task itself

## Task Routing Guidelines

### Delegate to Codex (cheap, fast)
- Boilerplate generation (new files, structs, interfaces, CRUD endpoints)
- Simple refactors (rename, extract function, move code between files)
- Test scaffolding and simple test writing
- Documentation and comment generation
- Formatting, linting fixes, import sorting
- Dependency version bumps
- Mechanical find-and-replace across files
- Adding error handling to straightforward functions
- Generating type definitions or interfaces from existing code
- Simple bug fixes where the issue is clearly identified

### Keep in Claude (complex, nuanced)
- Architectural decisions or design pattern selection
- Security-sensitive code review or vulnerability analysis
- Complex multi-step debugging requiring deep reasoning
- Performance optimization requiring profiling analysis
- Tasks requiring understanding of the full project context across many files
- Anything the user explicitly wants Claude to handle

### When in doubt
Ask the user: "This looks like it could be delegated to Codex to save tokens.
Want me to handle it myself or delegate?"

## CLI Usage

### Basic non-interactive execution

```bash
codex --approval-mode full-auto --quiet "<task description>"
```

### Flags reference

| Flag | Description | Recommended |
|------|-------------|-------------|
| `--approval-mode full-auto` | Auto-approve all tool use (non-interactive) | Always |
| `--quiet` | Suppress verbose output and thinking | Always (suppress stderr too) |
| `--model <model>` | Override model (default: codex-mini-latest) | Optional |
| `--no-project-doc` | Skip project doc injection | Optional |

### Important notes

- `--approval-mode full-auto` is required for non-interactive execution
- Suppress stderr with `2>/dev/null` to avoid bloating Claude Code's context window
- If the user explicitly asks to see Codex's thinking/reasoning, omit the `2>/dev/null`
- Run from the project's working directory so Codex has the right file context

## Execution Patterns

### Pattern 1: Simple task delegation (most common)

For a contained, single-purpose task:

```bash
cd /path/to/project && codex --approval-mode full-auto --quiet "Add JSDoc comments to all exported functions in src/utils.ts" 2>/dev/null
```

### Pattern 2: Read-only analysis

For tasks that only need to read code (no writes):

```bash
cd /path/to/project && codex --approval-mode full-auto --quiet "List all API endpoints in this project with their HTTP methods and paths. Do not modify any files." 2>/dev/null
```

### Pattern 3: Scoped file edits

For edits limited to specific files:

```bash
cd /path/to/project && codex --approval-mode full-auto --quiet "Refactor the error handling in src/api/client.rs to use thiserror instead of manual impl. Only modify files in src/api/." 2>/dev/null
```

### Pattern 4: Test generation

```bash
cd /path/to/project && codex --approval-mode full-auto --quiet "Generate unit tests for the functions in src/auth/token.go. Use the standard testing package. Place tests in src/auth/token_test.go" 2>/dev/null
```

## Prompt Engineering for Codex

When constructing the prompt string for Codex, follow these guidelines:

1. **Be specific and self-contained** — Codex does not have the conversation context that Claude has.
   Include all relevant details: file paths, function names, expected behavior, constraints.

2. **Specify scope explicitly** — Tell Codex which files/directories to work in and which to avoid.

3. **State the expected output** — "Create a new file at X", "Modify function Y in file Z",
   "Print a summary to stdout".

4. **Include language/framework context** — "This is a Rust project using Actix-web",
   "This is a SvelteKit app with TypeScript".

5. **Set boundaries** — "Do not modify any files outside src/api/",
   "Do not add new dependencies".

## Handling Codex Output

After Codex completes:

1. **Read the stdout output** — This contains Codex's actions and results
2. **Verify the changes** — Check modified files if needed (use `git diff` or read the files)
3. **Summarize for the user** — Tell them what Codex did, what files changed, and whether
   the task completed successfully
4. **Handle failures gracefully** — If Codex failed or produced incorrect results:
   - For minor issues: retry with a more specific prompt
   - For fundamental failures: fall back to handling the task directly with Claude
   - Always inform the user what happened

## Cost Management

Codex CLI uses `codex-mini-latest` (o4-mini) by default.
o4-mini pricing: $1.10/M input tokens, $4.40/M output tokens.
For simple tasks, the cost is typically well under $0.10.

| Task type | Typical cost |
|-----------|-------------|
| Boilerplate / scaffolding | $0.01–0.05 |
| Simple refactor | $0.02–0.08 |
| Test generation | $0.02–0.08 |
| Multi-file mechanical change | $0.05–0.20 |
| Read-only analysis | $0.01–0.03 |

## Example Workflow

**User says:** "Add error handling to the API calls in src/services/"

**Claude's process:**
1. Recognizes this as a mechanical task suitable for Codex
2. Constructs a specific prompt with project context
3. Runs:
```bash
cd /path/to/project && codex --approval-mode full-auto --quiet "Add proper error handling (try/catch or Result types as appropriate) to all API call functions in the src/services/ directory. Preserve existing function signatures. Use the project's existing error handling patterns. Do not modify files outside src/services/." 2>/dev/null
```
4. Reads output, verifies changes with `git diff --stat`
5. Reports to user: "Codex updated 4 files in src/services/ — added error handling to 12 API calls. Here's a summary of changes: ..."
