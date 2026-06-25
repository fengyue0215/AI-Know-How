# learn-claude-code 深度解析 —— 从 0 到 1 构建 AI Coding Agent

> 来源：[shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)（68.4k ⭐ / 11.1k forks）  
> 一个"从零构建 Claude Code 级 Agent Harness"的渐进式教程，20 章 / Python 实现 / MIT 开源

---

## 📋 目录

| 章节 | 内容 |
|------|------|
| 一 | 核心哲学：Agency 从哪里来 |
| 二 | 20 章渐进架构全景 |
| 三 | 核心实现深潜（Agent Loop / Tools / Permission / Hooks） |
| 四 | 上下文管理（Compact 4 层 / Subagent 隔离 / Memory 3 子系统） |
| 五 | 多 Agent 协作（Teams / Protocols / Worktree） |
| 六 | 扩展与组装（MCP / Skills / Comprehensive） |
| 七 | 教学版 vs 生产版对照 |
| 八 | 对 C++ 后端开发者的价值 |

---

## 一、核心哲学：Agency 从哪里来

### 1.1 核心理念

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   "Agency —— 感知、推理和行动的能力 ——                        │
│    来自模型训练，而非外部代码编排"                              │
│                                                             │
│   Agent 产品 = Model（驾驶员）+ Harness（载具）                │
│                                                             │
│   Harness = Tools + Knowledge + Observation                  │
│            + Action Interfaces + Permissions                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Agency 的历史脉络

| 年份 | 里程碑 | 关键启示 |
|------|--------|---------|
| 2013 | DeepMind DQN 从原始像素玩 Atari | Agent 行为来自训练，不是 if-else |
| 2019 | OpenAI Five 击败 Dota 2 世界冠军 | 45,000 年自我对弈训练出的 Agent |
| 2019 | DeepMind AlphaStar 达到星际争霸 II 宗师 | 多任务 Agent 来自大规模 RL |
| 2024-25 | LLM Agents 重塑软件工程 | **同样的模式**——Agency 是训练出来的 |

### 1.3 什么不是 Agent

教程明确批判了"prompt-plumbing"工具——拖拽式工作流构建器、无代码"AI Agent"平台——称之为"Rube Goldberg machines"，只是"披着宏大外衣的 shell 脚本"。

### 1.4 Harness 工程师的工作

```
Harness 工程师的职责（不是训练模型，而是为模型构建运行环境）：

  • 实现工具 —— 文件 IO、Shell、API 调用、浏览器控制、数据库查询
  • 编排知识 —— 产品文档、ADR、代码风格、合规要求
  • 管理上下文 —— 子 Agent 隔离、上下文压缩、任务系统
  • 控制权限 —— 沙盒、审批工作流、信任边界
  • 收集轨迹数据 —— 行动序列作为未来模型训练的信号
```

### 1.5 为什么研究 Claude Code

> "Claude Code 是我们见过的最优雅、最完整的 Agent Harness 实现。"

教程将 Claude Code 拆解为一组独立机制：Agent Loop、Tools、Skill Loading、Context Compaction、Subagent Spawning、Task System、Mailbox Team Coordination、Worktree Isolation、Permission Governance、Hooks、Memory、MCP Routing。

**核心结论**：最好的 Agent 产品来自理解"我的工作是 Harness，不是智能"的工程师。

---

## 二、20 章渐进架构全景

### 2.1 六阶段学习路径

```
Phase 1 — 核心能力（s01-s11）

  阶段 1：让 Agent 行动    s01 Loop → s02 Tools → s03 Permission → s04 Hooks
  阶段 2：处理复杂工作      s05 TodoWrite → s06 Subagent → s08 Context Compact
  阶段 3：记忆与恢复        s09 Memory → s10 System Prompt → s11 Error Recovery

Phase 2 — 高级能力（s12-s20）

  阶段 4：运行长任务        s12 Task System → s13 Background → s14 Cron
  阶段 5：协调多 Agent      s15 Teams → s16 Protocols → s17 Autonomous → s18 Worktree
  阶段 6：扩展与组装        s07 Skills → s19 MCP → s20 Comprehensive
```

