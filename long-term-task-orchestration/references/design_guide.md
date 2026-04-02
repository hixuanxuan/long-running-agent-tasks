# Long-Running Task Skill Design Guide

This document is a decision reference for designing long-running task Skills, focusing on "what to choose and why". For how to write specific scripts, see `script_patterns.md`.

---

## 1. Requirements Analysis Template

**Analysis principle**: Infer first, then confirm with the user. For each field, provide your own initial answer based on the user's description; state directly what you can already determine, and use option-style questions only for genuinely uncertain elements. Do **not** present a blank questionnaire for the user to fill in.

```
Task Name: ___
Task Description: One sentence (e.g., "Migrate all .less files in the project to PostCSS")

Input:
  - Input type: [code files / JSON data / diff / config files]
  - Input scope: [entire repository / specified directory / git diff / user-provided JSON]
    ↳ Scope support judgment: when both conditions below are met, the generated Skill must support scope variables by default (passed via environment variables):
       (a) Completable by sub-scope: processing a subset (e.g., a directory, some files) is logically equivalent to processing the full set
       (b) Independently verifiable within scope: success/failure of each task within the subset can be determined independently, without depending on tasks outside the scope
  - Estimated scale: [number of files / line count / number of task entries]

Operations:
  - What each subtask does: [migrate / review / fix / convert / generate]
  - Does it modify source files: [yes (needs validation + rollback) / no (generate report only)]
  - Is there a dependency order: [none / by dependency topological order / by priority]

Output:
  - Output type: [modified source files / JSON report / merged document]
  - Merge needed: [yes (merge multiple segments) / no (each independent)]

Validation:
  - Success criteria: [type check passes / build succeeds / JSON schema valid / AST unchanged]
  - Failure handling: [mark FAILED / revert files / keep for manual processing]

Environment:
  - Special dependencies needed: [TypeScript / Babel / PostCSS / ...]
  - Expected concurrency: [default 3-5 / high concurrency 10-20]
```

---

## 1b. Scope Variable Spec (Scope Env Vars)

When the "scope support judgment" in §1 concludes **yes**, the generated Skill must:

1. **Define scope variables in `.env`**, with defaults covering the full set, overridable by user:
   ```
   # Leave empty or unset to process all; when set, only process matching items
   SCOPE_DIR=          # Limit directory (relative to project root), e.g., src/components
   SCOPE_FILES=        # Glob pattern, e.g., "**/*.ts" (optional, combined with SCOPE_DIR)
   ```

2. **`discover.js` reads scope variables for filtering**, filtering logic completes at the scan stage, not at the dispatch stage:
   ```javascript
   const scopeDir = process.env.SCOPE_DIR || '';
   const scopeFiles = process.env.SCOPE_FILES || '**/*';
   // discover only writes files matching the scope to the task manifest
   ```

3. **Task manifest can be regenerated when scope variables change (idempotent)**: When re-running `discover.js`, already-DONE entries are preserved, new entries are appended, entries moved out of scope are not deleted (to avoid accidental loss of progress) but are marked `SKIPPED`.

4. **`setup-env.js` explains the purpose of scope variables**, guiding users to set them when they need partial debugging or incremental runs.

> Typical use case: run with `SCOPE_DIR=src/utils` first on a small directory to validate the rules, then clear the variable and run the full set.

---

## 2. State Storage Format Selection

| Format | Use Case | Advantages | Typical Case |
|--------|----------|-----------|-------------|
| TSV | Flat task list with fixed fields | Grep-friendly, git diff-friendly, human-readable | js-to-ts-migration |
| JSON manifest + inputs/ | Tasks with nested structure, chunks containing file lists | Manifest is small (no file content), subagent only reads its own input | codebase-review |
| JSON single file | Tasks with complex metadata (pid, commit hash, worktree) | Flexible structure, can track process state | review-fix |

### State Machine

**Basic state machine** (suitable for most tasks):

```
TODO → IN_PROGRESS → DONE ✅
                   ↘ FAILED ❌
                   ↘ SKIPPED ⏭️
```

**Extended state machine** (for tasks requiring merge validation, e.g., review-fix):

```
pending → running → success → merging → merged ✅
                  ↘ failed ❌       ↘ merge_failed ❌
```

The extended state machine adds the `success → merging → merged` phase — "subtask succeeded" and "merged into main branch" are two distinct actions.

---

## 3. Grouping Strategy

The grouping strategy determines "which files go into the same chunk for one subagent to handle". Good grouping balances two goals: files within a chunk have enough shared context for the subagent to make high-quality decisions, while the chunk is not so large that it blows up the context window or causes the subagent to miss files at the tail.

### By Directory + Line Count Limit (Most Common)

