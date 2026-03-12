---
name: codex
description: >
  Delegate coding tasks to OpenAI Codex CLI for autonomous execution. Use this skill
  whenever the user explicitly asks to use Codex, or for tasks that benefit from Codex's
  strengths: repo-scale refactoring across many files, multi-directory changes, generating
  release notes or changelogs, automated code review, test generation, boilerplate, linting
  fixes, documentation, dependency bumps, and any mechanical coding task. Codex handles
  larger and more complex delegatable tasks than simpler agents — it supports multi-agent
  parallelism, web search, image input, and cross-repository work. Do NOT use for tasks
  requiring Claude's live conversation context, security-sensitive architectural decisions,
  or when the user explicitly wants Claude to handle it.
---

# OpenAI Codex CLI Delegation Skill

Delegate coding tasks — from mechanical to repo-scale — to OpenAI Codex CLI running
locally in your terminal. Codex uses `codex exec` for non-interactive execution and
returns output that Claude summarizes for the user.

## Prerequisites

- `codex` CLI installed and on `PATH` (install: `npm install -g @openai/codex`)
- OpenAI API key set as `OPENAI_API_KEY` env var (or `CODEX_API_KEY` for CI contexts)
- Must be run inside a Git repository (use `--skip-git-repo-check` to override)
- Confirm installation: `codex --version`; resolve errors before using this skill

If `codex` is not installed or configured, inform the user and provide the installation
instructions above. Do NOT attempt to run Codex if prerequisites are not met.

## How It Works

1. Claude evaluates the user's request and identifies it as suitable for Codex
2. Claude constructs a specific, self-contained prompt and runs `codex exec`
3. Codex executes non-interactively with `--full-auto`, auto-approving file edits and commands
4. Codex streams progress to stderr (suppressed); final output goes to stdout
5. Claude reads the output, verifies changes, and summarizes results for the user
6. If Codex fails or produces incorrect results, Claude retries with a refined prompt or
   falls back to handling the task directly

## Task Routing Guidelines

### Delegate to Codex (broad capability)

Codex handles more than simple tasks — it can reason across large codebases:

- **Repo-scale refactoring** (rename identifiers, migrate APIs, update patterns project-wide)
- **Multi-directory changes** (cross-package edits using `--add-dir`)
- **Release notes and changelogs** (reads git history, writes structured output)
- **Boilerplate generation** (new files, structs, interfaces, CRUD endpoints)
- **Simple and moderate refactors** (extract function, rename, reorganize)
- **Test scaffolding and test writing** (unit, integration, property-based)
- **Documentation and comment generation**
- **Formatting, linting, import sorting**
- **Dependency version bumps**
- **Mechanical find-and-replace across files**
- **Code review** (read-only analysis using `--sandbox read-only`)
- **Tasks requiring web search** (Codex has built-in web search — use `--config features.web_search=true`)
- **Tasks with image/screenshot context** (design specs, UI bugs — use `--image`)
- **Generating structured output** (use `--output-schema` for JSON-shaped results)

### Keep in Claude (requires live context or deep reasoning)

- Tasks requiring the full conversation history Claude has accumulated
- Security architecture decisions and vulnerability analysis
- Complex debugging that requires iterative exploration with you
- Performance optimization requiring profiling and expert interpretation
- Anything the user explicitly wants Claude to handle

### When in doubt

Ask the user: "This could be delegated to Codex. Want me to handle it myself or delegate?"

## CLI Usage

### The correct non-interactive command

```bash
codex exec "<task description>"
```

`codex exec` is the purpose-built non-interactive entrypoint. It defaults to read-only
sandbox and streams progress to stderr, printing only the final message to stdout.

### Flags reference

| Flag | Description | When to use |
|------|-------------|-------------|
| `--full-auto` | Grants workspace-write sandbox + on-request approvals | File-editing tasks |
| `--sandbox workspace-write` | Allow edits within the working directory | Safer alternative to --full-auto |
| `--sandbox read-only` | No writes or commands (analysis only) | Code review, read-only queries |
| `--sandbox danger-full-access` | Unrestricted access including network | Only in isolated environments |
| `--add-dir <path>` | Grant write access to additional directory | Multi-repo/cross-package changes |
| `--model <model>` | Override model | See model table below |
| `--json` | Emit JSONL events stream to stdout | Structured/CI consumption |
| `-o <path>` | Write final message to file | Capture output to file |
| `--output-schema <path>` | Enforce JSON Schema on final response | Structured data generation |
| `--ephemeral` | Don't persist session files | CI/CD runs |
| `--skip-git-repo-check` | Allow running outside git repos | Non-git projects |
| `--image <path>` | Attach image file(s) to the prompt | Design specs, screenshots |

### Model options

