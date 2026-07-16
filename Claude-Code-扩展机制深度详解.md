# Claude Code 扩展机制深度详解 —— Skills / Plugins / MCP / Hooks

> 基于 Claude Code 官方文档、社区最佳实践、GitHub 开源生态。  
> 版本：v2.1.169+（2026.7）

---

## 一、Skills 技能系统

### 1.1 概念

Skill 是可移植的指令包，打包为 `SKILL.md` 文件。遵循开源 **Agent Skills 标准**，跨 Claude Code、Cowork 等工具兼容。旧的 `.claude/commands/*.md` 斜杠命令已并入 Skills 系统。

### 1.2 SKILL.md 完整结构

```markdown
---
name: deploy-check                                    # 技能名（/name 调用）
description: Pre-deployment checklist. Use when user   # 最关键字段——AI 用它判断自动激活
  mentions deploy, release, launch, ship.
argument-hint: "[service-name] [environment]"          # /help 中显示的参数提示
allowed-tools: Read, Grep, Bash(git:*,cmake:*)         # 工具白名单
disable-model-invocation: true                          # true=仅手动触发（危险操作）
context: fork                                           # 在隔离子Agent 中运行
model: sonnet                                           # 指定模型
agent: Explore                                          # 子Agent 类型
---
```

### 1.3 全部 Frontmatter 字段

| 字段 | 必需 | 说明 |
|------|:---:|------|
| `name` | 否 | 技能名（默认取目录名）。调用: `/name` |
| `description` | **强烈推荐** | 控制自动触发——写"pushy"一点，列出触发短语 |
| `allowed-tools` | 否 | 工具白名单。格式: `Read, Write, Bash(git:*)` |
| `argument-hint` | 否 | 参数提示，如 `[issue-number]` |
| `disable-model-invocation` | 否 | `true` = 仅手动触发。用于 deploy/commit/push 等危险操作 |
| `context: fork` | 否 | 在隔离子Agent中运行，不污染主对话 |
| `agent` | 否 | 子Agent类型: `Explore`, `Plan`, `general-purpose`, 或自定义 |
| `model` | 否 | 指定模型: `sonnet`, `opus`, `haiku` |

### 1.4 三种作用域

| 位置 | 路径 | 范围 | 优先级 |
|------|------|------|:---:|
| 项目 | `.claude/skills/<name>/SKILL.md` | 团队共享，Git 版本控制 | 最低 |
| 个人 | `~/.claude/skills/<name>/SKILL.md` | 所有项目 | 中 |
| 企业 | 托管策略 | 全组织 | 最高 |

### 1.5 高级特性

**参数传递**：
```markdown
# $ARGUMENTS — 捕获技能名后的所有参数
Fix GitHub issue $ARGUMENTS

# 位置参数: $0, $1, $2
Fix issue $0 in repository $1

# 动态 Shell 上下文（`!` 反引号，预处理时执行）
Current branch: `! git branch --show-current`
Last commit: `! git log -1 --format=%s`
```

**Router 模式（复杂技能结构）**：
```
deploy/
├── SKILL.md              # Router + 核心原则
├── workflows/            # 分步流程
├── references/           # 领域知识
├── templates/            # 输出模板
└── scripts/              # 可执行代码
```

### 1.6 技能开发最佳实践

| 原则 | 说明 |
|------|------|
| **Description 是成败关键** | Claude 倾向*低触发*。描述要显式列出触发短语和上下文 |
| **控制在 500 行内** | 详细 API 文档放 `references/`，从 SKILL.md 引用 |
| **提供输出示例** | 在 `examples/` 中放优秀输出，Claude 会校准质量 |
| **危险操作设 `disable-model-invocation: true`** | Deploy/commit/push——确保手动控制时机 |
| **大任务用 `context: fork`** | 隔离子Agent 运行，结果摘要回主会话 |
| **纳入 Git** | `.claude/skills/` 提交到仓库，团队自动同步 |
| **渐进式加载** | 仅 name+description (~30-50 token) 初始加载，全文按需激活 |

