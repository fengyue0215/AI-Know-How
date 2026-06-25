# Claude Code 功能深度详解

> Claude Code 是 Anthropic 的终端原生 AI 编程 Agent。本文档覆盖所有核心功能、命令体系、扩展机制和高级工作流。  
> 版本：v2.1.x（2026 年 6 月），模型：Fable 5 / Opus 4.8 / Sonnet 4.6 / Haiku 4.5

---

## 📋 目录

| 章节 | 内容 |
|------|------|
| 一 | 核心架构与设计哲学 |
| 二 | 命令体系：50+ 斜杠命令完整手册 |
| 三 | CLAUDE.md 记忆系统 |
| 四 | Hook 系统：11 种生命周期事件 |
| 五 | Skills 技能系统 |
| 六 | MCP 协议集成 |
| 七 | 子代理与 Agent Teams |
| 八 | Dynamic Workflows 动态工作流 |
| 九 | 权限与安全模型 |
| 十 | 配置体系与 settings.json |
| 十一 | 会话管理与上下文优化 |
| 十二 | C++ 后端实战配置模板 |

---

## 一、核心架构与设计哲学

### 1.1 极简主循环

```
while (true) {
    messages = buildPrompt(userInput, context, tools, memory)
    response = streamToModel(messages)
    tools = extractToolCalls(response)
    if (no tools) break
    results = executeTools(tools)   // 只读并行、写入串行
    messages.push(results)
    maybeCompress(messages)          // 4 层压缩
}
```

没有意图分类器、没有规划器、没有 RAG 管道。模型自主决策一切。

### 1.2 四层上下文压缩

| 层级 | 机制 | 触发条件 |
|------|------|---------|
| Snip | 裁剪冗余内容 | 自动 |
| Microcompact | 合并相似操作 | 自动 |
| Collapse | LLM 摘要早期对话 | 自动 |
| Autocompact | 触发 `/compact` | 上下文使用率 > 80% |

### 1.3 10 个关键架构模式

详见 `Claude-Code-源码分析汇总.md`。对使用者最重要的 4 个：

| 模式 | 对你的价值 |
|------|-----------|
| Fork Agent 共享 Cache | 并行子 Agent 的输入 token 成本 ≈ 1 个 Agent（~95% 节省） |
| Sticky Latches | CLAUDE.md 稳定导致 100% 缓存命中 |
| 两阶段技能加载 | 启动时只加载 frontmatter，调用时才加载全文 |
| 推测性工具执行 | 模型流式输出期间提前启动只读工具 |

---

## 二、命令体系：50+ 斜杠命令完整手册

### 2.1 项目初始化与记忆

| 命令 | 功能 |
|------|------|
| `/init` | 扫描仓库，生成 `CLAUDE.md`（构建命令、测试命令、代码风格） |
| `/memory` | 编辑记忆文件，启用/禁用自动记忆 |
| `/add-dir <path>` | 为会话添加额外工作目录（monorepo 场景） |
| `/cd <path>` | 切换会话工作目录（v2.1.169+） |

### 2.2 上下文与会话管理

| 命令 | 功能 |
|------|------|
| `/compact [instructions]` | 压缩对话上下文（减少 60-80% token），可选专注指令 |
| `/clear [name]` | 开启新对话，旧对话保存到 `/resume` |
| `/context [all]` | 可视化上下文使用情况（彩色网格展示各组件 token 占比） |
| `/btw <question>` | 快速提问，不污染对话历史 |
| `/resume [session]` | 恢复之前的会话 |
| `/rename [name]` | 重命名当前会话 |
| `/export [filename]` | 导出对话为纯文本/Markdown |
| `/branch [name]` | 在当前位置分叉对话（尝试不同方向） |
| `/fork <directive>` | 派生子 Agent 继承当前对话返回结果 |
| `/goal [condition\|clear]` | 设定目标，Claude 自主跨轮次工作直到条件满足 |

### 2.3 代码质量与审查

| 命令 | 功能 |
|------|------|
| `/review [PR]` | 审查本地 PR |
| `/code-review [effort] [--fix] [--comment]` | Diff 审查，等级 low/medium/high/xhigh/max/ultra |
| `/security-review` | 安全漏洞分析 |
| `/simplify` | 纯重构审查（4 个并行 Agent，不找 bug） |
| `/diff` | 交互式 diff 查看器 |
| `/verify` | 验证代码变更是否按预期工作 |

### 2.4 规划与执行

