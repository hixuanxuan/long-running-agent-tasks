# 已有长程任务 Skill 案例速查

三种典型模式的决策参考。设计新 Skill 时，先找到最接近的案例，再调整差异点。

## 决策速查表

| 维度 | js-to-ts (迁移类) | codebase-review (审查类) | review-fix (修复类) |
|------|-------------------|------------------------|-------------------|
| 修改源文件 | 是 | 否 | 是 |
| 依赖顺序 | 拓扑序 (P1→P4) | 无序 | 无序 |
| 状态存储 | TSV | JSON manifest+inputs | JSON 单文件 |
| 分组策略 | 按目录+行数+依赖 | 按目录+行数 | 每文件独立 |
| 隔离方式 | 无 | 无 | git worktree |
| 状态机 | 基础三态 | 基础三态 | 扩展 (含 merging) |
| 合并 | 无需 | 纯脚本 merge | Agent 参与 merge |
| 可选脚本 | — | merge.js | poll.js, merge.js |

## 各案例关键设计点

### js-to-ts-migration
- 按照拓扑序，dispatch 自动管理依赖顺序：只处理"所有 JS 依赖已 DONE"的文件，保证下游拿到的文件一定是上游处理过的
- AST 验证是硬约束：Babel 解析前后文件，去掉类型注解后 AST 必须相同
- crash-safe：IN_PROGRESS 残留时删除已创建的 .ts 文件，重置为 TODO（方案 A，因为操作幂等）
- 重试带错误信息：AST 失败附 diff，类型失败附 tsc 报错
- 架构复用验证：ts-strict-migration 复用了相同架构，只替换 discover 扫描逻辑和 build-prompt 约束

### codebase-review
- manifest 只存 id/status，不含 files 数组（避免 token 爆炸）。每个 chunk 独立 input 文件
- 分段独立：每个 segment 独立生成、校验、重跑，无跨 chunk 依赖
- dispatch 不解析 subagent 输出，只检查 segment 文件存在性
- IN_PROGRESS 残留直接重置为 TODO（方案 A，因为只读分析无副作用）

### review-fix
- **扩展状态机**：`pending → running → success → merging → merged`，"子任务成功"和"合并进主分支"是两个不同动作
- **worktree 隔离**：每个子任务在独立 git worktree 中操作，失败不影响主分支
- **poll.js 补位**：检查进程存活 → 读取 fix_result.json → 补位启动新任务
- **主 Agent 参与合并**：merge 需要 git merge + build 验证 + 冲突修复，不能纯脚本化
- **IN_PROGRESS 残留**：检查产出文件是否存在且合法（方案 B，因为修改了源文件）
- Phase 3 包含 API 回调：收集 merged 任务 ID，调用外部 API 标记 resolved
