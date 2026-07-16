# Work with AI —— Claude Code vs Kiro 深度对比 + 实战案例

> PPT 设计 + 案例文档 | 聚焦两大工具 | 约 40 张幻灯片 / 60 分钟

---

## 🎯 PPT 结构（40 张 / 60 分钟）

```
开场        4张/5min    冲击力 + 为什么只讲这两个
全景对比    8张/12min   哲学差异 + 架构差异 + 场景选择
Claude Code  6张/10min   深度演示 + 核心优势
Kiro         6张/10min   深度演示 + 核心优势
案例实战    8张/13min   同一需求，两种工具，两种路径
总结行动    8张/10min   选型指南 + 5 条原则 + 资料包
```

---

## Part 1：开场（4 张 / 5 min）

---

### Slide 1 · 封面

```
Work with AI

Claude Code vs Kiro —— 两种范式，一个目标

[姓名] · [日期]
```

---

### Slide 2 · 为什么只讲这两个

```
7 款工具 → 聚焦 2 款。为什么？

Claude Code                     Kiro
终端 Agent 之王                 Spec 驱动 IDE 新范式
代表"敏捷灵活"                 代表"结构化可审计"

两种截然不同的开发理念
理解它们的差异 = 理解 AI 编程的未来方向
```

---

### Slide 3 · 一句话定义

```
┌─────────────────────────┐  ┌─────────────────────────┐
│                         │  │                         │
│    Claude Code          │  │    Kiro                 │
│                         │  │                         │
│  "你对 AI 说一句话       │  │  "你先写需求文档        │
│   AI 自己去理解代码库    │  │   AI 生成结构化 Spec     │
│   读文件、写代码、跑测试 │  │   你逐阶段审批           │
│   循环迭代直到完成"      │  │   AI 按任务列表实施"     │
│                         │  │                         │
│  像和一个超级实习生结对   │  │  像管理一个工程团队      │
│                         │  │                         │
└─────────────────────────┘  └─────────────────────────┘
```

---

### Slide 4 · 今天的目标

```
听完这 60 分钟，你将能够：

✅ 准确判断什么场景用 Claude Code，什么场景用 Kiro
✅ 理解两种工具的底层哲学差异
✅ 看到一个真实 C++ 案例在两种工具下的完整实现路径
✅ 拿到两个工具的完整配置模板
```

---

## Part 2：全景对比（8 张 / 12 min）

---

### Slide 5 · 哲学差异 —— 核心

```
                    Claude Code                  Kiro

核心问题      "如何让 AI 更好地理解和        "如何让 AI 的行为可预测、
              修改代码？"                     可审计、可追溯？"

优化目标      推理自主性、速度、并行度        开发过程的状态管理和质量保证

代码定位      "代码是真相源"                  "Spec 是真相源，代码是派生品"

人类角色      架构师 + Reviewer               意图架构师 + 审批者

典型用法      claude -p "重构事件循环"         kiro spec create → 需求→设计→任务→执行
```

---

### Slide 6 · 哲学差异 —— 类比

```
Claude Code 像什么？              Kiro 像什么？

🏎️  F1 赛车                      🚂  高速列车

- 极快、极灵活                    - 有轨道、有站点
- 需要熟练的驾驶员                 - 按时按点到达
- 路线可以随时改变                 - 路线是规划好的
- 适合有经验的开发者               - 适合需要确保不出错的团队

- 价值在"速度"                    - 价值在"确定性"
```

---

### Slide 7 · 架构差异

```
Claude Code 架构                  Kiro 架构

while true:                      Spec Pipeline:
  messages = buildPrompt()         requirements.md ← EARS 格式
  response = stream(messages)          ↓ 你审批
  tools = extractTools(response)   design.md       ← 架构设计
  if no tools: break                   ↓ 你审批
  executeTools(tools)              tasks.md        ← 原子任务
  maybeCompact()                       ↓ 逐任务执行
                                  每个变更有 spec 溯源

模型自主决策一切                   Spec 约束模型的行为边界
```

---

### Slide 8 · 工作流对比 —— 同一需求的两种路径

