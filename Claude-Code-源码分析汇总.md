# Claude Code 源码分析汇总

> 社区对 Anthropic Claude Code（npm 包 `@anthropic-ai/claude-code`）的逆向工程与架构分析汇总  
> 基于 2025-2026 年间多个 GitHub 仓库的研究成果  
> 数据来源版本：v2.1.88（2026 年 3 月 npm 发布，含完整 source map）

---

## 📋 社区分析仓库一览

| 仓库 | 维护者 | 类型 | 亮点 |
|------|--------|------|------|
| [LetA-Tech/claude-code-from-source](https://github.com/LetA-Tech/claude-code-from-source) | LetA-Tech | 技术书籍 | 18 章 / ~400 页，10 个关键架构模式 |
| [ComeOnOliver/claude-code-analysis](https://github.com/ComeOnOliver/claude-code-analysis) | ComeOnOliver | 逆向文档 | 801 行详细文档，41 个工具 / 101 个命令 / 39 个服务 |
| [Yuyz0112/claude-code-reverse](https://github.com/Yuyz0112/claude-code-reverse) | Yuyz0112 | 运行时可视化 | API monkey-patching，追踪 Prompt / Tool Call / 压缩 |
| [PetrGuan/claude-code-anatomy](https://github.com/PetrGuan/claude-code-anatomy) | PetrGuan | 交互式可视化 | 26 页面 / 13 主题，架构地图 + 权限模拟器 |
| [bungphe/claude-code-source-code](https://github.com/bungphe/claude-code-source-code) | bungphe | 完整源码提取 | ~1,884 文件 / ~512K LOC，多语言分析报告 |
| [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) | VILA-Lab | 资源地图 | 链接 10+ 仓库、博客、复刻项目 |
| [sanbuphy/learn-coding-agent](https://github.com/sanbuphy/learn-coding-agent) | sanbuphy | 多层次研究 | 5 份报告（遥测 / 隐藏功能 / 卧底模式 / 远程控制 / 路线图） |
| [FlorianBruniaux/claude-code-ultimate-guide](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | FlorianBruniaux | 使用 + 架构指南 | 三级可信度标注，主循环 / 工具系统 / 上下文管理 |
| [@mseep/claude-code-source](https://www.npmjs.com/package/@mseep/claude-code-source) | mseep | 可运行重建 | 18 篇深度分析，支持多 LLM 提供商 / Ollama 本地模式 |

---

## 🏗️ 一、整体架构

### 1.1 项目规模

| 指标 | 数值 |
|------|------|
| TypeScript 源文件 | ~1,884 个 |
| 代码总行数 | ~512,664 行 |
| 最大单文件 | `query.ts`（~785KB / ~1,729 行） |
| 核心 Agent 循环 | `QueryEngine.ts`（~46KB） |
| 主 UI 组件 | `main.tsx`（~4,683 行） |
| 内置工具 | 40+ 个 |
| 斜杠命令 | 80+ 个（含 26 个隐藏命令） |
| React 组件 | 146+ 个 |
| 工具函数 | 564+ 个 |
| 服务模块 | 39 个 |
| npm 依赖 | ~192 个包 |
| 运行时 | Bun → 编译为 Node.js ≥ 18 |
| Feature Flag | 89（编译时） + GrowthBook 远程开关 |

### 1.2 目录结构（核心模块）

```
@anthropic-ai/claude-code/
├── lib/
│   ├── query.ts               # 🔥 核心 Agent 循环（最大文件）
│   ├── QueryEngine.ts          # 会话管理与状态机
│   ├── main.tsx                # 主 TUI 界面（Ink/React）
│   ├── tools/                  # 40+ 内置工具
│   │   ├── ReadTool.ts         # 文件读取
│   │   ├── WriteTool.ts        # 文件写入
│   │   ├── EditTool.ts         # 精确字符串替换
│   │   ├── BashTool.ts         # Shell 命令执行
│   │   ├── GrepTool.ts         # 代码搜索
│   │   ├── GlobTool.ts         # 文件名匹配
│   │   ├── TaskTool.ts         # 子 Agent 创建
│   │   ├── WebSearchTool.ts    # 网页搜索
│   │   └── ...                 # 30+ 更多工具
│   ├── context/                # 上下文管理
│   │   ├── compact/            # 4 层压缩（snip → microcompact → collapse → autocompact）
│   │   └── memory/             # 文件系统记忆
│   ├── hooks/                  # Hook 系统（事件驱动扩展）
│   ├── mcp/                    # MCP 协议客户端实现
│   ├── skills/                 # 技能系统（两阶段加载）
│   ├── permissions/            # 权限模型（allow/deny/ask）
│   └── telemetry/              # 遥测与隐私
├── vendor/                     # 编译后的依赖
└── package.json
```

---

## 🔄 二、Agent 主循环 —— 最核心的架构模式

### 2.1 循环而非 DAG

Claude Code 的架构极其简洁：**没有意图分类器、没有任务路由器、没有 RAG 管道、没有规划器**。

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent 主循环 (Master Loop)                  │
│                                                              │
│   while (true) {                                             │
│       messages = buildPrompt(userInput, context, tools)      │
│       response = streamToModel(messages)                     │
│       tools = extractToolCalls(response)                     │
│       if (no tools) break                                    │
│       results = executeTools(tools)  // 并行执行只读工具      │
│       messages.push(results)                                 │
│       maybeCompress(messages)                                │
│   }                                                          │
│                                                              │
│   模型自主决定：调用哪个工具、何时完成、是否需要更多信息       │
└──────────────────────────────────────────────────────────────┘
```

**设计哲学**：信任模型的判断力，不预设工作流。整个系统本质上是一个不断往 `messages[]` 数组追加内容的循环，直到模型返回纯文本（无 tool call）为止。

### 2.2 10 个关键架构模式

> 来源：`claude-code-from-source` 技术书籍（LetA-Tech）

| # | 模式 | 技术细节 | 价值 |
|---|------|---------|------|
| 1 | **AsyncGenerator 作为 Agent 循环** | 使用 JS `AsyncGenerator` yield Message 对象 | 天然支持背压和取消，代码简洁 |
| 2 | **推测性工具执行** | 在模型流式输出期间，提前启动只读工具（如文件读取） | 减少端到端延迟 |
| 3 | **并发安全批处理** | 按安全性分区：只读工具（Read/Grep/Glob）并行执行，写入工具（Write/Edit）串行执行 | 安全 + 性能 |
| 4 | **Fork Agent 共享 Prompt Cache** | 并行子 Agent 共享字节级相同的 prompt 前缀 | 节省 ~95% 输入 token 成本 |
| 5 | **4 层上下文压缩** | snip（裁剪） → microcompact（微量压缩） → collapse（折叠） → autocompact（自动压缩） | 渐进式降级，保持最大可用上下文 |
| 6 | **文件系统记忆 + LLM 召回** | 记忆以 Markdown 文件存储，Sonnet 侧查询选择相关记忆 | 轻量、透明、可手动编辑 |
| 7 | **两阶段技能加载** | 启动时只加载 frontmatter（名称+描述），调用时加载完整 skill 内容 | 节省 prompt 空间 |
| 8 | **Sticky Latches 保持缓存稳定** | Beta header 等特性开关一旦设置，整个会话期间永不取消 | 最大化 Anthropic API prompt cache 命中率 |
| 9 | **Slot 预留机制** | 默认请求 8K 输出 token，命中上限后自动升级到 64K | 避免输出截断 |
| 10 | **Hook 配置快照** | 启动时冻结 hook 配置，运行时不重新读取文件 | 防止运行时注入攻击 |

### 2.3 模式 4 详解 —— Fork Agent 共享 Prompt Cache

这是成本优化的核心设计：

```
主会话（messages 前缀 = 系统 prompt + CLAUDE.md + 工具定义）
  │
  ├── 子 Agent A（Fork）
  │   messages = [主会话前缀...] + [A 的专属 prompt]
  │   API 调用：前缀命中缓存 → 只对增量部分计费
  │
  ├── 子 Agent B（Fork）
  │   messages = [主会话前缀...] + [B 的专属 prompt]
  │   API 调用：前缀命中缓存 → 只对增量部分计费
  │
  └── 子 Agent C（Fork）
      messages = [主会话前缀...] + [C 的专属 prompt]
      API 调用：前缀命中缓存 → 只对增量部分计费

结论：并行创建 N 个子 Agent，输入 token 成本 ≈ 1 个 Agent 的成本
```

### 2.4 模式 5 详解 —— 4 层上下文压缩

```
会话长度增长 →
  │
  ├─ 第 1 层：Snip（裁剪）
  │   删除中间的空行、冗余注释、重复的工具结果
  │
  ├─ 第 2 层：Microcompact（微量压缩）
  │   合并连续的相似操作，摘要化工具输出
  │
  ├─ 第 3 层：Collapse（折叠）
  │   将早期对话转为 LLM 生成的摘要
  │
  └─ 第 4 层：Autocompact（自动压缩）
      触发条件：上下文使用率 > 80%
      自动调用 /compact，生成高度压缩的对话摘要
      旧消息替换为摘要，保留最近 N 轮完整信息
```

---

## 🔧 三、工具系统（Tool System）

### 3.1 工具分类

| 类别 | 工具 | 安全级别 |
|------|------|---------|
| **只读文件** | Read, Glob, Grep, LSP（goToDefinition/findReferences/hover） | 可并行 |
| **写入文件** | Write, Edit, NotebookEdit | 必须串行 |
| **Shell 执行** | Bash（含 git/gh/curl 等） | 需审批 |
| **Agent 创建** | TaskCreate, Agent, Workflow | 需审批 |
| **网络操作** | WebSearch, WebFetch | 只读 |
| **交互** | AskUserQuestion, SendMessage | 需用户参与 |
| **调度** | CronCreate, CronDelete, ScheduleWakeup, Monitor | 需审批 |
| **设计** | DesignSync | 需审批 |

### 3.2 工具执行的安全模型

```
┌──────────────────────────────────────────────────┐
│              工具调度器                            │
│                                                  │
│  收到 N 个 tool call                             │
│       │                                          │
│       ├─ 只读工具（Read/Grep/Glob）               │
│       │   → 并行执行（无副作用，不冲突）            │
│       │                                          │
│       ├─ 写入工具（Write/Edit）                    │
│       │   → 串行执行（保证文件一致性）             │
│       │                                          │
│       └─ Bash / 网络 / Agent                     │
│           → 按权限规则审批 → 串行执行              │
└──────────────────────────────────────────────────┘
```

### 3.3 工具定义的优化设计

每个工具使用 **Zod Schema** 定义参数：

```typescript
// 伪代码示例 —— 工具定义模式
const ReadTool = {
  name: "Read",
  description: "Reads a file from the local filesystem.",
  parameters: z.object({
    file_path: z.string().describe("Absolute path to the file"),
    offset: z.number().optional().describe("Start line"),
    limit: z.number().optional().describe("Max lines to read"),
  }),
  execute: async (params) => { /* 文件读取实现 */ }
}
```

工具定义本身的描述文本会被发送到模型，作为 function calling 的 tool schema。因此工具描述的精确性直接影响模型调用工具的准确性。

---

## 🧠 四、上下文与记忆管理

### 4.1 系统 Prompt 结构

```
┌────────────────────────────────────────────┐
│  系统 Prompt（每次 API 调用必带）            │
│                                            │
│  1. Claude 角色定义（"You are Claude..."）  │
│  2. 可用工具列表（名称 + 描述 + 参数 schema）│
│  3. 环境信息（OS/Shell/日期/工作目录）       │
│  4. 全局 CLAUDE.md（~/.claude/CLAUDE.md）   │
│  5. 项目 CLAUDE.md（./CLAUDE.md）           │
│  6. 本地 CLAUDE.md（./CLAUDE.local.md）     │
│  7. 记忆文件（Memory System）               │
│  8. 技能列表（Skills，仅 frontmatter）       │
│  9. 权限规则（Permissions）                 │
│  10. 当前会话状态                           │
└────────────────────────────────────────────┘
```

### 4.2 文件系统记忆设计

```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md              # 索引文件（每行一条记忆）
├── <memory-slug-1>.md     # 单条记忆（Markdown + frontmatter）
├── <memory-slug-2>.md
└── ...

每条记忆的结构：
---
name: short-kebab-case-slug
description: 一行摘要（用于 LLM 召回时的相关性判断）
metadata:
  type: user | feedback | project | reference
---

记忆正文内容...
```

**设计亮点**：
- 记忆为纯 Markdown 文件，开发者可以直接编辑
- 不在每次 API 调用时发送全部记忆，而是用一个**小型 Sonnet 侧查询**选择当前最相关的记忆
- 记忆之间支持 `[[wiki-link]]` 风格的相互引用

---

## 🎛️ 五、Hook 系统

### 5.1 架构

Hook 是事件驱动的扩展机制，配置在 `settings.json` 中：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "python3 ~/hooks/validate_bash.py"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "clang-format -i ${CLAUDE_ARG_FILE_PATH}"
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "command": "echo 'Claude Code task completed' | wall"
      }
    ]
  }
}
```

| Hook 事件 | 触发时机 |
|-----------|---------|
| `PreToolUse` | 工具执行前（可阻止） |
| `PostToolUse` | 工具执行成功后 |
| `Notification` | 任务完成通知 |
| `Stop` | 会话结束时 |
| `SessionStart` | 会话启动时 |

### 5.2 安全设计

- Hook 配置在**会话启动时快照**（snapshot），运行时不重新读取文件
- 防止运行时注入：即使有人修改了 hook 脚本文件，当前会话不受影响
- 所有 hook 命令在子进程中隔离执行

---

## 🕵️ 六、隐藏功能与 Feature Flag

> ⚠️ 以下内容来自社区对 source map 的分析，这些功能**未在公开版本中启用**

### 6.1 已知隐藏功能

| 代号 | 功能描述 | Feature Flag |
|------|---------|-------------|
| **KAIROS** | 常驻后台 AI 助手 —— 每日日志、"梦境"记忆整理、主动推送 | `KAIROS` |
| **BUDDY** | 虚拟 AI 伴侣宠物 —— 18 种物种、稀有度等级、由 userId 确定性生成 | `BUDDY` |
| **ULTRAPLAN** | 30 分钟远程规划模式（在 Opus 云实例上运行） | `ULTRAPLAN` |
| **Coordinator Mode** | 多 Agent 编排模式（工作分配、结果合并） | `COORDINATOR_MODE` |
| **Undercover Mode** | Anthropic 员工在公开仓库中自动去除 AI 署名 | `USER_TYPE === 'ant'` |
| **VOICE_MODE** | 语音交互管道 | `VOICE_MODE` |
| **DAEMON** | 后台进程管理器 | `DAEMON` |

### 6.2 未发布的内部模块（70+ 个）

这些模块只在 Anthropic 内部 monorepo 中存在，公开 npm 包中通过 `dead code elimination` 移除：

| 模块 | 功能推测 |
|------|---------|
| `daemon/main.js` | 后台守护进程 |
| `proactive/` | 主动式 AI（不等用户提问） |
| `coordinator/` | 多 Agent 协调 |
| `assistant/` | KAIROS 相关 |
| `REPLTool` | 交互式 REPL 工具 |
| `SnipTool` | 上下文裁剪工具 |
| `WebBrowserTool` | 浏览器自动化 |
| `PushNotificationTool` | 桌面推送通知 |
| `SubscribePRTool` | PR 订阅与自动审查 |

### 6.3 26 个隐藏斜杠命令

未在 `/help` 中列出但在源码中存在的命令（示例）：

| 命令 | 功能推测 |
|------|---------|
| `/ctx-viz` | 上下文可视化 |
| `/btw` | 隐藏调试信息 |
| `/good-claude` | 未知（可能是内部彩蛋） |
| `/naptime` | 暂停 Agent |
| ... | 其余 22 个 |

---

## 🔬 七、运行时逆向分析

### 7.1 可视化工具

[Yuyz0112/claude-code-reverse](https://github.com/Yuyz0112/claude-code-reverse) 通过 **API monkey-patching** 实现了对 Claude Code 运行时行为的可视化：

```
可视化内容：
├─ 每次 API 调用的完整 prompt（系统消息 + 工具定义 + 用户消息）
├─ 模型的完整响应（文本 + tool calls）
├─ 压缩（/compact）前后的上下文对比
├─ Token 消耗统计
└─ 工具调用链路图
```

这对理解 Claude Code **实际发送了什么 prompt** 和**模型如何决策**非常有价值。

### 7.2 关键发现

- **系统 Prompt 极其长** —— 单系统 Prompt（不含上下文）约 20K-50K token
- **压缩是常态** —— 长时间会话中，/compact 会被频繁触发
- **工具定义占据显著空间** —— 40+ 个工具的 JSON Schema 定义本身就是很大的上下文开销（约 8K-15K token）
- **权限规则被内联到系统 Prompt** —— 每个会话的权限规则是系统 Prompt 的一部分

---

## 🎯 八、对 C++ 后端开发者的启示

### 8.1 为什么要了解这些

| 了解的内容 | 实际价值 |
|-----------|---------|
| **Agent 主循环** | 理解 AI 如何决策，写出更精准的 Prompt |
| **上下文压缩机制** | 知道何时使用 `/compact`，避免关键信息被压缩丢失 |
| **工具执行模型** | 理解"只读并行 / 写入串行"的策略，优化自己的工具组合 |
| **Prompt Cache 共享** | 写 CLAUDE.md 时更有目的性（稳定前缀 = 节省成本） |
| **Hook 系统** | 可以写 Hook 实现自动格式化、自动测试、自动 commit |
| **文件系统记忆** | 善用 `/remember` 和 MEMORY.md，让 AI 在多次会话中积累知识 |

### 8.2 实战建议

```
1. 写 CLAUDE.md 时注意质量
   → CLAUDE.md 是每次 API 调用的"固定前缀"
   → 精确、无歧义的规范 = 更少的 token 浪费 = 更准确的 AI 行为
   → 避免废话，每个字都要有信息量

2. 理解工具调用成本
   → Read/Grep/Glob 是并行的（便宜）
   → Write/Edit/Bash 是串行的（每次都要等待）
   → 不要问"帮我看看这个项目" → AI 会串行读几百个文件
   → 应该问"帮我看看 src/network/ 下的事件循环代码"

3. 利用 /compact 时机
   → 当上下文使用率 >80% 时主动 compact
   → Compact 前确保关键信息已写入文件（CLAUDE.md 或记忆）
   → 压缩可能导致早期讨论的细节丢失

4. 善用 Hook
   → PostToolUse(Write) → clang-format 自动格式化
   → PostToolUse(Bash "cmake --build") → 自动运行 ctest
   → Notification → 长时间任务完成提醒
```

---

## 📊 技术栈一览

| 层级 | 技术 |
|------|------|
| 语言 | TypeScript |
| 运行时 | Bun → 编译为 Node.js ≥ 18 |
| UI 框架 | React + Ink（终端 TUI） |
| 状态管理 | React Context + 自定义 Store |
| 参数校验 | Zod |
| API 通信 | Anthropic SDK（SSE 流式） |
| 构建工具 | Bun bundler / esbuild |
| MCP 实现 | 自研 MCP Client |
| 测试 | Vitest + 自定义 harness |

---

## 📚 推荐阅读顺序

| 优先级 | 资源 | 适合人群 |
|--------|------|---------|
| ⭐⭐⭐ | `claude-code-from-source` 10 个模式 | 后端架构师（模式思维） |
| ⭐⭐⭐ | `Yuyz0112/claude-code-reverse` 可视化 | 想理解 AI 行为的人 |
| ⭐⭐ | `ComeOnOliver/claude-code-analysis` 文档 | 想全面了解系统的人 |
| ⭐⭐ | `FlorianBruniaux/claude-code-ultimate-guide` | 想写出更好 Prompt 的人 |
| ⭐ | `sanbuphy/learn-coding-agent` 隐藏功能分析 | 好奇宝宝 |
| ⭐ | `PetrGuan/claude-code-anatomy` 交互可视化 | 视觉学习者 |

---

> 📝 **免责声明**  
> 本文档汇总的内容来自社区对 `@anthropic-ai/claude-code` npm 包中 source map 的公开分析。  
> 所有仓库的维护者均声明其不包含 Anthropic 的原始专有源码，仅包含架构分析、伪代码和教学性注释。  
> 本文档仅用于技术学习目的。

---

*文档生成日期：2026-06-24 | 数据来源：GitHub 社区公开仓库*
