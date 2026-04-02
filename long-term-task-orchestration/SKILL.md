---
name: long-term-task-orchestration
description: >
  A meta-skill for designing and generating "long-running task Skills". Use when a user needs to create a Skill that involves a large number of files, requires cross-session resumption, or needs concurrent subagent scheduling.
  Trigger scenarios: (1) the user says "help me make a skill that bulk-processes XXX", (2) the user describes a batch task involving dozens to hundreds of files, even if they don't mention "skill",
  (3) the user says "create a long-running task", "make a long-running task skill", "a skill for bulk migration/review/fix".
  This skill drives the entire generation process end-to-end, calls skill-creator's eval capability in Step 5 for a smoke-test (user confirmation gate), and uses an independent grader agent in Step 6 for artifact quality review.
---

# Long-Term Task Orchestration Meta-Skill

A framework for designing and generating "long-running task Skills" for AI Coding Agents, with eval and multi-layer review to ensure quality delivery. Characteristics of long-running tasks: large number of files (dozens to tens of thousands), cannot complete in a single session, requires concurrent scheduling, requires cross-session resumption.

---

## Generation Workflow (End-to-End)

### Step 1 — Requirements Analysis

Based on the user's description, infer initial answers to the four key elements yourself, then confirm with the user (see full template in `references/requirements.md`).

**Important**: Always present your own inferred answers before asking questions. State directly what you can already determine; only use option-style questions for genuinely uncertain elements. Guide the user with directed questions — do not present a blank questionnaire for them to fill in.

### Step 2 — Architecture Decisions

Make three key choices based on requirements (decision matrix in `references/architecture.md`, existing case comparisons in `references/examples.md`):

1. **State storage format**: TSV / JSON manifest+inputs / JSON single file
2. **Grouping strategy**: By directory + line count / Per file / By dependency topological order
3. **State machine**: Basic (TODO→DONE/FAILED) / Extended (with merging phase)
4. **Isolation strategy**: Shared working directory / git worktree isolation (decision rules in `references/architecture.md` § git worktree)

### Step 3 — Generate Skill Files

Interface specs, implementation patterns, and common utility functions for each script are in `references/script_patterns.md`. Generate each in the following order, consulting the corresponding section before writing each script:

1. **setup-env.js** — Environment setup (see also `references/claude_cli.md` § Environment Initialization Spec)
2. **discover.js** — Scan + group + generate task manifest
3. **build-prompt.js** — Build subtask prompts (see also `references/prompt_design.md`)
4. **dispatch.js** — Concurrent scheduling core (see also `references/error_handling.md`)
5. **status.js** — Progress query
6. **merge.js** (optional) — Merge results
7. **poll.js** (optional, for worktree scenarios) — Poll and backfill

Generated directory structure:

```
<skill-name>/
├── SKILL.md                    # Phase definitions + resumption detection + completion criteria
├── scripts/
│   ├── setup-env.js            # Environment setup (unified template, rarely needs changes)
│   ├── discover.js             # Scan → generate task manifest (idempotent)
│   ├── dispatch.js             # Read manifest → group → concurrent dispatch
│   ├── build-prompt.js         # Programmatically build subtask prompts
│   ├── status.js               # Query progress
│   ├── merge.js                # Merge subtask results (optional)
│   └── poll.js                 # Poll + backfill (optional, for worktree isolation scenarios)
├── references/
│   ├── phase0_setup.md
│   ├── phase1_analyze.md
│   ├── phase2_dispatch.md
│   └── phase3_finalize.md
└── evals/
    └── evals.json
```

### Step 4 — Write SKILL.md

Assemble Phase definitions, resumption detection, and completion criteria into the generated SKILL.md. See `references/phase_template.md` for the Phase reference file writing pattern. Keep it under 500 lines.

### Step 5 — Eval Smoke Test (User Confirmation Gate)

