# Eval Grader：生成产物质量评估

生成完 Skill 全部文件后，启动独立 eval agent 审查产物质量。eval agent 是通过 `zulu run` 启动的完整 agent 会话，拥有文件读取、命令执行等全部工具能力，可以自行决定用什么手段检查。

---

## 流程

```
主 Agent 完成 Step 4（SKILL.md 编写完毕，所有脚本已生成）
    ↓
构建 grader prompt → 写入临时文件
    ↓
zulu run 启动 eval agent（独立会话，--cwd 指向生成的 skill 目录）
    ↓
eval agent 自行检查所有文件 → 将评估结果写入 eval-report.json
    ↓
主 Agent 读取 eval-report.json
    ↓
有 findings？
  → 是：抽象泛化问题 → 修复 → 再启动一轮 eval agent
  → 否：通过，进入 Step 5
    ↓
最多循环 3 轮。第 3 轮仍有问题 → 将剩余问题展示给用户
```

---

## Grader Prompt 模板

以下模板由主 Agent 在发起 eval 前组装。`{skillDir}` 替换为生成的 Skill 目录的绝对路径，`{reportPath}` 替换为评估报告的绝对路径。

````
你是一个独立的质量评审员，负责审查一个刚生成的长程任务 Skill 的全部文件。
你的目标是找出会导致这个 Skill 在实际运行时失败、效果不佳、或让使用它的 Agent 产生误解的隐含问题。

Skill 目录位于：{skillDir}

请自行读取目录中的所有文件（SKILL.md、scripts/*.js、references/*.md），用你认为合适的方式检查。
以下是重点关注的维度，但不限于此——如果你发现其他问题也请报告。

## 检查维度

### 1. 路径硬编码
脚本中是否存在写死的绝对路径（如 `/Users/xxx/...`、`/home/xxx/...`）。
所有路径应从命令行参数（`--root`）或 `.agent.env` 中读取，不应假设特定的目录结构。
检查范围包括脚本代码和 build-prompt 生成的 prompt 文本。

### 2. 指令语气
面向 Agent 的指令文本（SKILL.md、phase references、build-prompt 生成的 prompt）中，
是否存在大量"必须"、"严禁"、"不得"等强制性措辞而没有解释原因。

Agent 不是执行命令的机器人——它需要理解 **为什么** 要遵守某个约束，才能在边界情况下做出正确判断。
好的约束写法是"做 X，因为如果不做会导致 Y"，而不是"必须做 X"。

检查时注意：不是说完全不能用强制语气，而是每个强制约束都应该附带理由。
如果一个约束的理由显而易见（如"输出必须是合法 JSON"），简短的理由即可。
如果理由不显然，缺少解释就是一个问题。

### 3. 脚本可执行性
对 scripts/ 目录下的每个 .js 文件，尝试 `node --check <file>` 验证语法合法性。
如果发现语法错误，报告具体的错误信息。
也检查脚本之间的引用关系是否正确（如 dispatch.js require build-prompt.js 的路径）。

### 4. 契约一致性
dispatch.js 判定子任务成功时检查的产出文件路径，与 build-prompt.js 告诉 subagent 写入的路径是否一致。
这是最常见的隐含 bug——两边路径不匹配导致所有子任务都被判定为失败。

### 5. 自包含性
用 build-prompt.js 构建出的 prompt 是否真正自包含：
- 是否包含完整的文件路径列表（不能只有占位符）
- 是否明确告知了输出文件的完整路径
- 是否包含输出格式的示例
- 是否解释了验证标准（什么算成功）
subagent 是全新会话，不知道任何上下文，prompt 中不能假设它"已经知道"任何信息。

### 6. 幂等性
discover.js 重复运行时是否会覆盖已完成的任务状态。
正确的行为是：加载已有清单 → 保留已完成条目 → 只处理新增或变更的条目。

### 7. 其他
你认为值得报告的任何问题。信任你的判断——如果某个设计让你觉得"这在实际跑的时候可能会出问题"，就报告它。

## 输出要求

将评估结果写入 `{reportPath}`，JSON 格式：

```json
{
  "pass": false,
  "findings": [
    {
      "dimension": "路径硬编码",
      "severity": "error",
      "location": "scripts/dispatch.js:47",
      "description": "产出文件路径硬编码为 /tmp/task-data/...，应从 --root 参数派生",
      "suggestion": "改为 path.join(root, '.task-data', 'segments', ...)"
    }
  ],
  "summary": "发现 2 个 error、1 个 warning。主要问题集中在路径处理和 prompt 自包含性。"
}
```

severity 分两级：
- **error**：会直接导致运行时失败或产出错误（如路径不匹配、语法错误）
- **warning**：不会直接导致失败，但会降低效果或可维护性（如缺少理由的强制措辞、缺少正反例）

如果没有发现任何问题，输出 `{ "pass": true, "findings": [], "summary": "未发现问题" }`。

**不要报告风格偏好或锦上添花的建议。只报告你认为会实际造成问题的 findings。**
````

---

## 主 Agent 处理反馈的原则：抽象泛化

收到 eval 报告后，主 Agent 修复时遵循一个核心原则：**不针对单个 finding 打补丁，而是识别 finding 背后的模式，做泛化修复。**

### 示例

**eval 报告**：
> dispatch.js 第 47 行硬编码了 `/tmp/task-data/segments/`

**错误的修复方式**：只改第 47 行。

**正确的修复方式**：
1. 识别模式：脚本中可能多处存在路径拼接没有基于 `root` 参数的问题
2. 全局检查所有脚本中的路径拼接
3. 统一改为从 `--root` 参数派生
4. 如果 build-prompt 生成的 prompt 中也有类似问题，一并修复

### 示例

**eval 报告**：
> SKILL.md 中"严禁使用 delegate_subtask"缺少理由解释

**错误的修复方式**：只给这一条加上理由。

**正确的修复方式**：
1. 识别模式：所有面向 Agent 的指令文本中的强制约束都应检查是否附带了理由
2. 扫描 SKILL.md 和所有 phase references 中的强制措辞
3. 统一补充"因为..."的理由说明

### 泛化判断标准

拿到每个 finding 时问自己：**这个问题是否可能在其他位置也存在？** 如果是，先做全局排查再修复，不要只修 eval 指出的那一处。

---

## 循环终止条件

- eval agent 返回 `"pass": true` → 终止，进入 Step 5
- 已完成 3 轮 eval → 终止，将剩余 findings 展示给用户
- 连续两轮 findings 相同（修复未生效）→ 终止，展示给用户并说明修复尝试未成功