```
需求："实现 Redis 连接池，C++17, hiredis, 30s 超时, 自动重连"

┌─── Claude Code ─────────────┐  ┌─── Kiro ──────────────────┐
│                             │  │                            │
│ $ claude                    │  │ $ kiro spec create         │
│                             │  │   redis-pool               │
│ > 实现 Redis 连接池          │  │                            │
│   支持 5-20 连接             │  │ Phase 1: 生成              │
│   30s 超时，自动重连         │  │ requirements.md            │
│   先规划方案                 │  │   → 你审查 EARS 验收标准    │
│                             │  │                            │
│ AI 读代码库                  │  │ Phase 2: 生成              │
│ → 输出方案                   │  │ design.md                  │
│ → 你确认                     │  │   → 你审查架构决策          │
│ → 生成代码+测试+CMake        │  │                            │
│ → /review 审查               │  │ Phase 3: 生成              │
│                             │  │ tasks.md (7 个原子任务)     │
│ 总耗时：~5 分钟              │  │   → 你审批后逐任务执行      │
│ 产出：代码 + 测试 + CMake    │  │                            │
│                             │  │ 总耗时：~15 分钟            │
│                             │  │ 产出：Spec + 代码 + 审计链  │
└─────────────────────────────┘  └────────────────────────────┘
```

---

### Slide 9 · 场景决策矩阵

```
                    简单/低风险                    复杂/高风险
                    ──────────────────────────────────────────→

Claude Code ✅      Bug 修复              中型重构
                    "修这个编译错误"        "epoll → io_uring"
                    单文件小功能            跨模块 API 变更
                    快速原型                多文件协调修改

Kiro ✅             不划算                  大型新功能
                    Spec 流程对             "实现新的消息队列层"
                    一行修复太重            遗留代码分析
                                           安全关键代码
                                           需要审计追溯的功能

两者配合            用 Claude Code 快速探索 → 用 Kiro 固化成果
                    用 Kiro 写 Spec → 用 Claude Code 执行任务
```

---

### Slide 10 · 功能对比总表

```
┌──────────────────────┬─────────────────────┬─────────────────────┐
│ 维度                  │ Claude Code         │ Kiro                │
├──────────────────────┼─────────────────────┼─────────────────────┤
│ 类型                  │ 终端 Agent           │ AI IDE (Code OSS)   │
│ 安装                  │ curl | bash          │ 下载安装包          │
│ 价格                  │ $20-200/月           │ 免费-$200/月        │
│ 模型                  │ 仅 Claude 系列       │ Claude + Qwen + GLM │
│ 上下文                │ 1M token             │ 语义索引            │
│ 多文件编辑            │ 85-95% 成功率        │ Spec 驱动的协调修改 │
│ 自动补全              │ ❌ 无                │ ✅                  │
│ 终端集成              │ ✅ 原生              │ ✅ CLI 工具         │
│ Git 集成              │ ✅ 原生              │ ✅ PR 创建           │
│ Plan 模式             │ ✅ /plan             │ ✅ 强制 3 阶段      │
│ 回滚                  │ /rewind              │ ✅ 原子回滚         │
│ 事件自动化            │ Hook (11 种事件)     │ Agent Hooks (6 种)  │
│ 子 Agent              │ ✅ 1-数百并行        │ ✅ 自定义子代理     │
│ 离线                  │ ❌                   │ ❌                  │
│ CLion 支持            │ ❌                   │ ❌                  │
│ 学习曲线              │ 低（终端用户自然）   │ 中（需理解 Spec 流程│
│ 最佳场景              │ 复杂重构/探索/CI     │ 审计/合规/大团队    │
└──────────────────────┴─────────────────────┴─────────────────────┘
```

---

### Slide 11 · 成本对比

```
┌────────────────┬────────────┬────────────┬────────────┐
│ 使用强度        │ Claude Code│ Kiro       │ 差距        │
├────────────────┼────────────┼────────────┼────────────┤
│ 轻度            │ $20/mo     │ Free       │ Kiro 便宜   │
│ 中度            │ $100/mo    │ $20-40/mo  │ Kiro 便宜   │
│ 重度            │ $200/mo    │ $100-200/mo│ 持平        │
│ API 直连重度    │ $600-1500  │ -          │ Claude 贵   │
└────────────────┴────────────┴────────────┴────────────┘

关键差异：
- Claude Code: 订阅制（Max $100-200/月）包大量使用
- Kiro: Credit 制，Opus 消耗是 Auto 的 2.2x
- 两者都有永久免费层（Kiro 50 credits/mo, Claude Code 无）
```

---

