# AI 编程 Know-How —— C++ 后端开发者的 AI 工具实战指南

> **演讲时长**：60 分钟  
> **目标听众**：C++17 后端开发者（Socket / Redis / Oracle / ZMQ 分布式系统）  
> **日期**：2026 年 6 月

---

## 📋 目录

| 章节 | 主题 | 时长 |
|------|------|------|
| 一 | 开场 & 为什么现在必须学 AI 编程 | 5 min |
| 二 | 主流大模型概览与选型 | 10 min |
| 三 | AI 编程工具详解 | 20 min |
| 四 | 安装与配置实战 | 10 min |
| 五 | C++ 后端开发最佳实践 | 10 min |
| 六 | 总结与推荐方案 | 5 min |

---

## 一、开场 & 为什么现在必须学 AI 编程（5 min）

### 1.1 2026 年的现实

```
传统开发模式：需求 → 设计 → 编码 → 调试 → 测试 → 部署
                          ↑
                    耗时占比 40-60%
AI 辅助模式：需求 → 设计 → [AI 生成代码] → [AI 辅助调试] → 测试 → 部署
                          ↑                        ↑
                    耗时降低 50-70%          耗时降低 40-60%
```

**数据说话**（2026 年行业报告）：
- **SWE-bench Verified**：顶级模型已突破 88%，意味着 AI 能独立完成 88% 的真实开源项目 bug 修复
- **Cursor 自动补全接受率 72%** vs GitHub Copilot 30%
- **Claude Code 多文件重构成功率 85-95%**
- **开发者日均节省 2-4 小时**（Stack Overflow 2026 调查）

### 1.2 为什么 C++ 后端开发者更需要 AI

| 痛点 | AI 如何帮助 |
|------|------------|
| 模板元编程 / SFINAE 写法繁琐 | AI 生成模板代码，正确率远超手写 |
| CMake / Bazel 构建配置复杂 | AI 自动生成和调试构建脚本 |
| Socket / ZMQ 样板代码多 | AI 生成 epoll / io_uring 异步框架代码 |
| Oracle OCCI / Redis hiredis 接口冗长 | AI 快速生成 CRUD 封装 |
| 分布式系统调试困难 | AI 辅助分析日志、定位死锁/竞态 |
| 内存安全审计耗时 | AI 作为第一道防线扫描潜在问题 |

### 1.3 本次分享的目标

✅ 了解当前主流大模型的能力边界和选型依据  
✅ 掌握 3-5 款核心 AI 编程工具的安装和使用  
✅ 学会 C++ 后端的 AI 编程最佳实践  
✅ 明天就能在工作中用起来

---

## 二、主流大模型概览与选型（10 min）

### 2.1 2026 年 6 月模型格局

```
                   能力
                    ↑
     Claude Opus 4.8  ●  GPT-5.5
                    │
     Gemini 3.1 Pro  ●
                    │
     DeepSeek V4    ●
                    │
     Qwen3.7 Max    ●
                    │
     DeepSeek V4 Flash ●
                    │
                    └──────────────────────→ 成本
```

### 2.2 核心模型对比

| 维度 | Claude Opus 4.8 | GPT-5.5 | Gemini 3.1 Pro | DeepSeek V4 Pro | DeepSeek V4 Flash |
|------|:---:|:---:|:---:|:---:|:---:|
| **API 输入价格** ($/1M token) | 5.00 | 5.00 | 2.00 | 0.435 | **0.14** |
| **API 输出价格** ($/1M token) | 25.00 | 30.00 | 12.00 | 0.87 | **0.28** |
| **上下文窗口** | 1M | 1M | 1M | 1M | 1M |
| **SWE-bench (编码)** | **88.6%** | 88.7% | 80.6% | 80.6% | ~78% |
| **GPQA (推理)** | 91.3% | 81% | **94.3%** | - | - |
| **Terminal-Bench** | 74.6% | **82.7%** | 78.4% | - | - |
| **开源权重** | ❌ | ❌ | ❌ | ✅ MIT | ✅ MIT |
| **中文能力** | ✅ | ✅ | ✅ | ✅ **最强** | ✅ **最强** |

### 2.3 如何选择模型 —— 分层策略

```
┌─────────────────────────────────────────────┐
│  复杂度分层                                  │
│                                             │
│  高复杂度（架构设计/多文件重构）              │
│  → Claude Opus 4.8 / GPT-5.5                │
│  每次调用成本：$0.50 - $2.00                 │
│                                             │
│  中复杂度（单文件实现/调试）                  │
│  → Gemini 3.1 Pro / DeepSeek V4 Pro          │
│  每次调用成本：$0.05 - $0.20                 │
│                                             │
│  低复杂度（补全/格式化/简单重构）             │
│  → DeepSeek V4 Flash / 本地小模型            │
│  每次调用成本：<$0.01                        │
└─────────────────────────────────────────────┘
```

### 2.4 实际成本估算（C++ 后端日常使用场景）

| 使用强度 | 模型选择 | 月成本/人 |
|---------|---------|----------|
| 轻度（偶尔补全和问答） | DeepSeek + 免费额度 | $0 - $20 |
| 中度（日常编码 + 代码审查） | Gemini + Claude（混合） | $50 - $100 |
| 重度（全流程 AI 辅助） | Claude Max 订阅 | $100 - $200 |
| 重度 API 直连 | Claude Opus 全量调用 | $600 - $1,500 |

> 💡 **省钱技巧**：使用订阅制（Claude Max $100-200/月）而非 API 按量付费，重度使用可节省 60-80%

---

## 三、AI 编程工具详解（20 min）⭐ 核心章节

### 3.1 工具全景图

