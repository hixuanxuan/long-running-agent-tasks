---
name: long-term-task-orchestration
description: >
  用于设计和生成「长程任务 Skill」的元技能（meta-skill）。当用户需要创建一个涉及大量文件、需要跨会话恢复、需要并发 subagent 调度的 Skill 时使用。
  触发场景：(1) 用户说"帮我做一个批量 XXX 的 skill"，(2) 用户描述了一个需要处理几十到上百个文件的批量任务，即使没有提到"skill"一词，
  (3) 用户说"创建一个长程任务"、"make a long-running task skill"、"批量迁移/审查/修复的 skill"。
  本 skill 全程主导生成过程，仅在最后一步调用 skill-creator 的 eval 生成能力。
---

# 长程任务编排 Meta-Skill

为 AI Coding Agent 设计和生成「长程任务 Skill」的框架。长程任务的特征：文件数量多（几十到几万）、单次会话无法完成、需要并发调度、需要跨会话恢复。

---

## 生成工作流（端到端）

### Step 1 — 需求分析

基于用户描述，自行推断四要素的初版答案，再向用户确认（完整模板见 `references/requirements.md`）

**重要**：提问前先给出自己的推断结果。对已能判断的要素直接陈述，仅对真正不确定的要素以选项形式让用户确认，用户需要引导式的提问，不得用空白问卷式提问。

### Step 2 — 架构决策

根据需求做三个关键选择（决策矩阵见 `references/architecture.md`，已有案例对照见 `references/examples.md`）：

1. **状态存储格式**：TSV / JSON manifest+inputs / JSON 单文件
2. **分组策略**：按目录+行数 / 按文件独立 / 按依赖拓扑序
3. **状态机**：基础（TODO→DONE/FAILED）/ 扩展（含 merging 阶段）
4. **隔离策略**：共享工作目录 / git worktree 隔离（判断规则见 `references/architecture.md` § git worktree）

### Step 3 — 生成 Skill 文件

各脚本的接口规范、实现 pattern 和通用工具函数见 `references/script_patterns.md`。按以下顺序逐个生成，写每个脚本前先查阅对应章节：

1. **setup-env.js** — 环境准入（另见 `references/zulu_cli.md` § 环境初始化规范）
2. **discover.js** — 扫描+分组+生成任务清单
3. **build-prompt.js** — 构建子任务 prompt（另见 `references/prompt_design.md`）
4. **dispatch.js** — 并发调度核心（另见 `references/error_handling.md`）
5. **status.js** — 进度查询
6. **merge.js**（可选）— 合并结果
7. **poll.js**（可选，worktree 场景）— 轮询补位

生成的目录结构：

```
<skill-name>/
├── SKILL.md                    # Phase 定义 + 恢复检测 + 完成标准
├── scripts/
│   ├── setup-env.js            # 环境准入（统一模板，几乎不需改动）
│   ├── discover.js             # 扫描 → 生成任务清单（幂等）
│   ├── dispatch.js             # 读清单 → 分组 → 并发调度
│   ├── build-prompt.js         # 程序化构建子任务 prompt
│   ├── status.js               # 查询进度
│   ├── merge.js                # 合并子任务结果（可选）
│   └── poll.js                 # 轮询+补位（可选，用于 worktree 隔离场景）
├── references/
│   ├── phase0_setup.md
│   ├── phase1_analyze.md
│   ├── phase2_dispatch.md
│   └── phase3_finalize.md
└── evals/
    └── evals.json
```

### Step 4 — 编写 SKILL.md

将 Phase 定义、恢复检测、完成标准组装进生成的 SKILL.md。Phase reference 文件的编写模式见 `references/phase_template.md`。控制在 500 行以内。

### Step 5 — Eval 质量门禁

通过独立 eval agent 审查生成产物。完整流程和 grader prompt 模板见 `references/eval_grader.md`。

1. 按 `eval_grader.md` 模板组装 grader prompt，写入临时文件
2. `zulu run --cwd <skill 目录> --query <prompt>` 启动 eval agent（独立会话）
3. eval agent 自行读取文件、运行检查，将 findings 写入 `eval-report.json`
4. 读取报告：有 findings → **抽象泛化**问题模式，全局修复后重新发起 eval；无 findings → 进入 Step 6
5. 最多循环 3 轮，第 3 轮仍有问题则展示给用户

**修复原则**：不针对单个 finding 打补丁。先判断"这个问题是否在其他位置也存在"，做全局排查再统一修复。详见 `eval_grader.md` § 抽象泛化。

### Step 6 — 编写 evals

调用 skill-creator 的 eval 编写能力，为生成的 Skill 补充评估用例。

---

## 四个 Phase

| Phase | 职责 | 执行者 | 产出 |
|-------|------|--------|------|
| 0: 环境配置 | 按 `zulu_cli.md` 环境初始化规范完成 Node.js / zulu-cli / token / 模型选择，一次性将所有变量写入 `.agent.env` | 主 Agent + setup-env.js | `.agent.env` |
| 1: 分析规划 | 扫描目标、分析依赖、生成任务清单 | 主 Agent + discover.js | 任务清单文件 |
| 2: 批量执行 | 分组、调度 subagent、验证、重试 | dispatch.js + subagent | 各子任务产出 |
| 3: 收尾验证 | 合并结果、全局验证、生成报告 | 主 Agent + merge.js | 最终产物 |

每个 Phase 开始前，主 Agent 先 view 对应的 `references/phaseN_xxx.md` 获取详细指令。

**subagent 必须通过 `zulu run` CLI 以独立进程执行，严禁使用 `delegate_subtask`**