| 命令 | 功能 |
|------|------|
| `/plan [description]` | 进入规划模式（先出方案再改代码） |
| `/batch <instruction>` | 大规模并行编排（5-30 个独立单元，各自 worktree + PR） |
| `/background [prompt]` | 将会话转为后台 Agent 运行 |
| `/agents` | 管理子 Agent 配置 |
| `/tasks` | 查看后台运行的任务 |

### 2.5 模型与性能

| 命令 | 功能 |
|------|------|
| `/model [model]` | 切换模型（Sonnet/Opus/Haiku/Fable） |
| `/effort [level\|auto]` | 推理深度：low/medium/high/xhigh/max/ultracode |
| `/fast [on\|off]` | 快速模式开关 |
| `/cost` | 查看当前会话 token 消耗和成本估算 |
| `/config [key=value...]` | 打开设置或直接设置配置项 |

### 2.6 诊断与调试

| 命令 | 功能 |
|------|------|
| `/doctor` | 诊断安装、认证、网络、Node.js 版本问题 |
| `/debug [description]` | 启用调试日志 |
| `/rewind` | 回滚对话/代码到检查点 |
| `/heapdump` | 写入 JS 堆快照（诊断内存问题） |

### 2.7 集成与扩展

| 命令 | 功能 |
|------|------|
| `/mcp [reconnect\|enable\|disable]` | 管理 MCP 服务器连接 |
| `/skills` | 列出可用技能（按 `t` 按 token 数排序） |
| `/reload-skills` | 不重启重新扫描技能目录 |
| `/hooks` | 查看/配置 Hook（不需要手动编辑 JSON） |
| `/plugin [subcommand]` | 管理插件 |
| `/ide` | 管理 IDE 集成 |
| `/remote-control` | 从 claude.ai 远程控制当前会话 |
| `/login` / `/logout` | 账户管理 |
| `/permissions` | 管理 allow/ask/deny 规则 |

### 2.8 高级功能

| 命令 | 功能 |
|------|------|
| `/deep-research <question>` | 深度研究（Web 搜索→交叉验证→合成引用报告） |
| `/schedule [description]` | 创建云上定时任务 |
| `/loop [interval] [prompt]` | 循环执行 Prompt |
| `/run` | 启动和驱动项目应用 |
| `/autofix-pr [prompt]` | 监听 PR，CI 失败时自动推送修复 |
| `/insights` | 分析会话模式、摩擦点、优化建议 |
| `/stats` | 个人用量仪表板 |
| `/powerup` | 交互式学习 Claude Code 功能 |
| `/feedback` | 提交反馈或 bug 报告 |

### 2.9 命令分类速查

| 分类 | 命令数 | 核心命令 |
|------|:---:|------|
| 项目初始化 | 4 | `/init`, `/memory` |
| 上下文管理 | 9 | `/compact`, `/clear`, `/context`, `/btw` |
| 代码质量 | 6 | `/review`, `/code-review`, `/security-review` |
| 规划执行 | 5 | `/plan`, `/batch`, `/fork`, `/goal` |
| 模型性能 | 5 | `/model`, `/effort`, `/cost` |
| 诊断调试 | 4 | `/doctor`, `/rewind` |
| 集成扩展 | 8 | `/mcp`, `/skills`, `/hooks` |
| 高级功能 | 8 | `/deep-research`, `/schedule`, `/loop` |

---

## 三、CLAUDE.md 记忆系统

### 3.1 三层文件架构

| 文件 | 位置 | 作用 | 共享 |
|------|------|------|:---:|
| 全局 CLAUDE.md | `~/.claude/CLAUDE.md` | 个人偏好（所有项目） | ❌ |
| 项目 CLAUDE.md | `./CLAUDE.md` | 团队约定、构建命令 | ✅ Git |
| 本地 CLAUDE.md | `./CLAUDE.local.md` | 个人项目偏好 | ❌ gitignored |

### 3.2 文件系统记忆（Memory System）

```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md              # 索引（每行一条记忆）
├── coding-style.md        # 单条记忆
└── architecture.md        # 单条记忆
```

每条记忆是 Markdown + YAML frontmatter：

```markdown
---
name: coding-style
description: 项目 C++ 编码规范
metadata:
  type: project
---

使用 Google C++ Style Guide，snake_case 函数，PascalCase 类。
禁止异常，错误用 std::optional。详见 [[architecture]]
```

**关键设计**：不在每次 API 调用中发送全部记忆。用一个**小型 Sonnet 侧查询**选择当前最相关的记忆。

