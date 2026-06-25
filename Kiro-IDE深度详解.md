# Kiro IDE 深度详解 —— AWS 的 Spec 驱动 AI 开发环境

> Kiro 是 AWS 于 2025 年 7 月发布、2026 年 5 月正式国际化的 Agentic IDE。  
> 基于 Code OSS（VS Code 开源基础）构建，核心理念：**Spec 是源代码，代码是构建产物**。

---

## 📋 目录

| 章节 | 内容 |
|------|------|
| 一 | 架构哲学：Spec 驱动的本质 |
| 二 | Spec 驱动开发：完整三阶段工作流 |
| 三 | EARS 需求语法详解 |
| 四 | Steering 文件：项目级 AI 行为配置 |
| 五 | Agent Hooks：事件驱动自动化 |
| 六 | MCP 与 Powers：可扩展能力体系 |
| 七 | 子代理（Subagents）与 Agent Skills |
| 八 | 运行模式：Supervised vs Autopilot vs Auto |
| 九 | CLI、Web、IDE 三端统一 |
| 十 | 定价与 Credit 模型 |
| 十一 | 与其他工具的深度对比 |
| 十二 | C++ 后端团队的适用性分析 |

---

## 一、架构哲学：Spec 驱动的本质

### 1.1 核心范式

```
传统 AI 编程工具（Chat-and-Paste）：
  你在 Chat 里描述需求 → AI 生成代码 → 你粘贴到编辑器

Kiro 的 Spec 驱动：
  你描述需求 → Kiro 生成结构化 Spec → 你审查批准 →
  Kiro 生成设计文档 → 你审查批准 →
  Kiro 分解为原子任务 → 你审查批准 →
  Kiro 逐个执行任务（每次变更可独立审查和回滚）
```

### 1.2 核心理念

| 传统开发 | Kiro Spec 驱动 |
|---------|---------------|
| Spec 是辅助文档，代码是真相源 | **Spec 是真相源，代码是从 Spec 派生的产物** |
| 需求→代码之间缺少可追溯链路 | 每个代码变更都可追溯到 `tasks.md` 中的具体任务 |
| 代码审查时只能看 Diff | 代码审查时可以对照 Spec 验证意图是否实现 |
| AI 生成的代码散落在各处 | AI 的所有修改被组织在 Spec 目录下，结构化可审计 |

### 1.3 Spec 的文件结构

```
.kiro/
├── specs/
│   └── {feature-name}/
│       ├── requirements.md    # 需求文档（EARS 格式）
│       ├── design.md          # 技术设计文档
│       └── tasks.md           # 原子实现任务列表
├── steering/                  # 项目级 AI 行为配置
│   ├── product.md
│   ├── tech.md
│   ├── structure.md
│   └── ...
└── hooks/                     # 事件驱动自动化
    ├── auto-test.kiro.hook
    └── ...
```

---

## 二、Spec 驱动开发：完整三阶段工作流

### 2.1 阶段一：需求（Requirements）

**输入**：你描述想要什么功能
**输出**：`requirements.md` — 结构化的用户故事和验收标准

```
流程：
  你输入 → Kiro 生成 requirements.md → 你审查 → 
  通过 → 进入设计阶段
  不通过 → 你修改需求描述 → Kiro 重新生成 → 再次审查
```

**关键约束**：Kiro **不能跳过**需求阶段直接写代码。这是强制性的架构约束。

### 2.2 阶段二：设计（Design）

**输入**：已批准的需求文档
**输出**：`design.md` — 技术架构设计

设计文档包含：
- 架构决策和理由
- 组件边界和职责
- 数据模型
- API 契约（接口签名）
- 序列图（关键交互流程）
- 正确性属性（哪些情况必须成立）

**审查要点**：
- 设计是否满足所有需求？
- 技术选型是否符合项目 Steering 约束？
- 接口是否与现有代码兼容？

### 2.3 阶段三：任务（Tasks）

**输入**：已批准的设计文档
**输出**：`tasks.md` — 有序的原子实现任务列表

```
tasks.md 示例：
  1. 创建 include/network/redis_pool.h，定义 RedisPool 类接口
  2. 实现 src/network/redis_pool.cpp 的构造函数和析构函数
  3. 实现 getConnection() 方法（含超时和重试）
  4. 实现 returnConnection() 方法
  5. 实现健康检查 checkHealth()
  6. 更新 CMakeLists.txt
  7. 编写单元测试 tests/network/redis_pool_test.cpp
```

