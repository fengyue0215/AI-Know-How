# Kiro 扩展机制深度详解 —— Steering / Hooks / Skills / Powers / MCP

> 基于 Kiro 官方文档（kiro.dev/docs）、社区最佳实践、AWS Book of Kiro、GitHub 开源配置。  
> 版本：Kiro IDE 1.0.x / CLI v3（2026.7）

---

## 一、Steering 文件

### 1.1 概念

Steering 文件（`.kiro/steering/*.md`）是 Markdown + YAML frontmatter 的**持久化项目规则文件**。每次 AI 交互自动加载。替代"每次对话都要重复解释项目约定"。

### 1.2 四种 Inclusion 模式

| 模式 | Frontmatter | 触发条件 | 用例 |
|------|------------|---------|------|
| `always`（默认） | `inclusion: always` | 每次交互自动加载 | 技术栈、命名约定、安全策略 |
| `fileMatch` | `inclusion: fileMatch` + `fileMatchPattern: "glob"` | 匹配文件激活时加载 | 领域规则（仅网络层需要 IO 超时规则） |
| `manual` | `inclusion: manual` | 用户在 Chat 中显式引用 `#filename` | 故障排查指南、迁移流程 |
| `auto` | `inclusion: auto`（需 `name` + `description`） | AI 自动判断 | 较新模式，需验证规则（agnix KIRO-001~003） |

### 1.3 文件结构示例

```markdown
---
inclusion: always
---

# 技术栈
- C++17 (GCC 11+, Clang 14+)
- CMake 3.20+：cmake -B build && cmake --build build -j$(nproc)
- 测试：cd build && ctest --output-on-failure
- 依赖：hiredis, occi, cppzmq, protobuf, pthread

# 禁止事项
- MUST NOT: 使用裸 new/delete（用 std::make_unique）
- MUST NOT: 使用 C-style cast
- MUST NOT: 抛出异常（-fno-exceptions）
```

```markdown
---
inclusion: fileMatch
fileMatchPattern: "src/network/**/*.{cpp,h}"
---

# 网络层专属规则
- 所有 socket IO 必须设置超时（30 秒）
- 使用 non-blocking IO + epoll ET 模式
- 连接断开后必须自动重连
- 缓冲区大小上限 64KB
```

```markdown
---
inclusion: manual
domain: troubleshooting
---

# Redis 故障排查指南
1. 检查连接数：redis-cli CLIENT LIST | wc -l
2. 检查慢查询：redis-cli SLOWLOG GET 10
3. 检查内存：redis-cli INFO memory
4. 检查命中率：redis-cli INFO stats | grep keyspace
```

### 1.4 优先级规则

```
Workspace (.kiro/steering/) > Global (~/.kiro/steering/)

Multi-root Workspace：
  后添加的 Workspace > 先添加的 Workspace

同一目录内多个 always 文件：
  无显式优先级 → Kiro 根据上下文判断
  → 最佳实践：每个文件独立职责，不互相矛盾
```

> 来源：https://kiro.dev/docs/steering/
> https://www.qes.co.jp/media/aws/Kiro/a799

### 1.5 社区推荐的文件组织

| 文件 | 用途 | 更新频率 |
|------|------|:---:|
| `tech.md` | 技术栈、依赖版本、环境变量、全局常量 | 低（依赖变更时） |
| `structure.md` | 目录树、核心数据模型、公共函数、API 路由 | 中（功能开发时） |
| `lessons.md` | Bug 修复经验、最佳实践、业务逻辑"坑点" | 高（每次修 bug） |

**命名约定**：`00-`、`10-`、`20-` 前缀保持视觉顺序。

> 来源：https://blog.csdn.net/bingqise5193/article/details/157326451

### 1.6 Steering 最佳实践

| 原则 | 说明 |
|------|------|
| **纳入版本控制** | PR 审查 steering 变更，可审计、可调试 |
| **按关注点分离** | 编码约定 / 项目上下文 / 业务规则 / 行为边界 — 各自独立文件 |
| **保持精简** | 是给 AI 的摘要，不是文档数据库。只存高信号信息 |
| **审查 AI 更新** | Hook 自动更新 steering 后必须人工审查 |
| **全局设默认，项目覆盖** | `~/.kiro/steering/` 放中文偏好、通用安全规则；项目级覆盖具体规则 |
| **避免同目录矛盾** | 多个 `always` 文件不要给矛盾指令 |

