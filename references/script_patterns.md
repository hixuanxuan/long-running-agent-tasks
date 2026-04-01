# 脚本接口与实现 Pattern

本文件定义各脚本的接口规范和实现要点。Agent 生成具体 Skill 时，根据这些接口和 pattern 编写完整实现。

所有脚本使用 Node.js 编写，纯 `require` 标准库（fs/path/child_process），零外部依赖。

---

## 通用 Pattern

### .env 加载

所有脚本（除 setup-env.js 外）共用同一个 loadEnv pattern：

```javascript
function loadEnv(root) {
    // 从 root 读取 .env，解析 KEY=VALUE 格式
    // 只设置 process.env 中尚未定义的变量（CLI 参数优先于 .env）
}
```

### 参数解析

统一使用 `--key value` 格式，手动解析 `process.argv`。不引入 yargs/commander 等外部依赖。

### Chunk ID

路径转 ID：`/` → `__`，`.` → `_dot_`，溢出加 `_part_N`。用 `toChunkId(module, partIndex)` 函数统一。

### Shell 安全引号

所有拼接 shell 命令的地方统一使用 `shellQuote` 函数，防止路径或内容中的特殊字符导致命令注入或解析错误：

```javascript
function shellQuote(s) {
    return "'" + s.replace(/'/g, "'\\''") + "'";
}
```

---

## setup-env.js

环境准入脚本。所有长程任务 Skill 共用几乎相同的实现。

**接口**：
```
node scripts/setup-env.js --root <dir> [--ptoken <token>] [--model <model>] [--concurrency <N>]
```

**职责（按顺序）**：
1. 打印 Node.js 版本确认可用
2. 检查 zulu CLI，不存在则 `npm i @comate/zulu -g --registry=http://registry.npm.baidu-int.com`
3. 确定 ZULU_PTOKEN：优先 `--ptoken` 参数 → 已有 `.env` → 缺失则退出码 1
4. 写入/更新 `.env`：合并已有配置行，只覆盖 `ZULU_` 开头的字段

**退出码**：0 = 就绪，1 = 缺少必要参数，2 = 依赖安装失败

**关键约束**：dispatch.js 运行前不再重复检查这些依赖——只需 `loadEnv()` 读取 .env，信任 setup-env 已完成准入。

---

## discover.js

扫描目标文件，按分组策略生成任务清单。分组策略的选型依据见 `architecture.md` § 分组策略，本节只关注实现。

**接口**：
```
node scripts/discover.js --root <dir> [--max-lines <N>] [--extensions <.ts,.tsx>]
```

**职责**：
1. 递归扫描目录，按 extensions 过滤，排除 node_modules/dist/build/.git/__tests__
2. 统计每个文件行数
3. 按分组策略组成 chunk
4. 生成任务清单

**幂等 pattern**（这是 discover 最重要的实现特征）：
```
加载已有清单 → 对每个新 chunk:
  - chunk ID 已存在且状态为 done → 保留，不覆盖
  - chunk ID 已存在且状态为 pending → 更新文件列表（文件可能新增/删除）
  - 新 chunk → 标记为 pending
```

**产出文件**（以 JSON manifest+inputs 为例）：
- `.task-data/manifest.json` — chunk 列表（id/module/totalLines/fileCount/status），不含 files 数组
- `.task-data/inputs/{chunkId}-input.json` — 每个 chunk 的文件列表，subagent 只读自己的 input

---

## dispatch.js

读取任务清单，调度 subagent 并行执行。

**接口**：
```
node scripts/dispatch.js --root <dir> [--concurrency <N>] [--model <model>] [--dry-run] [--retry-failed] [--ptoken <token>]
```

**--concurrency并发数量确定**：优先从用户query分析提取，如果没有则默认20，标识同时并发多少个子Agent。


**内部流程**：
```
1. loadEnv() 读取 .env
2. 读取任务清单 → 过滤 ready 任务（pending，或 --retry-failed 时包含 error/failed）
3. 初始化并发池（大小 = concurrency）
4. 对每个 chunk:
   a. 标记为 IN_PROGRESS
   b. 调用 buildPrompt() 构建 prompt
   c. 启动 subagent（zulu run）
   d. 检查产出文件 → 标记 done 或 error
   e. 立即写盘（crash-safe）
   f. 池中有空位 → 启动下一个
5. 所有 chunk 处理完毕 → 输出摘要
```

**subagent 调用 pattern**：

> **⚠️ 强制约束**：subagent 必须通过 `zulu run` CLI 启动，**严禁使用 `delegate_subtask` 工具替代**。`zulu run` 的完整参数说明、`--display` 输出模式及调试技巧见 `references/zulu_cli.md`。

> **数据流向**：`discover.js` 生成 `inputs/{chunkId}-input.json` → `buildPrompt()` 读取文件列表并组装自包含 prompt → 临时文件中转 → `--query` 将 prompt 作为子Agent的用户输入 → 子Agent将结果写入 `segments/{chunkId}-output.json` → dispatch.js 检查该文件判定成功。

下方是 dispatch 的 pattern 示例，包含异步进程执行和并发池。单次调用是其中的 `runChunk` 函数，并发控制由 `runPool` 驱动。