```
                    IDE 集成度
                        ↑
              Cursor    │   Copilot
              Kiro      │   (插件)
              (AI IDE)  │
                        │
    ────────────────────┼────────────────────→ 自主能力
                        │
        Cline           │   Claude Code
        (VS Code 插件)  │   (终端 Agent)
                        │
              Aider     │
              (终端)    │
```

### 3.2 横向对比总表

| 特性 | Claude Code | Cursor | Kiro | GitHub Copilot | Cline | Aider |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| **类型** | 终端 Agent | AI IDE | AI IDE (AWS) | IDE 插件 | VS Code 插件 | 终端工具 |
| **安装方式** | curl / brew | 下载安装包 | 下载安装包 | 扩展商店 | 扩展商店 | pip install |
| **价格** | $20-200/月 | $20-60/月 | 免费(基础) | $10-39/月 | 免费开源 | 免费开源 |
| **多模型支持** | ❌ 仅 Claude | ✅ 多模型 | ✅ 多模型 | ✅ 多模型 | ✅ 任意 | ✅ 任意 |
| **多文件编辑** | ✅ 最强 | ✅ Composer | ✅ Spec 驱动 | ⚠️ Agent Mode | ✅ | ✅ |
| **自动补全** | ❌ 无 | ✅ 72% 接受率 | ✅ | ✅ 30% 接受率 | ❌ 无 | ❌ 无 |
| **终端集成** | ✅ 原生 | ⚠️ 内嵌终端 | ✅ CLI 工具 | ❌ | ⚠️ | ✅ 原生 |
| **Git 集成** | ✅ 原生 | ✅ | ✅ PR 创建 | ✅ PR Review | ⚠️ | ✅ |
| **MCP 支持** | ✅ | ✅ | ✅ 原生 | ✅ | ✅ 最强 | ❌ |
| **后台 Agent** | ❌ | ✅ 云异步 | ✅ 并行会话 | ❌ | ❌ | ❌ |
| **Spec/规划驱动** | ✅ Plan Mode | ⚠️ | ✅ **核心特色** | ❌ | ✅ Plan/Act | ❌ |
| **原子回滚** | ❌ | ⚠️ Checkpoints | ✅ **一键回滚** | ❌ | ❌ | ❌ |
| **离线/本地模型** | ❌ | ❌ | ❌ | ❌ | ✅ Ollama | ✅ Ollama |
| **C++ 代码理解** | ✅ 文件系统级 | ✅ 语义索引 | ✅ 语义索引 | ✅ LSP | ✅ | ✅ |
| **AWS 集成** | ❌ | ❌ | ✅ **原生深度** | ❌ | ❌ | ❌ |
| **中文支持** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **CLion 支持** | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |

### 3.3 Claude Code —— C++ 后端首选终端 Agent

**一句话概括**：终端原生的最强 AI 编程 Agent，适合大代码库、复杂重构、CI/CD 自动化。

#### 为什么 C++ 后端开发者应该首选 Claude Code

1. **文件系统级代码理解** —— 不依赖 IDE 索引，直接读文件。对 CMake/Bazel 等复杂构建系统无感知差异
2. **100 万 token 上下文** —— 能一次装入整个 C++ 服务模块（头文件 + 源文件 + 构建脚本）
3. **终端原生** —— 后端开发者日常工作环境，grep/log/gdb/valgrind 无缝衔接
4. **Plan Mode（规划模式）** —— 先出方案再改代码，避免 C++ 重构引入未定义行为
5. **Extended Thinking（深度思考）** —— 显式思维链，对复杂 C++ 问题（模板推导、内存模型、异步模式）有更强推理能力
6. **/init 一键生成 CLAUDE.md** —— 自动文档化项目规范，后续 AI 始终遵守团队约定

#### 核心功能速览

```
claude                    # 启动交互式会话
claude -p "修复这段代码"   # 单次提问模式（适合脚本化）
claude --plan             # 进入规划模式
claude --resume           # 恢复上次会话
```

**会话内部命令**：

| 命令 | 功能 |
|------|------|
| `/init` | 生成项目 CLAUDE.md |
| `/model` | 切换模型（Opus / Sonnet / Haiku） |
| `/clear` | 清空上下文 |
| `/compact` | 压缩历史（上下文 >80% 时使用） |
| `/cost` | 查看 token 消耗 |
| `/review` | Code Review 当前改动 |
| `/doctor` | 诊断配置问题 |
| `/mcp` | 管理 MCP 服务器连接 |

#### 典型工作流

```bash
# 1. 进入项目，启动 Claude Code
cd ~/projects/game-server
claude

# 2. 首次使用：生成项目知识文件
/init

# 3. 描述需求
> 帮我把这个 epoll 事件循环改成 io_uring，头文件在 include/，
> 源文件在 src/core/，构建脚本是 CMakeLists.txt。先规划再改。

# 4. AI 进入 Plan 模式 → 给出方案 → 你确认 → 执行修改

# 5. 验证
> 运行 cmake --build build && cd build && ctest

# 6. 审查
/review
```

#### 💡 技术内幕：Claude Code 是如何工作的？

> 社区通过 npm source map 对 Claude Code 进行了深入的逆向分析（详见 `Claude-Code-源码分析汇总.md`）。以下是几个对使用者有价值的关键发现：

**架构极简**：没有意图分类器、没有规划器、没有 RAG 管道。整个系统就是一个不断往 `messages[]` 数组追加内容的主循环，模型自主决定调用哪个工具、何时完成。

**10 个核心架构模式**中，对使用者最有价值的 3 个：

