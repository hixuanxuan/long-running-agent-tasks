# Architecture Decision Reference

This document is the sole reference for Step 2 (Architecture Decisions), focusing on "what to choose and why". For existing case comparisons, see `examples.md`.

---

## 1. State Storage Format Selection

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

## 2. Grouping Strategy

The grouping strategy determines "which files go into the same chunk for one subagent to handle". Good grouping balances two goals: files within a chunk have enough shared context for the subagent to make high-quality decisions, while the chunk is not so large that it blows up the context window or causes the subagent to miss files at the tail.

### By Directory + Line Count Limit (Most Common)

Files in the same directory share context (imports, type definitions), making this suitable for migration and review tasks. Set a cumulative line count ceiling to prevent context overflow.

How to set the ceiling: work backwards from the target model's context window. The fixed portion of the prompt (system instructions + constraint rules + output format) occupies a roughly fixed number of tokens; the remaining space is for file content. It is better to have smaller chunks and run more rounds than to have chunks so large that subagent output quality degrades or gets truncated. build-prompt.js also has a runtime fallback check (see `prompt_design.md` § Context Budget), so the grouping-phase ceiling can be set a bit loosely and let the runtime check make the final call.

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

## 3. File Isolation Tool: git worktree

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