> 每个子任务是一次独立的 CLI 进程和 Agent 会话，由 `dispatch.js` 脚本启动和管理，不在主 Agent 对话中嵌套调用。这一设计是长程任务框架的核心架构决策：
>
> 1. **Prompt 确定性**：子任务 Prompt 由 `build-prompt.js` 程序化组装，结构一致、内容可控。若由主 Agent 在会话中转述，会自行"重新理解"后改写指令——添加自身推断、改变内容组织方式，导致子任务接收到的指令偏离预期，产出质量不稳定。
> 2. **消除上下文累积**：每个 CLI 子任务的上下文里只有当前任务需要的信息。若在主 Agent 会话中串行调度，前序子任务的对话历史全部堆积在上下文中，浪费 Token 并干扰注意力。同时省去主 Agent 花 Token "思考怎么写子任务指令"的开销——Prompt 构建是程序化的零成本操作。
> 3. **并发可控**：CLI 进程由脚本控制并发数，可按资源和限流动态调整（3 路到 20+ 路）。在对话内让 Agent 自行调度并发，模型往往过于保守，难以达到真正的高并发。
> 4. **前后逻辑可编排**：脚本在子任务执行前后插入确定性逻辑（创建 worktree、校验产出、更新状态、清理临时文件），不需要 Agent 参与。
>

### 会话恢复检测（必须写入生成的 SKILL.md）

```
1. .agent.env 存在？       否 → Phase 0
2. 任务清单文件存在？      否 → Phase 1
3. 有 IN_PROGRESS 任务？  有 → 检查其产出文件是否存在且合法
                              → 合法：标记 DONE，继续
                              → 不合法或不存在：重置为 TODO，进入 Phase 2
4. 全部完成？             是 → Phase 3 / 否 → Phase 2（继续执行）
```

---

## 生成的 Skill 必须体现的六大设计原则

以下原则是生成产物的质量标准——生成出来的 Skill 要体现这些特征，而非本 meta-skill 工作流的约束。

1. **File As Progress** — 所有状态持久化到文件系统，恢复只依赖磁盘。每步操作完成后立即写盘。
   → 所有脚本中状态变更后必须立即 writeFileSync，不等批次结束。

2. **Context Reset** — 每个 subagent 是全新上下文，prompt 必须完全自包含，不能假设 subagent "已经知道"任何信息。
   → build-prompt 必须包含：完整文件内容、规则约束、输出格式、输出路径。

3. **Task Contract** — 每个子任务有输入/输出/验证三元组契约。dispatch 根据产出文件判定完成，不解析文本输出。
   → 成功判定 = 产出文件存在 + 格式合法（如 JSON.parse 通过），不依赖 stdout/stderr。

4. **Idempotent & Incremental** — 重复执行不覆盖已有结果。dispatch 只处理 pending/error，discover 只补充新文件。
   → 每个脚本运行前先加载已有状态，跳过已完成的条目。

5. **Programmatic over Agent** — 能用脚本解决的不让 Agent 做。分组、prompt 组装、状态更新、结果合并全部脚本化。
   → Agent 只做需要理解力的工作（代码修改、review 判断、修复决策）。
   → **subagent 通过CLI 以独立进程调度，严禁使用 `delegate_subtask`。** 独立进程保证 Prompt 程序化构建的确定性、消除上下文累积、支持真正的高并发控制和执行前后的脚本编排。

6. **Failure Isolation** — 错误在最小范围内解决，不将错误带到后续阶段，不将错误带出子任务。
   → **子任务内闭环**：subagent 内部发现校验失败，先在当前会话内尝试修复；修不好则 revert 环境、标记 FAILED，明确上报，不让问题产出混入完成队列。
   → **阶段内收敛**：Phase 2 执行阶段的失败任务必须在本阶段处理完毕（重试或标记 FAILED），进入 Phase 3 时只有"已完成"或"明确标记 FAILED"两种状态。
   → **三层重试机制**（详见 `references/error_handling.md`）：
      - 内层：进程崩溃/网络异常 → 用原 conversationId 恢复同一会话续传
      - 中层：产出校验失败 → 新会话附带错误信息针对性修复，最多 2-3 次；超限 revert + 标记 FAILED
      - 外层：主 Agent 评估 FAILED 数量，决策是否重新 dispatch（少则重跑，多则先排查原因）
   → 判断走哪层：检查产出文件状态（存在性 + 格式合法性），不解析 Agent 文本输出。

---

## 完成标准（通用模板）

```bash
# 所有任务清单中的条目都处于终态（done/failed/skipped）
node ${SKILL_DIR}/scripts/status.js --root .
# 退出码 0 = 全部完成，非 0 = 存在未完成
```

## References

| 文件 | 内容 |
|------|------|
| `references/requirements.md` | 需求分析模板、范围变量规范 |
| `references/architecture.md` | 架构决策：状态存储选型、分组策略、状态机、git worktree 隔离 |
| `references/prompt_design.md` | Prompt 设计哲学、质量原则、正反例、Context Budget |
| `references/phase_template.md` | Phase Reference 编写模式 |
| `references/error_handling.md` | 运行时错误处理：三层重试机制、成功判定、IN_PROGRESS 残留处理 |
| `references/script_patterns.md` | 各脚本的接口规范 + 实现 pattern |
| `references/zulu_cli.md` | zulu CLI 完整参考：`list-mode`（查看模型）、`run`（启动新的Agent会话）全部选项
| `references/examples.md` | 已有长程任务 Skill 的案例速查（js-to-ts、codebase-review、review-fix） |
| `references/eval_grader.md` | Eval 质量门禁：grader prompt 模板、检查维度、反馈抽象泛化原则 |