| 模式 | 对你的影响 |
|------|-----------|
| **Fork Agent 共享 Prompt Cache** | 并行创建多个子 Agent 查代码，token 成本几乎不增加（~95% cache 命中） |
| **4 层上下文压缩** | 长会话会自动压缩。压缩前确保关键信息已写入文件或记忆 |
| **Sticky Latches 保持缓存** | CLAUDE.md 等稳定前缀对成本影响巨大——它们是每次 API 调用的"固定部分"，100% 命中缓存 |

**实战建议**：
1. CLAUDE.md 写得越精确，AI 行为越一致，token 浪费越少
2. 长会话中主动 `/compact`，压缩前把关键决策写入记忆
3. 多并行子 Agent（`/workflow`）比单串行 Agent 更省 token

### 3.4 Cursor —— 最强 AI IDE

**一句话概括**：VS Code 的 AI 增强版，Composer 多文件编辑 + 72% 补全接受率是杀手功能。

#### 核心功能

| 功能 | 快捷键 | 说明 |
|------|--------|------|
| **Tab 补全** | `Tab` | 多行上下文感知补全，C++ 模板/STL 支持好 |
| **Inline Edit** | `Ctrl+K` | 选中代码 → 自然语言描述 → AI 改写 |
| **Chat** | `Ctrl+L` | 代码库感知问答 |
| **Composer** | `Ctrl+I` | **多文件编辑** —— 一次指令协调修改 10+ 文件 |
| **Agent Mode** | `Ctrl+Shift+I` | **全自主模式** —— 读文件→编辑→运行命令→修复错误，循环迭代 |
| **Background Agent** | - | 异步后台任务，编译时也能继续工作 |

#### Composer 提示技巧 —— @ 引用

```
@file:src/core/event_loop.cpp    # 引用特定文件
@folder:include/                 # 引用整个目录
@codebase                        # 语义搜索整个代码库
@web                             # 搜索互联网
@docs:cppreference               # 引用 C++ 参考文档
@git                             # 引用 git 历史
```

#### C++ 项目专属配置

**`.cursorrules` 示例**：

```yaml
# C++ 后端项目规范
Tech Stack:
  - C++17 (gcc 11+ / clang 14+)
  - CMake 3.20+
  - Redis (hiredis)
  - Oracle (OCCI)
  - ZMQ (cppzmq)
  - Protobuf 3.x

Code Style:
  - Google C++ Style Guide
  - 命名：snake_case 函数/变量，PascalCase 类名
  - 使用 const auto& 遍历容器
  - 禁止裸指针，用 std::unique_ptr / std::shared_ptr
  - 错误处理用 std::optional / absl::StatusOr

Constraints:
  - 不得使用异常（项目禁用 -fno-exceptions）
  - 头文件中不得包含不必要的依赖
  - 所有网络 IO 必须设置超时
  - 数据库连接必须使用连接池
  - 内存分配必须记录到内存池统计

Testing:
  - GoogleTest 框架
  - 每个公共接口至少 2 个测试用例
  - Mock 外部依赖（Redis/Oracle/ZMQ）
```

#### 典型工作流

```
1. 打开项目 → Cursor 自动索引代码库
2. Ctrl+I 打开 Composer → 描述需求
3. 审查 Diff → Accept/Reject
4. Ctrl+Shift+I Agent Mode → 让 AI 自动运行构建和测试
5. 发现问题 → Ctrl+L Chat 调试
```

### 3.5 Kiro —— AWS 出品的 Spec 驱动 AI IDE

**一句话概括**：AWS 打造的 Agentic IDE，核心特色是"先规划再编码"的 Spec 驱动开发 + 原子回滚 + 深度 AWS 集成。

#### 为什么值得关注

1. **Spec 驱动开发（核心差异化）** —— 不是"聊一句写一行"，而是：
   - `requirements.md` → 从自然语言生成 EARS 规范需求
   - `design.md` → 架构设计方案
   - `tasks.md` → 按依赖排序的任务列表
   - 然后再编码，**每一步都有文档可追溯**
2. **原子回滚** —— 每个 AI 操作都是原子事务，一键回滚到任意历史状态。C++ 重构错了也不怕
3. **Agent Hooks** —— 文件保存/创建/删除时自动触发 Agent（自动格式化、自动生成文档、自动 commit）
4. **Steering 文件** —— `.kiro/steering/` 目录下的项目说明书，比 `.cursorrules` 更结构化（支持 always/conditional/manual 三种加载模式）
5. **并行会话** —— 多个 Agent 同时工作，一个在重构网络层，一个在写测试
6. **CLI 工具** —— 终端内使用，支持 CI/CD 无头模式

#### 安装

```bash
# 1. 下载 IDE：https://kiro.dev/downloads
#    - macOS: 拖入 Applications
#    - Windows: 双击安装程序
#    - Linux: chmod +x kiro.AppImage && ./kiro.AppImage
#      或 sudo dpkg -i kiro_*.deb

# 2. CLI 安装（可选，用于终端/CI）
sudo dpkg -i kiro_*.deb

# 3. 启动
cd your-cpp-project
kiro .
```

#### 登录方式

| 方式 | 适用场景 |
|------|---------|
| **GitHub** | 个人开发者（推荐） |
| **AWS Builder ID** | 免费体验 AWS 生态 |
| **AWS IAM Identity Center** | 企业环境，SSO 统一管控 |

#### 核心交互

| 操作 | 快捷键 |
|------|--------|
| 打开 AI Chat | `Ctrl+L` |
| Plan 模式切换 | `Shift+Tab` |
| 引用文件 | `@文件名` |
| 引用上下文 | `#File` / `#Folder` / `#Problems` |
| 执行终端命令 | `!命令` |
| 多行输入 | `Shift+Enter` |

