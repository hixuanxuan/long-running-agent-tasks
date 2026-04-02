# Harness Engineering: Reliable Orchestration for Long-Running AI Agent Tasks

A framework for making AI Coding Agents work reliably at scale — across hundreds of files, multiple sessions, and concurrent execution.

## Background

AI Coding Agents handle single-file, single-function tasks well. But real engineering work includes a class of tasks that are far more complex:

- Migrating 21 frontend modules from JavaScript to TypeScript
- Running a full code review across dozens of modules and batch-fixing hundreds of findings
- Extracting all hardcoded strings scattered across a codebase into i18n resources

These **long-term tasks** share three characteristics: they touch hundreds or thousands of files, they cannot finish in a single session, and they consume tens of millions to hundreds of millions of tokens.

## The Problems

Three fundamental difficulties arise at this scale:

**1. Context exhaustion.** Model context windows are finite. As tasks grow longer, history accumulates and compression kicks in — each compression round degrades the Agent's understanding of earlier context. The Agent starts "forgetting" conventions established at step 10 by step 50, or repeats mistakes it already corrected. Worse, it detects a nearly-full context and preemptively declares the task done before it actually is.

**2. No cross-session continuity.** Network drops, token limits, model timeouts — these are not exceptions, they are the norm. Without a resumption mechanism, any interruption means starting over from scratch, compounding cost and potentially making the task impossible to finish.

**3. Uncontrollable behavior at scale.** What works on one file doesn't always work on a thousand. Format inconsistencies, broken builds, and individual file failures all occur. A single failure that crashes the entire pipeline is unusable in production.

## The Solution

Four core principles address these problems:

| Principle | Problem it solves |
|-----------|------------------|
| **Task decomposition** | Each subtask fits in a single session's context window |
| **Parallel execution** | Multiple subagents run concurrently, controlled by scripts |
| **File As Progress** | All state is persisted to disk immediately; any interruption resumes from the exact breakpoint |
| **Completion contracts** | Each subtask has a programmatic success criterion; failures are isolated and retried in layers |

Subtasks are executed as **independent CLI processes** (`claude -p`), not nested tool calls within the main Agent session. This eliminates context accumulation, enforces prompt determinism, enables true concurrency control, and allows scripts to inject deterministic pre/post logic around each execution.

## This Repository

This repository contains two skills that work together:

### `long-term-task-orchestration` — Meta-Skill

A meta-skill that generates new long-running task Skills on demand. Given a description of what you want to automate at scale, it produces a fully structured Skill with phase definitions, concurrent dispatch scripts, resumption logic, and programmatic validation — ready to run.

```
<generated-skill>/
├── SKILL.md                  # Phase definitions, resumption detection, completion criteria
├── scripts/
│   ├── setup-env.js          # Environment initialization
│   ├── discover.js           # Scan targets, generate task manifest (idempotent)
│   ├── dispatch.js           # Concurrent subagent scheduling
│   ├── build-prompt.js       # Programmatic prompt construction
│   ├── poll.js               # Poll status + backfill vacant slots
│   ├── merge.js              # Collect and merge results
│   └── status.js             # Progress query
├── references/
│   ├── phase0_setup.md
│   ├── phase1_analyze.md
│   ├── phase2_dispatch.md
│   └── phase3_finalize.md
└── evals/
    └── evals.json
```

After generating a skill, `long-term-task-orchestration` automatically invokes `skill-eval` for a smoke test, then runs an independent grader agent to review artifact quality before handing off to the user.

### `skill-eval` — Evaluation Tool

`skill-eval` is extracted from `skill-creator` — specifically the body eval stage of its generation pipeline. It runs an end-to-end benchmark:

```
Generate test cases → Run skill → Grade assertions → Aggregate benchmark → Present results
```

It runs each test case with and without the skill (or against an older version), grades the outputs against objective assertions, and presents the results in an interactive review viewer. `long-term-task-orchestration` invokes it automatically as a quality gate during Skill generation — you do not need to activate it explicitly.

## Installation

```bash
npx skills add hixuanxuan/long-running-agent-tasks -y
```

Both skills (`long-term-task-orchestration` and `skill-eval`) are installed in one command. `skill-eval` does not need to be activated manually — when `long-term-task-orchestration` reaches the eval stage, it invokes `skill-eval` automatically.

## Usage

```
/long-term-task-orchestration help me create a skill to introduce react-compiler and remove existing memo usage
/long-term-task-orchestration help me create a skill to remove dead code from the project
/long-term-task-orchestration help me create a skill to replace axios with fetch
/long-term-task-orchestration help me create a skill to migrate Vue components to React
```

## Design Philosophy

This is a form of **Harness Engineering** — building the scaffolding that compensates for current model limitations. Every design decision encodes an assumption about what the model cannot reliably do on its own. As models improve, the harness can be simplified. But the judgment of *which steps to give to the model and which to keep in the framework* remains valuable regardless of model capability.

The goal is not to replace the Agent's intelligence, but to give it a reliable structure within which that intelligence produces consistent, verifiable, and resumable results.