### 3.3 /init 最佳实践

```bash
# 在 Claude Code 会话中
/init

# 生成后手动优化：
# - 控制在 50 行以内（作为索引，不是文档转储）
# - 移除动态内容（时间戳、日期）
# - 详细文档放在 docs/ 中，AI 按需读取
# - 不要在会话中途编辑 CLAUDE.md（破坏 Cache）
```

---

## 四、Hook 系统：11 种生命周期事件

### 4.1 Hook 的核心价值

Hook 是**确定性的系统级强制执行**——不消耗上下文 token，不受 AI"忘记"规则的影响。

### 4.2 全部生命周期事件

| 事件 | 触发时机 | 可阻断 |
|------|---------|:---:|
| **PreToolUse** | 工具执行前 | ✅ |
| **PostToolUse** | 工具执行成功后 | ❌ |
| **PostToolUseFailure** | 工具执行失败后 | ❌ |
| **UserPromptSubmit** | 用户提交 Prompt 时 | ✅ |
| **SessionStart** | 会话启动/恢复时 | ❌ |
| **SessionEnd** | 会话结束时 | ❌ |
| **Stop** | 主 Agent 完成响应 | ❌ |
| **SubagentStop** | 子 Agent 完成 | ❌ |
| **PreCompact** | 上下文压缩前 | ❌ |
| **PostCompact** | 上下文压缩后 | ❌ |
| **ConfigChange** | 配置文件变更时 | ❌ |
| **Notification** | 发送通知时 | ❌ |
| **PermissionRequest** | 权限弹窗时 | ❌ |

### 4.3 配置结构

Hooks 配置在 `settings.json` 中，三层作用域：

| 文件 | 作用域 | Git 跟踪 |
|------|------|:---:|
| `~/.claude/settings.json` | 全局（所有项目） | ❌ |
| `.claude/settings.json` | 项目级 | ✅ |
| `.claude/settings.local.json` | 项目 + 个人覆盖 | ❌ |

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": ".claude/hooks/validate-bash.sh"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": "clang-format -i \"$CLAUDE_TOOL_INPUT_FILE_PATH\"",
          "timeout": 30,
          "statusMessage": "Auto-formatting C++..."
        }]
      }
    ]
  }
}
```

### 4.4 Matcher 模式

| 模式 | 匹配范围 |
|------|---------|
| `"Write"` | 仅 Write 工具 |
| `"Edit\|Write"` | Edit 和 Write |
| `"Bash"` | 仅 Bash 命令 |
| `"mcp__*"` | 所有 MCP 工具 |
| `"*"` 或 `""` | 所有工具 |

### 4.5 退出码

| 退出码 | 含义 |
|:---:|------|
| `0` | 允许操作 |
| `2` | **阻断操作**（stderr 反馈给 AI） |
| `1` | 脚本错误（操作被阻断） |

### 4.6 Hook 接收的环境变量

| 变量 | 内容 |
|------|------|
| `CLAUDE_TOOL_INPUT_FILE_PATH` | 被编辑文件的绝对路径 |
| `CLAUDE_PROJECT_DIR` | 项目根目录 |
| `CLAUDE_SESSION_ID` | 当前会话 ID |

**stdin 传入 JSON**：包含 `tool_name`、`tool_input`、`tool_output` 完整信息。

### 4.7 C++ 项目 Hook 实战模板

**自动格式化**：
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "clang-format -i \"$CLAUDE_TOOL_INPUT_FILE_PATH\""
      }]
    }]
  }
}
```

**阻断危险命令**：
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash .claude/hooks/block-dangerous.sh"
      }]
    }]
  }
}
```

`block-dangerous.sh` 检查 `rm -rf /`、`DROP TABLE`、`git push --force` 等模式，匹配时 `exit 2`。

**任务完成后自动构建+测试**：
```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "cmake --build build -j$(nproc) && cd build && ctest --output-on-failure"
      }]
    }]
  }
}
```

---

## 五、Skills 技能系统

### 5.1 两种技能类型

| 类型 | 位置 | 触发方式 |
|------|------|---------|
| **Slash Skill** | `.claude/skills/<name>/SKILL.md` | 手动 `/name` 或自动匹配描述 |
| **Agent Skill** | 同上，含 YAML frontmatter | AI 自动判断激活时机 |

### 5.2 SKILL.md 结构

```markdown
---
name: pre-deploy-check
description: Security checklist before deployment. Use when user mentions deploy, release, launch.
allowed-tools:
  - Read
  - Grep
  - Bash
  - Glob