```javascript
const { exec } = require('child_process');

// ── 异步执行：dispatch 需要同时管理多个子进程，用 exec 的 Promise 封装使并发池正常工作 ──
function execAsync(cmd, opts) {
    return new Promise((resolve, reject) => {
        exec(cmd, opts, (err, stdout, stderr) => {
            if (err) reject(Object.assign(err, { stdout, stderr }));
            else resolve({ stdout, stderr });
        });
    });
}

// ── 并发池：控制同时运行的子进程数量 ──
async function runPool(tasks, concurrency, handler) {
    const executing = new Set();
    for (const task of tasks) {
        const p = handler(task).then(() => executing.delete(p), () => executing.delete(p));
        executing.add(p);
        if (executing.size >= concurrency) {
            await Promise.race(executing);  // 等待任意一个完成，腾出槽位
        }
    }
    await Promise.all(executing);  // 等待尾部任务全部完成
}

// ── 单个 chunk 的完整处理流程 ──
async function runChunk(chunk) {
    const { chunkId } = chunk;
    markInProgress(chunkId);

    // 1. 读取该 chunk 的输入文件
    const inputPath = path.join(root, '.task-data', 'inputs', `${chunkId}-input.json`);
    const { files } = JSON.parse(fs.readFileSync(inputPath, 'utf-8'));

    // 2. 构建完整 prompt（见下方 build-prompt.js 章节）
    const outputPath = path.join(root, '.task-data', 'segments', `${chunkId}-output.json`);
    const prompt = buildPrompt(root, files, { outputPath, chunkId });

    // 3. 通过临时文件传递 prompt，避免 shell 转义问题
    const tmpFile = path.join(cwd, `.tmp-query-${chunkId}-${Date.now()}.txt`);
    fs.writeFileSync(tmpFile, prompt, 'utf-8');
    try {
        const cmd = [
            `QUERY_CONTENT=$(cat ${shellQuote(tmpFile)});`,
            `zulu run`,
            `--ptoken=${shellQuote(ptoken)}`,
            `-m=${shellQuote(model)}`,
            `--query="$QUERY_CONTENT"`,
            `--cwd=${shellQuote(cwd)}`
        ].join(' ');
        await execAsync(cmd, { timeout: 30 * 60 * 1000, maxBuffer: 50 * 1024 * 1024 });
    } finally {
        try { fs.unlinkSync(tmpFile); } catch {}
    }

    // 4. 成功判定（Task Contract）：检查产出文件存在性 + 格式合法性
    if (fs.existsSync(outputPath)) {
        JSON.parse(fs.readFileSync(outputPath, 'utf-8')); // 验证格式
        markDone(chunkId);
    } else {
        markFailed(chunkId, 'no output file');
    }
    // 不要解析 subagent 的 stdout/stderr 来判断成功与否
}

// ── 主入口 ──
const readyChunks = manifest.filter(c => c.status === 'pending');
await runPool(readyChunks, concurrency, runChunk);
```

**写盘时机**：每个 chunk 状态变更后立即写 manifest，不等批次结束（markInProgress / markDone / markFailed 内部都应 writeFileSync）。

---

## build-prompt.js

为单个 chunk 构建 subagent prompt。体现 Context Reset 原则：prompt 必须完全自包含。

> Prompt 的设计哲学、质量原则、正反例和 Context Budget 控制见 `prompt_design.md`。

**接口**（module.exports + 可独立运行）：
```javascript
/**
 * @param {string} root       项目根目录
 * @param {string[]} files    相对文件路径列表
 * @param {Object} options    { outputPath, chunkId, rules? }
 * @returns {string}          完整 prompt 文本
 */
function buildPrompt(root, files, options) { ... }
```

**运行时 token 检查**：组装完 prompt 后估算 token 数（字符数 / 3.5），如果超过模型上下文窗口的 60-70%，自动将该 chunk 拆分为 `{chunkId}_part_1`、`{chunkId}_part_2` 并各自生成独立的 prompt。

**返回值与消费方式**：`buildPrompt()` 返回的是 **内存中的字符串**（`@returns {string}`），调用方 dispatch.js 负责将该字符串写入临时文件。
---

## status.js

查询任务进度。

**接口**：
```
node scripts/status.js --root <dir>
```

**输出格式**：
```
Total: 120 | Done: 115 | Failed: 3 | Skipped: 2 | Pending: 0
Progress: 96% (115/120)
```

**退出码**：0 = 全部终态，1 = 有未完成，2 = 清单不存在

---

## merge.js（可选）

合并所有 segment 为最终结果。

**接口**：
```
node scripts/merge.js --root <dir>
```

**职责**：遍历 segments 目录，解析每个 chunk 产出，合并为统一的 result.json（含 summary 统计）。

部分完成时也应可运行，summary 中标注完成比例。

---

## poll.js（可选）

检查子任务进程状态，补位启动 pending task。用于 worktree 隔离场景。

**接口**：
```
node scripts/poll.js --root <dir> [--concurrency <N>]
```

**职责**：
1. 检查 running 状态的任务：`process.kill(pid, 0)` 判断进程是否存活
2. 进程已退出 → 检查产出文件 → 标记 success 或 failed
3. 计算空闲槽位 → 启动 pending 任务填充

**退出码**：0 = 有活跃任务，2 = 全部终态
