# Script Interface and Implementation Patterns

This file defines the interface specs and implementation key points for each script. When generating a specific Skill, use these interfaces and patterns to write the complete implementation.

All scripts are written in Node.js using only standard library `require` (fs/path/child_process), with zero external dependencies.

---

## Common Patterns

### .env Loading

All scripts (except setup-env.js) share the same loadEnv pattern:

```javascript
function loadEnv(root) {
    // Read .env from root, parse KEY=VALUE format
    // Only set variables not already defined in process.env (CLI args take priority over .env)
}
```

### Argument Parsing

Uniformly use `--key value` format, manually parse `process.argv`. Do not introduce external dependencies like yargs/commander.

### Chunk ID

Path to ID: `/` → `__`, `.` → `_dot_`, overflow appends `_part_N`. Use a unified `toChunkId(module, partIndex)` function.

### Shell-Safe Quoting

All places that concatenate shell commands must use the `shellQuote` function uniformly, to prevent special characters in paths or content from causing command injection or parsing errors:

```javascript
function shellQuote(s) {
    return "'" + s.replace(/'/g, "'\\''") + "'";
}
```

---

## setup-env.js

Environment setup script. Nearly identical implementation is shared across all long-running task Skills.

**Interface**:
```
node scripts/setup-env.js --root <dir> [--ptoken <token>] [--model <model>] [--concurrency <N>]
```

**Responsibilities (in order)**:
1. Print Node.js version to confirm availability
2. Check for claude CLI; if absent, run `npm install -g @anthropic-ai/claude-code`
3. Determine ANTHROPIC_API_KEY: priority `--api-key` argument → existing `.env` → if missing, exit with code 1
4. Write/update `.env`: merge existing config lines, only overwrite fields starting with `ANTHROPIC_`

**Exit codes**: 0 = ready, 1 = missing required parameters, 2 = dependency installation failed

**Key constraint**: dispatch.js does not re-check these dependencies before running — it only calls `loadEnv()` to read .env, trusting setup-env has completed the setup.

---

## discover.js

Scan target files and generate the task manifest according to the grouping strategy. For rationale behind grouping strategy selection, see `architecture.md` § Grouping Strategy; this section focuses only on implementation.

**Interface**:
```
node scripts/discover.js --root <dir> [--max-lines <N>] [--extensions <.ts,.tsx>]
```

**Responsibilities**:
1. Recursively scan directories, filter by extensions, exclude node_modules/dist/build/.git/__tests__
2. Count line numbers for each file
3. Group into chunks according to grouping strategy
4. Generate task manifest

**Idempotent pattern** (the most important implementation characteristic of discover):
```
Load existing manifest → for each new chunk:
  - Chunk ID already exists with status done → preserve, do not overwrite
  - Chunk ID already exists with status pending → update file list (files may have been added/removed)
  - New chunk → mark as pending
```

**Output files** (using JSON manifest+inputs as example):
- `.task-data/manifest.json` — chunk list (id/module/totalLines/fileCount/status), does not contain files array
- `.task-data/inputs/{chunkId}-input.json` — file list for each chunk, subagent only reads its own input

---

## dispatch.js

Read task manifest and dispatch subagents for parallel execution.

**Interface**:
```
node scripts/dispatch.js --root <dir> [--concurrency <N>] [--model <model>] [--dry-run] [--retry-failed]
```

**--concurrency determination**: First try to extract from user query; if not available, default to 20. Indicates how many sub-Agents to run concurrently.

**Internal flow**:
```
1. loadEnv() reads .env
2. Read task manifest → filter ready tasks (pending, or including error/failed when --retry-failed is set)
3. Initialize concurrency pool (size = concurrency)
4. For each chunk:
   a. Mark as IN_PROGRESS
   b. Call buildPrompt() to build prompt
   c. Launch subagent (claude --print)
   d. Check output file → mark done or error
   e. Write to disk immediately (crash-safe)
   f. Slot available in pool → start next one
5. All chunks processed → output summary
```

**Subagent invocation pattern**:

> **Important**: Subagents must be launched as independent processes via `claude -p` CLI; nested calls via Agent tools within the main Agent session are prohibited — nested calls cannot be managed by dispatch.js's concurrency pool, state tracking, and success determination mechanisms. For complete `claude` parameter documentation, see `references/claude_cli.md`.

> **Data flow**: `discover.js` generates `inputs/{chunkId}-input.json` → `buildPrompt()` reads the file list and assembles a self-contained prompt → passes via temporary file → `--query` uses the prompt as the sub-Agent's user input → sub-Agent writes results to `segments/{chunkId}-output.json` → dispatch.js checks this file to determine success.

Below is a dispatch pattern example, including async process execution and a concurrency pool. A single invocation is the `runChunk` function; concurrency control is driven by `runPool`.