### 2.2 全部 20 章速览

| 章节 | 主题 | 核心格言 | 关键概念 |
|------|------|---------|---------|
| s01 | Agent Loop | "One loop & Bash is all you need" | `while True`, `stop_reason`, 模型决策 + Harness 执行 |
| s02 | Tool Use | "Adding a tool = one handler" | 分发映射 `TOOL_HANDLERS`, 5 个工具, `safe_path` |
| s03 | Permission | "先设边界，再给自由" | 3 门审批管道, deny list, rule matching, user approval |
| s04 | Hooks | "围绕 Loop 钩入，永不重写 Loop" | PreToolUse / PostToolUse 扩展点 |
| s05 | TodoWrite | "没有计划的 Agent 会漂移" | `TodoItem`, 先规划再执行 |
| s06 | Subagent | "大任务拆小，每个子任务独立上下文" | 全新 `messages[]`, 上下文隔离, fork 模式 |
| s07 | Skill Loading | "按需加载知识，不预先装载" | `SkillManifest`, 两阶段加载 |
| s08 | Context Compact | "上下文总会满——要有腾空间的方法" | snipCompact, microCompact, toolResultBudget, autoCompact |
| s09 | Memory | "记住重要的，忘记不重要的" | 3 子系统：selection/extraction/consolidation |
| s10 | System Prompt | "Prompt 在运行时组装，不是硬编码" | 分段拼接, 条件注入 |
| s11 | Error Recovery | "错误不是终点，是重试的起点" | token 升级, fallback model, 重试策略 |
| s12 | Task System | "大目标拆成小任务，有序且持久化" | `TaskRecord`, `blockedBy`, JSON 持久化 |
| s13 | Background Tasks | "慢操作走后台，Agent 继续思考" | 线程执行, 通知队列 |
| s14 | Cron Scheduler | "按时间触发，无需人工踢一脚" | 持久化调度, 会话级触发 |
| s15 | Agent Teams | "一个 Agent 搞不定——委派给队友" | `MessageBus`, inbox, 权限冒泡 |
| s16 | Team Protocols | "队友需要共享通信规则" | 关闭握手, 计划审批 |
| s17 | Autonomous Agents | "队友自己看任务板，自己抢活" | 空闲循环, 自动认领, 自组织 |
| s18 | Worktree Isolation | "各干各的目录，互不干扰" | `WorktreeRecord`, 任务-目录绑定 |
| s19 | MCP Plugin | "能力不够？通过 MCP 插更多" | 多传输, 通道路由, 工具池 |
| s20 | Comprehensive | "许多机制，一个 Loop" | 全部机制在同一个 Loop 中协同 |

---

## 三、核心实现深潜

### 3.1 Agent Loop（s01）—— 最小的可运行内核

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return  # 模型没有请求工具 → 任务完成

        # 执行工具，结果追加回 messages
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
        # 循环回顶部，模型看到工具结果后继续推理
```

**关键洞察**：
- 这个 30 行的 Loop 就是 Claude Code 1729 行 `query.ts` 的**核心**
- 其余 1700 行全是保护机制（压缩、权限、Hook、重试）
- 教学版用 `stop_reason` 控制循环；生产版用 `needsFollowUp` flag（因为 streaming 中 `stop_reason === 'tool_use'` 不可靠）

### 3.2 Tool Use（s02）—— 分发模式

```python
TOOL_HANDLERS = {
    "bash":       run_bash,
    "read_file":  run_read,
    "write_file": run_write,
    "edit_file":  run_edit,
    "glob":       run_glob,
}

# Loop 中只改一行：
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS[block.name]   # 分发
        output = handler(**block.input)        # 执行
```

**添加工具 = `TOOLS` 中一个条目 + `TOOL_HANDLERS` 中一个映射。**

**并发执行模式**（生产版）：
- 不是简单的"读并行/写串行"
- 是 `isConcurrencySafe()` 函数按工具+输入判断
- 连续的安全调用形成并行批次；遇到不安全调用开始新串行批次

### 3.3 Permission（s03）—— 三道门审批管道

```
Tool Use Request
    │
    ▼