> 来源：https://aws.amazon.com/tw/events/taiwan/techblogs/kiro-best-practices/

---

## 二、Agent Hooks

### 2.1 概念

Hook 是 JSON 配置文件（`.kiro/hooks/*.kiro.hook`），在特定 IDE 事件发生时自动触发 AI Prompt 或 Shell 命令。

### 2.2 全部触发事件

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| `fileSaved` / `PostFileSave` | 匹配 glob 的文件被保存 | 自动 lint、格式化、更新测试 |
| `fileCreated` / `PostFileCreate` | 新文件创建 | 自动添加版权头、更新 CMakeLists |
| `fileDeleted` | 文件删除 | 自动清理引用 |
| `preTaskExecution` / `PreTaskExec` | Spec 任务开始前 | 前置条件检查 |
| `postTaskExecution` / `PostTaskExec` | Spec 任务完成后 | 自动运行测试、生成报告 |
| `promptSubmit` | 用户提交 Prompt 时 | Prompt 预处理 |
| `agentStop` | Agent 停止时 | 自动 commit、清理 |
| `preToolUse` | 工具执行前 | 安全拦截（阻断危险命令） |
| `postToolUse` | 工具执行后 | 格式化输出、日志记录 |
| `manual` | 手动触发 `/hook-name` | 安全扫描、架构审查 |

> 来源：https://kiro.dev/docs/hooks/
> https://kiro.dev/changelog/ide/0-9/

### 2.3 两种配置格式

**格式 A：单个 `.kiro.hook` 文件（`when` + `then`）**

```json
{
  "enabled": true,
  "name": "C++ Auto Format on Save",
  "version": "1.0.0",
  "description": "格式化保存的 C++ 文件",
  "when": {
    "type": "fileSaved",
    "patterns": ["src/**/*.cpp", "include/**/*.h"]
  },
  "then": {
    "type": "runCommand",
    "command": "clang-format -i ${FILE_PATH}"
  }
}
```

**格式 B：批量 JSON 数组（`trigger` + `matcher` + `action`）**

```json
{
  "version": "v1",
  "hooks": [
    {
      "name": "lint-on-save",
      "trigger": "PostFileSave",
      "matcher": "\\.cpp$",
      "action": { "type": "command", "command": "clang-tidy ${FILE_PATH} --fix" },
      "timeout": 30,
      "enabled": true
    }
  ]
}
```

### 2.4 Action 类型

| Action | 说明 | 示例 |
|--------|------|------|
| `runCommand` / `command` | 执行 Shell 命令 | `"clang-format -i ${FILE_PATH}"` |
| `askAgent` / `agent` | 发送 Prompt 给 AI | `"检查这段代码的内存安全性"` |

### 2.5 C++ 项目实战 Hook 集

**保存时自动 clang-format**：
```json
{
  "name": "cpp-format-on-save",
  "when": { "type": "fileSaved", "patterns": ["**/*.{cpp,h,hpp}"] },
  "then": { "type": "runCommand", "command": "clang-format -i \"$FILE\"" }
}
```

**新建 .cpp 自动更新 CMakeLists.txt**：
```json
{
  "name": "update-cmake-on-new-file",
  "when": { "type": "fileCreated", "patterns": ["src/**/*.cpp"] },
  "then": {
    "type": "askAgent",
    "prompt": "新文件 ${FILE_PATH} 已创建。找到对应的 CMakeLists.txt，将新文件添加到合适的 target 中。"
  }
}
```

**Spec 任务完成后自动构建+测试**：
```json
{
  "name": "build-and-test-after-task",
  "when": { "type": "postTaskExecution" },
  "then": {
    "type": "runCommand",
    "command": "cmake --build build -j$(nproc) && cd build && ctest --output-on-failure"
  }
}
```

**"自进化 AI 记忆"模式（社区推崇）**：
```json
{
  "name": "self-evolving-memory",
  "when": { "type": "manual" },
  "then": {
    "type": "askAgent",
    "prompt": "分析当前会话内容：1) 提取架构决策更新 tech.md；2) 提取结构变化更新 structure.md；3) 提取经验教训更新 lessons.md；4) git commit 并推送。"
  }
}
```

