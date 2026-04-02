# Prompt Design Spec

This document consolidates all design decisions related to subagent prompts: quality principles, structural spec, and Context Budget control.

---

## 1. Core Prompt Quality Principle

Every output item must come with a justification for "why it must be addressed". If a subagent cannot write a specific trigger scenario and impact explanation for a finding, it has not thought through why it should report it — that finding is most likely noise. Therefore, a good prompt should, through output format and rule design, compel the subagent to perform causal reasoning for each output item, rather than simple pattern matching.

---

## 2. Five Components of a Good Prompt

1. **Clear task description** (1-2 sentences, no lengthy exposition)
2. **Core constraints + examples and counter-examples** (rules explain what should be reported; examples and counter-examples demonstrate what qualifies — examples and counter-examples have far greater impact on output quality than the rule text itself)
3. **Files to process** (complete paths)
4. **Output requirements** (explicit output path + format example; subagent doesn't need to guess where or how to write)
5. **Validation criteria** (what result counts as success, aligned with dispatch's success determination logic)

### Complete Example (codebase-review as example)

```
Please complete a code review task on the following files, finding bugs, security risks, spec violations, and maintainability issues.

## Review Rules
- Only report issues for which you can construct a specific trigger scenario
- Each issue must answer: under what conditions does it trigger? What is the direct consequence of triggering?
- If you cannot describe a reasonable trigger scenario, do not report it
- If no issues are found, output an empty array

### Positive Example
trigger: "Race condition under concurrent requests causes duplicate balance deduction"
impact: "User fund loss, affects all concurrent order scenarios"
suggestion: "Add distributed lock in deductBalance()"
→ Qualified: trigger condition is clear, consequence is serious and verifiable

### Negative Example
message: "Line 12 uses let, should be changed to const"
→ Not qualified. Missing trigger scenario and impact explanation. Even for spec issues, must explain which rule is violated and what consequence allowing it would have.

Correct version of the same issue:
trigger: "Variable that is never reassigned after declaration uses let"
impact: "Violates the team's oxlint prefer-const spec; as it accumulates, lint rules become meaningless, new members cannot learn correct patterns from existing code"
suggestion: "Change to const"

## Files to Review
src/utils/auth.ts
src/api/user.ts

## Output Requirements
Write review results to `.task-data/segments/chunk_src__utils-output.json`, format:
{
  "chunkId": "chunk_src__utils",
  "findings": [
    {
      "file": "src/utils/auth.ts",
      "line": 42,
      "trigger": "When user token expires and user logs in again, old session is not cleared",
      "impact": "Attacker can use leaked old token to continuously access, affecting all users with remember-me enabled",
      "suggestion": "Call invalidateAllSessions(userId) at the entry of renewToken()"
    }
  ]
}

## Validation
Output file must be valid JSON; every entry in the findings array must contain file, line, trigger, impact, suggestion fields.

## IMPORTANT
**You are the leaf executor of this task; you must complete all work directly and must not invoke Agent tools to launch sub-Agents.**
```
**When the Agent judges a task as complex, it tends to call Agent tools to break it down; subagent prompts must explicitly prohibit this and explain why — no recursive nesting.**

---

## 3. Context Budget (subagent prompt token budget)

The most common failure mode in long-running tasks is subagent prompts that are too long. Defense operates on two layers:

**Layer 1: Grouping limit at the discover stage.** This is coarse-grained control — chunk by line count or file count so most chunks naturally fall within the safe range. For grouping limit selection, see `architecture.md` § Grouping Strategy.

**Layer 2: Runtime check in build-prompt.js.** This is the fallback — after assembling the full prompt, estimate the actual token count (character count / 3.5 as a rough approximation); if it exceeds 60-70% of the model's context window, automatically split the chunk into `{chunkId}_part_1`, `{chunkId}_part_2`.

The two layers complement each other: the grouping stage does not need to calculate tokens precisely (token density varies greatly across languages and comment density), just makes a reasonable rough cut; the runtime check uses actual character count for the final decision.

Other key points:
- **Smaller is better**. Slightly smaller chunks just mean more rounds; chunks too large cause subagents to miss tail files or have output truncated.
- **Do not stuff other chunks' results into the prompt**. Follow the Context Reset principle: each subagent sees only its own input.