Files in the same directory share context (imports, type definitions), making this suitable for migration and review tasks. Set a cumulative line count ceiling to prevent context overflow.

How to set the ceiling: work backwards from the target model's context window. The fixed portion of the prompt (system instructions + constraint rules + output format) occupies a roughly fixed number of tokens; the remaining space is for file content. It is better to have smaller chunks and run more rounds than to have chunks so large that subagent output quality degrades or gets truncated. build-prompt.js also has a runtime fallback check (see §4), so the grouping-phase ceiling can be set a bit loosely and let the runtime check make the final call.

Empirical reference: for most scenarios, **3000 lines/chunk** is a safe starting point (approximately 15K-24K tokens of file content). Read-only review tasks can go up to 5000 lines.

### Per File

Each file is an independent task, suitable for fix-type tasks. Often paired with worktree isolation — each task operates in an independent git worktree without interfering with others.

### By Dependency Topological Order

Suitable for migration tasks with inter-file dependencies. Process leaf nodes first (files with no dependencies), then their dependents.

```
P1-No-Deps    → Process first, no blocking
P2-Low-Deps   → Depends on a few already-migrated files
P3-Mid-Deps   → Depends on more files
P4-Circular   → Circular dependencies, require special handling (interface extraction / force same group)
```

---

## 3.4 File Isolation Tool: git worktree

**Problem context**: Multiple subagents share the same working directory when running concurrently. If any subagent modifies files, two types of problems may occur:
- One subagent's write operations corrupt files being read by another subagent
- Multiple subagents writing simultaneously cause git state conflicts

**git worktree capability**: `git worktree add <path> HEAD` creates an independent working directory copy from the current commit. Each worktree has its own file tree and staging area, independent of each other. After completing work in their respective worktrees, subagents merge or cherry-pick changes back to the main branch.

```javascript
// Example: create an independent worktree for a subtask
const worktreePath = path.join(root, `.worktrees/${chunkId}`);
execSync(`git worktree add ${shellQuote(worktreePath)} HEAD`);
// subagent operates within worktreePath, then merges/cherry-picks back to main branch
```

Whether worktree isolation is needed depends on the actual read/write patterns of the subtasks.

---

## 4. Context Budget (subagent prompt token budget)

The most common failure mode in long-running tasks is subagent prompts that are too long. Defense operates on two layers:

**Layer 1: Grouping limit at the discover stage.** This is coarse-grained control — chunk by line count or file count so most chunks naturally fall within the safe range.

**Layer 2: Runtime check in build-prompt.js.** This is the fallback — after assembling the full prompt, estimate the actual token count (character count / 3.5 as a rough approximation); if it exceeds 60-70% of the model's context window, automatically split the chunk into `{chunkId}_part_1`, `{chunkId}_part_2`.

The two layers complement each other: the grouping stage does not need to calculate tokens precisely (token density varies greatly across languages and comment density), just makes a reasonable rough cut; the runtime check uses actual character count for the final decision.

Other key points:
- **Smaller is better**. Slightly smaller chunks just mean more rounds; chunks too large cause subagents to miss tail files or have output truncated.
- **Do not stuff other chunks' results into the prompt**. Follow the Context Reset principle: each subagent sees only its own input.

---

## 5. Error Handling Spec

See `references/error_handling.md` (three-layer retry mechanism, success determination, IN_PROGRESS residual handling).

---

## 6. Phase Reference Writing Pattern

When writing `references/phaseN_xxx.md` for the generated Skill, each file follows a unified structure:

```
# Phase N: [Phase Name]
Phase goal: one sentence.

## N.1 [First Step]
(Specific instructions, including commands to run)

## N.2 [Second Step]
...

## Completion Criteria
[What file/condition exists] → Phase N complete, can proceed to Phase N+1.
```

Core content for each Phase:

- **Phase 0**: Follow the 5-step environment initialization spec in `references/claude_cli.md` to complete environment checks, write all variables needed by subsequent Phases **to `.agent.env` in one shot**, and run `setup-env.js` to verify the exit code. After Phase 0 ends, each Phase reads only from `.agent.env` and does not prompt the user again.
- **Phase 1**: Run `discover.js` → check chunking results (chunk count, file count, line distribution) → confirm configuration → optional dependency analysis
- **Phase 2**: Confirm model → run `dispatch.js` → progress check → retry failed tasks → interruption recovery instructions. dispatch.js dispatches subagents as independent processes via `claude -p` (must not use Agent tool nested calls, as that bypasses concurrency control and progress tracking); see `references/claude_cli.md` for command options
- **Phase 3**: Run `merge.js` → global validation (customized by task type: tsc/build/schema) → generate report → optional cleanup

Phase 3's "global validation" is the most critical customization point — migration tasks run tsc + build, review tasks check JSON completeness, fix tasks run build + clean up worktrees.
