# 本地离线 AI 部署方案 —— 代码不出内网的 AI 编程

> 适用于代码安全要求高、无法使用云 API 的企业内网环境。  
> 全部工具开源，完全离线可用，零 API 费用。

---

## 🎯 一、方案概览

```
┌──────────────────────────────────────────────┐
│              离线 AI 编程架构                  │
│                                              │
│   VS Code / Cursor / CLion                   │
│         ↓                                    │
│   Continue 插件 / Cline 插件                  │
│         ↓                                    │
│   Ollama 本地服务 (localhost:11434)           │
│         ↓                                    │
│   DeepSeek-Coder / CodeQwen / Llama          │
└──────────────────────────────────────────────┘
```

---

## 🖥️ 二、硬件需求

| 模型 | 参数规模 | 最低显存 | 推荐显存 | 适用场景 |
|------|---------|---------|---------|---------|
| DeepSeek-Coder V2 Lite | 16B | 8GB | 16GB | 日常 C++ 补全 |
| DeepSeek-Coder V2 | 21B | 16GB | 24GB | 复杂代码生成 |
| CodeQwen 2.5-Coder | 7B | 6GB | 8GB | 轻量补全（速度快） |
| CodeQwen 2.5-Coder | 14B | 12GB | 16GB | 中等复杂度 |
| Llama 4 Scout (编程优化版) | 17B | 12GB | 16GB | 通用+编码 |

**推荐配置**：
- **个人开发机**：RTX 3060 12GB + DeepSeek-Coder V2 Lite 16B
- **团队共享**：RTX 4090 24GB + DeepSeek-Coder V2 21B
- **预算方案**：RTX 2060 6GB + CodeQwen 2.5-Coder 7B

> 量化模型（Q4_K_M）可将显存需求降低约 40-60%，质量损失 <3%。

---

## 📦 三、环境搭建

### 步骤 1：安装 Ollama

```bash
# Linux
curl -fsSL https://ollama.com/install.sh | sh

# macOS
brew install ollama

# Windows
# 从 https://ollama.com/download 下载 .exe 安装包
# 需要先安装 Visual Studio Build Tools（C++ 桌面开发工作负载）
```

启动服务：
```bash
ollama serve    # 默认监听 localhost:11434
```

### 步骤 2：拉取模型

```bash
# 推荐：DeepSeek-Coder V2 Lite（16B，平衡性能和质量）
ollama pull deepseek-coder-v2:16b

# 轻量：CodeQwen 2.5-Coder（7B，速度优先）
ollama pull codeqwen:7b

# 最强但需要大显存：DeepSeek-Coder V2（21B）
ollama pull deepseek-coder-v2:21b

# 验证安装
ollama list
```

### 步骤 3：测试模型

```bash
ollama run deepseek-coder-v2:16b
>>> 用 C++17 写一个线程安全的 LRU 缓存
```

---

## 🔌 四、IDE 集成

### 方案 A：VS Code + Continue 插件（推荐）

1. 安装 **Continue** 扩展（`Ctrl+Shift+X` → 搜索 "Continue"）
2. 配置 `~/.continue/config.json`：

```json
{
  "models": [
    {
      "title": "DeepSeek-Coder 16B",
      "provider": "ollama",
      "model": "deepseek-coder-v2:16b",
      "apiBase": "http://localhost:11434"
    }
  ],
  "tabAutocompleteModel": {
    "title": "CodeQwen 7B (Autocomplete)",
    "provider": "ollama",
    "model": "codeqwen:7b",
    "apiBase": "http://localhost:11434"
  },
  "systemMessage": "你是一个专业的 C++17 后端开发助手。优先使用 Modern C++、RAII、智能指针。注重性能和内存安全。回答用中文。"
}
```

> ⚠️ 坑：在 Continue 面板中**直接选择模型**，不要点 "Connect" 按钮，否则会 404。

### 方案 B：VS Code + Cline 插件（功能更强）

1. 安装 **Cline** 扩展
2. 配置 API Provider → **Ollama**
3. 选择模型 `deepseek-coder-v2:16b`
4. Cline 支持 Plan/Act 模式 + MCP 扩展，功能更接近 Claude Code

```json
// Cline 设置
{
  "apiProvider": "ollama",
  "ollamaModelId": "deepseek-coder-v2:16b",
  "ollamaBaseUrl": "http://localhost:11434"
}
```

### 方案 C：JetBrains CLion + Continue

1. 从 JetBrains 插件市场安装 Continue
2. 配置 `config.json` 同上
3. 如果 UI 配置不生效，直接编辑 `%USERPROFILE%\.continue\config.json`

### 方案 D：终端中使用（类似 Claude Code 体验）

```bash
# 安装 aider
pip install aider-chat

# 使用本地模型
aider --model ollama/deepseek-coder-v2:16b

# 编辑模式
aider --model ollama/deepseek-coder-v2:16b src/core/event_loop.cpp
```

---

## 🔒 五、完全离线部署（内网环境）

### 外网准备（在能上网的机器上）

