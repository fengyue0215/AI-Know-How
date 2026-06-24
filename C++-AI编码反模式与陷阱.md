# C++ AI 编码反模式与陷阱

> 基于 Jason Turner（CppCon 2025）和社区实战经验整理。  
> 核心原则：AI 会复现互联网上的劣质 C++ 代码。你的工作是识别并拒绝它们。

---

## 🚨 一、十大常见陷阱

### 陷阱 1：裸 new/delete

```cpp
// ❌ AI 常生成的（互联网上的过时代码）
class Connection {
    Socket* socket_;
public:
    Connection() { socket_ = new Socket(); }
    ~Connection() { delete socket_; }  // 如果构造函数抛异常？如果被复制？
};

// ✅ 正确写法
class Connection {
    std::unique_ptr<Socket> socket_;
public:
    Connection() : socket_(std::make_unique<Socket>()) {}
    // 自动析构，不可复制（或显式实现移动语义）
};
```

**为什么 AI 会错**：互联网上大量 C++98/03 教材和代码仍在使用裸指针。

### 陷阱 2：资源泄漏（忘记 RAII）

```cpp
// ❌ AI 常生成的
void processFile(const char* path) {
    FILE* f = fopen(path, "r");
    if (!f) return;
    char buf[256];
    fread(buf, 1, 256, f);
    if (some_condition) return;  // 泄漏！忘了 fclose(f)
    fclose(f);
}

// ✅ 正确写法
void processFile(const char* path) {
    std::unique_ptr<FILE, decltype(&fclose)> f(fopen(path, "r"), &fclose);
    if (!f) return;
    // 自动关闭，即使提前 return 也不泄漏
}
```

### 陷阱 3：未检查返回值

```cpp
// ❌ AI 常生成的
void sendData(int fd, const std::vector<char>& data) {
    write(fd, data.data(), data.size());  // 没检查返回值！
    // write 可能只写了部分数据，甚至返回 -1
}

// ✅ 正确写法
void sendData(int fd, const std::vector<char>& data) {
    size_t total = 0;
    while (total < data.size()) {
        ssize_t n = write(fd, data.data() + total, data.size() - total);
        if (n < 0) {
            if (errno == EINTR) continue;
            throw std::runtime_error("write failed: " + std::string(strerror(errno)));
        }
        total += n;
    }
}
```

### 陷阱 4：悬空引用/迭代器

```cpp
// ❌ AI 常生成的
std::vector<int>& getData() {
    std::vector<int> data = {1, 2, 3};
    return data;  // 返回局部变量的引用！UB！
}

// ✅ 正确写法
std::vector<int> getData() {
    return {1, 2, 3};  // RVO / move
}
```

### 陷阱 5：多线程数据竞争

```cpp
// ❌ AI 常生成的（看起来"差不多对"）
class Counter {
    int count_ = 0;
public:
    void increment() { count_++; }  // 不是原子的！
    int get() const { return count_; }
};

// ✅ 正确写法
class Counter {
    std::atomic<int> count_{0};
public:
    void increment() { count_.fetch_add(1, std::memory_order_relaxed); }
    int get() const { return count_.load(std::memory_order_relaxed); }
};
```

### 陷阱 6：移动后的对象被使用

```cpp
// ❌ AI 常生成的
std::string data = std::move(other);
process(data);
process(other);  // other 已被移动！UB！

// ✅ 正确写法
std::string data = std::move(other);
process(data);
// 不再使用 other，或者明确重置它
```

### 陷阱 7：虚析构函数缺失

```cpp
// ❌ AI 常生成的
class Base {
public:
    // 没有虚析构函数！
    virtual void doWork() = 0;
};

class Derived : public Base {
    std::vector<int> data_;
public:
    void doWork() override { /* ... */ }
};

std::unique_ptr<Base> ptr = std::make_unique<Derived>();
// ptr 析构时只调用 Base::~Base()，Derived::data_ 泄漏！

// ✅ 正确写法
class Base {
public:
    virtual ~Base() = default;  // 有虚函数的基类必须有虚析构函数
    virtual void doWork() = 0;
};
```

### 陷阱 8：不安全的类型转换

```cpp
// ❌ AI 常生成的
void processPacket(const uint8_t* data, size_t len) {
    if (len < sizeof(Header)) return;
    Header* hdr = (Header*)data;  // C-style cast，对齐问题！UB！
    hdr->process();
}

// ✅ 正确写法
void processPacket(const uint8_t* data, size_t len) {
    if (len < sizeof(Header)) return;
    Header hdr;
    std::memcpy(&hdr, data, sizeof(Header));  // 安全拷贝，避免对齐问题
    hdr.process();
}
```

