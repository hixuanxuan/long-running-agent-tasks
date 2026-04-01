# 长程任务 Skill 设计指南

本文档是设计长程任务 Skill 的决策参考，关注"选什么、为什么"。具体脚本怎么写看 `script_patterns.md`。

---

## 1. 需求分析模板

**分析原则**：先独立推断，再向用户确认。基于用户描述，对每个字段给出自己的初版答案；已能确定的直接陈述，真正不确定的以选项形式提问，**不得** 以空白问卷形式抛给用户填写。

```
任务名称: ___
任务描述: 一句话描述（如"将项目中所有 .less 文件迁移到 PostCSS"）

输入:
  - 输入类型: [代码文件 / JSON 数据 / diff / 配置文件]
  - 输入范围: [整个仓库 / 指定目录 / git diff / 用户提供的 JSON]
    ↳ 范围支持判断：同时满足以下两点时，生成的 Skill 必须默认支持范围变量（通过环境变量传入）：
       (a) 可按子范围完成：处理整体的一个子集（如某个目录、部分文件）与处理全量在逻辑上等价
       (b) 范围内可独立验证：子集内每个任务的成功/失败可以单独判断，不依赖范围外的任务结果
  - 预估规模: [文件数量 / 行数 / 任务条目数]

操作:
  - 每个子任务做什么: [迁移 / 审查 / 修复 / 转换 / 生成]
  - 是否修改源文件: [是（需要验证+回退）/ 否（只生成报告）]
  - 是否有依赖顺序: [无序 / 按依赖拓扑序 / 按优先级]

输出:
  - 输出类型: [修改后的源文件 / JSON 报告 / 合并文档]
  - 是否需要合并: [是（merge 多个 segment）/ 否（各自独立）]

验证:
  - 成功标准: [类型检查通过 / 构建成功 / JSON schema 合法 / AST 不变]
  - 失败处理: [标记 FAILED / 回退文件 / 保留供手动处理]

环境:
  - 是否需要特殊依赖: [TypeScript / Babel / PostCSS / ...]
  - 并发度预期: [默认 3-5 / 高并发 10-20]
```

---

## 1b. 范围变量规范（Scope Env Vars）

当 §1 中"范围支持判断"结论为**是**时，生成的 Skill 必须：

1. **在 `.env` 中定义范围变量**，默认值覆盖全量，用户可覆盖：
   ```
   # 留空或不设表示处理全量；指定时只处理匹配项
   SCOPE_DIR=          # 限定目录（相对于项目根），如 src/components
   SCOPE_FILES=        # glob 模式，如 "**/*.ts"（可选，与 SCOPE_DIR 叠加）
   ```

2. **`discover.js` 读取范围变量做过滤**，过滤逻辑在扫描阶段完成，不在 dispatch 阶段：
   ```javascript
   const scopeDir = process.env.SCOPE_DIR || '';
   const scopeFiles = process.env.SCOPE_FILES || '**/*';
   // discover 只将匹配范围的文件写入任务清单
   ```

3. **范围变量变更时任务清单可重新生成（幂等）**：`discover.js` 已有的 DONE 条目在重新 discover 时保留，新增条目追加，移出范围的条目不删除（避免意外丢失进度），但标记为 `SKIPPED`。

4. **`setup-env.js` 中说明范围变量用途**，引导用户在需要局部调试或增量运行时设置。

> 典型使用场景：先用 `SCOPE_DIR=src/utils` 跑通一个小目录验证规则，确认无误后清空变量跑全量。

---

## 2. 状态存储格式选择

| 格式 | 适用场景 | 优势 | 典型案例 |
|------|---------|------|---------|
| TSV | 任务列表扁平、字段固定 | 可 grep、git diff 友好、人类直接可读 | js-to-ts-migration |
| JSON manifest + inputs/ | 任务有嵌套结构、chunk 包含文件列表 | manifest 小（不含文件内容），subagent 只读自己的 input | codebase-review |
| JSON 单文件 | 任务包含复杂元数据（pid、commit hash、worktree） | 结构灵活、可跟踪进程状态 | review-fix |

### 状态机

**基础状态机**（适用于大多数任务）：

```
TODO → IN_PROGRESS → DONE ✅
                   ↘ FAILED ❌
                   ↘ SKIPPED ⏭️
```

**扩展状态机**（需要合并验证的任务，如 review-fix）：

```
pending → running → success → merging → merged ✅
                  ↘ failed ❌       ↘ merge_failed ❌
```

扩展状态机多了 `success → merging → merged` 阶段——"子任务成功"和"合并进主分支"是两个不同的动作。

---

## 3. 分组策略

分组策略决定"哪些文件放进同一个 chunk 让一个 subagent 处理"。好的分组平衡两个目标：chunk 内文件有足够的共享上下文让 subagent 做出高质量决策，同时 chunk 不大到撑爆上下文窗口或导致 subagent 漏处理尾部文件。

### 按目录 + 行数限制（最常用）

同目录文件有共享上下文（import、类型定义），适合迁移和审查类任务。累计行数设一个上限，防止上下文溢出。

