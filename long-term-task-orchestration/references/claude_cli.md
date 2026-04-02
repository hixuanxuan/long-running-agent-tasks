# Claude CLI Reference

`claude` is the CLI tool used by the long-running task framework (i.e., the Claude Code CLI).

For full CLI parameter documentation, see the official docs: https://code.claude.com/docs/en/cli-reference.md

---

## Environment Initialization Spec

The generated Skill's `setup-env.js` / Phase 0 must complete environment checks and configuration in the following order, and **write all variables needed by subsequent phases to `.agent.env`** (not just export them), ensuring each Phase does not depend on the current shell state when reading them.

### Step 1 — Ensure Node.js is Available

```bash
node --version
```

If not present, install via fnm:

```bash
curl -fsSL https://fnm.vercel.app/install | bash
fnm install --lts && fnm use lts-latest
# After installation, source the corresponding shell config (~/.bashrc or ~/.zshrc)
```

### Step 2 — Ensure claude CLI is Available

```bash
claude --version
```

If not present, install globally:

```bash
npm install -g @anthropic-ai/claude-code
```

### Step 3 — Obtain and Persist ANTHROPIC_API_KEY

First check the user's query, then check environment variables. If not set, prompt the user to obtain an API Key from the Anthropic Console, and:
1. Persist to shell config (`~/.bashrc` or `~/.zshrc`)
2. Write to the `.agent.env` file in the working directory (**must write to file, cannot just export**)

```bash
# .agent.env format
ANTHROPIC_API_KEY=your_api_key_here
MODEL=claude-sonnet-4-6
PROJECT_DIR=/absolute/path/to/project
# ... other variables needed by subsequent phases
```

### Step 4 — Model

Use the latest Sonnet model and analyze the currently valid model name; write it to the `MODEL` field in `.agent.env`.

### Step 5 — Confirm All Paths Are Literal Absolute Paths

All paths passed to claude subcommands **must be resolved to literal absolute paths**; shell variable forms (e.g., `$PROJECT_DIR`) are prohibited — shell variables in `.agent.env` are not expanded, causing silent failures.

### .agent.env One-Shot Collection Principle

When Phase 0 ends, `.agent.env` must contain all variables needed by all subsequent Phases. Each Phase reads from `.agent.env` and must not prompt the user again mid-run.

---

## Calling Pattern in dispatch.js

Key point: pass prompts via temporary files (to avoid shell escaping), **success is determined by output files, not by parsing stdout** (see Task Contract principle).

```bash
# ⚠️ claude CLI has no --cwd flag; change working directory with cd
QUERY_CONTENT=$(cat /path/to/tmp-query.txt)
cd /path/to/project && claude -p "$QUERY_CONTENT" \
  --model claude-sonnet-4-6 \
  --allowedTools "Read,Edit,Bash,Grep,Glob" \
  --output-format json
```

- `--allowedTools`: Pre-approve tools to avoid interactive confirmation prompts during subagent execution
- `--output-format json`: Output contains `session_id` in JSON, used by `--resume` for inner-layer retry
- Working directory is switched via `cd`, not `--cwd` (that flag does not exist)

**Inner-layer retry (resume session)**:

```bash
cd /path/to/project && claude -p "Continue previous task" \
  --resume <session_id> \
  --model claude-sonnet-4-6 \
  --allowedTools "Read,Edit,Bash,Grep,Glob" \
  --output-format json
```

For full invocation examples and concurrency pool implementation, see `script_patterns.md` § dispatch.js.
