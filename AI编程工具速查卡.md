# AI 编程工具速查卡 🚀

> 打印或分享给团队，一页看完核心信息

---

## 🎯 一分钟选型

| 你的情况 | 推荐工具 | 一句话理由 |
|---------|---------|-----------|
| 用 CLion | **GitHub Copilot** | 唯一支持 CLion 的主流工具 |
| 终端党 / 大项目重构 | **Claude Code** | 最强 Agent，文件系统级理解 |
| 想要 AI IDE | **Cursor** | VS Code 增强版，补全最准 |
| 大代码库 + 自主 Agent | **Windsurf** | Cascade 更主动，跨文件感知强 |
| AWS 生态 + 结构化流程 | **Kiro** | Spec 驱动开发，原子回滚 |
| 预算有限 | **Cline + DeepSeek** | 免费开源 + 最便宜模型 |
| 企业合规要求 | **Copilot Enterprise** | SSO + 审计 + IP 保障 |

---

## 📦 快速安装

```bash
# Claude Code（推荐）
curl -fsSL https://claude.ai/install.sh | bash
claude                        # 启动，浏览器登录

# Aider（预算友好备选）
pip install aider-chat
aider --model deepseek/deepseek-chat

# Cursor：https://cursor.com 下载安装包
# Kiro：https://kiro.dev/downloads 下载安装包
# Copilot：VS Code/CLion 扩展商店搜索安装
# Cline：VS Code 扩展商店搜索安装
```

---

## 🧠 模型分层策略

```
高复杂度（架构/多文件重构）
  → Claude Opus 4.8 / GPT-5.5
  → $0.50-2.00/次

中复杂度（单文件实现/调试）
  → Gemini 3.1 Pro / DeepSeek V4 Pro
  → $0.05-0.20/次

低复杂度（补全/格式化）
  → DeepSeek V4 Flash / 免费模型
  → <$0.01/次
```

---

## ✍️ C++ Prompt 黄金模板

```
你是一个 C++17 [领域] 专家。当前项目使用 [技术栈]。

任务：[一句话描述]
约束：
- C++17 标准，[编译选项]
- [其他约束]

输出格式：
1. [期望的输出结构]
```

---

## 🔑 核心快捷键

| 工具 | 快捷键 | 功能 |
|------|--------|------|
| **Claude Code** | `/init` | 生成项目知识文件 |
| | `/review` | Code Review |
| | `/cost` | 查看用量 |
| **Cursor** | `Ctrl+K` | 行内编辑 |
| | `Ctrl+L` | AI Chat |
| | `Ctrl+I` | Composer（多文件编辑） |
| | `Ctrl+Shift+I` | Agent Mode |
| **Kiro** | `Ctrl+L` | AI Chat |
| | `Shift+Tab` | Plan 模式切换 |
| | `@文件名` | 引用文件 |
| **Copilot** | `Tab` | 接受补全 |
| | `Ctrl+I` | 行内 Chat |
| | `Ctrl+Shift+I` | Chat 面板 |

---

## ⚠️ 安全红线

| 场景 | 必须人工审查 |
|------|------------|
| `new`/`delete` | 改用 `make_unique`/`make_shared` |
| 网络数据解析 | 检查缓冲区边界 |
| SQL 拼接 | 检查注入风险 |
| 多线程锁 | 验证死锁可能性 |
| 加密/哈希 | 安全专家审查 |
| 外部输入解析 | 格式校验 + 模糊测试 |

---

## 📋 落地检查清单

- [ ] 安装 1-2 个 AI 编程工具
- [ ] 运行 `/init` 生成项目 CLAUDE.md
- [ ] 为项目编写 `.cursorrules`
- [ ] 建立团队 Prompt 模板库
- [ ] Code Review 流程加入 AI 审查
- [ ] 设置 API 用量预算和告警

---

## 🎓 CppCon 2025 AI + C++ 关键共识

| 演讲者 | 核心观点 |
|--------|---------|
| **Daisy Hollman** (Anthropic) | 开发者从"写代码"转为"审代码"，判断力 > 打字速度 |
| **Jason Turner** | AI 会复现劣质 C++ 代码，Sanitizer + 测试是底线 |
| **Ion Todirel** (Microsoft) | LLM 擅长模式识别/样板代码，架构决策仍需人类 |
| **Meta** | 百万行级 C++ 代码库的 LLM 重构流水线已投产 |

> 完整 7 场演讲汇总见：`CppCon-2025-AI-C++-演讲汇总.md`

---

## 🔬 Claude Code 架构速览

| 设计要点 | 对你的价值 |
|---------|-----------|
| 极简主循环（无分类器/规划器） | 模型自主决策，精准 Prompt 是关键 |
| Fork Agent 共享 Cache（~95% 节省） | 并行子任务几乎不增加 token 成本 |
| 4 层上下文压缩 | 长会话自动压缩，关键信息提前写入记忆 |
| Stable 前缀 = Cache 命中 | CLAUDE.md 质量决定成本和 AI 行为一致性 |

> 完整源码分析见：`Claude-Code-源码分析汇总.md`

---

## 🔮 2026 前沿动向速览

| 前沿方向 | 核心发现 | 对你的意义 |
|---------|---------|-----------|
| **专用探索 Agent** | token -60%，准确率 +5.5% | 大项目先搜后修，不要边搜边修 |
| **自进化 Agent** | 3 轮自进化提升明显 | 未来的 AI 工具会越用越好 |
| **过程纪律** | Plan Mode 提升 17% 正确率 | 坚持先规划再编码 |
| **形式化验证 C++** | AI 生成 + 数学证明 | 安全关键代码有望获得保证 |
| **AaaS 范式** | 软件 = Agent，人类 = 意图架构师 | 后端服务需设计为 Agent 可调用的工具 |

> 完整研究汇总见：`前沿研究方向与行业动态.md`

---

*2026-06-23 | 配套《AI 编程 Know-How 演讲内容》使用*
