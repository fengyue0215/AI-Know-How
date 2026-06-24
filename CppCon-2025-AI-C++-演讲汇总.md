# CppCon 2025 AI + C++ 演讲汇总

> CppCon 2025 于 2025 年 9 月 15-18 日在美国科罗拉多州奥罗拉举行。  
> 本文件汇总了所有与 AI/C++ 开发相关的演讲，提炼核心观点供团队学习参考。  
> 视频已陆续在 CppCon YouTube 频道和 Class Central 免费发布。

---

## 📋 演讲清单

| # | 演讲者 | 所属 | 标题 | 类型 | 日期 |
|---|--------|------|------|------|------|
| 1 | Daisy Hollman | Anthropic (Claude Code) | Crafting the Code You Don't Write: Sculpting Software in an AI World | Keynote | 9/16 |
| 2 | Jason Turner | C++Weekly | Best Practices for AI Tool Use in C++ | Talk | 9/17 |
| 3 | Ion Todirel | Microsoft | LLMs in the Trenches: Boosting C++ System Programming with AI | Talk | 9/17 |
| 4 | Jubin Chheda | Meta | Refactoring at Scale: LLM-Powered Pipelines for Large C++ Codebases | Talk | 9/16 |
| 5 | Alexandra Kemper | Microsoft | What's New for VS Code: Performance, Debugging, & GitHub Copilot Agents | Talk | 9/16 |
| 6 | Mark Williamson + Daisy Hollman | Undo + Anthropic | Agentic C++ Debugging Live! – Without a Safety Net | Live Demo | 9/18 |
| 7 | 多人 Panel | Anthropic + Microsoft + 社区 | What We Learned About AI Tools For C++ Engineers | Panel | 9/16 |

---

## 一、Daisy Hollman — Keynote：在 AI 时代雕刻软件

