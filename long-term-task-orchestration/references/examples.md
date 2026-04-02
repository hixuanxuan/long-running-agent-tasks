# Existing Long-Running Task Skill Case Reference

Decision reference for three typical patterns. When designing a new Skill, first find the closest case, then adjust the differences.

## Decision Quick Reference

| Dimension | js-to-ts (migration) | codebase-review (review) | review-fix (fix) |
|-----------|---------------------|-------------------------|-----------------|
| Modifies source files | Yes | No | Yes |
| Dependency order | Topological order (P1→P4) | None | None |
| State storage | TSV | JSON manifest+inputs | JSON single file |
| Grouping strategy | By directory+lines+dependencies | By directory+lines | Per file |
| Isolation | None | None | git worktree |
| State machine | Basic three-state | Basic three-state | Extended (with merging) |
| Merge | Not needed | Pure script merge | Agent participates in merge |
| Optional scripts | — | merge.js | poll.js, merge.js |

## Key Design Points for Each Case

### js-to-ts-migration
- Follows topological order; dispatch automatically manages dependency order: only processes files where "all JS dependencies are DONE", guaranteeing downstream files are always processed upstream
- AST validation is a hard constraint: Babel parses the file before and after; removing type annotations, the AST must be identical
- Crash-safe: when IN_PROGRESS residuals exist, delete already-created .ts files and reset to TODO (Option A, because the operation is idempotent)
- Retry includes error information: AST failure attaches diff, type failure attaches tsc errors
- Architecture reuse verified: ts-strict-migration reused the same architecture, only replacing the discover scan logic and build-prompt constraints

### codebase-review
- manifest only stores id/status, no files array (avoids token explosion). Each chunk has an independent input file
- Segments are independent: each segment is generated, validated, and re-run independently, with no cross-chunk dependencies
- dispatch does not parse subagent output; only checks segment file existence
- IN_PROGRESS residuals are reset directly to TODO (Option A, because read-only analysis has no side effects)

### review-fix
- **Extended state machine**: `pending → running → success → merging → merged`; "subtask succeeded" and "merged into main branch" are two distinct actions
- **Worktree isolation**: each subtask operates in an independent git worktree; failures do not affect the main branch
- **poll.js backfill**: check process liveness → read fix_result.json → backfill and launch new tasks
- **Main Agent participates in merge**: merging requires git merge + build validation + conflict resolution, which cannot be fully scripted
- **IN_PROGRESS residuals**: check if output file exists and is valid (Option B, because source files were modified)
- Phase 3 includes API callback: collect merged task IDs, call external API to mark resolved