```bash
# 1. 下载 Ollama 安装包
# Linux:   https://ollama.com/download/linux
# Windows: https://ollama.com/download/windows
# macOS:   https://ollama.com/download/mac

# 2. 安装 Ollama 并拉取模型
ollama pull deepseek-coder-v2:16b
ollama pull codeqwen:7b

# 3. 下载 VS Code 插件
# Continue: https://marketplace.visualstudio.com/items?itemName=Continue.continue
# Cline:    https://marketplace.visualstudio.com/items?itemName=saoudrizwan.claude-dev
```

### 打包传输

```bash
# 模型文件位置
# Linux/macOS: ~/.ollama/
# Windows:     C:\Users\<用户名>\.ollama\

# 打包
tar -czf ollama-models.tar.gz ~/.ollama/

# 传输到内网机器（U盘 / 内部文件服务器）
```

### 内网安装

```bash
# 1. 安装 Ollama
# 双击安装包 或 dpkg -i ollama*.deb

# 2. 恢复模型文件
tar -xzf ollama-models.tar.gz -C ~/

# 3. 验证
ollama list    # 应该能看到模型列表
```

### 断网验证

```bash
# 拔掉网线后测试
ollama run deepseek-coder-v2:16b
>>> 用 C++ 写一个 epoll echo server
# 应该正常响应
```

---

## ⚡ 六、性能参考

| 硬件 | 模型 | 补全延迟 | Chat (256 token) |
|------|------|---------|-----------------|
| RTX 4090 24GB | DeepSeek-Coder 16B | ~150ms | ~3s |
| RTX 3060 12GB | DeepSeek-Coder 16B | ~250ms | ~6s |
| RTX 2060 6GB | CodeQwen 7B | ~200ms | ~4s |
| M4 Max 64GB | DeepSeek-Coder 16B | ~200ms | ~5s |
| M3 Pro 36GB | CodeQwen 7B | ~180ms | ~3.5s |
| **对比：GitHub Copilot** | 云端 | ~80ms | ~1.5s |

> 本地比云端慢 2-3x，但**零隐私泄露、零 API 费用、零网络依赖**。

---

## 🛠️ 七、高级配置

### 多模型分层策略（本地版）

```json
// ~/.continue/config.json
{
  "models": [
    {
      "title": "DeepSeek-Coder 16B (Chat)",
      "provider": "ollama",
      "model": "deepseek-coder-v2:16b",
      "apiBase": "http://localhost:11434"
    }
  ],
  "tabAutocompleteModel": {
    "title": "CodeQwen 7B (Fast Autocomplete)",
    "provider": "ollama",
    "model": "codeqwen:7b",
    "apiBase": "http://localhost:11434"
  }
}
```

小模型做补全（快），大模型做 Chat/Refactor（质量高）。

### 量化级别选择

```bash
# 查看可用量化版本
ollama show deepseek-coder-v2:16b

# 不同量化级别
ollama pull deepseek-coder-v2:16b-q4_K_M   # 推荐：质量损失 <3%，显存减半
ollama pull deepseek-coder-v2:16b-q8_0     # 质量最高，显存需求大
ollama pull deepseek-coder-v2:16b-q2_K     # 最小显存，质量下降明显
```

### 团队共享 Ollama 服务

```bash
# 在一台大显存机器上部署 Ollama，开放局域网访问
# 编辑 Ollama 服务配置
sudo systemctl edit ollama

# 添加：
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"

# 重启
sudo systemctl restart ollama

# 团队成员配置（替换 localhost 为服务器 IP）
{
  "apiBase": "http://192.168.1.100:11434"
}
```

---

## 🆚 八、本地 vs 云端方案对比

| 维度 | 本地方案 | 云端方案 |
|------|---------|---------|
| **隐私** | ✅ 100% 本地，代码不出机器 | ⚠️ 需信任云服务商 |
| **成本** | ✅ 一次性硬件投入，零持续费用 | ⚠️ $20-200/月/人 |
| **质量** | ⚠️ 低于 Claude Opus/GPT-5.5 | ✅ 最新最强模型 |
| **延迟** | ⚠️ 150-300ms | ✅ 50-100ms |
| **离线** | ✅ 完全离线可用 | ❌ 必须联网 |
| **维护** | ⚠️ 需要自己维护硬件和模型更新 | ✅ 云服务商维护 |

### 混合策略（推荐）

```
日常简单任务（补全/格式化/单行修改）
→ 本地 DeepSeek-Coder / CodeQwen（零成本）

中等复杂度（单文件实现/代码审查）
→ 本地大模型 或 DeepSeek API（极低成本）

复杂任务（多文件重构/架构设计）
→ Claude Code / Cursor（云端，按需付费）
```

---

## 🔧 九、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| CUDA Out of Memory | 显存不足 | 换更小的模型或更低的量化级别 |
| 模型响应极慢 | 模型被 swap 到系统内存 | 确保模型完全在 GPU 显存中 |
| Continue 404 | 误点了 Connect 按钮 | 直接在下拉菜单选择模型 |
| Windows 找不到 ollama | 未添加 PATH | 重新运行安装程序 |
| 补全延迟 >500ms | 模型太大 / GPU 太慢 | 换 CodeQwen 7B 做补全 |

---

*文档生成日期：2026-06-24 | 数据来源：Ollama 官方文档、Continue 官方文档、社区实战验证*