> **演讲者**：Daisy Hollman，Anthropic 杰出工程师，C++ 标准委员会成员（2016 年至今），Claude Code 团队成员  
> **视频**：[Class Central](https://www.classcentral.com/course/youtube-crafting-the-code-you-don-t-write-sculpting-software-in-an-ai-world-daisy-hollman-cppcon-2025-515704) | [Undo.io](https://undo.io/all-types/videos/crafting-the-code-you-dont-write-sculpting-software-in-an-ai-world-daisy-hollman-cppcon-2025/)

### 核心观点

```
┌─────────────────────────────────────────────────────────────┐
│  软件开发者的角色正在从"代码写手"转变为"代码雕刻家"           │
│                                                             │
│  类比：雕塑家面对电动工具的出现                               │
│       画家面对摄影术的发明                                   │
│                                                             │
│  AI 不会取代工匠 —— 它改变了工匠的工作方式                    │
│  你的价值不再是你写了多少行代码，而是你判断什么代码值得保留   │
└─────────────────────────────────────────────────────────────┘
```

### 关键洞察

1. **问题已经从"为什么"变成"怎么做"**
   > "When I agreed to give this talk, the narrative involved *why you should be using agents*. Since then, the world has evolved such that the interesting question is no longer 'why' but 'how'."
   - 6 个月内，Agentic Coding 从概念验证进化到几乎自主的软件工程师

2. **软件工艺的价值不降反升**
   | 变得更重要 | 变得不那么重要 |
   |-----------|--------------|
   | 架构思维 | 逐行编写代码 |
   | Code Review 和判断力 | 记住 API 细节 |
   | 理解正确性和性能取舍 | 编写样板代码 |
   | 知道什么代码**不该接受** | 从零实现数据结构 |

3. **C++ 的特殊挑战**
   > "How do we leverage these tools where correctness and performance are non-negotiable?"
   - C++ 的容错率远低于脚本语言
   - 内存安全 + 性能 + 严格正确性 = 更高的验证门槛

4. **安全特性必须内置到工具中** —— 就像电动工具的安全防护装置，AI 编码 Agent 需要防护栏（Plan Mode、Diff Review、自动化测试）

### 🔗 与本次分享的关联

- 对应主文档 **第一章 1.1-1.2**：为什么 C++ 后端开发者需要 AI
- 对应主文档 **第五章 5.5**：AI 编码安全守则
- 对应主文档 **第六章 6.3**：关键词回顾——"人类兜底"
- **强化观点**：在 C++ 领域，AI 工具的使用不仅是效率问题，更是**安全性工程**问题

---

## 二、Jason Turner — C++ 中使用 AI 工具的最佳实践

> **演讲者**：Jason Turner，C++Weekly 主持人，CppCast 联合创始人  
> **视频**：[Class Central](https://www.classcentral.com/course/youtube-best-practices-for-ai-tool-use-in-c-jason-turner-cppcon-2025-520699)

### 核心警告

```
┌─────────────────────────────────────────────────────────────┐
│  ⚠️ AI/LLM 会复现互联网上的劣质 C++ 代码                     │
│                                                             │
│  互联网上充斥着：                                             │
│  - 过时的内存管理模式（裸 new/delete）                        │
│  - 未定义行为                                                  │
│  - 资源泄漏                                                   │
│  - 不完整的错误处理                                            │
│                                                             │
│  AI 不会区分好代码和坏代码 —— 它会从两者的混合中"平均"输出     │
└─────────────────────────────────────────────────────────────┘
```

### 安全实践框架

| 原则 | 具体做法 |
|------|---------|
| **绝不盲信** | AI 生成的每一行代码都必须通过编译 + 静态分析 + 测试 |
| **类型安全第一** | 用强类型和 RAII 让错误代码无法编译（`std::unique_ptr` 替代裸指针，`[[nodiscard]]` 强制检查返回值） |
| **工具链兜底** | CMake + Ninja + `ctest` 是必须的自动化安全网 |
| **Sanitizer 必跑** | AddressSanitizer / UndefinedBehaviorSanitizer 对 AI 生成代码必跑 |
| **小步验证** | 每次只接受小批量 AI 生成代码，立即编译测试，不要积攒大 PR |

### 金句

> "Build with CMake + Ninja, run `ctest`. Automation is your safety net. The absolute minimum default."

### 🔗 与本次分享的关联

- 对应主文档 **第五章 5.4**：任务粒度铁律
- 对应主文档 **第五章 5.5**：AI 编码安全守则表格
- **强化观点**：AI 工具在 C++ 中的使用需要比脚本语言多一层安全验证

---

## 三、Ion Todirel — AI 实战：提升 C++ 系统编程效率

> **演讲者**：Ion Todirel，Microsoft Lead Engineering Manager  
> **视频**：[Class Central](https://www.classcentral.com/course/youtube-llms-in-the-trenches-boosting-c-system-programming-with-ai-ion-todirel-cppcon-2025-522182)

### 演讲主题覆盖

```
┌─────────────────────────────────────────────────────────────┐
│  LLM 在 C++ 系统编程中的应用场景                              │
│                                                             │
│  ✅ 性能调优 —— 分析热点代码，建议优化策略                     │
│  ✅ 底层调试 —— 分析 coredump、汇编级别的异常                 │
│  ✅ 重构 —— 识别代码异味，建议结构化改进                      │
│  ✅ 代码生成 —— 模板代码、接口封装、测试用例                   │
│  ⚠️ 架构决策 —— 仍需人类判断                                 │
│  ⚠️ 安全关键代码 —— 不可依赖 AI                              │
└─────────────────────────────────────────────────────────────┘
```

### 跨平台实战要点

- 示例覆盖 **Linux 和 Windows** 两个平台
- 真实的 C++ 系统编程场景（不是 toy example）
- 包含 Windows 特定的调试技巧和 Linux perf 工具结合 AI 分析

### AI 擅长 vs 人类必须做的

| AI 擅长（可以放心用） | 人类必须做（AI 辅助但不替代） |
|----------------------|---------------------------|
| 识别重复代码模式 | 最终架构决策 |
| 生成 CRUD/样板代码 | 理解复杂业务领域逻辑 |
| 建议可能的优化方向 | 确认性能优化的正确性 |
| 生成测试用例骨架 | 安全关键代码审查 |
| 格式化和风格统一 | 权衡取舍（性能 vs 可维护性） |
| 翻译代码（C++11→C++17） | 定义编码规范和红线 |

### 🔗 与本次分享的关联

- 对应主文档 **第三章工具详解**：Claude Code / Cursor 实战
- 对应主文档 **第五章 5.2**：C++ 专项 Prompt 模板（网络/数据库/调试）
- 对应主文档 **第五章 5.6**：AI 辅助开发全流程
- **强化观点**：系统编程是 AI 最能产生价值的领域之一（因为是模式密集型的），但也是风险最高的领域之一

---

## 四、Jubin Chheda (Meta) — 大规模 LLM 驱动的 C++ 重构流水线

> **演讲者**：Jubin Chheda，Meta Software Engineer（负责 Facebook Feed）  
> **议题页**：[CppCon 2025 Schedule](https://cppcon2025.sched.com/event/27bOi/directory/)

### 核心架构

```
                    LLM 驱动的重构流水线
                    
  ┌──────────┐    ┌──────────────┐    ┌──────────────┐
  │ 静态分析   │───→│ LLM Prompt   │───→│ 自动化 PR    │
  │ (Lint/    │    │ (代码异味     │    │ (建议修复    │
  │  Clang)   │    │  + 建议修复)   │    │  + 测试)     │
  └──────────┘    └──────────────┘    └──────────────┘
       ↑                                      │
       └──────── 人工审查（Guardrail）←────────┘
```

### 演讲亮点

1. **静态分析信号 + LLM Prompt 结合** —— 不是简单地"让 AI 看代码"，而是先用 Clang/静态分析工具找出问题区域，再让 LLM 针对具体问题生成修复
2. **现代化改造**：`std::unique_ptr` 替代裸指针、ranges 替代传统循环、coroutines 替代回调
3. **测试加固**：AI 为高风险代码区域生成/扩展单元测试
4. **CI 集成**：可作为 nightly pipeline 或按需触发的 on-demand 任务
5. **效果度量**：缺陷密度变化、测试覆盖率提升、ROI 评估

### 防护栏设计

```
防护栏（Guardrail）机制：
├─ 编译检查 —— 生成的代码必须通过编译
├─ 测试验证 —— 现有测试一个不能少
├─ 静态分析 —— 不能引入新的 lint 警告
├─ Diff Review —— 人类最终确认
└─ 灰度发布 —— 先在小范围验证，再扩到整个代码库
```

### 🔗 与本次分享的关联

- 对应主文档 **第五章 5.6**：AI 辅助开发全流程中的 CI/CD 集成
- 对应主文档 **第六章 6.2**：落地路线图第 2 个月——CI/CD 集成
- **新增启示**：对于大型 C++ 项目，AI + 静态分析 + CI 是一条经过 Meta 验证的可行路径

---

## 五、Alexandra Kemper (Microsoft) — VS Code 的 AI 增强

> **演讲者**：Alexandra Kemper，Microsoft  
> **议题页**：[isocpp.org](https://isocpp.org/blog/2025/08/cppcon-2025-whats-new-for-vs-code-perf.-debugging-github-copilot-agents-ale)

### 核心内容

1. **GitHub Copilot Agent Mode** —— 处理多步骤复杂 C++ 编码任务
   - 现代化 C++ 仓库
   - 自动理解代码库结构，跨文件编辑
2. **LLDB 调试改进** —— VS Code 中的 C++ 调试体验提升
3. **CMake 语言服务** —— 智能补全、错误提示、跳转定义

### 🔗 与本次分享的关联

- 对应主文档 **第三章 3.5（原 3.6）**：GitHub Copilot
- 对应主文档 **第四章 4.4**：Copilot 安装配置
- **补充信息**：VS Code 的 C++ 工具链在快速进化，Agent Mode 不再是 Cursor 的独有功能

---

## 六、Mark Williamson + Daisy Hollman — AI 时间旅行调试

> **演讲者**：Mark Williamson（Undo CTO）+ Daisy Hollman（Anthropic）  
> **视频**：[Class Central](https://www.classcentral.com/course/youtube-agentic-c-debugging-live-without-a-safety-net-daisy-hollman-mark-williamson-cppcon-2025-521318)

### 核心概念：Agentic Debugging = TTD + AI Agent

```
┌─────────────────────────────────────────────────────────────┐
│  时间旅行调试（TTD）+ AI Agent = 确定性根因分析              │
│                                                             │
│  1. TTD 录制程序完整执行过程（每条指令/线程/变量/IO）         │
│  2. AI Agent 拥有回放控制权                                   │
│  3. Agent 从故障点"逆流而上"追踪因果链                        │
│  4. 所有分析基于确定性录制 —— 消除幻觉                        │
│                                                             │
│  关键洞察：以确定性运行录制作为"真相源"，AI 不需猜测          │
└─────────────────────────────────────────────────────────────┘
```

### MCP 工具集设计哲学 —— "少即是多"

只有 **8-10 个反向操作**：

| 工具 | 功能 |
|------|------|
| `reverse_step_instruction` | 反向执行一条指令 |
| `reverse_step_line` | 反向执行一行代码 |
| `reverse_finish_function` | "反调用"一个函数 |
| `get_value(variable_name)` | 检查当前变量值 |
| `last_value(variable_name)` | 追踪变量最后变化点 |
| `get_source_code` | 获取当前位置源码 |
| `get_stack_trace` | 获取当前调用栈 |

**为什么只用反向操作？** —— 强制 Agent 走科学路径：从观测到的故障（果）逆推根本原因（因），消除猜测和前向探索。

### 实战演示 —— Doom 游戏调试

以 Doom 为例演示：
> 提问："第二个僵尸是什么时候被杀死的？"

Agent 执行过程：
1. 翻译人类概念为代码：`players[0].kill_count`
2. 反向迭代录制直到 kill_count == 2
3. 返回精确时间点（游戏 tick + 实际时间 + 源码位置）
4. 生成可检查的书签供人类验证

### 性能开销

| 指标 | 开销 |
|------|------|
| CPU 密集型 | 2× - 5× 减速 |
| 内存 | ~2× 额外开销 |
| 录制数据量 | 数 MB/秒 |
| 适用场景 | 后验分析、非延迟敏感生产环境 |

### 核心洞察

> "85% 的工程时间花在理解和修复代码上，而非编写新代码。但迄今为止的 AI 投资几乎全在代码生成上。这个项目填补了工程师花费最多时间的环节。"

### 🔗 与本次分享的关联

- 对应主文档 **第五章 5.6**：AI 辅助开发全流程中的调试阶段
- 对应主文档 **第四章 4.6**：MCP 协议
- 对应主文档现场演示脚本 **Demo 3**：AI 辅助调试
- **新增启示**：AI + 确定性录制是 C++ 调试的未来方向，值得长期关注

---

## 七、Panel 讨论 — AI 工具在 C++ 中的经验与教训

> **Panel 嘉宾**：Daisy Hollman (Anthropic) · Inbal Levi (Microsoft) · Jason Turner · Matt Godbolt · Michael Wong  
> **主持人**：Guy Davidson  
> **议题页**：[CppCon 2025 Schedule](https://cppcon2025.sched.com/event/28Pda/)

### 讨论话题

- **Claude / Copilot / Gemini / Intellicode** 各自的优劣势
- C++ 工程师的真实工作流改进
- AI 工具的短板和不足
- 未来发展方向

### 社区共识（从 Panel 讨论提炼）

```
关于 AI + C++ 开发的 2025 共识：

✅ 已经达成的共识：
  - AI 补全/生成对 C++ 开发确实有生产力提升
  - AI 审查（Code Review）作为第一道防线可行
  - CLAUDE.md / .cursorrules 能显著改善 AI 的上下文理解
  - "不信任但要验证"是所有工具使用的前提

⚠️ 仍在探索的：
  - AI 全自动生成生产级 C++ 代码的安全性保障
  - 如何量化 AI 带来的代码质量变化
  - C++ 特有的 AI 训练数据集质量提升
  - AI 对初级 C++ 开发者的学习路径影响

❌ 明确的反对：
  - 让 AI 独立 Merge 代码（所有嘉宾一致反对）
  - 完全跳过人工 Code Review
  - 不跑 Sanitizer 就信任 AI 生成的代码
```

### 🔗 与本次分享的关联

- 整体对应主文档的全部章节
- **核心共识与本次分享的"人类兜底"原则完全一致**

---

## 📊 CppCon 2025 AI 演讲主题聚类

```
                    CppCon 2025 AI + C++ 全景

  代码生成与重构              调试与诊断            工具与平台
  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
  │ Daisy Hollman │      │ Williamson + │      │ A. Kemper    │
  │ (雕刻软件)    │      │ Hollman      │      │ (VS Code +   │
  │              │      │ (TTD+Agent)  │      │  Copilot)    │
  │ J. Chheda    │      │              │      │              │
  │ (Meta 重构)  │      │ Ion Todirel  │      │ Panel        │
  │              │      │ (LLM 调试)   │      │ (工具对比)   │
  │ Jason Turner │      │              │      │              │
  │ (安全实践)   │      │              │      │              │
  └──────────────┘      └──────────────┘      └──────────────┘
```

---

## 🎯 对 C++ 后端团队的核心启示

### 1. 行业共识已形成
CppCon 2025 的 AI 演讲密度前所未有。整个 C++ 社区已经确认：AI 不是可选项，是必选项。

### 2. C++ 的特殊要求
- **验证门槛高于其他语言**：内存安全 + 未定义行为 + 性能约束
- **Sanitizer 是底线**：AI 生成代码必须经过 AddressSanitizer / UBSan
- **类型系统是安全网**：善用 `std::unique_ptr` / `[[nodiscard]]` / `std::optional`

### 3. 从"写代码"到"审代码"
- 你的主要工作不再是写出每一行代码，而是判断 AI 生成的代码是否正确
- Code Review 能力成为比编码速度更重要的技能

### 4. AI 调试是下一个前沿
- 时间旅行调试 + AI Agent 可能彻底改变 C++ 调试方式
- MCP 协议是连接 AI Agent 和 C++ 工具链的关键桥梁

### 5. 立即可以做的三件事
1. **建立验证流水线**：编译 + Sanitizer + 测试，自动化运行
2. **写项目规范文件**：CLAUDE.md / .cursorrules / .kiro/steering，投入 10 分钟，每次交互受益
3. **小步快跑**：每次只让 AI 处理 1-2 个函数 / 一个完整的垂直切片

---

## 📺 视频资源索引

| 演讲 | 视频链接 |
|------|---------|
| Daisy Hollman Keynote | [Class Central](https://www.classcentral.com/course/youtube-crafting-the-code-you-don-t-write-sculpting-software-in-an-ai-world-daisy-hollman-cppcon-2025-515704) |
| Jason Turner | [Class Central](https://www.classcentral.com/course/youtube-best-practices-for-ai-tool-use-in-c-jason-turner-cppcon-2025-520699) |
| Ion Todirel | [Class Central](https://www.classcentral.com/course/youtube-llms-in-the-trenches-boosting-c-system-programming-with-ai-ion-todirel-cppcon-2025-522182) |
| Williamson + Hollman | [Class Central](https://www.classcentral.com/course/youtube-agentic-c-debugging-live-without-a-safety-net-daisy-hollman-mark-williamson-cppcon-2025-521318) |
| CppCon 官方 YouTube | [youtube.com/@CppCon](https://youtube.com/@CppCon) |

---

*文档生成日期：2026-06-23 | 数据来源：isocpp.org、CppCon 2025 Schedule、Class Central、Undo.io*