上限怎么定：根据目标模型的上下文窗口反推。prompt 的固定部分（系统指令+约束规则+输出格式）大约占一个固定 token 量，剩余空间留给文件内容。宁可 chunk 小一些多跑几轮，也不要 chunk 过大导致 subagent 输出质量下降或被截断。build-prompt.js 里还有一道运行时兜底检查（见 §4），所以分组阶段的上限可以设得稍宽松，让运行时检查做最终裁决。

经验值参考：大多数场景 **3000 行/chunk** 是安全起点（约 15K-24K tokens 的文件内容）。只读审查类可放宽到 5000 行。

### 按文件独立

每个文件独立一个 task，适合修复类任务。常搭配 worktree 隔离——每个 task 在独立 git worktree 中操作，互不干扰。

### 按依赖拓扑序

适用于有文件间依赖的迁移任务。先处理叶子节点（无依赖的文件），再处理依赖者。

```
P1-No-Deps    → 优先处理，无阻塞
P2-Low-Deps   → 依赖少量已迁移文件
P3-Mid-Deps   → 依赖较多文件
P4-Circular   → 循环依赖，需要特殊处理（接口提取 / 强制同组）
```

---

## 3.4 文件隔离工具：git worktree

**问题背景**：多个 subagent 并发时共享同一份工作目录。如果其中有 subagent 会修改文件，就可能出现两类问题：
- 一个 subagent 的写操作污染其他 subagent 正在读取的文件内容
- 多个 subagent 同时写入导致 git 状态冲突

**git worktree 的能力**：`git worktree add <path> HEAD` 会从当前 commit 创建一个独立的工作目录副本。每个 worktree 拥有自己的文件树和暂存区，互不干扰。subagent 在各自的 worktree 中完成工作后，通过 merge 或 cherry-pick 将变更合回主分支。

```javascript
// 示例：为子任务创建独立 worktree
const worktreePath = path.join(root, `.worktrees/${chunkId}`);
execSync(`git worktree add ${shellQuote(worktreePath)} HEAD`);
// subagent 在 worktreePath 下操作，完成后 merge/cherry-pick 回主分支
```

根据子任务的实际读写模式决定是否需要worktree隔离。

---

## 4. Context Budget（subagent prompt token 预算）

长程任务最常见的失败模式是 subagent 的 prompt 过长。防御分两层：

**第一层：discover 阶段的分组上限。** 这是粗粒度控制——按行数或文件数切分 chunk，让大多数 chunk 天然落在安全范围内。

**第二层：build-prompt.js 的运行时检查。** 这是兜底——组装完完整 prompt 后，估算实际 token 数（字符数 / 3.5 可粗略近似），如果超过模型上下文窗口的 60-70%，自动将该 chunk 拆分为 `{chunkId}_part_1`、`{chunkId}_part_2`。

两层配合的效果：分组阶段不需要精确计算 token（不同语言、不同注释密度的 token 密度差异很大），只做一个合理的粗切；运行时检查用实际字符数做最终裁决。

其他关键点：
- **宁小勿大**。chunk 偏小只是多跑几轮，chunk 过大会导致 subagent 漏处理尾部文件或输出被截断。
- **不要在 prompt 中塞其他 chunk 的结果**。遵守 Context Reset 原则，每个 subagent 只看自己的输入。

---

## 5. 错误处理规范

详见 `references/error_handling.md`（三层重试机制、成功判定、IN_PROGRESS 残留处理）。

---

## 6. Phase Reference 编写模式

为生成的 Skill 编写 `references/phaseN_xxx.md` 时，每个文件遵循统一结构：

```
# Phase N：[阶段名]
本阶段目标：一句话。

## N.1 [第一步]
（具体指令，含要运行的命令）

## N.2 [第二步]
...

## 完成标志
[什么文件/条件存在] → Phase N 完成，可进入 Phase N+1。
```

各 Phase 的核心内容：

- **Phase 0**：按 `references/zulu_cli.md`「环境初始化规范」5 步完成环境检查，将所有后续 Phase 需要的变量**一次性写入 `.agent.env`**，运行 `setup-env.js` 验证退出码。Phase 0 结束后各 Phase 只从 `.agent.env` 读取变量，不再询问用户。
- **Phase 1**：运行 `discover.js` → 检查分块结果（chunk 数、文件数、行数分布）→ 确认配置 → 可选的依赖分析
- **Phase 2**：确认 model → 运行 `dispatch.js` → 进度检查 → 重试失败任务 → 中断恢复说明。dispatch.js 通过 `zulu run` 调度 subagent（**严禁使用 `delegate_subtask`**），命令选项详见 `references/zulu_cli.md`
- **Phase 3**：运行 `merge.js` → 全局验证（按任务类型定制：tsc/build/schema）→ 生成报告 → 可选清理

Phase 3 的"全局验证"是最关键的定制点——迁移类跑 tsc + build，审查类检查 JSON 完整性，修复类跑 build + 清理 worktree。