#### Steering 文件示例（C++ 后端项目）

```markdown
<!-- .kiro/steering/backend.md -->

## 项目名称
Game Server Backend

## 技术栈
- C++17 (GCC 11+)
- CMake 3.20+
- Redis (hiredis)
- Oracle (OCCI)
- ZMQ (cppzmq)

## 代码规范
- Google C++ Style Guide
- 禁用异常 (-fno-exceptions)
- 使用 std::unique_ptr 管理资源
- 所有网络 IO 必须设置超时
- 数据库连接使用连接池

## 约束
- MUST NOT: 裸 new/delete
- MUST NOT: 阻塞主事件循环
- MUST: 所有公共 API 带 Doxygen 注释
```

#### 适用场景

- ✅ 团队已经或计划使用 AWS 云服务（深度集成 Bedrock/Lambda/DynamoDB）
- ✅ 需要结构化的开发流程（需求→设计→任务→编码→回滚）
- ✅ 在意 AI 操作可审计、可回滚的企业环境
- ✅ 需要并行处理多个任务的开发者
- ⚠️ 纯 C++ 后端与 AWS 生态无关时，Spec 驱动的优势仍然存在，但 AWS 集成优势浪费

#### 与 Cursor 对比

| 维度 | Kiro | Cursor |
|------|------|--------|
| **开发范式** | Spec 驱动（先规划再编码） | Composer 即时编辑 |
| **回滚能力** | 原子回滚到任意历史 | Checkpoints 快照 |
| **云生态** | AWS 深度集成 | 无特定云绑定 |
| **社区成熟度** | 较新（2025 年 7 月发布） | 最成熟的 AI IDE |
| **C++ 生态** | 通用支持 | 更成熟的 C++ 社区配置 |
| **自动化钩子** | Agent Hooks（保存时触发） | 无等价功能 |
| **VS Code 兼容** | 基于 Code OSS，可导入配置 | VS Code 分支，完全兼容 |

---

### 3.6 Windsurf —— OpenAI 收购的 AI IDE 新贵

**一句话概括**：被 OpenAI 以约 30 亿美元收购（2026 年 3 月），Cascade Agent 比 Cursor Composer 更自主，对大型 C++ 代码库的跨文件感知更强。

#### 核心差异化

| 特性 | Windsurf | Cursor |
|------|---------|--------|
| **核心 Agent** | Cascade（更自主） | Composer（更可控） |
| **代码库索引** | 语义地图 + LLM 搜索 | 向量存储 + 编码器 |
| **跨文件感知** | 始终在线（自动感知最近编辑） | 需要显式 @-mention |
| **跨会话记忆** | ✅ 持久化"Memories" | ❌ 会话级 |
| **Tab 到跳转** | ✅ Supercomplete + Tab to Jump | ❌ 无 |
| **价格** | $15/月（Pro） | $20/月（Pro） |
| **并行 Agent** | ❌ 串行 Cascade 步骤 | ✅ 云异步后台 Agent |

#### 为什么对 C++ 后端团队有吸引力

1. **跨文件感知更适合 C++** —— 自动发现 `.h` ↔ `.cpp` 的对应关系，修改头文件时自动提示更新实现文件
2. **持久化记忆** —— 跨会话记住项目约定，不需要每次重新描述
3. **大型 monorepo 更友好** —— 云端索引 + 语义地图处理大型代码库比向量搜索更准确
4. **性价比** —— $15/月比 Cursor 便宜 25%

#### 适用场景

- ✅ 大型 C++ monorepo，有很多头文件/源文件对的跨文件修改
- ✅ 偏好 Agent 自主性更强的工作流
- ✅ 需要跨会话保持项目上下文
- ⚠️ Cursor 的 Composer 更成熟，社区资源更丰富
- ⚠️ 被 OpenAI 收购后的产品方向仍在整合中

---

### 3.7 GitHub Copilot —— 企业团队标配

**一句话概括**：最广泛的 IDE 支持（包括 CLion），企业治理最完善。

#### IDE 支持矩阵

| IDE | 代码补全 | Chat | Agent Mode | Code Review |
|-----|:---:|:---:|:---:|:---:|
| VS Code | ✅ | ✅ | ✅ | ✅ |
| Visual Studio | ✅ | ✅ | ⚠️ | ✅ |
| JetBrains CLion | ✅ | ✅ | ✅ (2026) | ✅ |
| Neovim | ✅ | ✅ | ❌ | ❌ |
| JetBrains IDEA | ✅ | ✅ | ✅ | ✅ |

#### CLion 配置要点（对 C++ 团队最重要）

```
Settings → Plugins → Marketplace → 搜索 "GitHub Copilot" → 安装
→ Tools → GitHub Copilot → Login to GitHub
→ 首次打开 Copilot Chat 需要额外授权
```

**C++ 上下文增强**（Microsoft 2026 年 5 月发布）：

```json
// .vscode/settings.json 或 CLion 等效配置
{
  "github.copilot.advanced": {
    "customInstructions": "./copilot-instructions.md"
  }
}
```

```markdown
<!-- copilot-instructions.md -->
## C++ 项目上下文
- 使用 C++17 标准，CMake 构建
- 禁用异常，使用错误码返回
- 命名空间：project::module
- 所有接口在 .h 声明，.cpp 实现
```

#### 2026 年 6 月计费变更（重要！）

- Agent Mode 和高级模型使用 **AI Credits** 计费
- 重度 Agent 用户账单从 $29 → 最高 $750/月
- 2026 年 8 月前有促销 credits 缓冲
- **建议**：团队评估后设置用量上限

### 3.8 Cline —— 开源免费 + 最强 MCP 生态