| Model | Best for |
|-------|---------|
| `gpt-5.4` (default) | Complex tasks, strong reasoning, agentic workflows |
| `gpt-5.3-codex` | Advanced coding specialist, repo-scale work |
| `gpt-5.3-codex-spark` | Near-instant iteration (ChatGPT Pro only) |

### Stderr suppression

Codex streams progress/thinking to stderr. Suppress with `2>/dev/null` to keep Claude's
context clean. If the user wants to see Codex's reasoning, omit the redirection.

## Execution Patterns

### Pattern 1: File-editing task (most common)

```bash
cd /path/to/project && codex exec --full-auto "Add JSDoc comments to all exported functions in src/utils.ts" 2>/dev/null
```

### Pattern 2: Read-only code review

```bash
cd /path/to/project && codex exec --sandbox read-only "Review the error handling in src/api/ and list all places where errors are silently swallowed. Do not modify any files." 2>/dev/null
```

### Pattern 3: Repo-scale refactoring

```bash
cd /path/to/project && codex exec --full-auto "Migrate all uses of the deprecated fetchData() API to the new fetchDataV2() API throughout the entire codebase. Update call sites, types, and tests." 2>/dev/null
```

### Pattern 4: Multi-directory change

```bash
cd /path/to/backend && codex exec --full-auto --add-dir /path/to/frontend "Update the UserProfile type in backend/src/types.ts and propagate the new `displayName` field to all frontend components that consume it." 2>/dev/null
```

### Pattern 5: Release notes generation (pipe to file)

```bash
cd /path/to/project && codex exec --sandbox read-only "Generate a CHANGELOG entry for all commits since the last git tag. Group by feat/fix/chore. Format as Markdown." 2>/dev/null | tee CHANGELOG_DRAFT.md
```

### Pattern 6: Structured output with schema

```bash
cd /path/to/project && codex exec --sandbox read-only --output-schema /tmp/schema.json "Audit all API endpoints and return a JSON list with fields: path, method, authenticated, description." 2>/dev/null
```

### Pattern 7: Task with image context

```bash
cd /path/to/project && codex exec --full-auto --image /tmp/design-spec.png "Implement the component shown in the design spec. Place it in src/components/HeroSection.tsx using the project's existing Tailwind setup." 2>/dev/null
```

### Pattern 8: Resume a previous session

```bash
codex exec resume --last "Now also update the tests for the files you just changed." 2>/dev/null
```

## Prompt Engineering for Codex

When constructing the prompt for `codex exec`, follow these guidelines:

1. **Be specific and self-contained** — Codex does not have Claude's conversation context.
   Include file paths, function names, expected behavior, and constraints.

2. **Specify scope explicitly** — Tell Codex which files/directories to work in and which to avoid.
   For read-only tasks, say "Do not modify any files."

3. **Include language/framework context** — "This is a Rust project using Actix-web",
   "This is a Next.js 15 app with TypeScript and Tailwind."

4. **State the expected output format** — "Create a new file at X", "Modify function Y in Z",
   "Print a Markdown summary to stdout."

5. **Set boundaries** — "Do not modify files outside src/api/", "Do not add new dependencies."

6. **For repo-scale tasks, be explicit about scope** — "Across the entire codebase",
   "Only in packages/ui/", "In all *.test.ts files."

## Handling Codex Output

After Codex completes:

1. **Read stdout** — Contains Codex's final message and results
2. **Verify changes** — Use `git diff --stat` or read modified files
3. **Summarize for the user** — What files changed, what Codex did, success/failure
4. **Handle failures gracefully:**
   - Minor issues: retry with a more specific prompt
   - Wrong scope: add more constraints to the prompt
   - Fundamental failure: fall back to handling the task directly with Claude
   - Always inform the user what happened and why

## Multi-Agent Usage (Experimental)

For large parallel workloads, Codex supports spawning sub-agents. Enable via config:

```bash
codex exec --config features.multi_agent=true "Refactor all modules in src/ in parallel: update imports, fix linting issues, and add missing type annotations." --full-auto 2>/dev/null
```

Use `agents.max_threads` in `~/.codex/config.toml` to control concurrency (default: 6).

## Example Workflow

**User says:** "Generate release notes for this sprint"

**Claude's process:**
1. Recognizes this as a read-only, structured-output task ideal for Codex
2. Runs:
```bash
cd /path/to/project && codex exec --sandbox read-only "Read the git log since the tag v2.3.0. Write release notes grouped by: New Features, Bug Fixes, Breaking Changes, and Internal Changes. Use the commit messages and touched files to infer each category. Output as Markdown." 2>/dev/null | tee release-notes.md
```
3. Reads release-notes.md
4. Reports: "Codex generated release notes covering 23 commits since v2.3.0 — saved to release-notes.md. Here's a preview: ..."
