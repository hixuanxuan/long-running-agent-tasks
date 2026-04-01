# Phase Reference 编写模式

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

---

## 各 Phase 核心内容

- **Phase 0**：按 `zulu_cli.md`「环境初始化规范」5 步完成环境检查，将所有后续 Phase 需要的变量**一次性写入 `.agent.env`**，运行 `setup-env.js` 验证退出码。Phase 0 结束后各 Phase 只从 `.agent.env` 读取变量，不再询问用户。
- **Phase 1**：运行 `discover.js` → 检查分块结果（chunk 数、文件数、行数分布）→ 确认配置 → 可选的依赖分析
- **Phase 2**：确认 model → 运行 `dispatch.js` → 进度检查 → 重试失败任务 → 中断恢复说明。dispatch.js 通过 `zulu run` 调度 subagent（**严禁使用 `delegate_subtask`**），命令选项详见 `zulu_cli.md`
- **Phase 3**：运行 `merge.js` → 全局验证（按任务类型定制：tsc/build/schema）→ 生成报告 → 可选清理

Phase 3 的"全局验证"是最关键的定制点——迁移类跑 tsc + build，审查类检查 JSON 完整性，修复类跑 build + 清理 worktree。