**一句话概括**：VS Code 最强开源 AI 插件，支持任意模型 + 本地模型 + MCP 工具市场。

#### 核心优势

- ✅ **完全免费开源**（500 万+ VS Code 安装量）
- ✅ **支持任意模型**：Claude / GPT / DeepSeek / Gemini / Ollama 本地模型
- ✅ **Plan/Act 双模式**：规划和执行分离，可分配不同模型
- ✅ **最强 MCP 生态**：内置 MCP Marketplace，一键安装社区工具
- ✅ **Browser Automation**：可视化调试 Web 页面

#### 适用场景

- 想免费使用 AI 编程能力
- 需要接入本地/私有模型（代码安全要求高）
- 需要 MCP 扩展数据库、监控、部署等后端工具

### 3.9 Aider —— 终端 AI 结对编程

**一句话概括**：Python 实现的终端 AI 编程工具，支持 Git 原生集成，轻量灵活。

```bash
# 安装
pip install aider-chat

# 使用（会自动检测 Git 仓库）
aider --model deepseek/deepseek-chat

# 编辑模式
aider src/core/event_loop.cpp src/core/connection.cpp
```

**适用场景**：喜欢终端、需要精确控制 Git 变更、成本敏感的团队。

### 3.10 工具选型决策树

```
开始
 │
 ├─ 使用 CLion 作为主 IDE？
 │   ├─ 是 → GitHub Copilot（唯一选择）
 │   └─ 否 ↓
 │
 ├─ 团队用 AWS 云服务？需要结构化开发流程（需求→设计→任务→回滚）？
 │   ├─ 是 → Kiro（Spec 驱动 + AWS 集成）
 │   └─ 否 ↓
 │
 ├─ 偏好终端工作？大代码库？复杂重构？
 │   ├─ 是 → Claude Code（首选）+ 可选 Cursor
 │   └─ 否 ↓
 │
 ├─ 想要 AI IDE 全体验？
 │   ├─ 偏好自主 Agent → Windsurf（Cascade 更主动）
 │   ├─ 偏好手动控制 → Cursor（Composer 更可控）
 │   └─ 否 ↓
 │
 ├─ 预算有限？需要本地模型？
 │   ├─ 是 → Cline + DeepSeek / Ollama
 │   └─ 否 ↓
 │
 └─ 企业合规要求？
     └─ GitHub Copilot Enterprise
```

---

## 四、安装与配置实战（10 min）🔥 现场演示

### 4.1 Claude Code 安装（推荐方式）

```bash
# Linux / macOS（原生安装器，无需 Node.js）
curl -fsSL https://claude.ai/install.sh | bash

# macOS Homebrew
brew install --cask claude-code

# 验证安装
claude --version

# 首次启动
cd your-cpp-project
claude

# 浏览器 OAuth 登录 → 完成
# 公司网络限制？使用 API Key：
export ANTHROPIC_API_KEY=sk-ant-xxxxx
```

**WSL2 用户注意**：项目放在 WSL 原生文件系统（`~/projects/`），不要放在 `/mnt/c/`，否则文件监控性能极差。

### 4.2 Cursor 安装

```bash
# 1. 官网下载安装包：https://cursor.com
# 2. 安装后导入 VS Code 配置
#    Settings → General → Import VS Code Configuration

# 3. 安装 C++ 扩展
#    Ctrl+Shift+X → 搜索 "C/C++" (Microsoft) → 安装
#    或搜索 "clangd" (LLVM) → 安装（更推荐）

# 4. 配置 C++ 项目（生成 compile_commands.json）
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build
# 将 compile_commands.json 软链到项目根目录
ln -sf build/compile_commands.json .
```

### 4.3 Kiro 安装

```bash
# 1. 官网下载安装包：https://kiro.dev/downloads
#    - macOS: 拖入 Applications
#    - Windows: 双击安装程序
#    - Linux: chmod +x kiro.AppImage && ./kiro.AppImage
#      或 sudo dpkg -i kiro_*.deb

# 2. CLI 安装（可选，用于终端/CI）
sudo dpkg -i kiro_*.deb

# 3. 首次启动 → 选择登录方式
#    - GitHub（推荐个人开发者）
#    - AWS Builder ID（免费体验）
#    - AWS IAM Identity Center（企业 SSO）

# 4. 一键导入 VS Code 配置（可选）
#    Settings → Import VS Code Configuration

# 5. 配置 C++ 项目
cd your-cpp-project && kiro .
# AI Chat: Ctrl+L → 开始 Spec 驱动开发
```

### 4.4 GitHub Copilot 安装

**VS Code**：
```
Ctrl+Shift+X → 搜索 "GitHub Copilot" → 安装 → 登录 GitHub
```

**JetBrains CLion**：
```
Settings → Plugins → Marketplace → "GitHub Copilot" → 安装
→ 重启 → Tools → GitHub Copilot → Login
```

### 4.5 Cline 安装

```
VS Code → Ctrl+Shift+X → 搜索 "Cline" → 安装
→ 左下角 Cline 图标 → 选择 API Provider
→ 推荐：OpenRouter（统一接入 Claude/GPT/DeepSeek）
或直接接 DeepSeek API（成本最低）
```

**免费方案配置**（OpenRouter + DeepSeek 免费模型）：

```json
{
  "apiProvider": "openrouter",
  "openRouterApiKey": "sk-or-xxxxx",
  "openRouterModelId": "deepseek/deepseek-chat-free"
}
```

### 4.6 MCP 协议 —— AI 工具的后端瑞士军刀 ⭐

**什么是 MCP**：Model Context Protocol，让 AI 直接调用外部工具（数据库、API、文件系统等）。

