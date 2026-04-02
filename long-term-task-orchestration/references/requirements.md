# Requirements Analysis Reference

This document is the sole reference for Step 1 (Requirements Analysis).

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

## 2. Scope Variable Spec (Scope Env Vars)

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