### Slide 12 · 过渡页

```
Part 3

Claude Code 深度演示

"一句话 → Plan → 确认 → 代码"
```

---

## Part 3：Claude Code 深度演示（6 张 / 10 min）

---

### Slide 13-18 · Claude Code 6 张演示

```
Slide 13: Claude Code 核心工作流动图
  $ claude → > 描述需求 → AI Plan → 确认 → 生成代码 → /review

Slide 14: 终端截图 —— Claude Code 启动 + /init
  展示 CLAUDE.md 生成过程

Slide 15: Plan Mode 截图
  "实现 Redis 连接池" → AI 输出架构方案（不写代码）

Slide 16: 执行截图
  确认后 AI 读取代码库、生成头文件+实现+测试+CMake

Slide 17: /review 截图
  AI 自动审查刚生成的代码

Slide 18: Hook 自动化
  PostToolUse → clang-format → 保存即格式化
```

**演讲者备注**：这部分直接 Live Demo，slides 是备份。强调 3 个要点：1) Plan Mode 先规划再改；2) /review 自动审查；3) Hook 零成本自动化。

---

## Part 4：Kiro 深度演示（6 张 / 10 min）

---

### Slide 19-24 · Kiro 6 张演示

```
Slide 19: Kiro 核心工作流动图
  kiro spec create → requirements.md → 审批 → design.md → 审批
  → tasks.md → 审批 → 逐任务执行 → Agent Hooks 自动测试

Slide 20: Phase 1 截图 —— requirements.md
  展示 EARS 格式的验收标准
  WHEN getConnection() THEN RedisPool SHALL 30秒内返回连接
  IF 连接池满 THEN RedisPool SHALL 返回超时错误

Slide 21: Phase 2 截图 —— design.md
  架构决策、组件边界、API 契约、数据模型

Slide 22: Phase 3 截图 —— tasks.md
  7 个原子任务，有序、可独立审批和回滚

Slide 23: 执行截图 —— 逐任务实施
  每个任务完成后 Agent Hooks 自动运行 cmake + ctest

Slide 24: Steering 文件展示
  .kiro/steering/tech.md, structure.md, network-standards.md
  条件加载: inclusion: fileMatch on "src/network/**"
```

**演讲者备注**：强调 Kiro 的独特价值——"每次变更都知道为什么做"。这不是快的问题，是确定性和可追溯性的问题。

---

## Part 5：案例实战（8 张 / 13 min）⭐ 核心

---

### Slide 25 · 案例引入

```
案例：C++ Game Server —— 实现 Redis 缓存层

背景：
- C++17, CMake, hiredis 异步 API
- 项目禁用异常，Google 代码风格
- 现有：裸 Redis 连接，无连接池，无重试
- 目标：连接池 + 自动重连 + 健康检查 + 内存统计集成

我们用 Claude Code 和 Kiro 分别实现这个需求
看两种工具在同一个真实场景下的差异
```

---

### Slide 26 · Claude Code 路径 —— 概览

```
Claude Code 路径：5 分钟，一句话启动

$ cd ~/projects/game-server
$ claude

> 我要实现 Redis 连接池。当前用裸 hiredis 连接，
> 没有连接池和重试。请：
> 1. 先分析现有代码（include/db/redis_client.h, src/db/）
> 2. 规划方案
> 3. 我确认后实施

整个过程：
  T+0:00  AI 读取现有代码
  T+0:30  AI 输出方案（头文件接口、连接池设计、重试策略）
  T+1:00  确认方案
  T+3:00  AI 生成 redis_pool.h, redis_pool.cpp, 测试, CMakeLists.txt
  T+4:00  AI 运行 cmake --build && ctest → 通过
  T+5:00  /review → AI 审查自己的代码 → 发现 2 个改进点
```

---

### Slide 27 · Claude Code 路径 —— 关键截图

```
[Claude Code 终端截图]

Plan 阶段：
  AI 分析了 redis_client.h 和 src/db/
  当前模式：每次请求创建新连接，用完不回收 → 连接数随时间增长
  建议：hiredis async API + 连接池 + shared_ptr 管理生命周期
  接口：RedisPool(5, 20, 30s), getConnection(), returnConnection()

执行阶段：
  → include/db/redis_pool.h    (78 行)
  → src/db/redis_pool.cpp      (215 行)
  → tests/db/redis_pool_test.cpp (142 行)
  → CMakeLists.txt 更新

/review 审查结果：
  🟡 redis_pool.cpp:127 — checkHealth 中未检查 hiredis 返回值
  🔵 redis_pool.h:34   — 可以加 [[nodiscard]] 防止忽略返回值
```