**关键特性**：
- 每个任务是**原子**的——可独立审查、独立回滚
- 任务是**有序**的——有依赖关系的任务排序正确
- 任务是**可验证**的——每个任务有明确的完成标准

### 2.4 两种创建路径

| 路径 | 命令 | 适用场景 |
|------|------|---------|
| **需求优先** | `kiro spec create <name>` | 新功能开发 |
| **实现优先** | `kiro spec create <name> --from-existing` | 分析现有代码库，生成描述现有实现的 Spec |

**实现优先路径的价值**：对遗留 C++ 代码库非常有意义。Kiro 会先分析代码，生成理解现有实现的 Spec，让你在不熟悉代码的情况下也能快速理解架构。

### 2.5 Quick Plan 模式

对于简单、理解透彻的功能，可以跳过三阶段的审批门：

```bash
# 一次性生成 requirements + design + tasks，无审批门
kiro spec create <name> --quick-plan
```

**适用场景**：bug 修复、小重构、格式调整等低风险变更。

---

## 三、EARS 需求语法详解

EARS（Easy Approach to Requirements Syntax）是 Kiro 用于编写验收标准的形式化语法。

### 3.1 五种句式

| 句式 | 模板 | 用途 |
|------|------|------|
| **事件驱动** | `WHEN [event] THEN [system] SHALL [response]` | 描述对事件的响应 |
| **条件驱动** | `IF [condition] THEN [system] SHALL [response]` | 描述条件分支行为 |
| **状态驱动** | `WHILE [state] [system] SHALL [response]` | 描述持续状态下的行为 |
| **无条件** | `[system] SHALL [response]` | 描述始终成立的行为 |
| **特征驱动** | `WHERE [feature] [system] SHALL [response]` | 描述特定特征组合下的行为 |

### 3.2 C++ 后端项目示例

```markdown
# Redis 连接池需求

## User Story: 开发者获取 Redis 连接

### Acceptance Criteria

**事件驱动：**
WHEN 开发者调用 getConnection() 
THEN RedisPool SHALL 在 30 秒内返回一个可用连接

**条件驱动：**
IF 连接池中没有空闲连接 AND 当前连接数小于最大值
THEN RedisPool SHALL 创建新连接并返回

IF 连接池达到最大连接数
THEN RedisPool SHALL 等待至少 30 秒后返回超时错误

**状态驱动：**
WHILE Redis 服务器不可达
RedisPool SHALL 每 5 秒尝试重连，最多重试 3 次

**无条件：**
RedisPool SHALL 使用 std::unique_ptr 管理所有连接资源
RedisPool SHALL 在析构时正确关闭所有连接
```

### 3.3 EARS 的优势

| 优势 | 说明 |
|------|------|
| **可测试性** | 每条验收标准可直接转化为测试用例 |
| **无歧义** | 结构化的 WHEN/IF/WHILE 消除自然语言的模糊性 |
| **完整性检查** | 缺少某种句式 = 遗漏了某类需求（如无条件要求） |
| **AI 友好** | LLM 对结构化格式的解析准确率远高于自由文本 |

---

## 四、Steering 文件：项目级 AI 行为配置

Steering 文件是放在 `.kiro/steering/` 下的 Markdown 文件，定义 AI 在所有交互中遵守的规则。

### 4.1 两种加载模式

| 模式 | 配置 | 行为 |
|------|------|------|
| **始终加载** | `inclusion: always` | 每次 AI 交互都注入这些规则 |
| **条件加载** | `inclusion: fileMatch` + `fileMatch: "**/*.cpp"` | 仅在匹配的文件处于活动状态时注入 |

### 4.2 完整示例：C++ 后端项目

```markdown
<!-- .kiro/steering/tech.md -->
---
inclusion: always
---

# 技术栈
- C++17 (GCC 11+, Clang 14+)
- CMake 3.20+，构建命令：cmake -B build && cmake --build build -j$(nproc)
- GoogleTest 测试框架
- 依赖库：hiredis, occi, cppzmq, protobuf, pthread

# 禁止事项
- MUST NOT: 使用裸 new/delete
- MUST NOT: 使用 C-style cast
- MUST NOT: 在头文件中暴露第三方库头文件（Pimpl 惯用法）
- MUST NOT: 使用异常（项目编译选项 -fno-exceptions）
```