Gate 1: Hard Deny List（静态黑名单）
    ├─ 匹配 → ⛔ 立即阻断
    │  例：rm -rf /, sudo, shutdown, mkfs, dd if=
    └─ 未匹配 → 继续
                │
                ▼
      Gate 2: Rule Matching（上下文相关）
          ├─ 无规则匹配 → ✅ 直接执行（大多数操作的快速路径）
          └─ 规则匹配 → 继续
             例：写入工作区外 → 触发
                 包含 "rm " 的命令 → 触发
                        │
                        ▼
              Gate 3: User Approval
                  ├─ y/yes → ✅ 执行
                  └─ 其他 → ⛔ 拒绝
```

**生产版关键差异**：
- 4 种行为：`allow/deny/ask` + **`passthrough`**（工具不表态，传递给泛型管道）
- 8 个规则来源（userSettings, projectSettings, localSettings, flagSettings, policySettings, cliArg, command, session），有优先级排序
- **YoloClassifier**：Auto 模式下用分类器 LLM 判断工具调用安全性
- **Permission Bubbling**：子 Agent 的权限提示冒泡到父终端

### 3.4 Hooks（s04）—— 围绕 Loop 的扩展点

```
         PreToolUse Hook
              │
              ▼
    [模型请求工具] → [审批管道] → [执行工具] → [PostToolUse Hook]
                                              │
                                              ▼
                                        [格式化/通知/日志]
```

Hook 的核心原则：**"围绕 Loop 钩入，永不重写 Loop"**。Loop 本身不变，Hook 是独立的外部函数。

---

## 四、上下文管理

### 4.1 Compact 4 层 + 1 层应急（s08）

```
执行顺序（Claude Code query.ts 源码顺序）：

  L3: toolResultBudget  →  将大结果持久化到磁盘（先做，保证内容还在）
  L1: snipCompact       →  裁剪中间旧对话
  L2: microCompact      →  旧 tool_result 替换为占位符
  L4: autoCompact       →  LLM 生成全文摘要（唯一需要 API 调用的）
  
  应急层：Reactive Compact → API 返回 413 时更激进地压缩
```

| 层级 | 机制 | 触发条件 | API 调用 |
|:---:|------|---------|:---:|
| L1 | snipCompact | 消息数 > 50 | 0 |
| L2 | microCompact | 只有最近 3 个 tool_result 保持完整 | 0 |
| L3 | toolResultBudget | 最后一条 user msg 的 tool_result 总大小 > 200KB | 0 |
| L4 | autoCompact | L1-L3 后 token 仍超阈值 | **1** |
| 应急 | Reactive Compact | API 返回 413 `prompt_too_long` | 0-1 |

**设计哲学**：**"便宜的先来，贵的最后"**。前 3 层零 API 调用。

**压缩后恢复**（生产版特有）：压缩后不只是保留摘要 —— 自动重新附加最近文件、计划、Agent/Skill/Tool 上下文，预算 50,000 token，最多 5 个文件，每个最多 5K token。

### 4.2 Subagent（s06）—— 上下文隔离

```
主 Agent                         子 Agent
  │                                │
  │ task("分析 event_loop.cpp")    │
  ├──────────────────────────────→│
  │                                │ 全新 messages[] = [任务描述]
  │                                │ 自己的 Loop（最多 30 轮）
  │                                │ 受限工具（无 task，防止递归）
  │                                │ 读文件、搜索、分析...
  │                                │
  │        只返回最终结论           │
  │←──────────────────────────────│
  │ 中间过程全部丢弃                │ 销毁