> 来源：https://sherlock.xyz/post/how-to-write-skills-for-claude-code-and-cowork
> https://www.morphllm.com/claude-code-skills-mcp-plugins
> https://skywork.ai/blog/ai-bot/add-skills-to-claude-code-ultimate-guide/

---

## 二、Plugin 插件系统

### 2.1 概念

Plugin 是 Skills + Commands + Agents + Hooks + MCP 的**打包分发单元**。社区称之为"AI 编程工具的 npm moment"——标准化了 AI Agent 能力的打包、分发和治理。

### 2.2 插件目录结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       # 必需：元数据清单
├── .mcp.json             # 可选：MCP 配置
├── commands/*.md         # 可选：斜杠命令
├── skills/*/SKILL.md     # 可选：Agent Skills
├── agents/*.md           # 可选：子Agent 定义
├── hooks/*.py            # 可选：自动化脚本
└── README.md
```

**最小 `plugin.json`**：
```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "C++ backend development toolkit",
  "author": "team-name"
}
```

### 2.3 市场生态

| 市场 | 地址 | 类型 |
|------|------|------|
| **官方** | `code.claude.com/plugins`（31K+ ⭐） | 策展、质量受控 |
| **社区** | `kossakovsky/cc-plugins` | 社区维护 |
| **社区** | `svgsponer/claude-bazaar` | 社区维护 |
| **企业自建** | `oglimmer/plugin-skill-hosting` | Token 门控、Web UI、Postgres 存储 |

### 2.4 插件命令

```bash
# 添加市场
/plugin marketplace add kossakovsky/cc-plugins

# 安装插件
/plugin install <name>@<marketplace>

# 列出已安装
/plugin list

# 手动更新
/plugin marketplace update <name>

# 从本地目录加载
claude --plugin-dir /path/to/plugin
```

### 2.5 命名空间

Skills 在插件内自动加插件名前缀：`/my-plugin:hello`。避免冲突。

### 2.6 安全考量

| 原则 | 说明 |
|------|------|
| **无硬编码密钥** | 用环境变量：`process.env.SECRET` |
| **审查 plugin.json** | 检查声明的 MCP/Hooks/Binaries |
| **从低风险开始** | LSP → Commit → Review → 再扩展到外部集成 |
| **企业用白名单** | `pluginSuggestionMarketplaces` 控制可见市场 |

> 来源：https://github.com/kossakovsky/cc-plugins
> https://developer.aliyun.com/article/1747753
> https://github.com/KimYx0207/AI-Coding-Guide-Zh/blob/main/docs/claude-code/08-Plugins生态完整指南.md

---

## 三、MCP 集成

### 3.1 配置方式

```bash
# CLI 添加
claude mcp add <name> -- <command>          # -s user（全局）或 -s project（Git 共享）

# 手动编辑
# ~/.claude/mcp.json（用户全局）
# .mcp.json（项目级，Git 共享）
```

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

### 3.2 MCP Gateway 模式（2026 生产级推荐）

单一联邦端点替代多 MCP Server 直连：

```json
{
  "mcpServers": {
    "gateway": {
      "transport": "streamable-http",
      "url": "https://gateway.example.com/v1/mcp",
      "headers": {
        "Authorization": "Bearer ${FI_API_KEY}",
        "X-FI-User": "${USER}@example.com"
      }
    }
  }
}
```

**Gateway 价值**：

| 问题 | Gateway 方案 |
|------|------------|
| 工具调用不可观测 | OpenTelemetry span（工具名、耗时、用户归属） |
| 凭证分散 | 集中密钥管理，客户端不接触下游 token |
| Prompt 注入 | 内联扫描器（文本 ~65ms, 图像 ~107ms） |
| 上下文膨胀 | 选择性注册 + 描述压缩 |

> 来源：https://futureagi.com/blog/using-mcp-gateway-with-claude-code-practical-guide-2026/

### 3.3 MCP Token 成本优化

2026 年对 22 个团队的调查：MCP 相关输入 token 占总消耗的 **41-58%**。5 个杠杆：

| 杠杆 | 节省 | 方法 |
|------|:---:|------|
| 按会话选择性注册 | 8-15% | 首条用户消息分类器返回相关工具 |
| 工具结果语义缓存 | 4-7% | 工具名+内容哈希缓存（35-55% 命中率） |
| 编译工具执行 | 25-45% | 编译为 Python 模块，暴露为单工具 |
| 工具描述压缩 | ~40% | 重写冗长描述为结构化格式 |
| 首轮固定工具集 | 避免重复 | Turn 1 确定工具集后不再重新分类 |

> 来源：https://futureagi.com/blog/how-to-reduce-mcp-token-costs-claude-code-scale-2026/

### 3.4 常用 MCP Server 推荐

| Server | 用途 | 推荐度 |
|--------|------|:---:|
| **GitHub** | 多仓库开发、PR Review、Issue 管理 | ⭐⭐⭐ |
| **Playwright** | 浏览器自动化、E2E 测试 | ⭐⭐⭐ |
| **PostgreSQL** | Schema 探索、查询生成（建议只读） | ⭐⭐ |
| **Filesystem** | 受限目录的文件访问 | ⭐⭐ |
| **Sequential Thinking** | 强制分步推理、捕获矛盾 | ⭐⭐ |
| **Linear / Jira** | Issue 跟踪 | ⭐ |
| **Slack** | 开发沟通同步（先从只读开始） | ⭐ |

### 3.5 MCP 最佳实践

| 原则 | 说明 |
|------|------|
| **从零开始** | 内置工具（Read/Write/Edit/Bash/Grep/Glob）已覆盖大部分场景。MCP 只在碰壁时加 |
| **上下文警告** | 5 个 Server + 58 个工具 ≈ 55K+ token 开销。保持在 10 个 Server / 80 个工具以内 |
| **HTTP 优于 STDIO** | 2026.4 的 STDIO RCE 披露后，`streamable-http` 成为推荐传输 |
| **环境变量引用** | `${VAR}` 语法，不在 `.mcp.json` 中硬编码密钥 |
| **数据库用只读副本** | 先连 staging/read replica 测试，再考虑生产写权限 |
| **文件系统最小范围** | Scope 到必要最小目录 |
| **先 `enforce: false` 跑一周** | 再加 allowlist 规则 |

> 来源：https://github.com/justinwlin/claude-mcp-guide
> https://github.com/anipotts/claude-code-tips/blob/main/docs/tips/mcp-integration.md

---

## 四、Hooks 系统

### 4.1 概念

Hooks 是**确定性的系统级扩展**——不消耗上下文 token，不受 AI "忘记"规则影响。配置在 `settings.json`。

### 4.2 全部生命周期事件（13 种）

| 事件 | 触发时机 | 可阻断 |
|------|---------|:---:|
| `PreToolUse` | 工具执行前 | ✅ |
| `PostToolUse` | 工具执行成功后 | ❌ |
| `PostToolUseFailure` | 工具执行失败后 | ❌ |
| `UserPromptSubmit` | 用户提交 Prompt 时 | ✅ |
| `SessionStart` | 会话启动/恢复时 | ❌ |
| `SessionEnd` | 会话结束时 | ❌ |
| `Stop` | Agent 完成响应时 | ❌ |
| `SubagentStop` | 子 Agent 完成时 | ❌ |
| `PreCompact` | 上下文压缩前 | ❌ |
| `PostCompact` | 上下文压缩后 | ❌ |
| `ConfigChange` | 配置变更时 | ❌ |
| `Notification` | 发送通知时 | ❌ |
| `PermissionRequest` | 权限弹窗时 | ❌ |

### 4.3 配置结构

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": ".claude/hooks/validate-bash.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "clang-format -i \"$CLAUDE_TOOL_INPUT_FILE_PATH\"",
        "timeout": 30
      }]
    }]
  }
}
```

### 4.4 退出码

| 码 | 含义 |
|:---:|------|
| `0` | 允许操作 |
| `2` | **阻断操作**（stderr 反馈给 AI） |
| `1` | 脚本错误（操作被阻断） |

### 4.5 C++ 项目 Hook 模板

**阻断危险命令**：
```bash
#!/bin/bash
# .claude/hooks/block-dangerous.sh
INPUT=$(cat)
if echo "$INPUT" | grep -qE "rm -rf /|DROP TABLE|git push --force"; then
  echo "BLOCKED: dangerous command detected" >&2
  exit 2
fi
exit 0
```

### 4.6 Skills vs MCP vs Hooks vs Plugins 对比

| 机制 | 用途 | 上下文成本 | 激活 |
|------|------|:---:|------|
| **Skills** | 过程性知识（怎么做） | ~30-50 token/skill | AI 自动或手动 |
| **MCP** | 外部工具连接（访问什么） | 高（工具 schema） | 启动时加载 |
| **Hooks** | 确定性自动化（何时触发） | **零** | 事件驱动 |
| **Plugins** | 以上三者的打包分发 | 按内含 | 安装后激活 |

> 社区共识：大多数开发者需要 2-3 个 MCP Server + 几个自定义 Skills。

---

## 五、C++ 项目完整 `.claude/` 配置模板

```
.claude/
├── settings.json                # 权限 + Hooks + 模型
├── settings.local.json          # 个人覆盖（gitignored）
├── skills/
│   ├── cmake-build/SKILL.md     # 构建技能
│   ├── deploy-check/SKILL.md    # 部署前检查
│   └── cpp-review/SKILL.md      # C++ 代码审查技能
├── agents/
│   ├── code-reviewer.md         # 代码审查子代理
│   ├── test-writer.md           # 测试生成子代理
│   └── security-auditor.md      # 安全审计子代理（Opus）
├── hooks/
│   ├── auto-format.sh           # clang-format 脚本
│   └── block-dangerous.sh       # 危险命令拦截
└── commands/
    └── ship.md                  # 传统斜杠命令（可选）
```

---

## 六、关键来源索引

| 资源 | URL |
|------|-----|
| Skills 开发完整指南 (Sherlock) | https://sherlock.xyz/post/how-to-write-skills-for-claude-code-and-cowork |
| Skills vs MCP vs Plugins 对比 | https://www.morphllm.com/claude-code-skills-mcp-plugins |
| Claude Code Skills 终极指南 | https://skywork.ai/blog/ai-bot/add-skills-to-claude-code-ultimate-guide/ |
| disable-model-invocation 设定 | https://dev.classmethod.jp/en/articles/disable-model-invocation-claude-code/ |
| 插件生态完整指南（中文） | https://github.com/KimYx0207/AI-Coding-Guide-Zh/blob/main/docs/claude-code/08-Plugins生态完整指南.md |
| 社区插件市场 | https://github.com/kossakovsky/cc-plugins |
| 社区 Bazaar | https://github.com/svgsponer/claude-bazaar |
| 阿里云插件市场分析（中文） | https://developer.aliyun.com/article/1747753 |
| MCP Gateway 实践指南 | https://futureagi.com/blog/using-mcp-gateway-with-claude-code-practical-guide-2026/ |
| MCP Token 成本优化 | https://futureagi.com/blog/how-to-reduce-mcp-token-costs-claude-code-scale-2026/ |
| MCP 集成最佳实践 | https://github.com/anipotts/claude-code-tips/blob/main/docs/tips/mcp-integration.md |
| MCP 配置参考指南 | https://github.com/justinwlin/claude-mcp-guide |
| MCP 最佳实践 | https://github.com/MuhammadUsmanGM/claude-code-best-practices/blob/main/guides/mcp-servers.md |
| Skills Marketplace 指南 | https://skywork.ai/blog/ai-bot/claude-code-skills-marketplace-ultimate-guide/ |
| community skills 集合 | https://github.com/alexknowshtml/claude-skills |

---

*文档生成日期：2026-07-16 | 来源：Claude Code 官方文档 + 社区最佳实践 + GitHub 开源生态*