### 陷阱 9：异常安全缺失

```cpp
// ❌ AI 常生成的（强异常保证都没了）
void updateConfig(const std::string& key, const std::string& value) {
    db_.insert(key, value);          // 如果这里抛异常
    cache_.invalidate(key);          // 这行不会执行
    notifyListeners(key, value);     // 这行也不会执行
    // 但 db_ 已经部分更新了！
}

// ✅ 正确写法（copy-and-swap / scope guard）
void updateConfig(const std::string& key, const std::string& value) {
    auto oldValue = db_.get(key);  // 保存旧值
    db_.insert(key, value);
    try {
        cache_.invalidate(key);
        notifyListeners(key, value);
    } catch (...) {
        db_.insert(key, oldValue);  // 回滚
        throw;
    }
}
```

### 陷阱 10：使用已过时的 C++ 惯用法

```cpp
// ❌ AI 常生成的
std::string result;
for (size_t i = 0; i < items.size(); i++) {       // 索引循环
    result += items[i].name() + ", ";              // 字符串拼接
}
if (!result.empty()) result.pop_back(); result.pop_back();  // 去掉末尾逗号

// ✅ 现代 C++ 写法
#include <ranges>
auto names = items | std::views::transform(&Item::name);
std::string result = std::accumulate(
    std::next(names.begin()), names.end(), std::string(*names.begin()),
    [](std::string a, const std::string& b) { return std::move(a) + ", " + b; }
);
// 或等 ranges::to 在 C++23 更简洁
```

---

## ✅ 二、AI 生成代码的安全验证清单

收到 AI 生成的 C++ 代码后，按此清单逐项检查：

### 编译层

- [ ] 编译通过（`-Wall -Wextra -Werror`）
- [ ] 无 narrowing conversion 警告
- [ ] 无 sign-compare 警告

### 静态分析层

- [ ] `clang-tidy` 无新增警告
- [ ] `cppcheck` 无新增问题
- [ ] 无裸 `new`/`delete`
- [ ] 析构函数为 `virtual` 或 `final` 类不需要

### Sanitizer 层（必须跑！）

- [ ] AddressSanitizer：`-fsanitize=address` 无报错
- [ ] UndefinedBehaviorSanitizer：`-fsanitize=undefined` 无报错
- [ ] ThreadSanitizer：`-fsanitize=thread` 无报错（多线程代码）

### 逻辑层

- [ ] 所有系统调用返回值已检查
- [ ] 所有 IO 操作设置了超时
- [ ] 资源由 RAII 管理（无手动 `close`/`free`/`delete`）
- [ ] 异常安全（或项目禁用异常时有替代方案）
- [ ] 移动语义使用正确

---

## 🎯 三、让 AI 少犯错的 Prompt 技巧

### 技巧 1：在 Prompt 中禁止常见反模式

```
约束：
- 禁止使用裸 new/delete（用 std::make_unique/std::make_shared）
- 禁止使用 C-style cast（用 static_cast/reinterpret_cast）
- 禁止返回局部变量的引用或指针
- 所有系统调用必须检查返回值
- 使用 RAII 管理所有资源
```

### 技巧 2：要求 AI 标注潜在风险点

```
输出格式要求：
在每个函数的注释中标注：
// @risk: [可能的风险点，如"这里假定 data 非空"]
```

### 技巧 3：让 AI 对自己的代码做安全审查

```
现在对你刚生成的代码做安全审查，检查：
1. 资源泄漏
2. 数据竞争
3. 未定义行为
4. 异常安全
列出每个问题并给出修复。
```

---

## 📚 四、Jason Turner 的核心建议

> 来自 CppCon 2025 "Best Practices for AI Tool Use in C++"

| 建议 | 解释 |
|------|------|
| **"Build with CMake + Ninja, run ctest"** | 自动化构建+测试是最低安全网 |
| **"Sanitizer 不是可选项"** | AI 生成的代码必须跑 ASan + UBSan |
| **"类型系统是最好的文档"** | 用 `[[nodiscard]]`、`std::optional`、强类型让错误代码无法编译 |
| **"小步验证"** | 每次接受 1-2 个函数的 AI 生成代码，立即编译测试 |
| **"不要修改 AI 代码的风格"** | 让 clang-format 统一格式化，AI 的风格偏好不重要 |

---

*文档生成日期：2026-06-24 | 参考：Jason Turner CppCon 2025、C++ Core Guidelines、社区实战经验*