Invoke skill-eval to obtain eval capabilities, run a smoke test round on the generated Skill to verify end-to-end completeness — confirm the generated artifacts can be triggered normally and phases connect without obvious gaps. **This step is executed only once; the result is handed to the user for judgment.**

1. Invoke skill-eval to obtain generation and execution capabilities, run eval against the current generated Skill directory, execute only once
2. Present the full eval results to the user, including: passing test cases, failing test cases, covered scenarios
3. **Wait for user decision**:
   - User accepts current results → proceed directly to Step 6
   - User requests improvements → modify corresponding files based on feedback, then proceed to Step 6

### Step 6 — Artifact Quality Review

Conduct a file-by-file quality review of generated artifacts through an independent eval agent. See `references/eval_grader.md` for the full flow and grader prompt template.

1. Assemble grader prompt from the `eval_grader.md` template, write to a temporary file
2. `claude -p <prompt> --cwd <skill directory>` to launch eval agent (independent session)
3. Eval agent reads files and runs checks autonomously, writes findings to `eval-report.json`
4. Read report: findings present → **abstract and generalize** problem patterns, apply global fixes, then re-run review; no findings → done
5. Maximum 3 rounds; if problems remain after round 3, present to user

**Fix principle**: Do not patch individual findings. First determine "whether this problem also exists elsewhere", do a global sweep, then fix uniformly. See `eval_grader.md` § Abstract Generalization.

---

## Four Phases

| Phase | Responsibility | Executor | Output |
|-------|--------------|--------|------|
| 0: Environment Setup | Follow `claude_cli.md` environment initialization spec to complete Node.js / claude-cli / API Key / model selection, write all variables to `.agent.env` in one shot | Main Agent + setup-env.js | `.agent.env` |
| 1: Analysis & Planning | Scan targets, analyze dependencies, generate task manifest | Main Agent + discover.js | Task manifest file |
| 2: Batch Execution | Group, dispatch subagents, validate, retry | dispatch.js + subagent | Per-subtask output |
| 3: Finalization & Validation | Merge results, global validation, generate report | Main Agent + merge.js | Final artifacts |

Before each Phase begins, the main Agent first views the corresponding `references/phaseN_xxx.md` for detailed instructions.

**Subagents must be executed as independent processes via `claude -p` CLI, and must NOT be called via Agent tools nested within the main Agent session.**

> Each subtask is an independent CLI process and Agent session, launched and managed by the `dispatch.js` script via `claude --print`, not nested within the main Agent conversation. This design is a core architectural decision of the long-running task framework:
>
> 1. **Prompt Determinism**: Subtask prompts are assembled programmatically by `build-prompt.js`, with consistent structure and controlled content. If the main Agent were to relay prompts through the session, it would "re-interpret" and rewrite the instructions — adding its own inferences, changing content organization — causing the subtask to receive instructions that deviate from intent, leading to unstable output quality.
> 2. **Eliminate Context Accumulation**: Each CLI subtask's context contains only the information needed for that task. If dispatched serially within the main Agent session, all preceding subtask conversation history accumulates in context, wasting tokens and distracting attention. This also avoids the main Agent spending tokens "thinking about how to write subtask instructions" — prompt construction is a programmatic zero-cost operation.
> 3. **Controlled Concurrency**: CLI processes have their concurrency controlled by scripts, adjustable dynamically based on resources and rate limits (3 to 20+ lanes). Having the Agent self-schedule concurrency in a conversation tends to be overly conservative, making true high concurrency hard to achieve.
> 4. **Pre/Post Logic Orchestration**: Scripts insert deterministic logic before and after subtask execution (create worktree, validate output, update state, clean up temp files) without requiring Agent involvement.
>

### Session Resumption Detection (must be written into generated SKILL.md)

```
1. Does .agent.env exist?         No → Phase 0
2. Does task manifest exist?      No → Phase 1
3. Any IN_PROGRESS tasks?         Yes → Check if their output files exist and are valid
                                       → Valid: mark DONE, continue
                                       → Invalid or missing: reset to TODO, enter Phase 2
4. All complete?                  Yes → Phase 3 / No → Phase 2 (continue)
```