```markdown
<!-- .kiro/steering/structure.md -->
---
inclusion: always
---

# 项目结构
- include/network/ — 网络层公共头文件
- src/network/ — 网络层实现
- include/db/ — 数据库层公共头文件
- src/db/ — 数据库层实现
- include/mq/ — 消息队列层公共头文件
- src/mq/ — 消息队列层实现
- tests/ — 测试文件（镜像 src/ 结构）

# 命名约定
- 文件名：snake_case
- 类名：PascalCase
- 函数名：snake_case
- 成员变量：trailing_underscore
- 命名空间：project_name::module_name
```

```markdown
<!-- .kiro/steering/network-standards.md -->
---
inclusion: fileMatch
fileMatch: "src/network/**/*.{cpp,h}"
---

# 网络层规范
- 所有 socket IO 必须设置超时
- 使用 non-blocking IO + epoll
- 连接断开后必须自动重连
- 缓冲区大小上限 64KB，超出需分片
- 所有网络数据必须校验长度和格式
```

---

## 五、Agent Hooks：事件驱动自动化

### 5.1 触发事件

| 事件类型 | 触发时机 | 典型用途 |
|---------|---------|---------|
| `fileSaved` | 匹配 glob 的文件被保存 | 自动格式化、自动生成测试、更新文档 |
| `fileCreated` | 新文件被创建 | 自动添加到 CMakeLists.txt、自动添加版权头 |
| `fileDeleted` | 文件被删除 | 自动清理 CMakeLists.txt 引用 |
| `task:pre` | 任务开始执行前 | 检查前置条件 |
| `task:post` | 任务执行完成后 | 运行测试、检查覆盖率 |
| `manual` | 手动触发 | 安全扫描、全量代码审查 |

### 5.2 完整示例

```json
// .kiro/hooks/auto-format.kiro.hook
{
  "name": "C++ Auto Format on Save",
  "version": "1",
  "when": {
    "type": "fileSaved",
    "patterns": ["src/**/*.{cpp,h}", "include/**/*.{cpp,h}"]
  },
  "then": {
    "type": "runShell",
    "command": "clang-format -i ${FILE_PATH}"
  }
}
```

```json
// .kiro/hooks/auto-test-on-task.kiro.hook
{
  "name": "Auto Test After Task",
  "version": "1",
  "when": {
    "type": "task:post"
  },
  "then": {
    "type": "askAgent",
    "prompt": "任务已完成。请：1. 检查对应的测试文件是否存在且覆盖了新代码 2. 运行 cmake --build build && cd build && ctest 3. 如有失败，分析原因并修复"
  }
}
```

```json
// .kiro/hooks/add-to-cmake.kiro.hook
{
  "name": "Auto Update CMakeLists on New File",
  "version": "1",
  "when": {
    "type": "fileCreated",
    "patterns": ["src/**/*.cpp", "include/**/*.h"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "新文件 ${FILE_PATH} 已创建。请：1. 找到对应的 CMakeLists.txt 2. 将新文件添加到合适的 target 中 3. 确保 include 目录被正确配置"
  }
}
```

---

## 六、MCP 与 Powers：可扩展能力体系

### 6.1 MCP Server 支持

Kiro 原生支持 MCP 协议，可直接配置社区或自建的 MCP Server。

**AWS 官方 MCP Server 列表**：

| MCP Server | 功能 |
|-----------|------|
| `aws-api-mcp-server` | AWS 服务管理（创建/查询资源） |
| `cdk-mcp-server` | CDK 最佳实践 + CDK Nag 安全扫描 |
| `cloudwatch-mcp-server` | 监控指标查询、日志搜索 |
| `cloudtrail-mcp-server` | 审计日志分析 |
| `aws-pricing-mcp-server` | 资源成本估算 |
| `knowledge-mcp-server` | AWS 文档检索 |
| `bedrock-kb-retrieval-mcp-server` | 知识库 RAG 检索 |

**配置位置**：`.vscode/mcp.json` 或 `.cursor/mcp.json`

### 6.2 Powers：打包好的能力套件

Powers 是比 MCP Server 更高层的抽象——预打包了 MCP Server + Steering 文件 + Hooks 的组合。

```
Power = MCP Server(s) + Steering Files + Hooks + Agent Skills
```

**示例：AWS Amplify Power**
- 引导式工作流：认证 → 数据模型 → 存储 → 函数 → 沙盒 → 部署
- 分阶段编排器确保依赖顺序正确
- 自动处理 AWS 资源的创建和配置

**示例：AI-DLC Power**（AI 驱动开发生命周期）
- 12 个绿地开发阶段 / 13 个棕地开发阶段
- 自动工作区检测和逆向工程
- 适用于任何技术栈（不仅是 AWS）

