# 团队 Prompt 模板库 —— C++ 后端开发专用

> 以下模板针对 C++17 后端技术栈（Socket / Redis / Oracle / ZMQ / Protobuf）预配置。  
> 直接替换 `[占位符]` 即可使用。每个模板都配有使用说明。

---

## 📋 模板索引

| # | 场景 | 适用工具 |
|---|------|---------|
| 1 | 网络编程（epoll / io_uring） | Claude Code / Cursor |
| 2 | Redis 客户端封装 | Claude Code / Cursor |
| 3 | Oracle 数据库访问层 | Claude Code / Cursor |
| 4 | ZMQ 消息队列 | Claude Code / Cursor |
| 5 | Protobuf 序列化 | Claude Code / Cursor |
| 6 | CMake 构建脚本 | Claude Code / Cursor |
| 7 | 单元测试生成 | Claude Code / Cursor |
| 8 | 代码审查 | Claude Code `/review` |
| 9 | 调试分析（coredump / 日志） | Claude Code / Cursor |
| 10 | 架构方案设计 | Claude Code（Plan Mode） |

---

## 模板 1：网络编程（epoll ET 模式）

### 使用说明
适用于需要实现或重构网络层的场景。明确指定 epoll 参数、超时、无阻塞要求。

### Prompt 模板

```
你是一个 C++17 网络编程专家。当前项目使用 epoll + 非阻塞 IO。

## 技术栈
- C++17, CMake 3.20+, GCC 11+
- 项目禁用异常（-fno-exceptions），错误使用 std::optional 返回

## 任务
实现一个 TCP [服务器/客户端/连接池]，具体要求：
- 使用 epoll_create1 + epoll_wait，边缘触发（EPOLLET）
- 支持 [N] 个并发连接
- 所有 IO 操作设置 [30] 秒超时
- 使用 non-blocking socket + fcntl
- 使用 std::unique_ptr 管理资源

## 约束
- 禁止裸 new/delete（用 std::make_unique）
- 禁止 C-style cast
- 所有系统调用必须检查返回值
- 使用 RAII 管理所有文件描述符
- 不得阻塞调用线程

## 输出格式
1. 头文件声明（include/network/[文件名].h）
2. 实现文件（src/network/[文件名].cpp）
3. 单元测试骨架（GoogleTest）
4. CMakeLists.txt 更新
5. 使用示例
```

---

## 模板 2：Redis 客户端封装

### 使用说明
适用于 hiredis 异步 API 的封装。区分连接池、管道、哨兵等模式。

### Prompt 模板

```
你是一个 C++17 后端开发专家，精通 Redis 和 hiredis。

## 技术栈
- C++17, CMake, hiredis 异步 API
- 项目禁用异常，错误使用 std::optional / absl::StatusOr

## 任务
实现 Redis [连接池/管道/哨兵客户端]，具体要求：
- 基于 hiredis 异步 API
- 连接池：最小 [5] 个，最大 [20] 个连接
- 所有操作 [3] 秒超时，失败自动重试（最多 [3] 次）
- 支持 String / Hash / List / Set 操作
- 连接断开自动重连
- 集成到现有内存池统计系统（MemoryPool::record）

## 约束
- 线程安全（std::shared_mutex）
- 不得阻塞主事件循环
- 使用 std::unique_ptr 管理 redisContext
- 所有 hiredis 回调必须处理错误情况

## 输出格式
1. 头文件（include/db/redis_[功能].h）
2. 实现文件（src/db/redis_[功能].cpp）
3. 连接生命周期状态机说明
4. 单元测试（GoogleTest + redis mock）
```

---

## 模板 3：Oracle 数据库访问层

### 使用说明
适用于 Oracle OCCI 封装。注意 Oracle 特有的连接管理、LOB 处理。

### Prompt 模板

```
你是一个 C++17 企业级后端开发专家，精通 Oracle 数据库和 OCCI。

## 技术栈
- C++17, Oracle OCCI, CMake
- 项目禁用异常，错误使用 std::optional

## 任务
实现 Oracle 数据库访问层，具体要求：
- 基于 OCCI 连接池（occi::Environment::createConnectionPool）
- 支持参数化查询和预编译语句
- 支持 CLOB/BLOB 读写
- 事务支持（RAII 风格的事务对象）
- 连接池参数可配置（最小/最大连接数、超时）

## 关键表结构
[在此粘贴你的表 DDL]

## 约束
- 禁止裸指针管理 OCCI 对象（用自定义 deleter 的 unique_ptr）
- 所有数据库操作必须有超时控制
- 不得在头文件中暴露 OCCI 头文件（Pimpl 惯用法）
- 线程安全

## 输出格式
1. 头文件（include/db/oracle_[功能].h）—— 使用 Pimpl 隐藏 OCCI 细节
2. 实现文件（src/db/oracle_[功能].cpp）
3. RAII 事务对象设计
4. CRUD 使用示例
```

---

## 模板 4：ZMQ 消息队列

### 使用说明
注意 ZMQ 的 socket 不是线程安全的，必须在创建线程中使用。

### Prompt 模板

```
你是一个 C++17 分布式系统专家，精通 ZeroMQ。

## 技术栈
- C++17, cppzmq, CMake
- 项目禁用异常

## 任务
实现 ZMQ [PUB-SUB / PUSH-PULL / ROUTER-DEALER] 模式，具体要求：
- 使用 cppzmq（C++ 封装）
- PUB 端支持 [N] 个 topic，消息速率 [10K/s]
- SUB 端支持 topic 过滤和消息反序列化
- 实现高水位标记（HWM）防止内存溢出
- 连接断开自动重连
- 消息序列化使用 Protobuf

## 约束
- ZMQ socket 必须在创建线程中使用（非线程安全）
- 使用 zmq::poll 或集成到 epoll 事件循环
- 设置 ZMQ_LINGER 为 [0] 避免关闭阻塞
- 所有消息必须校验长度和格式

## 输出格式
1. 头文件（include/mq/zmq_[模式].h）
2. 实现文件（src/mq/zmq_[模式].cpp）
3. 消息格式文档
4. 异常场景处理说明
```