```

**核心设计决策**：

| 维度 | 选择 | 理由 |
|------|------|------|
| 上下文 | 全新 `messages[]` | 子 Agent 中间步骤不污染主上下文 |
| 返回值 | 仅最终结论文本 | 不返回整个消息列表 |
| 递归防护 | 子 Agent 无 `task` 工具 | 不能再创建子 Agent |
| 安全 | 工具调用经过 PreToolUse Hook | 上下文隔离 ≠ 权限隔离 |

**生产版的 Fork 模式**：不是创建全新上下文，而是通过 `buildForkedMessages()` 构建缓存友好的前缀，让父子共享 Prompt Cache。5 个条件必须逐字节相同：system prompt、tools、model、message prefix、thinking config。

### 4.3 Memory（s09）—— 3 个子系统

```
┌─────────────────────────────────────────────────────────────┐
│  Memory 3 个子系统                                           │
│                                                             │
│  1. Selection（选择）                                        │
│     每次用户请求开始时触发                                    │
│     MEMORY.md 索引导入 SYSTEM prompt（可被 Prompt Cache）    │
│     用 Sonnet 侧查询选择相关记忆（最多 5 个）                 │
│     非 embedding 相似度 —— CC 用 Sonnet 自己来选             │
│                                                             │
│  2. Extraction（提取）                                       │
│     每轮对话自然结束后触发（stop_reason != tool_use）         │
│     监控对话中的隐含偏好和约束                                │
│     用户不必说"记住这个"——系统自动发现                        │
│     去重检查后才写入                                          │
│                                                             │
│  3. Consolidation（整合 / "Dream"）                          │
│     文件数 ≥ 10 时触发                                       │
│     去重、合并矛盾、修剪过时记忆                              │
│     生产版有 4 层门控：时间门（≥24h）→ 扫描节流 →             │
│     会话门（≥5 个新 transcript）→ 锁门（.consolidate-lock）  │
└─────────────────────────────────────────────────────────────┘
```

---

## 五、多 Agent 协作

### 5.1 Agent Teams（s15-s17）

```
┌─────────────────────────────────────────────────────────────┐
│  Team 架构                                                   │
│                                                             │
│                    ┌─────────────┐                          │
│                    │  Team Lead  │                          │
│                    │ 分配+汇总   │                          │
│                    └──┬───┬───┬─┘                          │
│                       │   │   │                             │
│              ┌────────┘   │   └────────┐                    │
│              ▼            ▼            ▼                    │
│        ┌─────────┐  ┌─────────┐  ┌─────────┐              │
│        │Teammate │  │Teammate │  │Teammate │              │
│        │  A      │  │  B      │  │  C      │              │
│        └─────────┘  └─────────┘  └─────────┘              │
│                                                             │
│  通信：MessageBus + inbox（点对点直连，非层级汇报）          │
│  工具：TaskCreate, SendMessage, Worktree 隔离                │
│  自主：空闲 Agent 自动扫描任务板、认领工作（s17）             │
└─────────────────────────────────────────────────────────────┘
```

**s17 自主 Agent 的循环**：
```
while True:
    task = scan_board()     # 扫描任务板
    if task:
        claim(task)         # 认领
        execute(task)       # 执行
    else:
        idle()              # 等待新任务