---

## 七、子代理（Subagents）与 Agent Skills

### 7.1 自定义子代理

Kiro 支持创建专门的子代理，每个可以有自己的：
- 系统 Prompt（定义角色和专长）
- 可用工具集（限制工具范围）
- 模型选择（轻量任务用 Haiku，复杂任务用 Opus）

```
主 Agent
  ├── Code Review 子代理（Opus，只读工具，专注安全审查）
  ├── Test Writer 子代理（Sonnet，读写工具，专注测试生成）
  └── Doc Writer 子代理（Haiku，只读+写入，专注文档更新）
```

### 7.2 并行会话

Kiro 支持**多个 Tab 并行运行**不同的 Agent/会话：

```
Tab 1: Spec 驱动开发——实现 Redis 连接池
Tab 2: Chat——调试 epoll 事件循环的性能问题
Tab 3: Code Review 子代理——审查 Tab 1 生成的代码
```

每个 Tab 独立管理上下文、模型选择、工具权限。

---

## 八、运行模式：Supervised vs Autopilot vs Auto

### 8.1 三种模式对比

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| **Supervised** | Agent 每步暂停，显示变更 + 理由，等待你批准 | 复杂功能、陌生代码库、敏感系统 |
| **Autopilot** | Agent 按 tasks.md 自主执行，不每步暂停 | 定义明确、范围可控、已批准 Spec |
| **Auto（默认）** | 混合模式：Kiro 自动判断何时需要你审查，何时可以自主执行 | 日常使用 |

### 8.2 原子回滚

无论使用哪种模式，每个 AI 操作都是**原子的**——可以一键回滚到任意历史状态（类似 Git，但粒度更细）。

---

## 九、CLI、Web、IDE 三端统一

### 9.1 Kiro CLI

```bash
# 启动 CLI
kiro .

# Spec 命令
kiro spec create <feature-name>
kiro spec create <name> --from-existing   # 从现有代码反向生成 Spec
kiro spec create <name> --quick-plan      # 快速规划模式
kiro spec list
kiro spec run <feature> --task <n>        # 运行指定任务

# CLI v3 (Early Access)
kiro-cli --v3
```

### 9.2 Kiro Web

浏览器中使用的完整 Kiro 体验：
- GitHub 集成和 PR 创建
- 支持 Spec 驱动开发（功能略少于 IDE）
- 无需安装，浏览器即用

### 9.3 统一引擎

CLI v3、IDE 和 Kiro Web 共享同一个 Agent 引擎。引擎层面的改进（工具、规划、模型选择）同时对所有端生效。

---

## 十、定价与 Credit 模型

### 10.1 个人计划

| 计划 | 月费 | Credits/月 | 超额 | 关键解锁 |
|------|:---:|:---:|------|------|
| **Free** | $0 | 50 | 不可超额 | 基础补全、有限模型 |
| **Pro** | $20 | 1,000 | $0.04/credit | Claude Opus、子代理、Powers、Hooks、CLI |
| **Pro+** | $40 | 2,000 | $0.04/credit | 同上，2x credits |
| **Pro Max** | $100 | 5,000 | $0.04/credit | 同上，5x credits（**2026.6 新增**） |
| **Power** | $200 | 10,000 | $0.04/credit | 同上，最高额度 |

### 10.2 Credit 消耗倍率

| 模型 | 倍率 | 说明 |
|------|:---:|------|
| Auto（推荐） | 1.0x | 自动路由到最佳模型 |
| Claude Sonnet 4.6 | 1.3x | 日常高性价比 |
| Claude Opus 4.8 | 2.2x | 最强推理，复杂任务用 |
| Claude Haiku 4.5 | 0.4x | 轻量任务 |
| Qwen3 Coder Next | 0.05x | 预算选项 |

### 10.3 团队计划

| 特性 | 个人 | 团队 |
|------|:---:|:---:|
| 每席位价格 | $20-200 | $20-200 |
| 集中账单 | ❌ | ✅ |
| 使用分析 | ❌ | ✅ |
| SAML/SCIM SSO | ❌ | ✅ |
| 共享 Credit 池 | ❌ | ❌（每席位独立） |

> ⚠️ **注意**：团队没有共享 Credit 池。每个开发者需要自己的订阅。

---

## 十一、与其他工具的深度对比

### 11.1 哲学对比