> 来源：https://blog.csdn.net/badao_liumang_qizhi/article/details/162073745
> https://blog.csdn.net/bingqise5193/article/details/157326451

### 2.6 Hook 最佳实践

| 原则 | 说明 |
|------|------|
| **从 manual 开始** | 破坏性操作（commit、push）先用 manual 验证，再自动化 |
| **精确匹配** | 避免 `**/*`，用 `src/**/*.cpp` 精确限定 |
| **审查 AI 产出** | Hook 自动更新的内容必须人工审查 |
| **已知限制** | 每次文件保存只触发一个 hook；不支持链式触发 |

---

## 三、Skills 与 Powers

### 3.1 Skills（Agent Skills）

| 维度 | 说明 |
|------|------|
| **位置** | `.kiro/skills/<name>/SKILL.md`（workspace）或 `~/.kiro/skills/`（global） |
| **标准** | 遵循开源 **Agent Skills 标准**，跨 AI 工具兼容 |
| **加载** | 渐进式披露：启动时只加载 name + description，按需加载全文 |
| **与 Steering 区别** | Steering 始终在线；Skills 按需激活 |

```markdown
<!-- .kiro/skills/cmake-build/SKILL.md -->
---
name: cmake-build
description: Build C++ project with CMake. Use when user mentions build, compile, cmake.
---

# CMake Build
1. Run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
2. Run: cmake --build build -j$(nproc)
3. If errors, read error output, locate source, fix, retry
4. Report: PASS or FAIL with summary
```

### 3.2 Powers

Power = Skill 的超集，额外打包 MCP 配置：

```
Power = POWER.md（知识 + 工作流）
      + mcp.json（可选，MCP 服务器配置）
      + steering/（可选，额外引导文件）
```

**两种类型**：

| 类型 | 有 MCP? | 示例 |
|------|:---:|------|
| Guided MCP Power | ✅ | Supabase, Stripe, Datadog |
| Knowledge Base Power | ❌ | Terraform CLI 指南, 测试策略 |

**粒度规则**：
- 默认：一个工具一个 Power
- 拆分条件：工作流完全独立 / 用户从不同时需要 / 不同环境
- 合在一起：terraform, git, docker, kubectl — 即使命令多

**安装**：`@powers install supabase`

> 来源：https://github.com/aToy0m0/kiro_powers
> https://kiro.dev/docs/web/sandbox/mcp/

### 3.3 Steering vs Skills vs Powers vs Subagents —— 选择指南

| 机制 | 用途 | 激活方式 | 上下文占用 |
|------|------|---------|:---:|
| **Steering** | 始终在线的项目规则 | 自动（always/fileMatch/manual） | 每次交互 |
| **Skills** | 按需加载的能力包 | AI 自动或显式调用 | 仅使用时 |
| **Powers** | MCP 服务器文档+工作流 | 用户激活 / 关键词匹配 | 仅使用时 |
| **Subagents** | 专用 Agent（独立上下文） | 自动选择或手动 | 隔离上下文 |

---

## 四、MCP 集成

### 4.1 配置方式

| 方式 | 位置 | 适用 |
|------|------|------|
| 内联 Agent 配置 | `.kiro/agents/*.md` YAML frontmatter | 子 Agent 专用 MCP |
| `mcp.json` | `.kiro/settings/mcp.json`（workspace）或 `~/.kiro/settings/mcp.json`（global） | 全局 MCP |
| Power 内 | Power 的 `mcp.json` | Power 专用 MCP |
| Kiro Web | Settings → Agent → MCP | Web 界面配置 |

### 4.2 传输支持

- **Stdio**：本地进程（Kiro IDE + Kiro Web 均支持）
- **HTTP/SSE**：远程服务（Kiro Web 当前仅支持 Stdio）
- 环境变量展开：支持 `${VAR}` 语法

### 4.3 MCP 配置示例