---

## 模板 5：Protobuf 序列化

### Prompt 模板

```
你是一个 C++17 后端开发专家，精通 Protocol Buffers。

## 技术栈
- C++17, Protobuf 3.x, CMake
- 项目禁用异常

## 任务
实现 Protobuf 消息的序列化/反序列化工具类，具体要求：
- 支持从二进制/JSON 反序列化
- 消息校验（必填字段、值域检查）
- 大消息的零拷贝处理（使用 std::string_view 或 arena allocation）
- 与 ZMQ 消息系统集成

## 约束
- 使用 Arena allocation 减少内存分配（高频消息场景）
- 所有反序列化必须校验输入数据
- 不得在热路径上使用反射

## [粘贴 .proto 文件内容]
```

---

## 模板 6：CMake 构建脚本

### 使用说明
适用于生成或修改 CMakeLists.txt。C++ 项目的构建系统往往复杂，让 AI 生成可以减少手动配置错误。

### Prompt 模板

```
你是一个 C++17 构建系统专家，精通 CMake。

## 项目结构
src/
├── core/          # 事件循环、内存池
├── network/       # TCP/UDP、ZMQ 封装
├── db/            # Redis、Oracle
├── mq/            # 消息队列
include/
└── [对应的 public headers]
tests/
└── [GoogleTest]

## 任务
生成 CMakeLists.txt，要求：
- CMake 3.20+
- C++17 标准
- 启用 -Wall -Wextra -Werror
- 链接库：hiredis, occi, zmq, protobuf, pthread, gtest
- 支持 Debug/Release 构建类型
- 支持 compile_commands.json 生成

## 特殊要求
- 禁用异常：-fno-exceptions
- AddressSanitizer 选项：-fsanitize=address（Debug 模式）
- 头文件依赖自动扫描
```

---

## 模板 7：单元测试生成（GoogleTest）

### Prompt 模板

```
你是一个 C++ 测试专家，精通 GoogleTest 框架。

## 技术栈
- C++17, GoogleTest, GMock
- 项目禁用异常

## 待测试代码
[粘贴头文件和实现文件]

## 任务
为以上代码生成完整的单元测试，要求：
- 每个公共接口至少 [2] 个测试用例
- 覆盖正常路径 + 边界条件 + 错误路径
- 使用 GMock mock 外部依赖（Redis/Oracle/ZMQ）
- 使用 TEST_F 测试夹具管理共享资源
- 测试命名遵循 Given_When_Then 风格

## 约束
- Mock 对象必须是线程安全的（如果原代码是多线程）
- 测试不得依赖外部服务（全部 mock）
- 每个测试独立，不依赖执行顺序
```

---

## 模板 8：代码审查

### 使用说明
在 Claude Code 中直接使用 `/review`，或粘贴 diff 到 Cursor Chat。

### Prompt 模板

```
请审查以下代码变更，从以下维度分析：

1. **内存安全**：有无泄漏、悬空指针、双重释放
2. **并发安全**：数据竞争、死锁风险、锁粒度
3. **性能**：不必要的拷贝、热路径上的分配、缓存友好性
4. **错误处理**：返回值检查、异常安全、边界条件
5. **API 设计**：接口一致性、const 正确性、noexcept 标记
6. **现代 C++**：是否可以利用 C++17 特性简化

## 代码变更
[粘贴 git diff 或代码]

## 输出格式
每个问题标注：严重程度（🔴致命 / 🟡警告 / 🔵建议）+ 修复方案
```

---

## 模板 9：调试分析

### Prompt 模板

```
## 崩溃信息
[粘贴 coredump / GDB backtrace / valgrind 输出]

## 相关代码
[粘贴崩溃位置的源码]

## 系统环境
- OS: [Linux / Windows]
- 编译器: [GCC 11 / Clang 14]
- 编译选项: [-O2 -fno-exceptions]

## 请分析
1. 最可能的崩溃原因（给出 2-3 个假设，按可能性排序）
2. 每个假设的验证方法（GDB 命令 / 添加日志 / repro 步骤）
3. 修复方案（优先选择不需要重构的快速修复）
```

---

## 模板 10：架构方案设计（Plan Mode）

### 使用说明
在 Claude Code 中先描述问题，AI 会进入 Plan Mode 给出方案。确认后再生成代码。

### Prompt 模板

```
## 背景
当前系统使用 [现有架构描述]，遇到以下问题：
[问题 1]
[问题 2]

## 目标
[期望达成的效果]

## 约束
- 不得破坏现有接口兼容性
- 性能不低于现有方案的 [90%]
- 改动范围控制在 [指定模块/目录]

## 请先规划方案
1. 分析现有架构的瓶颈
2. 给出 2-3 个可选方案（包含优缺点对比）
3. 推荐方案及实施步骤

我确认方案后再开始编码。
```

---

## 💡 使用建议

1. **首次使用时替换 `[占位符]`**，然后保存为团队共享的 Prompt 片段
2. **在 CLAUDE.md 中引用常用模板**：`Prompt templates: see docs/prompts/`
3. **持续迭代**：每次发现好用的 Prompt 变体，更新到模板库
4. **记录效果**：在模板注释中标注"XX 场景下效果最好"
5. **分类管理**：按模块（网络/数据库/消息队列/测试）组织模板文件

---

*文档生成日期：2026-06-24 | 建议将此文件纳入 Git 管理，团队共同维护*