| 维度 | Kiro | Cursor | Windsurf | Claude Code |
|------|------|--------|---------|-------------|
| **核心范式** | Spec 驱动 | Chat + Composer | Cascade（自主 Agent） | 终端 Agent |
| **Spec 强制** | ✅ 强制三阶段 | ⚠️ Plan Mode 可选 | ❌ 无 | ⚠️ Plan Mode 可选 |
| **变更可追溯** | ✅ 每个变更有 spec 溯源 | ❌ 依赖 git history | ❌ | ❌ |
| **原子回滚** | ✅ 每个 AI 操作 | ⚠️ Checkpoints | ❌ | ❌ |
| **事件自动化** | ✅ Agent Hooks | ❌ | ❌ | ⚠️ Hook 系统 |
| **代码库分析** | 语义索引 | 向量索引 | 语义地图 | 文件系统读取 |

### 11.2 技术栈对比

| 维度 | Kiro | Cursor | Windsurf | Claude Code |
|------|------|--------|---------|-------------|
| **基础** | Code OSS | VS Code 分支 | VS Code 分支 | 终端 TUI |
| **AI 引擎** | Amazon Bedrock | 多 API | 多 API + 自研 | Anthropic API |
| **模型选择** | Claude + Qwen + DeepSeek + GLM | Claude + GPT + Gemini + Grok | Claude + GPT + Gemini | 仅 Claude |
| **安装** | 下载安装包 | 下载安装包 | 下载安装包 | curl / brew |

### 11.3 Kiro 的独特优势

| 优势 | 为什么重要 |
|------|-----------|
| **强制的 Spec 流程** | 防止 AI "直接写代码"而不思考架构 |
| **EARS 形式化需求** | 需求可测试、可审计、可作为测试用例来源 |
| **Agent Hooks** | 让 AI 融入开发流程而非独立工具（保存即触发） |
| **Spec 纳入 Git** | 需求→设计→任务→代码的完整可追溯链 |
| **从现有代码反向生成 Spec** | 对遗留 C++ 代码库非常有价值 |
| **AWS 深度集成** | 如果用 AWS 生态，这是无可替代的优势 |

---

## 十二、C++ 后端团队的适用性分析

### 12.1 强场景匹配

| 场景 | 为什么 Kiro 合适 |
|------|-----------------|
| **遗留 C++ 代码库** | `--from-existing` 反向生成 Spec，理解大型 C++ 项目的架构 |
| **多人协作的复杂功能** | Spec 作为"合同"，评审通过后 AI 按合同执行，减少理解偏差 |
| **需要审计追溯** | 需求→设计→任务→代码的完整链，每个变更都知道"为什么做" |
| **安全关键代码** | EARS 验收标准 + 设计审查 + 任务审查 = 比直接生成多两层安全门 |
| **CMake 构建自动化** | Agent Hooks 可以在保存 .cpp 时自动更新 CMakeLists.txt |

### 12.2 弱场景

| 场景 | 为什么 Kiro 可能不合适 |
|------|---------------------|
| **简单 bug 修复** | 三阶段 Spec 流程对于一行修复来说太重 |
| **非 AWS 生态** | AWS 深度集成优势浪费（但 Spec 驱动的价值仍在） |
| **CLion 用户** | Kiro 基于 Code OSS，CLion 用户需要切换 IDE |
| **偏好终端** | Kiro 的核心体验在 IDE 中，CLI 功能不如 IDE 完整 |

### 12.3 混合使用策略

```
大型新功能 / 重构：
  → Kiro Spec 驱动（需求→设计→任务→实施）

日常单文件修改 / bug 修复：
  → Cursor 或 Claude Code（快速、轻量）

代码审查：
  → Kiro Code Review 子代理（Opus，只读工具）
    或 Claude Code /review

构建自动化：
  → Kiro Agent Hooks（保存时自动触发 cmake + ctest）
```

---

## 📚 推荐资源

| 资源 | 链接 |
|------|------|
| Kiro 官网 | https://kiro.dev |
| Kiro 官方文档 | https://kiro.dev/docs |
| Kiro 定价 | https://kiro.dev/pricing |
| SDD Workshop（动手教程） | https://github.com/094459/sdd-workshop |
| Kiro 系统 Prompt（开源） | https://github.com/jasonkneen/kiro |
| Kiro Changelog | https://kiro.dev/changelog |
| Kiro vs Cursor 对比 | https://www.morphllm.com/comparisons/kiro-vs-cursor |

---

*文档生成日期：2026-06-24 | 数据来源：kiro.dev 官方文档、AWS 官方博客、社区分析报告*
