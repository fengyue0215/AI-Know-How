# CI/CD AI 集成方案 —— 让 AI 参与自动化代码审查

> 基于 Meta CppCon 2025 演讲 "Refactoring at Scale" 的启示：  
> 将 LLM 驱动的代码审查集成到 CI 流水线，作为第一道防线。

---

## 🎯 一、整体架构

```
                        PR 提交
                           │
                           ▼
                  ┌────────────────┐
                  │  静态分析层     │  ← 快速、确定性
                  │  clang-tidy    │
                  │  cppcheck      │
                  │  clang-format  │
                  └───────┬────────┘
                          │ 通过
                          ▼
                  ┌────────────────┐
                  │  AI 审查层      │  ← 语义理解、上下文感知
                  │  Claude Code   │
                  │  /review       │
                  └───────┬────────┘
                          │
                    ┌─────┴─────┐
                    │           │
                 通过        发现问题
                    │           │
                    ▼           ▼
              ┌─────────┐  ┌──────────┐
              │ 人工审查 │  │ AI 建议   │
              │ (最终)   │  │ 附在 PR  │
              └─────────┘  └──────────┘
```

分层策略：静态分析快速过滤（秒级）→ AI 语义审查（分钟级）→ 人工最终确认。

---

## 🔧 二、Claude Code 无头模式集成

### 基础用法

```bash
# 非交互模式 —— 适合 CI
claude -p "Review this git diff for C++ memory safety issues" \
       --allowedTools "Read,Grep,Glob" \
       --max-turns 10 \
       --output-format json
```

| 参数 | 说明 |
|------|------|
| `-p "prompt"` | 单次 Prompt 模式（非交互） |
| `--allowedTools` | 限制工具（CI 中只允许只读工具） |
| `--max-turns` | 最大工具调用轮数 |
| `--output-format json` | 结构化输出（方便 CI 解析） |
| `--model` | 指定模型（CI 中建议用 sonnet） |
| `--no-permission-prompt` | CI 无头模式 |

### GitHub Actions 示例

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 需要完整 git 历史

      - name: Install Claude Code
        run: curl -fsSL https://claude.ai/install.sh | bash

      - name: AI Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # 获取 PR 的 diff
          git diff origin/${{ github.base_ref }}...HEAD > pr.diff

          # AI 审查
          claude -p "$(cat <<EOF
          Review this C++ PR diff. Focus on:
          1. Memory safety (leaks, dangling pointers, double free)
          2. Thread safety (data races, deadlocks)
          3. Undefined behavior
          4. Exception safety
          5. Resource management (RAII compliance)
          6. Performance regressions

          For each issue found, output:
          - Severity: CRITICAL / WARNING / SUGGESTION
          - File and line number
          - Description
          - Suggested fix

          PR diff:
          $(cat pr.diff)
          EOF
          )" --allowedTools "Read" --max-turns 5 --output-format json > review.json

      - name: Post Review Comment
        uses: actions/github-script@v7
        with:
          script: |
            const review = require('./review.json');
            const body = formatReview(review);
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: body
            });
```

### GitLab CI 示例

```yaml
# .gitlab-ci.yml
ai-review:
  stage: review
  image: ubuntu:latest
  before_script:
    - apt-get update && apt-get install -y curl git
    - curl -fsSL https://claude.ai/install.sh | bash
  script:
    - git diff origin/main...HEAD > pr.diff
    - |
      claude -p "Review this C++ PR diff for safety issues: $(cat pr.diff)" \
        --allowedTools "Read" \
        --max-turns 5 \
        --output-format json > review.json
    - python3 scripts/post_review.py review.json
  only:
    - merge_requests
```

---

## 🤖 三、静态分析 + LLM 组合策略

这是 Meta 验证过的方案：先让静态分析工具定位问题区域，再让 LLM 针对具体问题生成修复。

```bash
#!/bin/bash
# ai_refactor_pipeline.sh

# 步骤 1：静态分析
echo "=== Step 1: Static Analysis ==="
clang-tidy src/**/*.cpp --export-fixes=fixes.yaml

# 步骤 2：提取问题
echo "=== Step 2: Extract Issues ==="
python3 -c "
import yaml
with open('fixes.yaml') as f:
    data = yaml.safe_load(f)
# 提取前 10 个最严重的问题
"

# 步骤 3：让 AI 生成修复
echo "=== Step 3: AI Fix Generation ==="
claude -p "$(cat <<EOF
The following clang-tidy issues were found in our C++ codebase.
For each issue, read the relevant source file and propose a fix.

Issues:
$(cat fixes.yaml | head -200)

For each fix, output:
- The exact edit (old_string → new_string)
- Why this fix is correct
- Any remaining risk
EOF
)" --allowedTools "Read,Edit" --max-turns 10

# 步骤 4：验证修复
echo "=== Step 4: Verify Fixes ==="
cmake --build build && cd build && ctest
```

### 与 Meta 方案的对比

| 维度 | Meta 方案 | 我们的方案 |
|------|----------|-----------|
| 代码规模 | 百万行级 | 数千到数十万行 |
| LLM 调用 | 内部定制流水线 | Claude Code API / CLI |
| 执行频率 | Nightly pipeline | 每次 PR |
| 防护栏 | 多层 Guardrail | 编译 + Sanitizer + 测试 |

---

## 📊 四、AI 审查的效果度量

```yaml
# 建议在 CI 中收集以下指标
metrics:
  ai_review:
    issues_found: 12          # AI 发现的问题数
    true_positives: 9         # 人工确认为真实问题
    false_positives: 3        # AI 误报
    precision: 0.75           # 精确率

  time_saved:
    ai_review_time_sec: 45    # AI 审查耗时
    human_review_before_min: 30  # 引入 AI 前人工审查耗时
    human_review_after_min: 15   # 引入 AI 后人工审查耗时
```

---

## 🛡️ 五、安全考量

| 关注点 | 措施 |
|--------|------|
| **API Key 安全** | 使用 GitHub Secrets / GitLab CI Variables，绝不硬编码 |
| **代码泄露** | CI 中只传 diff 而非完整源文件（除非必要） |
| **AI 自动 Merge** | ❌ 绝对禁止。AI 只能建议，Merge 必须人工 |
| **模型幻觉** | AI 审查结果标记为"suggestion"，不作为阻断条件 |
| **资源限制** | `--max-turns` 限制 AI 调用次数，防止 CI 超时/超预算 |

---

## 🚀 六、渐进式落地路线

```
Phase 1（第 1 周）：影子模式
  - AI 审查在 CI 中运行，但不阻断 merge
  - 收集数据：精确率、召回率、耗时
  - 调整 Prompt 和参数

Phase 2（第 2-3 周）：建议模式
  - AI 审查结果作为 PR comment 自动发布
  - Reviewer 可以参考 AI 建议
  - 建立人工确认流程

Phase 3（第 4 周+）：增强模式
  - 特定类型问题（格式化、明显的内存泄漏）AI 可自动修复
  - 复杂问题仍需人工审查
  - 持续度量效果
```

---

*文档生成日期：2026-06-24 | 参考：Meta CppCon 2025、GitHub Actions 文档、Claude Code CLI 文档*
