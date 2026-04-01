# Zulu CLI 参考

`zulu` 是长程任务框架的核心 CLI 工具。本文档记录 dispatch.js 调度 subagent 时用到的命令，以及 setup-env.js 环境准入时的依赖检查方式。

---

## 环境初始化规范

生成的 Skill 的 `setup-env.js` / Phase 0 必须按以下顺序完成环境检查与配置，并将所有后续阶段需要的变量**写入 `.agent.env`**（而非仅 export），确保各 Phase 读取时不依赖当前 shell 状态。

### Step 1 — 确保 Node.js 可用

```bash
node --version
```

若不存在，通过 fnm 安装：

```bash
curl -fsSL https://fnm.vercel.app/install | bash
fnm install --lts && fnm use lts-latest
# 安装后 source 对应 shell 配置文件（~/.bashrc 或 ~/.zshrc）
```

### Step 2 — 确保 zulu-cli 可用

使用`--version`命令获取版本，若不存在，或版本低于 `1.0.0`，则全局安装：
```bash
zulu --version
```
全局安装：

```bash
npm i @comate/zulu -g --registry=http://registry.npm.baidu-int.com
```

### Step 3 — 获取并持久化 ZULU_PTOKEN

分析并检查 `ZULU_PTOKEN` 环境变量是否已存在，优先从用户query中获取，其次从`~/.comate/login`文件获取。若都未设置，提示用户从团队平台(https://oneapi-comate.baidu-int.com/token)获取 token，并：
1. 持久化写入 shell 配置（`~/.bashrc` 或 `~/.zshrc`）
2. 写入工作目录下的 `.agent.env` 文件（**必须写文件，不能仅 export**）

```bash
# .agent.env 格式（后续所有脚本通过 dotenv 读取）
ZULU_PTOKEN=your_token_here
MODEL=Claude-3-7-Sonnet
PROJECT_DIR=/absolute/path/to/project
# ... 其他后续阶段需要的变量
```

> **执行 zulu 命令前**：若环境存在代理，必须先 unset：
> ```bash
> unset http_proxy https_proxy all_proxy
> ```

### Step 4 — 选择并确认模型

```bash
zulu list-model --ptoken="${ZULU_PTOKEN}"
```

按以下优先级选取：**Claude Sonnet > GPT > Gemini > GLM > MiniMax**，同类中优先选版本最新且具备 thinking 能力的。将结果写入 `.agent.env` 的 `MODEL` 字段。

### Step 5 — 确认所有路径为字面绝对路径

所有传入 zulu 子命令的路径（项目目录、输出目录等）**必须解析为字面绝对路径**，写入 `.agent.env` 时禁止使用 shell 变量形式（如 `$PROJECT_DIR`）。shell 变量在 `.agent.env` 文件中不会被展开，会导致静默失败。

```bash
# 错误（静默失败）
PROJECT_DIR=$HOME/myproject

# 正确
PROJECT_DIR=/Users/username/myproject
```

### .agent.env 一次性收集原则

Phase 0 结束时，`.agent.env` 中必须包含后续所有 Phase 需要的全部变量。各 Phase 从`.agent.env`读取环境变量，不得在中途再次询问用户。

---

## list-model — 查看可用模型

```bash
zulu list-model [-t,--ptoken <token>]
```

列出当前账号下可用的模型 ID 与显示名。Phase 0 用此命令让用户选择模型并写入 `.agent.env`。

---

## run — 启动 subagent

```bash
zulu run [options]
```

### 完整选项

| 选项 | 说明 |
|------|------|
| `-t, --ptoken <token>` | 内部 PToken（必填，来自 `.env` 的 `ZULU_PTOKEN`） |
| `-q, --query <text>` | 用户提交的内容（prompt 文本） |
| `-m, --model <name>` | 模型名称，支持 ID 和显示名，**推荐使用显示名** |
| `-c, --conversation <id>` | 对话 ID，用于多轮会话恢复 |
| `--mode <mode>` | 工作模式：`Agent`（默认）、`Ask`、`Plan` |
| `--cwd <path>` | 当前工作区路径 |
| `--ripgrep <path>` | ripgrep 可执行文件路径 |
| `--activate-rule <path>` | 激活的 Rule 文件路径 |
| `--activate-skill <name>` | 激活的 Skill 名称 |
| `-d, --display <mode>` | 输出模式（见下节） |

---

## --display 输出模式

### `task`（默认）

任务完成后输出一个带 Frontmatter 的 Markdown，正文为最后一轮文本。适合作为 subagent 消费。

```
---
conversationId: xxx
sessionStorePath: ~/.comate-engine/store/chat_session_xxx
status: ok
---
[本次任务的总结性文本]
```

dispatch.js 使用默认 `task` 模式时，**不依赖 stdout 文本判断成功与否**——始终通过检查产出文件来判定（见 Task Contract 原则）。

### `event-stream`

以每行一个 JSON 的形式流式输出增量数据，适合第三方界面对 CLI 的封装。

**事件类型**：

| type | 说明 | 关键字段 |
|------|------|----------|
| `conversation-messages` | 同步整轮消息列表 | `conversationInfo`, `messages` |
| `conversation-status` | 同步会话状态 | `conversationInfo` |
| `partial-message-data` | agent 消息级别增量信息 | `conversationInfo`, `messages` |
| `partial-message-elements` | 消息内 elements 的增量结构 | `conversationInfo`, `messageData` |

示例：

```jsonl
{"type":"conversation-messages","conversationInfo":{"id":"","status":"Running","type":"AgentConversation","utime":1773653613239,"ctime":1773653613239},"messages":[]}
{"type":"conversation-status","conversationInfo":{"id":"","status":"Running","type":"AgentConversation","utime":1773653613239,"ctime":1773653613239}}
{"type":"partial-message-data","conversationInfo":{"id":"","status":"Running","type":"AgentConversation","utime":1773653613243,"ctime":1773653613239},"messages":[{"id":"","v2":true,"role":"assistant","status":"inProgress","content":{},"elements":[{"id":"","discard":false,"children":[]}]}]}
{"type":"partial-message-elements","conversationInfo":{"id":"","status":"Running","type":"AgentConversation","utime":1773653613243,"ctime":1773653613239},"messageData":{"id":"","elements":[{"id":"","discard":false,"children":[{"id":"","type":"REASON","end":false,"content":"The","startTime":1773653617642,"lastModifiedTime":1773653617642}]}]}}
```

### `record-stream`

以每行一个 JSON 的形式输出**已闭合**的元素块（Reason、Text、ToolCall、ToolResult 结束后各输出一次）。与 `event-stream` 的区别是不输出中间状态，适合观察执行轨迹、调试 subagent 行为。

```jsonl
{"conversationId":"", "messages":[{"id":"", "type":"REASON", "content":""}]}
{"conversationId":"", "messages":[{"id":"", "type":"TOOL", "toolName":"", "params":{}, "toolState":"", "result":{}, "children":[]}]}
```

---

## dispatch.js 中的调用方式

> 完整调用示例、数据流向及成功判定 pattern 见 `script_patterns.md` § dispatch.js。

核心要点：通过临时文件传递 prompt（避免 shell 转义），使用默认 `task` 输出模式，**成功判定依赖产出文件，不解析 stdout**。

**何时使用 `--display record-stream`**：需要调试 subagent 的执行轨迹时，可将 `zulu run` 的输出重定向到日志文件，事后用 `jq` 分析。不推荐在生产调度中使用——会显著增加 stdout 体积，触发 `maxBuffer` 限制。

**何时使用 `-c, --conversation`**：dispatch.js 通常每个 chunk 启动独立会话（不传 `-c`）。如需在同一 chunk 的重试中保留上下文，可从上次 `task` 输出的 Frontmatter 中解析 `conversationId` 并传入。
