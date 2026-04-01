# Prompt 设计规范

本文档聚合 subagent prompt 相关的所有设计决策：质量原则、结构规范、Context Budget 控制。

---

## 1. Prompt 质量核心原则

每一条产出都必须自带"为什么必须处理"的论证。如果 subagent 无法为某个发现写出具体的触发场景和影响说明，说明它自己也没有想清楚为什么要报告——这条发现大概率是噪音。因此，好的 prompt 应当通过输出格式和规则设计，迫使 subagent 对每条产出进行因果推理，而非简单的模式匹配。

---

## 2. 好 prompt 的五个组成部分

1. **清晰的任务描述**（1-2 句，不要长篇大论）
2. **核心约束 + 正反例**（规则说明什么该报告，正反例示范什么样的报告合格——正反例对产出质量的影响远大于规则条文本身）
3. **待处理文件**（完整的路径）
4. **输出要求**（明确的输出路径 + 格式示例，subagent 不用猜写到哪、写成什么样）
5. **验证标准**（什么样的结果算成功，和 dispatch 的成功判定逻辑对齐）

### 完整示例（以 codebase-review 为例）

```
请对以下文件完成代码审查任务，找出 bug、安全风险、规范违反和可维护性问题。

## 审查规则
- 只报告你能构造出具体触发场景的问题
- 每个问题必须回答：什么条件下触发？触发后的直接后果是什么？
- 如果你无法描述一个合理的触发场景，不要报告它
- 如果没有发现问题，输出空数组

### 正例
trigger: "并发请求时 race condition 导致余额重复扣减"
impact: "用户资金损失，影响所有并发下单场景"
suggestion: "在 deductBalance() 中加分布式锁"
→ 合格：触发条件明确，后果严重且可验证

### 反例
message: "第 12 行使用了 let，应改为 const"
→ 不合格。缺少触发场景和影响说明。即使是规范问题，也必须说明违反的是哪条规范、放任不管会导致什么后果。

同一问题的正确写法：
trigger: "声明后未被重新赋值的变量使用了 let"
impact: "违反团队 oxlint prefer-const 规范，积累后导致 lint 规则形同虚设，新成员无法通过现有代码学习正确写法"
suggestion: "改为 const"

## 待审查文件路径
src/utils/auth.ts
src/api/user.ts

## 输出要求
将审查结果写入 `.task-data/segments/chunk_src__utils-output.json`，格式：
{
  "chunkId": "chunk_src__utils",
  "findings": [
    {
      "file": "src/utils/auth.ts",
      "line": 42,
      "trigger": "当用户 token 过期后重新登录时，旧 session 未被清除",
      "impact": "攻击者可利用泄露的旧 token 持续访问，影响所有启用了 remember-me 的用户",
      "suggestion": "在 renewToken() 入口处调用 invalidateAllSessions(userId)"
    }
  ]
}

## 验证
输出文件必须是合法 JSON，findings 数组中每个条目必须包含 file、line、trigger、impact、suggestion 字段。

## IMPORTANT
**不得使用 delegate_subtask 工具**，所有工作必须由你直接完成**。
```
**Agent判定为复杂任务后倾向delegate_task去完成，禁止subagent自行delegate_subtask套娃，IMPORTANT章节的内容是必要的**。
---

## 3. Context Budget（subagent prompt token 预算）

长程任务最常见的失败模式是 subagent 的 prompt 过长。防御分两层：

**第一层：discover 阶段的分组上限。** 这是粗粒度控制——按行数或文件数切分 chunk，让大多数 chunk 天然落在安全范围内。分组上限的选型见 `architecture.md` § 分组策略。

**第二层：build-prompt.js 的运行时检查。** 这是兜底——组装完完整 prompt 后，估算实际 token 数（字符数 / 3.5 可粗略近似），如果超过模型上下文窗口的 60-70%，自动将该 chunk 拆分为 `{chunkId}_part_1`、`{chunkId}_part_2`。

两层配合的效果：分组阶段不需要精确计算 token（不同语言、不同注释密度的 token 密度差异很大），只做一个合理的粗切；运行时检查用实际字符数做最终裁决。

其他关键点：
- **宁小勿大**。chunk 偏小只是多跑几轮，chunk 过大会导致 subagent 漏处理尾部文件或输出被截断。
- **不要在 prompt 中塞其他 chunk 的结果**。遵守 Context Reset 原则，每个 subagent 只看自己的输入。
