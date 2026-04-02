# Error Handling Spec

This document defines the runtime error handling strategy for long-running task Skills. For design decisions (state storage, grouping, etc.), see `architecture.md`.

---

## Core Principle: Resolve Errors at the Smallest Possible Scope

"Smallest scope" is layered:

- **Subtask self-containment**: If a subagent detects invalid output during execution (e.g., compilation fails, JSON schema validation fails), it should fix the issue within the current session rather than leaving errors for subsequent steps. When unfixable, revert the workspace, mark FAILED, and explicitly inform the upper layer — do not let problematic output mix into the completion queue.
- **Phase-level convergence**: Failed tasks during Phase 2 (batch execution) must be handled within that phase (retried or marked FAILED); when entering Phase 3 (finalization), only "completed" or "explicitly marked FAILED" states are allowed. Handoffs between phases must be clean — otherwise downstream work continues on problematic output, and issues only surface during acceptance, wasting the entire chain.

## Success Determination

Check the output file as specified by the Task Contract: file exists + format valid (e.g., JSON.parse passes) = success.

Do not parse subagent text output — text is unreliable (may be truncated, format inconsistent).

## Three-Layer Retry Mechanism

**Inner layer: Resume session (continuation)**

Applicable when: process crash, network failure, compression error, or other abnormal exit where the task itself is not at fault.

dispatch records the `session_id` from each execution (parsed via `--output-format json`). When an abnormal exit is detected (`execAsync` rejects and output file does not exist), use `claude --resume <session-id>` to resume the same session and send "continue" to let the Agent pick up from the breakpoint.

```javascript
// Inner layer: resume session (in the catch branch of runChunk, still using execAsync to maintain async)
// ⚠️ claude CLI has no --cwd flag; change working directory with cd
const resumeCmd = [
    `cd ${shellQuote(cwd)} &&`,
    `claude -p "Continue previous task"`,
    `--resume ${shellQuote(lastSessionId)}`,  // resume same session
    `--model claude-sonnet-4-6`,
    `--allowedTools "Read,Edit,Bash,Grep,Glob"`
].join(' ');
await execAsync(resumeCmd, { timeout: 30 * 60 * 1000, maxBuffer: 50 * 1024 * 1024 });
```

This is the lowest-cost retry — the Agent does not need to re-understand the task, it continues directly from the interruption point. Note that the retry also uses `execAsync` (see `script_patterns.md` § dispatch.js) to ensure other tasks in the concurrency pool are not blocked.

**Middle layer: New session with feedback (targeted fix)**

Applicable when: subagent completes normally, but output validation fails (tsc errors, AST diff not empty, JSON schema invalid).

Start a fresh subagent session and attach the complete error information (error content, failed validation rules) as context in the prompt. The Agent only fixes the error point, not the entire group of files.

- Maximum 2-3 retries
- Each time attach accumulated error information (not just the most recent), to help the Agent understand patterns of repeated failure
- If limit exceeded: revert workspace (delete or restore modified files), clean up environment, mark FAILED
- At this point, no more attempts at the subtask level

**Outer layer: Main Agent re-dispatch (decision layer)**

Applicable when: after the middle layer is exhausted, a batch of FAILED tasks has accumulated.

The main Agent checks the FAILED count and error characteristics, and makes a decision:

- **Few FAILEDs (e.g., ≤5%)**: Errors may be intermittent and worth re-dispatching this batch of files (re-group, generate new prompts, launch new subagents) — essentially rerunning them as brand-new tasks.
- **Many FAILEDs (e.g., >10%)**: May indicate the task rules themselves have problems (rules unclear, target files have systemic issues); blindly retrying only wastes tokens. The main Agent should first analyze the failure patterns, possibly needing to adjust build-prompt rule descriptions or exclude certain files.

You need to judge whether a retry is worthwhile. Non-recoverable errors do not need to be retried.

### Three-Layer Reference Table

| Layer | Trigger Condition | Implementation | Cost | Limit |
|-------|------------------|---------------|------|-------|
| Inner | Process crash/network failure, output file missing | `claude --resume` resume same session | Lowest (continuation) | 1-2 times |
| Middle | Output exists but validation fails | New session + complete error info | Medium | 2-3 times |
| Outer | After middle exhausted, batch of FAILEDs | Main Agent decides to re-dispatch | High | Main Agent judgment |

## IN_PROGRESS Residual Handling

When the Agent is interrupted, some tasks may be stuck in IN_PROGRESS. New sessions resuming must handle this:

- **Option A (reset to TODO)**: Suitable for idempotent, side-effect-free tasks (e.g., read-only review, revertible migration operations)
- **Option B (check output file to decide)**: Suitable for tasks that modify source files. Check if output file exists and is valid; if so, mark DONE, otherwise reset to TODO

Which option to choose depends on "whether re-executing this subtask is safe". Read-only tasks use A; tasks that modify files and whose output is verifiable use B.
