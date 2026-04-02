# Phase Reference Writing Pattern

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

---

## Core Content for Each Phase

- **Phase 0**: Follow the 5-step environment initialization spec in `claude_cli.md` to complete environment checks, write all variables needed by subsequent Phases **to `.agent.env` in one shot**, and run `setup-env.js` to verify the exit code. After Phase 0 ends, each Phase reads only from `.agent.env` and does not prompt the user again.
- **Phase 1**: Run `discover.js` → check chunking results (chunk count, file count, line distribution) → confirm configuration → optional dependency analysis
- **Phase 2**: Confirm model → run `dispatch.js` → progress check → retry failed tasks → interruption recovery instructions. dispatch.js dispatches subagents as independent processes via `claude -p` (must not use Agent tool nested calls, as that bypasses concurrency control and progress tracking); see `claude_cli.md` for command options
- **Phase 3**: Run `merge.js` → global validation (customized by task type: tsc/build/schema) → generate report → optional cleanup

Phase 3's "global validation" is the most critical customization point — migration tasks run tsc + build, review tasks check JSON completeness, fix tasks run build + clean up worktrees.