```javascript
const { exec } = require('child_process');

// ── Async execution: dispatch needs to manage multiple child processes simultaneously; use exec's Promise wrapper for the concurrency pool to work properly ──
function execAsync(cmd, opts) {
    return new Promise((resolve, reject) => {
        exec(cmd, opts, (err, stdout, stderr) => {
            if (err) reject(Object.assign(err, { stdout, stderr }));
            else resolve({ stdout, stderr });
        });
    });
}

// ── Concurrency pool: control the number of simultaneously running child processes ──
async function runPool(tasks, concurrency, handler) {
    const executing = new Set();
    for (const task of tasks) {
        const p = handler(task).then(() => executing.delete(p), () => executing.delete(p));
        executing.add(p);
        if (executing.size >= concurrency) {
            await Promise.race(executing);  // wait for any one to complete, freeing a slot
        }
    }
    await Promise.all(executing);  // wait for all tail tasks to complete
}

// ── Complete processing flow for a single chunk ──
async function runChunk(chunk) {
    const { chunkId } = chunk;
    markInProgress(chunkId);

    // 1. Read this chunk's input file
    const inputPath = path.join(root, '.task-data', 'inputs', `${chunkId}-input.json`);
    const { files } = JSON.parse(fs.readFileSync(inputPath, 'utf-8'));

    // 2. Build complete prompt (see build-prompt.js section below)
    const outputPath = path.join(root, '.task-data', 'segments', `${chunkId}-output.json`);
    const prompt = buildPrompt(root, files, { outputPath, chunkId });

    // 3. Pass prompt via temp file to avoid shell escaping issues
    const tmpFile = path.join(cwd, `.tmp-query-${chunkId}-${Date.now()}.txt`);
    fs.writeFileSync(tmpFile, prompt, 'utf-8');
    try {
        // ⚠️ claude CLI has no --cwd; switch working directory via cd
        const cmd = [
            `cd ${shellQuote(cwd)} &&`,
            `QUERY_CONTENT=$(cat ${shellQuote(tmpFile)});`,
            `claude -p "$QUERY_CONTENT"`,
            `--model claude-sonnet-4-6`,
            `--allowedTools "Read,Edit,Bash,Grep,Glob"`,
            `--output-format json`
        ].join(' ');
        await execAsync(cmd, { timeout: 30 * 60 * 1000, maxBuffer: 50 * 1024 * 1024 });
    } finally {
        try { fs.unlinkSync(tmpFile); } catch {}
    }

    // 4. Success determination (Task Contract): check output file existence + format validity
    if (fs.existsSync(outputPath)) {
        JSON.parse(fs.readFileSync(outputPath, 'utf-8')); // validate format
        markDone(chunkId);
    } else {
        markFailed(chunkId, 'no output file');
    }
    // Do not parse subagent stdout/stderr to determine success
}

// ── Main entry point ──
const readyChunks = manifest.filter(c => c.status === 'pending');
await runPool(readyChunks, concurrency, runChunk);
```

**Write timing**: Write manifest immediately after each chunk state change; do not wait until the batch ends (markInProgress / markDone / markFailed should all internally call writeFileSync).

---

## build-prompt.js

Build the subagent prompt for a single chunk. Embodies the Context Reset principle: the prompt must be fully self-contained.

> For prompt design philosophy, quality principles, examples/counter-examples, and Context Budget control, see `prompt_design.md`.

**Interface** (module.exports + can run standalone):
```javascript
/**
 * @param {string} root       Project root directory
 * @param {string[]} files    List of relative file paths
 * @param {Object} options    { outputPath, chunkId, rules? }
 * @returns {string}          Complete prompt text
 */
function buildPrompt(root, files, options) { ... }
```

**Runtime token check**: After assembling the full prompt, estimate the token count (character count / 3.5 as rough approximation); if it exceeds 60-70% of the model's context window, automatically split the chunk into `{chunkId}_part_1`, `{chunkId}_part_2` and generate independent prompts for each.

**Return value and consumption**: `buildPrompt()` returns an **in-memory string** (`@returns {string}`); the caller dispatch.js is responsible for writing that string to a temporary file.

---

## status.js

Query task progress.

**Interface**:
```
node scripts/status.js --root <dir>
```

**Output format**:
```
Total: 120 | Done: 115 | Failed: 3 | Skipped: 2 | Pending: 0
Progress: 96% (115/120)
```

**Exit codes**: 0 = all in terminal state, 1 = incomplete entries exist, 2 = manifest not found

---

## merge.js (optional)

Merge all segments into the final result.

**Interface**:
```
node scripts/merge.js --root <dir>
```

**Responsibilities**: Iterate over the segments directory, parse each chunk's output, merge into a unified result.json (with summary statistics).

Should also be runnable when partially complete; include completion ratio in the summary.

---

## poll.js (optional)

Check subtask process status and backfill pending tasks. Used for worktree isolation scenarios.

**Interface**:
```
node scripts/poll.js --root <dir> [--concurrency <N>]
```

**Responsibilities**:
1. Check tasks in running state: use `process.kill(pid, 0)` to determine if the process is alive
2. Process has exited → check output file → mark success or failed
3. Calculate available slots → start pending tasks to fill

**Exit codes**: 0 = active tasks exist, 2 = all in terminal state
