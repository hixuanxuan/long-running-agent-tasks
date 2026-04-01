# 架构决策参考

本文档是 Step 2（架构决策）的唯一参考，关注"选什么、为什么"。已有案例对照见 `examples.md`。

---

## 1. 状态存储格式选择

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

## 2. 分组策略

分组策略决定"哪些文件放进同一个 chunk 让一个 subagent 处理"。好的分组平衡两个目标：chunk 内文件有足够的共享上下文让 subagent 做出高质量决策，同时 chunk 不大到撑爆上下文窗口或导致 subagent 漏处理尾部文件。

### 按目录 + 行数限制（最常用）

同目录文件有共享上下文（import、类型定义），适合迁移和审查类任务。累计行数设一个上限，防止上下文溢出。

上限怎么定：根据目标模型的上下文窗口反推。prompt 的固定部分（系统指令+约束规则+输出格式）大约占一个固定 token 量，剩余空间留给文件内容。宁可 chunk 小一些多跑几轮，也不要 chunk 过大导致 subagent 输出质量下降或被截断。build-prompt.js 里还有一道运行时兜底检查（见 `prompt_design.md` § Context Budget），所以分组阶段的上限可以设得稍宽松，让运行时检查做最终裁决。

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

## 3. 文件隔离工具：git worktree

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

根据子任务的实际读写模式决定是否需要 worktree 隔离。