---

### Slide 28 · Kiro 路径 —— 概览

```
Kiro 路径：15 分钟，结构化三步

$ cd ~/projects/game-server
$ kiro .
$ kiro spec create redis-pool

Phase 1: requirements.md
  Kiro 分析现有代码 → 生成 EARS 验收标准
  你审查: "连接数范围应该是 5-20 不是 10-50" → Kiro 更新

Phase 2: design.md
  Kiro 生成架构设计
  你审查: "用 hiredis async API，不是同步" → Kiro 更新
  你审查: "接口不要暴露 hiredis 类型（Pimpl）" → Kiro 更新

Phase 3: tasks.md（7 个任务）
  你审批 → Autopilot 逐任务执行
  每个任务完成后: Hook 自动 cmake + ctest
```

---

### Slide 29 · Kiro 路径 —— requirements.md 展示

```markdown
# Redis 连接池需求
.kiro/specs/redis-pool/requirements.md

## User Story: 获取可用连接
**Event-driven:**
WHEN 组件调用 getConnection()
THEN RedisPool SHALL 在 30 秒内返回一个可用连接

**Conditional:**
IF 池中有空闲连接 THEN RedisPool SHALL 立即返回该连接
IF 池空 AND count < max THEN RedisPool SHALL 创建新连接
IF 池满 THEN RedisPool SHALL 等待最多 30 秒后返回超时错误

**State-driven:**
WHILE Redis 服务器不可达
RedisPool SHALL 每 5 秒尝试重连，最多 3 次后标记连接失效

## User Story: 连接生命周期
**Unconditional:**
RedisPool SHALL 使用 std::unique_ptr 管理 hiredis 连接
RedisPool SHALL 在析构时关闭所有连接
RedisPool SHALL 通过 MemoryPool::record() 记录每次分配
```

---

### Slide 30 · Kiro 路径 —— 审计链展示

```
Kiro 产出的完整审计链：

requirements.md  →  "为什么做这个功能"
    ↓
design.md        →  "为什么选择这个方案"
    ↓
tasks.md         →  "做了什么、按什么顺序"
    ↓
git commits      →  "每次变更对应的任务"

审查时：
  代码变更 → 追溯 tasks.md → 追溯 design.md → 追溯 requirements.md
  "这行代码为什么这么写？" → 答案在审计链中

合规审计时：
  "你们的 Redis 连接池有哪些验收标准？"
  → 直接指向 requirements.md（EARS 格式）
  → 可追溯、可验证、可复现
```

---

### Slide 31 · 同一需求 —— 两种工具对比总结

```
┌──────────────────┬──────────────────┬──────────────────┐
│                   │ Claude Code      │ Kiro              │
├──────────────────┼──────────────────┼──────────────────┤
│ 启动方式          │ 一句话 Prompt    │ 写需求描述        │
│ 总耗时            │ ~5 分钟          │ ~15 分钟          │
│ 前期 overhead     │ 几乎为零         │ 3 阶段审批        │
│ 代码质量保证      │ /review + Hook   │ Spec 驱动的强制   │
│                   │                  │ 审查              │
│ 可追溯性          │ git history      │ git + Spec 文档   │
│ 团队协作          │ CLAUDE.md 共享   │ Spec 共享审查     │
│ 适合              │ 快速迭代         │ 需要确保不出错    │
│ 不适合            │ 需要合规审计     │ 一行修 bug        │
│                   │                  │                   │
│ 类比              │ 超级实习生       │ 工程团队          │
│                   │ 你说→他做→你审   │ 你写需求→团队评审  │
│                   │                  │ →逐任务执行→你验收│
└──────────────────┴──────────────────┴──────────────────┘
```

---

### Slide 32 · 混合使用 —— 最佳实践

```
实际项目中最优策略：两者配合

场景 A：快速探索 + 迭代
  Claude Code 快速生成原型
  → 验证可行后
  → Kiro 生成 Spec 固化方案

场景 B：大功能开发
  Kiro 写 Spec（确保需求完整）
  → Claude Code 执行 tasks.md（快）

场景 C：后期维护
  Claude Code 日常修改
  → Kiro 保留 Spec 作为"活文档"

场景 D：CI/CD 集成
  Claude Code 无头模式：PR 自动审查
  Kiro Hook：保存时自动测试 + 格式化
```