```
┌──────────┐    MCP 协议    ┌──────────────┐
│ AI 模型   │ ←------------→ │ MCP Server   │
│ (Claude)  │  工具调用/响应  │ (自己写的服务) │
└──────────┘                └──────┬───────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                ┌───▼───┐    ┌────▼────┐   ┌────▼────┐
                │ Redis │    │ Oracle  │   │  ZMQ    │
                └───────┘    └─────────┘   └─────────┘
```

**对 C++ 后端团队的价值**：可以写 MCP Server 让 AI 直接：
- 查询 Redis 缓存状态 → 辅助调试
- 查询 Oracle 表结构 → 生成数据访问代码
- 调用 ZMQ 接口 → 验证消息格式
- 读取日志文件 → 定位线上问题

**Claude Code 中配置 MCP**：

```bash
# 添加 MCP Server
claude mcp add redis-inspector -- python3 ~/mcp-servers/redis_mcp.py

# 在会话中管理
/mcp    # 查看和管理所有 MCP 连接
```

---

## 五、C++ 后端开发最佳实践（10 min）

### 5.1 Prompt 工程四要素

```
┌─────────────────────────────────────────────┐
│  1. 角色定位：你是谁，AI 是什么角色           │
│     例："你是一个 C++17 后端架构师，有 10 年   │
│           分布式系统经验"                     │
│                                             │
│  2. 完整上下文：技术栈 + 约束 + 代码上下文     │
│     例："使用 C++17, CMake, hiredis, OCCI,   │
│           禁用异常，Google 代码风格"           │
│                                             │
│  3. 强制结构化输出：明确格式要求               │
│     例："按以下格式输出：                     │
│           1. 变更文件列表                     │
│           2. 每个文件的具体修改               │
│           3. 潜在风险点"                     │
│                                             │
│  4. 验收红线：明确不可逾越的底线               │
│     例："不得引入新的动态内存分配              │
│           不得修改现有接口签名                 │
│           所有网络调用必须设置超时"            │
└─────────────────────────────────────────────┘
```

### 5.2 C++ 专项 Prompt 模板

#### 模板 1：网络编程（Socket / epoll）

```
你是一个 C++17 网络编程专家。当前项目使用 epoll + 非阻塞 IO。

任务：为以下需求生成代码
- 实现一个 TCP 连接池，支持最多 1000 个并发连接
- 使用 epoll ET 模式
- 所有 IO 操作设置 30 秒超时
- 使用 std::unique_ptr 管理资源
- 错误处理使用 std::optional 返回

约束：
- C++17 标准，禁用异常
- 头文件在 include/network/，实现文件在 src/network/
- 遵循项目现有的命名约定（参见 CLAUDE.md）

输出格式：
1. 头文件声明
2. 实现文件
3. 单元测试骨架
4. 使用示例
```

#### 模板 2：数据库访问（Redis / Oracle）

```
你是一个 C++ 后端数据库专家。

任务：为 Redis 缓存层生成代码
- 使用 hiredis 异步 API
- 实现连接池（最少 5，最大 20 连接）
- 支持 String / Hash / List 操作
- 所有操作带超时重试（最多 3 次）
- 集成到现有的内存池统计系统

技术栈：C++17, hiredis, CMake

约束：
- 线程安全（使用 std::shared_mutex）
- 连接断开自动重连
- 不得阻塞主事件循环
```

#### 模板 3：分布式调试

```
我有一个 ZMQ PUB-SUB 系统，偶尔出现消息丢失。

# 相关文件
@file:src/mq/publisher.cpp
@file:src/mq/subscriber.cpp
@file:include/mq/config.h

# 现象
- 消息量 ~10K/s 时正常
- >50K/s 时随机丢失 0.1-1% 消息
- 无错误日志

# 任务
1. 分析可能的原因
2. 建议诊断步骤（添加哪些日志/指标）
3. 给出修复方案
```

### 5.3 CLAUDE.md —— 项目 AI 知识库

**这是投入产出比最高的 10 分钟**。生成后，所有 AI 工具都会遵守你的项目规范。

```bash
# Claude Code 中一键生成
/init
```

**手动优化的 CLAUDE.md 模板**（C++ 后端项目）：

```markdown
# Project: Game Server Backend

## Build
- CMake 3.20+, C++17, GCC 11+
- Build: `cmake -B build && cmake --build build -j$(nproc)`
- Test: `cd build && ctest --output-on-failure`
- Format: `clang-format -i **/*.cpp **/*.h`

## Architecture
- `src/core/` — Event loop, connection pool, memory pool
- `src/network/` — TCP/UDP abstraction, ZMQ wrappers
- `src/db/` — Redis (hiredis), Oracle (OCCI)
- `src/mq/` — ZMQ PUB-SUB, PUSH-PULL patterns
- `include/` — Public headers, one .h per module
- `tests/` — GoogleTest, mirror src/ structure

## Conventions
- Google C++ Style Guide
- snake_case functions/variables, PascalCase classes
- No exceptions (disabled via -fno-exceptions)
- Error handling: `std::optional<T>` or `absl::StatusOr<T>`
- RAII for all resources, `std::unique_ptr` preferred
- All network I/O must have timeouts
- DB connections must use the connection pool (never direct)
- Memory allocations tracked via `MemoryPool::record()`

## Constraints
- MUST NOT: new raw `new`/`delete` (use `std::make_unique`)
- MUST NOT: block the main event loop
- MUST NOT: throw exceptions
- MUST: all public APIs documented with Doxygen
- MUST: each PR include unit tests
```

### 5.4 任务粒度铁律