```json
{
  "mcpServers": {
    "redis-inspector": {
      "command": "python3",
      "args": ["~/mcp-servers/redis_mcp.py"]
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

> 来源：https://kiro.dev/docs/web/sandbox/mcp/
> https://kiro.dev/docs/custom-agents/

---

## 五、自定义子代理

### 5.1 配置

位置：`.kiro/agents/<name>.md`（workspace）或 `~/.kiro/agents/`（global）

```markdown
---
name: cpp-code-reviewer
description: Senior C++ code reviewer. Use after implementing changes.
model: claude-sonnet-4
tools: read, grep, glob
permissions:
  allow_paths: ["${PROJECT_ROOT}/src"]
  deny_tools: ["shell_execute"]
---

You are a senior C++ code reviewer. Review code for:
1. Memory safety (RAII, smart pointers, no raw new/delete)
2. Thread safety (data races, deadlocks)
3. Performance (unnecessary copies, hot-path allocations)
4. Error handling (return value checks, timeout enforcement)
5. Style compliance (Google C++ Style Guide)
```

### 5.2 子代理自动选择

Kiro 根据子代理的 `description` 字段自动路由任务。也可在 Agent 选择器中手动指定。

> 来源：https://kiro.dev/blog/custom-subagents-skills-and-enterprise-controls/
> https://kiro.dev/changelog/ide/0-9/

---

## 六、C++ 项目完整 Kiro 配置模板

```
.kiro/
├── steering/
│   ├── 00-tech.md              # always — 技术栈、禁止事项
│   ├── 10-structure.md         # always — 目录结构、命名约定
│   ├── 20-network-rules.md     # fileMatch: src/network/** — IO 超时规则
│   └── 30-troubleshooting.md   # manual — Redis/DB 故障排查
├── hooks/
│   ├── format-on-save.kiro.hook    # fileSaved → clang-format
│   ├── cmake-on-create.kiro.hook   # fileCreated → 更新 CMakeLists
│   └── test-after-task.kiro.hook   # postTaskExecution → cmake+ctest
├── skills/
│   └── cmake-build/
│       └── SKILL.md            # 构建技能
├── agents/
│   ├── code-reviewer.md        # 代码审查子代理
│   └── test-writer.md          # 测试生成子代理
└── settings/
    └── mcp.json                # MCP 服务器（redis/oracle 检查器）
```

---

## 七、关键来源索引

| 资源 | URL |
|------|-----|
| Kiro Steering 官方文档 | https://kiro.dev/docs/steering/ |
| Kiro Hooks 官方文档 | https://kiro.dev/docs/hooks/ |
| Kiro Hooks Types | https://kiro.dev/docs/hooks/types/ |
| Kiro Custom Agents | https://kiro.dev/docs/custom-agents/ |
| Kiro Powers & MCP (Web) | https://kiro.dev/docs/web/sandbox/mcp/ |
| Kiro 0.9 Changelog (Subagents/Skills) | https://kiro.dev/changelog/ide/0-9/ |
| Kiro 1.0 Changelog (Hooks/Agents) | https://kiro.dev/changelog/ide/1-0/ |
| AWS Book of Kiro 社区实践 | https://aws.amazon.com/tw/events/taiwan/techblogs/kiro-best-practices/ |
| Kiro Steering 优先级指南（QES） | https://www.qes.co.jp/media/aws/Kiro/a799 |
| Steering 配置最佳实践（QES） | https://www.qes.co.jp/media/ai/a845 |
| CSDN Kiro Hooks 完全指南 | https://blog.csdn.net/badao_liumang_qizhi/article/details/162073745 |
| CSDN 自进化 AI 记忆 | https://blog.csdn.net/bingqise5193/article/details/157326451 |
| Kiro Powers GitHub | https://github.com/aToy0m0/kiro_powers |
| Kiro Subagents Sample (AWS) | https://github.com/aws-samples/sample-aidlc-toolset-with-subagents-and-custom-skills |
| Kiro Config Control (GUI) | https://github.com/kirodotdev-labs/config-control-kiro |
| Dev.to Kiro 实战指南 | https://dev.to/galian/kiro-a-practical-guide-to-awss-spec-driven-agentic-ide-26o9 |
| JR Academy Kiro 核心功能 | https://jiangren.com.au/en/wiki/kiro-guide/03-core-features |

---

*文档生成日期：2026-07-16 | 来源：Kiro 官方文档 + 社区最佳实践 + GitHub 开源配置*