context: fork
model: sonnet
---

# Pre-Deploy Security Checklist

1. Read all source files in `src/`
2. Run `cmake --build build`
3. Run `cd build && ctest --output-on-failure`
4. Check for: raw new/delete, missing timeout, unhandled errors
5. Output structured report with PASS/FAIL/WARN
```

### 5.3 关键 Frontmatter 字段

| 字段 | 作用 |
|------|------|
| `name` | 斜杠命令名（`/name`） |
| `description` | **最关键字段**——AI 用它判断何时自动激活。写"pushy"一点 |
| `allowed-tools` | 限制工具范围（安全） |
| `disable-model-invocation: true` | 仅手动触发（部署、提交等危险操作） |
| `context: fork` | 在隔离子 Agent 中运行（不污染主对话） |
| `model` | 指定模型（如 `claude-opus-4-8`） |

### 5.4 参数传递

```markdown
# 使用 $ARGUMENTS
Fix GitHub issue $ARGUMENTS

# 位置参数：$0, $1, $2
Fix issue $0 in repository $1

# 动态 shell 上下文（! 反引号，预处理时执行）
Current git branch: `! git branch --show-current`
Last commit message: `! git log -1 --format=%s`
```

### 5.5 Skills 作用域

| 位置 | 作用域 | 优先级 |
|------|------|:---:|
| 企业策略 | 组织级 | 最高 |
| `~/.claude/skills/` | 个人 | 中 |
| `.claude/skills/` | 项目 | 最低 |

---

## 六、MCP 协议集成

### 6.1 传输类型

| 类型 | 用途 | 状态 |
|------|------|:---:|
| **HTTP** | ⭐ 远程服务推荐方案 | 推荐 |
| **Stdio** | 本地进程 | 支持 |
| **SSE** | — | ❌ 已弃用 |

### 6.2 安装作用域

| 作用域 | 配置文件 | 共享方式 |
|------|------|------|
| `local`（默认） | `~/.claude.json` | 私有，仅当前项目 |
| `project` | `.mcp.json` | ✅ 通过 Git 团队共享 |
| `user` | `~/.claude.json` | 跨所有项目 |

### 6.3 上下文警告

```
5 个 MCP Server + 58 个工具 ≈ 55,000+ token 开销

建议：
- 同时激活 ≤ 10 个 MCP Server
- 活跃工具总数 ≤ 80 个
- 使用 Tool Search 功能自动按需加载（减少 ~85% 开销）
```

### 6.4 C++ 后端常用 MCP Server

```bash
# GitHub 集成
claude mcp add github -- npx -y @modelcontextprotocol/server-github
# 环境变量: GITHUB_PERSONAL_ACCESS_TOKEN

# PostgreSQL（或 Oracle 兼容查询）
claude mcp add postgres -- npx -y @modelcontextprotocol/server-postgres
# 环境变量: POSTGRES_URL

# 文件系统访问
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /path/to/project

# 自建 MCP（参考 MCP-Server开发入门.md）
claude mcp add redis-inspector -- python3 ~/mcp-servers/redis_mcp.py
claude mcp add oracle-inspector -- python3 ~/mcp-servers/oracle_mcp.py
```

---

## 七、子代理与 Agent Teams

### 7.1 子代理（Subagents）

子代理是独立的 Claude 实例，拥有自己的上下文窗口、系统 Prompt、工具列表和权限。

| 特性 | 说明 |
|------|------|
| **上下文隔离** | 独立上下文窗口 |
| **通信方式** | 仅向父 Agent 返回摘要 |
| **Token 成本** | 较低（只有摘要回到父会话） |
| **状态** | GA（正式发布） |

### 7.2 创建自定义子代理

```markdown
<!-- .claude/agents/code-reviewer.md -->
---
name: code-reviewer
description: Use proactively after implementing changes. Review for safety, correctness, and style.
tools: Read, Grep, Glob
model: sonnet
---