---

## Six Design Principles the Generated Skill Must Embody

The following principles are quality standards for generated artifacts — the generated Skill must exhibit these characteristics, not constraints on this meta-skill's workflow.

1. **File As Progress** — All state is persisted to the filesystem; resumption relies only on disk. Write to disk immediately after each operation completes.
   → In all scripts, state changes must be followed by immediate `writeFileSync` — do not wait until batch ends.

2. **Context Reset** — Each subagent has a fresh context; prompts must be fully self-contained and cannot assume the subagent "already knows" any information.
   → build-prompt must include: complete file content, rule constraints, output format, output path.

3. **Task Contract** — Each subtask has an input/output/validation triplet contract. dispatch determines completion based on output files, not by parsing text output.
   → Success = output file exists + format valid (e.g., JSON.parse passes); does not rely on stdout/stderr.

4. **Idempotent & Incremental** — Repeated execution does not overwrite existing results. dispatch only processes pending/error; discover only supplements new files.
   → Each script loads existing state before running, skipping already-completed entries.

5. **Programmatic over Agent** — What can be solved by scripts should not be delegated to the Agent. Grouping, prompt assembly, state updates, and result merging are all scripted.
   → The Agent only does work requiring comprehension (code modification, review judgment, fix decisions).
   → **Subagents are dispatched as independent processes via `claude -p` CLI; nested calls via Agent tools within the main Agent session are prohibited.** Independent processes guarantee the determinism of programmatic prompt construction, eliminate context accumulation, support true high-concurrency control, and enable pre/post script orchestration.

6. **Failure Isolation** — Errors are resolved at the smallest possible scope; errors must not carry over to subsequent phases or escape from subtasks.
   → **Subtask self-containment**: If a subagent detects validation failure internally, it first attempts to fix within the current session; if unfixable, revert the environment, mark FAILED, explicitly report — do not let problematic output mix into the completion queue.
   → **Phase-level convergence**: Failed tasks during Phase 2 (batch execution) must be handled within that phase (retried or marked FAILED); when entering Phase 3, only "completed" or "explicitly marked FAILED" states are allowed.
   → **Three-layer retry mechanism** (see `references/error_handling.md`):
      - Inner layer: process crash / network failure → resume the same session with original conversationId
      - Middle layer: output validation failure → new session with error context for targeted fix, max 2-3 times; if exceeded, revert + mark FAILED
      - Outer layer: main Agent evaluates FAILED count, decides whether to re-dispatch (few → retry, many → diagnose first)
   → To determine which layer to use: check output file state (existence + format validity); do not parse Agent text output.

---

## Completion Criteria (Generic Template)

```bash
# All entries in the task manifest are in a terminal state (done/failed/skipped)
node ${SKILL_DIR}/scripts/status.js --root .
# Exit code 0 = all complete, non-0 = incomplete entries remain
```

## References

| File | Content |
|------|------|
| `references/requirements.md` | Requirements analysis template, scope variable spec |
| `references/architecture.md` | Architecture decisions: state storage selection, grouping strategy, state machine, git worktree isolation |
| `references/prompt_design.md` | Prompt design philosophy, quality principles, examples and counter-examples, Context Budget |
| `references/phase_template.md` | Phase Reference writing pattern |
| `references/error_handling.md` | Runtime error handling: three-layer retry mechanism, success determination, IN_PROGRESS residual handling |
| `references/script_patterns.md` | Interface specs + implementation patterns for each script |
| `references/claude_cli.md` | Complete claude CLI reference: `--print` (start a new Agent session), `--resume` (resume session), all options |
| `references/examples.md` | Quick reference for existing long-running task Skill cases (js-to-ts, codebase-review, review-fix) |
| `references/eval_grader.md` | Eval quality gate: grader prompt template, review dimensions, abstract generalization feedback principle |
