# Codex CLI Integration for Claude Code

Delegate simple coding tasks from Claude Code to [OpenAI Codex CLI](https://github.com/openai/codex) for cost-efficient execution.

## Purpose

Enable Claude Code to invoke the Codex CLI (`codex --approval-mode full-auto`) for automated
code generation, refactoring, test scaffolding, and other mechanical coding workflows —
saving Claude tokens for tasks that actually need them.

## Prerequisites

- `codex` CLI installed and available on `PATH`
- OpenAI API key configured as `OPENAI_API_KEY` env var or in `~/.config/openai/api_key`
- Confirm the installation by running `codex --version`; resolve any errors before using the skill

### Install Codex CLI

```bash
npm install -g @openai/codex
```

## Installation

### Option 1: Standalone Skill Installation

```bash
git clone --depth 1 https://github.com/Eliott/codex-plugin.git /tmp/skill-codex-tmp && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/skill-codex-tmp/plugins/skill-codex/skills/codex ~/.claude/skills/codex && \
rm -rf /tmp/skill-codex-tmp
```

### Option 2: Plugin Installation

If Claude Code's plugin marketplace is available:

```
/plugin marketplace add Eliott/codex-plugin
/plugin install skill-codex@codex-plugin
```

## Usage

### Automatic delegation

Claude Code will automatically delegate simple tasks to Codex when the skill is installed.
Tasks like boilerplate generation, simple refactors, test scaffolding, and documentation
updates will be routed to Codex CLI.

### Explicit delegation

You can explicitly ask Claude to use Codex:

```
Use codex to add JSDoc comments to all files in src/utils/
```

### Thinking tokens

By default, Codex's verbose output (stderr) is suppressed with `2>/dev/null` to avoid
bloating Claude Code's context window. If you want to see them for debugging, ask Claude
to show Codex's output.

## Cost

Codex CLI uses `codex-mini-latest` (o4-mini) by default.
Pricing: $1.10/M input tokens, $4.40/M output tokens.
Typical simple tasks cost $0.01–0.10 — significantly cheaper than using Claude for
mechanical work.

## How It Works

1. You give Claude Code a task
2. Claude evaluates if the task is simple enough to delegate
3. Claude constructs a specific prompt and runs `codex --approval-mode full-auto --quiet "..."`
4. Codex executes the task using o4-mini
5. Claude reads the output, verifies the changes, and summarizes results
6. If Codex fails, Claude falls back to handling it directly

## License

MIT License