You are a senior C++ code reviewer. Review code for:
1. Memory safety (leaks, dangling pointers, RAII)
2. Thread safety (data races, deadlocks)
3. Performance (unnecessary copies, hot path allocations)
4. Style compliance (Google C++ Style Guide)
5. Error handling (return value checks, timeout enforcement)
```

### 7.3 调用方式

| 方式 | 说明 |
|------|------|
| **自动委派** | Claude 根据 `description` 自动路由 |
| **自然语言** | "用 code-reviewer 子代理审查最近的改动" |
| **@-mention** | `@agent-code-reviewer` 强制调用 |
| **会话级别** | `claude --agent code-reviewer` 让整个会话使用该子代理 |
| **前台/后台** | 后台并发运行；`Ctrl+B` 后台化 |

### 7.4 Agent Teams（实验性）

```
Team Lead（分配任务、接收汇总）
  ├── Teammate A（独立上下文、独立工具）
  ├── Teammate B
  └── Teammate C
      通信方式：点对点直接消息（非层级汇报）
      工具：TaskCreate, SendMessage, Worktree 隔离
```

**启用**：
```json
{
  "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }
}
```

**最佳实践**：3-5 个 teammate 推荐，每人 5-6 个任务，明确文件所有权。

### 7.5 子代理 vs Agent Teams vs Dynamic Workflows 对比

| 维度 | 子代理 | Agent Teams | Dynamic Workflows |
|------|:---:|:---:|:---:|
| **上下文** | 独立 | 独立 | 独立 |
| **通信** | 仅返回父会话 | 点对点直连 | 编排脚本控制 |
| **规模** | 1-7 并行 | 3-8 个 teammate | 数百个并行 |
| **成本** | 低 | 高 | 极高 |
| **状态** | GA | 实验性 | GA（ultracode） |

---

## 八、Dynamic Workflows 动态工作流

### 8.1 ultracode 模式

`/effort ultracode` 结合 `xhigh` 推理深度 + 自动 Workflow 编排，支持数百个并行子代理。

### 8.2 常见编排模式

| 模式 | 说明 | C++ 场景示例 |
|------|------|------------|
| **Fan-out-Synthesize** | 拆分→并行处理→汇总 | 并行分析 20 个模块的代码质量 |
| **Adversarial Verify** | 生成→独立验证 | 3 个 Agent 独立验证 AI 生成的代码安全性 |
| **Tournament** | 生成多个方案→投票选最优 | 3 种 epoll vs io_uring 架构方案对比 |
| **Loop-Until-Done** | 持续工作直到条件满足 | 扫描代码库直到找齐所有内存泄漏 |
| **Classify-and-Act** | 分类→路由到专用 Agent | 按模块类型分派到不同审查 Agent |

### 8.3 成本警告

> 200 个并行 worker × `xhigh` effort = 数百万 token。  
> 控制策略：收窄 worker 范围、降低非关键 worker 的 effort、设置 `max_tokens` 上限、利用共享 Cache。

### 8.4 `/batch` 命令

```bash
# 大规模迁移
/batch 将 src/ 下所有裸指针改为 std::unique_ptr

# 每个模块独立 worktree + 独立 PR
# Agent 自行测试、自行生成 commit
```

---

## 九、权限与安全模型

### 9.1 三级权限

| 级别 | 行为 |
|------|------|
| `allow` | 自动允许 |
| `ask` | 每次弹窗确认 |
| `deny` | 自动拒绝 |

### 9.2 权限配置

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",           // 所有 git 命令自动允许
      "Bash(cmake:*)",         // 所有 cmake 命令自动允许
      "Bash(ctest:*)",         // 所有 ctest 命令自动允许
      "Read(*)",               // 读取所有文件自动允许
      "Grep(*)", "Glob(*)"     // 搜索自动允许
    ],
    "ask": [
      "Bash(docker:*)",        // Docker 命令弹窗确认
      "WebSearch(*)"           // 网络搜索弹窗确认
    ],
    "deny": [
      "Bash(rm:*)",            // 禁止删除命令
      "Bash(sudo:*)",          // 禁止 sudo
      "Bash(ssh:*)",           // 禁止 SSH
      "Bash(scp:*)"            // 禁止文件传输
    ]
  }
}
```

### 9.3 目录边界

```json
{
  "allowedDirectories": ["/home/user/projects/game-server"],
  "deniedDirectories": ["/home/user/projects/game-server/third_party"]
}
```

---

## 十、配置体系与 settings.json

### 10.1 配置文件层级（后覆盖前）

```
1. 托管设置（组织策略，最高优先级）
2. ~/.claude/settings.json        （用户全局）
3. .claude/settings.json          （项目，可 Git 共享）
4. .claude/settings.local.json    （项目个人覆盖，gitignored）
```

### 10.2 完整配置示例（C++ 项目）