```
❌ 错误做法：
  "帮我实现整个用户管理系统，包括注册/登录/权限/数据库"

✅ 正确做法（拆分为 8-10 个小任务）：
  1. "实现 User 数据模型和 Oracle 表映射"
  2. "实现 Redis 用户会话缓存"
  3. "实现注册接口，包含参数验证"
  4. "实现登录接口，包含密码哈希验证"
  5. "实现 JWT Token 生成和验证"
  6. "实现权限中间件"
  7. "为以上功能编写单元测试"
  8. "生成 CMakeLists.txt 更新"
```

**原则**：一个 Prompt 控制在 150-300 行代码产出，10 个小 Prompt 远好于 1 个大 Prompt。

### 5.5 AI 编码安全守则

| 场景 | 安全措施 |
|------|---------|
| **内存操作** | AI 生成的 `new`/`delete` 必须人工审查，建议用 `make_unique` |
| **网络数据** | AI 生成的解析代码必须验证缓冲区边界 |
| **数据库查询** | AI 生成的 SQL 必须检查注入风险 |
| **多线程代码** | AI 生成的锁逻辑必须人工验证死锁风险 |
| **加密相关** | 任何加密/哈希代码必须由安全专家审查 |
| **配置解析** | AI 生成的配置文件解析必须有格式校验 |

### 5.6 AI 辅助开发流程

```
┌──────────────────────────────────────────────┐
│             AI 辅助开发全流程                  │
│                                              │
│  1. 需求分析                                 │
│     ├─ AI 辅助：分析需求文档，提取技术要点     │
│     └─ 人工：确认需求理解正确                 │
│                                              │
│  2. 方案设计                                 │
│     ├─ AI 辅助：生成架构方案，对比优劣         │
│     └─ 人工：选择方案，审核可行性             │
│                                              │
│  3. 编码实现                                 │
│     ├─ AI 辅助：生成代码框架/样板/测试         │
│     └─ 人工：核心逻辑，代码审查               │
│                                              │
│  4. 代码审查                                 │
│     ├─ AI 辅助：第一轮自动化审查               │
│     │   - /review (Claude Code)              │
│     │   - 内存安全 / 并发 / 性能扫描          │
│     └─ 人工：确认 AI 发现的问题，补充审查      │
│                                              │
│  5. 测试                                     │
│     ├─ AI 辅助：生成测试用例，边界条件覆盖     │
│     └─ 人工：确认测试覆盖率，补充业务场景     │
│                                              │
│  6. 调试                                     │
│     ├─ AI 辅助：分析 coredump / 日志 /        │
│     │   valgrind 输出                        │
│     └─ 人工：确认根因，实施修复               │
│                                              │
│  🚨 核心原则：人类始终是最终责任人             │
│     AI 是"超级实习生"，你是"架构师+Reviewer"   │
└──────────────────────────────────────────────┘
```

---

## 六、总结与推荐方案（5 min）

### 6.1 C++ 后端团队推荐工具组合

| 角色 | 推荐工具 | 预估成本 |
|------|---------|---------|
| **全员标配** | GitHub Copilot Pro | $10/人/月 |
| **核心开发** | Claude Code (Max 5x) + Copilot | $110/人/月 |
| **架构师/Tech Lead** | Claude Code (Max 20x) + Cursor Pro | $220/人/月 |
| **大型 C++ monorepo** | Windsurf Pro ($15) + Claude Code | $115/人/月 |
| **AWS 生态团队** | Kiro + Claude Code | $20-100/人/月 |
| **预算敏感** | Cline + DeepSeek API | $5-20/人/月 |

### 6.2 落地路线图

```
第 1 周：
  ✅ 全员安装 GitHub Copilot
  ✅ 在 1-2 个项目试用 Claude Code
  ✅ 生成项目 CLAUDE.md

第 2 周：
  ✅ 学习 Prompt 工程基础
  ✅ 建立 C++ 专项 Prompt 模板库
  ✅ 团队 Code Review 加入 AI 辅助

第 3-4 周：
  ✅ 核心开发者切换到 Cursor / Claude Code / Kiro
  ✅ 建立 AI 编码规范（.cursorrules / .kiro/steering / CLAUDE.md 纳入 Git）
  ✅ 评估 MCP Server 需求（Redis / Oracle / ZMQ 调试工具）

第 2 个月：
  ✅ 全面推广 AI 辅助开发流程
  ✅ 建立 AI 代码质量度量体系
  ✅ 探索 CI/CD 集成（AI 自动 Code Review）
```

### 6.3 关键要点回顾

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  1. 模型选型：复杂任务用 Claude/GPT，             │
│     日常用 DeepSeek/Gemini，分层省钱              │
│                                                 │
│  2. 工具选型：Claude Code（终端大项目）+          │
│     Cursor（日常 IDE）+ Copilot（团队标配）        │
│                                                 │
│  3. 项目规范：花 10 分钟写好 CLAUDE.md，           │
│     之后的每一次 AI 交互都受益                    │
│                                                 │
│  4. 小步快跑：每次 150-300 行，10 个小任务        │
│     远好于 1 个大任务                            │
│                                                 │
│  5. 人类兜底：AI 是"超级实习生"，                  │
│     你是"架构师+Reviewer"，最终责任永远在人类      │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 6.4 推荐资源