---

## Part 6：总结行动（8 张 / 10 min）

---

### Slide 33 · 选型决策树

```
开始
 │
 ├─ 需要审计追溯？合规要求？
 │   ├─ 是 → Kiro
 │   └─ 否 ↓
 │
 ├─ 快速迭代？未知领域探索？
 │   ├─ 是 → Claude Code
 │   └─ 否 ↓
 │
 ├─ 大型新功能？多人协作？
 │   ├─ 是 → Kiro（Spec 驱动） + Claude Code（执行）
 │   └─ 否 ↓
 │
 ├─ 日常维护？bug 修复？
 │   ├─ 是 → Claude Code
 │   └─ 否 ↓
 │
 ├─ AWS 生态？
 │   ├─ 是 → Kiro（原生集成）
 │   └─ 否 ↓
 │
 └─ 预算有限？
     └─ Claude Code Pro ($20) or Kiro Pro ($20)
```

---

### Slide 34 · 两个工具的快速上手

```
Claude Code 3 步开始：
  1. curl -fsSL https://claude.ai/install.sh | bash
  2. cd project && claude
  3. /init

Kiro 3 步开始：
  1. https://kiro.dev/downloads 下载安装
  2. kiro . 打开项目
  3. kiro spec create <name>  创建第一个 Spec

混合配置建议：
  .claude/settings.json     ← Claude Code Hook + 权限
  .kiro/steering/           ← Kiro 项目规范
  CLAUDE.md                 ← 两者共享的项目约定
```

---

### Slide 35 · 5 条核心原则

```
Work with AI：5 条原则

1. 分层用模型
   复杂 Claude/GPT，日常 Gemini/DeepSeek

2. 选对范式
   快速探索 → Claude Code
   可靠交付 → Kiro

3. 写好项目规范
   CLAUDE.md + .kiro/steering/
   10 分钟 → 每次交互受益

4. 小步验证
   每次 150-300 行，编译+Sanitizer+测试

5. 人类兜底
   AI 是工具，你是最终责任人
```

---

### Slide 36 · 资料包

```
会后资料包（16 个文档）

今天的核心：
  📄 PPT设计方案-Work-with-AI.md       ← 本 PPT 源文件
  📄 Kiro-IDE深度详解.md              ← Kiro 完整手册
  📄 Claude-Code-功能深度详解.md       ← Claude Code 完整手册

补充深度：
  📄 C++-AI编码反模式与陷阱.md
  📄 团队Prompt模板库.md
  📄 成本优化实战指南.md
  📄 本地离线AI部署方案.md
  📄 CI-CD-AI集成方案.md
  📄 MCP-Server开发入门.md
  📄 learn-claude-code-深度解析.md
  📄 Claude-Code-源码分析汇总.md
  📄 CppCon-2025-AI-C++-演讲汇总.md
  📄 前沿研究方向与行业动态.md
```

---

### Slide 37-40 · Q&A + 结束

```
Slide 37: 常见疑问
  Q: 我应该先学哪个？
  A: Claude Code。学习成本更低，先感受 AI 编程的价值。
     然后用 Kiro 解决 Claude Code 解决不好的问题（审计、合规、大团队协作）。

  Q: 两个工具能否同时用？
  A: 可以。同一个项目同时配置 .claude/ 和 .kiro/。不冲突。

  Q: C++ 支持好吗？
  A: 两者都支持 C++。Kiro 的 --from-existing 对遗留代码分析特别有用。

Slide 38-40: Thank You + Q&A
```

---

## 🎬 演讲节奏

```
0:00-0:05   开场（4 张）         —— 为什么只讲这两个
0:05-0:17   全景对比（8 张）     —— 哲学 + 架构 + 场景矩阵
0:17-0:27   Claude Code（6 张）  —— Live Demo 或截图演示
0:27-0:37   Kiro（6 张）         —— Live Demo 或截图演示
0:37-0:50   案例实战（8 张）      —— 同一需求，两种路径 ⭐
0:50-1:00   总结行动（8 张）     —— 选型 + 原则 + Q&A
```

---

*设计日期：2026-06-24*
