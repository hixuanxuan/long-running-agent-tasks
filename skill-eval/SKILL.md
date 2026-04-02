---
name: skill-eval
description: >
  Skill 执行质量单次评测工具。对已有 Skill 进行一次端到端质量评估：生成测试用例、运行 Skill 产出、
  通过 Grader 对 assertions 评分、聚合 benchmark 统计、启动可视化评审页面呈现结果。
  当用户说"帮我评测这个 skill"、"跑一下测试看看效果"、"对比一下有没有 skill 的区别"、"run evals"、
  "benchmark this skill"、"评估 skill 质量"时使用。只做单次评测出报告，不含迭代改进循环。
---

# Skill Eval

对 Skill 的执行流程进行单次端到端质量评测：

```
生成测试用例 → 运行 Skill → Grading 评分 → Benchmark 聚合 → 呈现结果
```

## Phase 1: 生成测试用例

与用户确认后，生成 `evals/evals.json`：

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "用户的任务 prompt",
      "expected_output": "期望结果的描述",
      "files": [],
      "expectations": [
        "输出包含 X",
        "使用了 skill 的 Y 脚本"
      ]
    }
  ]
}
```

要求：
- 2-5 个真实用户会说的 prompt
- 让用户确认："这几个测试用例看着对吗？要加更多吗？"
- 完整 schema 见 `references/schemas.md`

## Phase 2: 运行测试 + 起草 Assertions

**目录结构**：

```
<skill-name>-workspace/
├── eval-0-descriptive-name/
│   ├── eval_metadata.json
│   ├── with_skill/
│   │   └── outputs/
│   └── without_skill/       # 或 old_skill/
│       └── outputs/
├── benchmark.json
└── benchmark.md
```

**Step 1: 同时发起所有运行**

对每个测试用例，同一轮发起两个 subAgent：

With-skill 运行：
```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/eval-<ID>/with_skill/outputs/
- Outputs to save: <用户关心的产出>
```

Baseline 运行（同 prompt，无 skill 或旧版 skill）：
- 新 skill → baseline 不带 skill，存到 `without_skill/outputs/`
- 改进 skill → baseline 用旧版（先 `cp -r` snapshot），存到 `old_skill/outputs/`

为每个 eval 创建 `eval_metadata.json`：
```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

eval_name 要有描述性，用它做目录名。

**Step 2: 等待期间起草 Assertions**

不要干等运行结束，趁这段时间：
1. 为每个测试用例起草 assertions，向用户解释
2. 好的 assertion 是客观可验证的，名字有描述性（在 benchmark viewer 里一目了然）
3. 主观性强的 skill（写作风格、设计质量）不要硬塞 assertions
4. 更新 `eval_metadata.json` 和 `evals/evals.json`

**Step 3: 捕获运行时数据**

每个 subAgent 完成时，通知中包含 `total_tokens` 和 `duration_ms`，立即保存到 `timing.json`：
```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```
错过就没了。

## Phase 3: Grading + Benchmark + 呈现结果

**1. Grade 每个运行**

读取 `agents/grader.md`，为每个运行评估 assertions，结果存为 `grading.json`。

关键：expectations 数组必须使用字段 `text`、`passed`、`evidence`（不是 `name`/`met`/`details`），viewer 依赖这些字段名。

能用脚本验证的 assertion，写脚本跑。

**2. 聚合 Benchmark**

```bash
python -m scripts.aggregate_benchmark <workspace> --skill-name <n>
```

生成 `benchmark.json` + `benchmark.md`。手动生成时参照 `references/schemas.md`。

**3. Analyzer 分析**

读取 `agents/analyzer.md`，分析 benchmark 数据：
- 无论有无 skill 都 100% 通过的 assertion（无区分度）
- 高方差 eval（可能 flaky）
- 时间/token 取舍

**4. 启动评审 Viewer**

```bash
nohup python <skill-eval-path>/eval-viewer/generate_review.py \
  <workspace> \
  --skill-name "my-skill" \
  --benchmark <workspace>/benchmark.json \
  > /dev/null 2>&1 &
VIEWER_PID=$!
```

无 display 环境用 `--static <output_path>` 生成静态 HTML。

务必用 `generate_review.py`，不要自己写 HTML。

告诉用户："评审页面已打开。'Outputs' tab 逐个查看测试用例，'Benchmark' tab 看定量对比。"

**5. 收尾**

用户看完后关掉 viewer：
```bash
kill $VIEWER_PID 2>/dev/null
```

汇总评测结论：哪些 case 表现好、哪些有问题、benchmark 数据的关键发现。

## Blind Comparison（可选）

需要更严谨的 A/B 对比时，读 `agents/comparator.md` 和 `agents/analyzer.md`。
两个输出匿名交给独立 agent 评判。大多数情况正常评测流程就够了。

## Claude.ai 适配

没有 subAgent 时：
- 逐个执行测试，自己读 SKILL.md 按指令完成任务
- 在对话中直接展示每个 prompt 和 output，文件存到 filesystem 让用户下载
- 跳过定量 benchmark，聚焦定性反馈

## 参考文件

- `agents/grader.md` — Grader Agent 指令
- `agents/comparator.md` — Blind A/B Comparator 指令
- `agents/analyzer.md` — Benchmark 分析器指令
- `references/schemas.md` — 所有 JSON schema（evals.json, grading.json, benchmark.json, timing.json, metrics.json, comparison.json, analysis.json）