| 资源 | 链接 |
|------|------|
| Claude Code 官方文档 | https://support.claude.com |
| Cursor 官方文档 | https://docs.cursor.com |
| GitHub Copilot 文档 | https://docs.github.com/copilot |
| Cline (开源) | https://github.com/cline/cline |
| Aider (开源) | https://github.com/Aider-AI/aider |
| SWE-bench 排行榜 | https://www.swebench.com |
| LLM 价格对比 | https://www.morphllm.com/llm-cost-calculator |
| OpenRouter (多模型接入) | https://openrouter.ai |
| **成本优化实战指南** | `成本优化实战指南.md`（同目录） |
| **本地离线 AI 部署方案** | `本地离线AI部署方案.md`（同目录） |
| **C++ AI 编码反模式与陷阱** | `C++-AI编码反模式与陷阱.md`（同目录） |
| **团队 Prompt 模板库** | `团队Prompt模板库.md`（同目录） |
| **CI/CD AI 集成方案** | `CI-CD-AI集成方案.md`（同目录） |
| **MCP Server 开发入门** | `MCP-Server开发入门.md`（同目录） |
| **CppCon 2025 AI+C++ 演讲汇总** | `CppCon-2025-AI-C++-演讲汇总.md`（同目录） |
| **Claude Code 源码架构分析** | `Claude-Code-源码分析汇总.md`（同目录） |
| **前沿研究方向与行业动态** | `前沿研究方向与行业动态.md`（同目录） |

---

## 附录：CppCon 2025 AI + C++ 演讲精华 ⭐

> 详见同目录文件 **`CppCon-2025-AI-C++-演讲汇总.md`**（7 场 AI 相关演讲 + Panel 完整汇总）

### CppCon 2025 的行业共识

2025 年的 CppCon 上，AI 演讲密度前所未有。整个 C++ 社区已经达成共识：

| 共识 | 详情 |
|------|------|
| **AI 不是可选项** | Daisy Hollman (Anthropic) 主题演讲："问题已从'为什么用'变成'怎么用'" |
| **开发者角色转变** | 从"代码写手"→"代码雕刻家"，判断力比打字速度更重要 |
| **验证门槛更高** | Jason Turner：AI 会复现互联网上的劣质 C++ 代码，Sanitizer 是底线 |
| **调试是下一个前沿** | Undo + Anthropic 联合演示时间旅行调试 + AI Agent |
| **规模化已验证** | Meta：百万行级 C++ 代码库的 LLM 驱动重构流水线已在生产运行 |

### CppCon 金句

> "When I agreed to give this talk, the narrative involved *why you should be using agents*. Since then, the world has evolved such that the interesting question is no longer 'why' but 'how'."
> — **Daisy Hollman**, Anthropic (Claude Code 团队), CppCon 2025 Keynote

> "AI tools simply regurgitate what they find on the internet. The internet is full of terrible examples of outdated memory leaks, undefined behavior, and worse. How do we safely use these tools while ensuring good code quality?"
> — **Jason Turner**, C++Weekly 主持人, CppCon 2025

> "85% of engineering is debugging, not writing new code. Yet most AI investment has gone into code generation. We're filling the gap where engineers spend most of their time."
> — **Mark Williamson**, Undo CTO, CppCon 2025

> "Build with CMake + Ninja, run `ctest`. Automation is your safety net."
> — **Jason Turner**, CppCon 2025

---

## Q&A —— 常见问题预案

**Q1：公司代码不能上传外网怎么办？**
> A：使用 Cline + Ollama 本地部署 DeepSeek V4 或 Qwen3-Coder，完全离线。或申请公司级 API 代理 + 数据脱敏。

**Q2：AI 生成的 C++ 代码质量如何？**
> A：模板/框架代码正确率 >90%，复杂算法约 70-80%。关键是人工审查 + 编译 + 测试三道防线。

**Q3：Claude Code 的 CLAUDE.md 会不会泄露项目信息？**
> A：CLAUDE.md 只在本地存储，随每次 API 调用发送。如果担心，可以：
> - 不放入敏感信息（IP、密钥等）
> - 使用 `.claude/local.md`（加入 .gitignore）放个人偏好
> - 企业版支持数据不用于训练

**Q4：你们团队实际用了多久适应？**
> A：1-2 周习惯基本操作，1 个月形成有效工作流，2 个月后效率提升明显。

**Q5：工具选太多会不会增加认知负担？**
> A：建议从 1 个开始（推荐 Copilot，最无感），2 周后加入第二个（Claude Code 或 Cursor），逐步扩展。

**Q6：CppCon 上的人怎么看 AI + C++？**
> A：2025 年 CppCon 上 AI 是最大热点之一。社区共识：AI 工具必须用，但 C++ 的特殊性（内存安全、性能约束）意味着验证门槛更高。Daisy Hollman（Anthropic）的主题演讲指出：开发者的角色从"写代码"转变为"审代码"，判断力比打字速度更重要。详见附录 `CppCon-2025-AI-C++-演讲汇总.md`。

---

> 📝 **演讲备注**  
> - 第三章（工具详解）建议穿插 2-3 个现场演示  
> - 现场演示 1：Claude Code 修复一个 C++ 编译错误  
> - 现场演示 2：Cursor Composer 生成 Redis 连接池代码  
> - 第五章（最佳实践）建议结合实际项目代码演示 Prompt 技巧  
> - 准备一个 Demo 项目，提前配好环境，避免现场翻车
> - **补充素材**：可引用 CppCon 2025 演讲中的金句增加说服力（附录文件已整理）
> - **进阶学习**：推荐团队成员观看 CppCon 2025 的 Daisy Hollman Keynote 和 Jason Turner 演讲视频
> - **前沿视野**：`前沿研究方向与行业动态.md` 涵盖 SWE-bench 最新突破、Agent 架构创新、C++ 形式化验证、AaaS 范式变革等，适合 Tech Lead 阅读

---

*文档生成日期：2026-06-23 | 数据来源：Anthropic、OpenAI、Google、DeepSeek 官方文档及 morphllm.com、futureagi.com 等独立评测平台*