```

---

## 六、扩展与组装

### 6.1 Skill Loading（s07）—— 两阶段加载

```
启动时：只加载 SkillManifest（名称 + 描述）→ ~30-50 token/skill
调用时：匹配到 skill → 加载完整 SKILL.md 内容
```

### 6.2 MCP Plugin（s19）

- 多传输支持（HTTP / Stdio，SSE 已弃用）
- 通道路由
- 工具池动态组装
- 上下文警告：5 个 MCP + 58 个工具 ≈ 55,000+ token

### 6.3 Comprehensive（s20）

所有机制在一个 Loop 中协同："Many mechanisms, one loop."

---

## 七、教学版 vs 生产版对照

教程在每个章节都有"与生产 CC 的差异"附录，以下是关键对照：

| 维度 | 教学版（learn-claude-code） | 生产版（Claude Code 源码） |
|------|---------------------------|--------------------------|
| **Loop 控制** | `stop_reason` | `needsFollowUp` flag（streaming 更可靠） |
| **工具并发** | 顺序执行 | `partitionToolCalls()` 分区并行 |
| **权限行为** | 3 种（allow/deny/ask） | 4 种（+ passthrough） |
| **权限来源** | 1 个 deny list | 8 个优先级排序的来源 |
| **权限分类器** | 无 | YoloClassifier（Auto 模式） |
| **子 Agent fork** | 全新 messages | 3 种模式（Normal/Fork/General-Purpose） |
| **压缩后恢复** | 无 | 自动重附加文件+计划（50K token 预算） |
| **Memory 整合** | 文件数阈值 | 4 层门控（时间/扫描/会话/锁） |
| **验证管道** | 2 步（deny + rule） | ~8 步（Zod + validateInput + hooks + ...） |
| **运行环境** | Python（教学清晰度优先） | TypeScript on Bun（性能优先） |

---

## 八、对 C++ 后端开发者的价值

### 8.1 为什么值得学

| 学习内容 | 对你写 C++ 的实际帮助 |
|---------|---------------------|
| **Agent Loop 原理** | 理解 AI 如何决策 → 写出更精准的 Prompt |
| **权限模型设计** | 类比：如何为你自己的 CLI 工具设计安全边界 |
| **上下文压缩 4 层** | 知道何时 `/compact`，避免关键信息被压缩丢失 |
| **Subagent 隔离模式** | 理解 `/fork` 和 `/workflow` 的真实开销和收益 |
| **Memory 3 子系统** | 善用 `/memory` 和 CLAUDE.md，让 AI 跨会话积累知识 |
| **Hook 扩展点** | 写出更精准的 PostToolUse hook（clang-format、ctest） |
| **Tool 设计模式** | 为自己的 MCP Server 设计更合理的工具接口 |

### 8.2 核心收获

```
1. Claude Code 的本质是一个 Harness
   → 你的 CLAUDE.md、MCP Server、Hook 脚本、Skills
   → 都是这个 Harness 的组成部分
   → 你的工作是"为 AI 提供更好的运行环境"

2. 复杂度是逐层叠加的，Loop 本身永远不变
   → 就像 C++ 的"零开销抽象"原则
   → 每个机制都是可选模块，按需启用

3. 上下文管理是 Agent 工程的核心挑战
   → 4 层压缩 + Subagent 隔离 + Memory 3 子系统
   → 这三个机制共同解决了"AI 的注意力窗口"
```

### 8.3 与现有 know-how 的关联

| 本教程章节 | 对应的 know-how 文档 |
|-----------|-------------------|
| s01-s02 Agent Loop + Tools | `Claude-Code-功能深度详解.md` 第一章 |
| s03 Permission | `Claude-Code-功能深度详解.md` 第九章 |
| s04 Hooks | `Claude-Code-功能深度详解.md` 第四章 |
| s06 Subagent | `Claude-Code-功能深度详解.md` 第七章 |
| s08 Context Compact | `Claude-Code-源码分析汇总.md` 模式 5 |
| s09 Memory | `Claude-Code-功能深度详解.md` 第三章 |
| s15-s18 Agent Teams | `Claude-Code-功能深度详解.md` 第七-八章 |
| s19 MCP | `MCP-Server开发入门.md` |
| s07 Skills | `Claude-Code-功能深度详解.md` 第五章 |

---

## 📦 快速开始

```bash
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env   # 填入 ANTHROPIC_API_KEY

# 从最简单的 Loop 开始
python s01_agent_loop/code.py

# 体验上下文压缩
python s08_context_compact/code.py

# 看完整 Agent
python s20_comprehensive/code.py
```

---

## 📚 相关资源

| 资源 | 链接 |
|------|------|
| 仓库主页 | https://github.com/shareAI-lab/learn-claude-code |
| 中文 README | https://github.com/shareAI-lab/learn-claude-code/blob/main/README-zh.md |
| Kode Agent CLI（作者新项目） | `npm i -g @shareai-lab/kode` |
| OpenClaw（姊妹项目） | 常驻 Agent + 心跳 + cron + 多渠道 IM |

---

*文档生成日期：2026-06-24 | 来源：shareAI-lab/learn-claude-code（MIT 开源）*