```json
{
  "effortLevel": "medium",
  "model": "claude-sonnet-4-6",
  "allowedDirectories": ["/home/dev/projects/game-server"],
  "deniedDirectories": ["/home/dev/projects/game-server/third_party"],
  "permissions": {
    "allow": [
      "Read(*)", "Grep(*)", "Glob(*)",
      "Bash(git:*)", "Bash(cmake:*)", "Bash(ctest:*)",
      "Bash(ninja:*)", "Bash(make:*)"
    ],
    "ask": ["WebSearch(*)", "WebFetch(*)"],
    "deny": ["Bash(rm:-rf*)", "Bash(sudo:*)", "Bash(ssh:*)"]
  },
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "clang-format -i \"$CLAUDE_TOOL_INPUT_FILE_PATH\""
      }]
    }]
  },
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.anthropic.com/"
  }
}
```

---

## 十一、会话管理与上下文优化

### 11.1 会话生命周期

```
创建（/init）→ 工作（model calls）→ 上下文增长 →
/compact（压缩）→ /clear（重置）→ /resume（恢复）
```

### 11.2 最佳实践

| 实践 | 原因 |
|------|------|
| 一个会话一个任务 | 避免上下文污染 |
| 70-80% 上下文时 compact | 在压缩丢失细节前主动控制 |
| 休息 >1 小时开新会话 | Cache TTL 为 1 小时 |
| 不编辑 CLAUDE.md 中途 | 破坏 Prompt Cache |
| 不中途切换模型 | 不同模型独立缓存 |
| 用 `/btw` 快速提问 | 不污染主对话历史 |
| 用 `/context` 监控 | 可视化 token 消耗分布 |

### 11.3 成本监控

```bash
# 当前会话
/cost

# 用量仪表板
/stats

# 分析优化建议
/insights

# 导出
/export session.md
```

---

## 十二、C++ 后端实战配置模板

### 12.1 完整项目配置包

将此复制到 C++ 项目的 `.claude/` 目录：

```
.claude/
├── settings.json              # 权限 + Hooks + 模型配置
├── settings.local.json        # 个人覆盖（gitignored）
├── agents/
│   ├── code-reviewer.md       # 代码审查子代理
│   ├── test-writer.md         # 测试生成子代理
│   └── security-auditor.md    # 安全审计子代理
├── skills/
│   ├── cmake-build/SKILL.md   # CMake 构建技能
│   └── deploy-check/SKILL.md  # 部署前检查技能
└── hooks/
    ├── auto-format.sh         # clang-format 脚本
    └── block-dangerous.sh     # 危险命令拦截脚本
```

### 12.2 settings.json 模板

参见第十章完整配置。

### 12.3 code-reviewer 子代理模板

参见第七章示例。

### 12.4 cmake-build Skill

```markdown
---
name: cmake-build
description: Build the C++ project with CMake. Use when user mentions build, compile, cmake.
allowed-tools:
  - Bash
---

# CMake Build

Current build command: `cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON && cmake --build build -j$(nproc)`

1. Execute the build command
2. If compilation errors, read the error output and locate the source
3. Fix the errors and rebuild
4. Report: PASS (0 errors) or FAIL with error summary
```

### 12.5 推荐安装步骤

```bash
# 1. 在项目根目录启动
cd ~/projects/game-server
claude

# 2. 生成 CLAUDE.md
/init

# 3. 手动优化 CLAUDE.md（控制 50 行）

# 4. 配置 MCP（按需）
claude mcp add redis -- python3 ~/mcp-servers/redis_mcp.py

# 5. 安装 Hook
/hooks  # 交互式配置

# 6. 创建自定义技能
# 编辑 .claude/skills/<name>/SKILL.md

# 7. 开始工作
```

---

## 📚 推荐资源

| 资源 | 链接 |
|------|------|
| 官方文档 | https://code.claude.com/docs |
| 支持中心 | https://support.claude.com |
| All-in-One 指南（社区） | https://github.com/wesammustafa/Claude-Code-Everything-You-Need-to-Know |
| Mastery 指南（社区） | https://github.com/TheDecipherist/claude-code-mastery |
| 10 步教程（社区） | https://github.com/luongnv89/claude-howto |
| 技能商店（社区） | https://github.com/alexknowshtml/claude-skills |
| Agent Orchestration 对比 | https://www.morphllm.com/ai-agent-orchestration |

---

*文档生成日期：2026-06-24 | 数据来源：Anthropic 官方文档、社区最佳实践、source map 逆向分析*
