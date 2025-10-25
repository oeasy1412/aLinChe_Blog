---
title: 协程
published: 2025-10-16
description: co_
tags: [OS, 线程, cpp]
category: OS
draft: false
---
## 核心概念：什么是协程？
协程是一种可以暂停执行并在稍后恢复执行的函数，将数据分配在堆上保存。与常规函数一旦调用就必须执行到返回不同，协程提供了协作式的多任务处理能力，使得编写异步代码如同编写同步代码一样直观。
在 C++20 中，一个函数只要其内部包含了 `co_await`、`co_yield` 或 `co_return` 这三个关键字中的任何一个，它就成为了一个协程函数。
*   `co_await`: 暂停协程，等待 `expression` 操作完成。这是实现异步的核心。
*   `co_yield`: 暂停协程，并产生一个值。主要用于实现生成器（Generator）。
*   `co_return`: 结束协程的执行，并返回一个最终值。

## 协程的基石：协程框架 (`Coroutine Frame`)
协程能够暂停和恢复的关键在于它的完整执行状态​​（包括挂起点、局部变量、参数、promise对象等）被保存在了堆上分配的`协程帧`，而不是像常规函数一样完全依赖于线程栈。

**当一个协程被调用时，并不会立即执行其函数体内的代码**。编译器会执行以下关键步骤：
1.  **在堆上分配内存**：编译器会生成代码，通过 `operator new` 在堆上分配一块内存，这块内存被称为“协程框架”或“协程状态”。
2.  **存储协程状态**：这个框架是一个复合对象，它存储了协程恢复执行所需的一切信息：
    *   **函数参数**：所有传递给协程的参数都会被复制或移动到协程框架中。
    *   **局部变量**：在挂起点之间需要保持状态的局部变量。
    *   **Promise 对象**：一个用户定义的对象，用于控制协程的行为，并与调用者通信。
    *   **恢复点信息**：记录协程暂停的位置，以便下次能从正确的地方恢复。
    *   **其他编译器内部状态**。

这种机制被称为 “无栈协程” (`Stackless Coroutines`)，因为协程一旦挂起，它就不再占用调用线程的栈空间，将自身的执行状态存储在堆上的协程帧中，使得系统可以高效地管理成千上万个并发的协程。

## 调用者与协程的交互：堆栈行为
理解协程与常规线程栈的交互至关重要：
*   **初始调用**：当一个普通函数（调用者）调用一个协程时，仍然会在当前线程的栈上创建一个临时的栈帧。
*   **挂起与返回**：当协程执行到第一个挂起点（通常是 `co_await`），并且该挂点决定需要暂停时（`await_ready()` return false），协程会`挂起`。此时，控制权连同协程的返回值（由 `promise_type::get_return_object()`提供）会一起`返回`给调用者。
*   **栈帧弹出**：随着控制权返回，为调用协程而创建的那个临时栈帧就会被pop弹出，**调用线程的栈空间被释放**。
*   **恢复执行**：协程的恢复是通过 `std::coroutine_handle<>` 的 `resume()` 方法触发的。恢复操作可以在任何线程上执行，协程并不会回到原来的调用栈，而是**在调用resume()恢复它的线程上下文中继续执行**。

**示例代码**
```cpp
// 示例：协程自行管理生命周期（不存储句柄）
struct AsyncTask {
    struct promise_type {
        // 返回空任务对象
        AsyncTask get_return_object() { return {}; } 
        // 立即开始执行
        std::suspend_never initial_suspend() noexcept { return {}; }
        // 协程结束后立即销毁
        std::suspend_never final_suspend() noexcept { return {}; }
        // co_return; 的处理
        void return_void() {} // or void return_value(int) {}
        void unhandled_exception() { std::terminate(); /* 在实际应存储异常 */ }
    };
};

struct TimerAwaitable {
    int delay_ms;
    int m_val{0};

    bool await_ready() const { return false; } // 总是需要等待

    void await_suspend(std::coroutine_handle<> handle) {
        thread([handle, ms = delay_ms, this] {
            std::this_thread::sleep_for(std::chrono::milliseconds(ms));
            m_val = 1;
            handle.resume(); // 恢复协程
        }).detach();
    }

    int await_resume() { return m_val; }
};
```

---

## **深入剖析：协程的组成部分与工作流程**
C++20 协程的设计更偏向于一个框架，让库的开发者来定义具体的行为。这套框架的核心是 `Promise` 对象和 `Awaitable` 对象。

### **控制中心：Promise 对象 (`promise_type`)**
每个协程都与一个 Promise 对象关联。这是一个由程序员定义的类（必须命名为 `promise_type`），它像一个控制中心，**规定**了协程的整个生命周期行为。 编译器通过 `std::coroutine_traits<>` 模板，根据协程的返回类型来找到对应的 `promise_type`。 最常见的做法是在协程的返回类型中内嵌一个 `promise_type` 定义。

一个典型的 `promise_type` 包含以下关键函数：

*   `get_return_object()`: 在协程帧分配完成并初始化后，在执行 initial_suspend() 之前，第一个被调用的函数。它负责创建一个返回给调用者的对象（例如一个 `Task` 或 `Generator` 对象）。调用者通过这个对象与协程交互。
*   `initial_suspend() noexcept`: 返回一个 Awaitable 对象，决定协程在开始执行前是否要立即挂起。
    *   若返回 `std::suspend_always{}`：协程创建后立即挂起，等待被手动 `resume()`。
    *   若返回 `std::suspend_never{}`：协程创建后立即执行，直到遇到第一个真正的挂起点。
*   `final_suspend() noexcept`: 返回一个 Awaitable 对象，决定协程在执行完毕（到达 `co_return` 或函数体末尾）后是否要挂起。
    *   通常返回 `std::suspend_always{}`，以保持协程框架存活，使得调用者可以安全地获取协程的结果或异常。之后需要手动销毁句柄。
*   `return_void()` / `return_value(value)`: 当协程执行 `co_return;` / `co_return value;` 时被调用。用于将结果存入 Promise 对象中，以便调用者获取。
*   `yield_value(value)`: 当协程执行 `co_yield value;` 时被调用。这是 `co_yield` 表达式的底层实现。
*   `unhandled_exception()`: 如果协程内部抛出未被捕获的异常，该函数会被调用，通常用于将异常信息保存起来。

**示例：一个返回类型的基本骨架**
```cpp
// 示例：外部控制协程（存储句柄）
struct AsyncTask {
    struct promise_type {
        // 返回包含句柄的任务对象
        AsyncTask get_return_object() { return AsyncTask(std::coroutine_handle<promise_type>::from_promise(*this)); }
        // 初始挂起，让调用者获得控制权控制何时开始
        std::suspend_always initial_suspend() noexcept { return {}; }
        // 最终挂起，等待调用者销毁
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {} // or void return_value(int) {}
        void unhandled_exception() { terminate(); /* 在实际应存储异常 */ }
    };

    std::coroutine_handle<promise_type> handle; // 持有协程的句柄

    explicit AsyncTask(std::coroutine_handle<promise_type> h) : handle(h) {}
    bool done() const { return handle.done(); }
    void resume() {
        if (handle && !handle.done()) {
            handle.resume();
        }
    }
    ~AsyncTask() { // RAII
        if (handle) {
            handle.destroy();
        }
    }
};

struct suspend_always {
    constexpr bool await_ready() const noexcept { return false; }
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
};
struct suspend_never { 
    constexpr bool await_ready() const noexcept { return true; }
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
};
```

### **挂起的艺术：Awaitable 对象与 `co_await`**
`co_await` 关键字后面跟的是一个 Awaitable 对象。这个对象定义了“**如何等待**”的具体逻辑。 任何实现了特定接口的类型都可以是 Awaitable。这个接口由三个函数组成：

*   `await_ready() -> bool`: 在挂起协程之前被调用。这个可以做一个优化：如果等待的操作已经完成（例如，数据已在缓存中），它可以返回 `true`，这样协程就不会发生实际的挂起和恢复，避免了上下文切换的开销。
*   `await_suspend(std::coroutine_handle<> handle)`: 当 `await_ready()` 返回 `false` 时，协程状态被保存，然后此函数被调用。它的核心任务是安排未来的恢复操作。它接收当前协程的句柄 `handle`，可以在一个异步操作（如网络请求、文件IO）的回调函数中调用 `handle.resume()` 来唤醒协程。
*   `await_resume()`: 当协程被 `resume()` 恢复后，此函数被调用。它的返回值就是整个 `co_await` 表达式的结果。

**`co_await` 的完整流程：**
1.  计算 `co_await` 后的表达式，得到一个 Awaitable 对象。
2.  调用 Awaitable 的 `await_ready()`。
3.  如果返回 `true`，流程继续到第 5 步，不发生挂起。
4.  如果返回 `false`，协程挂起，并调用 Awaitable 的 `await_suspend(handle)`。这个函数负责启动异步操作，并安排在操作完成时调用 `handle.resume()`。然后控制权返回给调用者/恢复者。
5.  当协程最终被恢复时，执行 `await_resume()`，其返回值成为 `co_await` 表达式的值。
```c
tasks.emplace_back(coroutine(i, 1000));
task.resume();
// [Task]: get_return_object()
// [Task]: initial_suspend(): wait for caller...
// [Coroutine0]: started.
// [Awaitable]: await_ready()
// [Awaitable]: await_suspend()
// ...
// [Awaitable]: await_resume()
// [Coroutine0]: will finish and wait for destruction
// [Task]: return_value()
// [Task]: final_suspend(): wait for destruction...
// 协程 0 完成
// [Task]: Destructor called
```

### **`co_*` 语法糖的本质**
`co_await` `co_yield` `co_return` 其本质是语法糖。编译器会将这些简洁的关键字**解糖(`desugar`)**，转换成一套复杂的、基于协程框架、Promise 对象和 Awaitable 对象的底层调用。 这种“语法糖”极大地简化了异步编程的复杂性，让程序员可以专注于业务逻辑，而不是繁琐的状态机和回调管理。


## 剖析 std::coroutine_handle 的实现代码
- `coroutine_handle` 的本质：**一个围绕着 `void*` 指针的轻量级、非拥有式的包装器，其所有“魔法”操作都委托给了编译器内置函数（`__builtin_coro_*`）**。

### **核心作用：协程的“遥控器”**
您可以将 `coroutine_handle` 想象成一个指向堆上协程状态的“遥控器”或“裸指针”。它本身非常小（只有一个指针的大小），并且**不拥有**它所指向的协程状态。这意味着 `coroutine_handle` 的创建和销毁不会影响协程状态的生命周期。你必须手动管理协程状态的销毁。

它的主要职责有两个：
1.  **控制协程**：恢复（`resume`）或销毁（`destroy`）协程。
2.  **访问协程的 `promise` 对象**：作为与协程内部通信的桥梁。

### **成员详解**
#### 构造与赋值 (Construction and Assignment)
```cpp
template <typename _Promise>
struct coroutine_handle { ... };
```
*   `static coroutine_handle from_promise(_Promise& __p)` 从一个 `promise` 对象的引用创建一个有效的 `coroutine_handle`。
*   **工作原理**：编译器在堆上分配协程状态时，`promise` 对象是整个状态数据块的一部分。这个函数通过调用编译器内置的 `__builtin_coro_promise`，传入 `promise` 对象的地址，编译器就能通过指针运算**反向推算出整个协程状态数据块的起始地址**，并用这个地址来初始化 `coroutine_handle`。
*   **使用场景**：它几乎只在 `promise_type` 的 `get_return_object()` 方法中被调用，用于创建那个将要返回给调用者的任务对象（例如我们之前的 `AsyncTask`）。

#### 导出与导入 (Export and Import)
*   `constexpr void* address() const noexcept`
*   `constexpr static coroutine_handle from_address(void* __a) noexcept`
*   `address()`：将句柄内部的 `void*` 指针暴露出来。你可以将这个裸指针传递给一个 C函数。
*   `from_address()`：从一个 `void*` 指针重建 `coroutine_handle`。
> **警告**：这是一个不安全的操作。如果你从一个无效的地址创建句柄，后续操作将导致严重错误。

#### 类型转换 (Conversion)
*   `constexpr operator coroutine_handle<>() const noexcept`
这个转换运算符允许你将一个**特化版本**的句柄（例如 `coroutine_handle<MyPromise>`）转换为一个**非特化（类型擦除）版本**的句柄（`coroutine_handle<>`）。
*   **使用场景**：当你需要编写一个可以处理任何类型协程的通用调度器或执行器时，这个功能非常有用。调度器不需要知道 `promise` 的具体类型，它只需要能够 `resume()` 或 `destroy()` 协程即可。

#### 状态观察 (Observers)
*   `constexpr explicit operator bool() const noexcept`
一个标准的布尔转换，用于检查句柄是否为空。等价于 `address() != nullptr`。
*   `bool done() const noexcept`
这是一个非常重要的状态检查函数。它通过调用 `__builtin_coro_done` 来检查协程是否**执行完毕**。
*   **`true` 的条件**：协程已经执行到 `co_return` 或函数体的末尾，并且**当前正挂起在 `final_suspend` 点**。
*   **`false` 的条件**：协程还未执行完毕，或者协程已经执行完毕但已被 `destroy()`。

#### 执行控制 (Resumption)
*   `void operator()() const`
*   `void resume() const`
这是**恢复协程执行**的两个方法（`operator()` 只是 `resume()` 的语法糖）。它通过调用 `__builtin_coro_resume` 来执行恢复操作。这个编译器内置函数会负责恢复协程的上下文（寄存器等），然后跳转到上次暂停的位置继续执行。
> **警告**：只能对处于挂起状态的协程调用 `resume()`。对一个未挂起或已结束的协程调用 `resume()` 是未定义行为。
*   `void destroy() const`
这是**销毁协程状态**的函数。它会调用 `__builtin_coro_destroy`，这个内置函数会：
1.  调用 `promise_type` 的析构函数。
2.  调用协程参数和局部变量的析构函数。
3.  通过 `operator delete` 释放整个协程状态占用的堆内存。
> **警告**：每个协程状态**必须且只能被 `destroy()` 一次**，否则会导致内存泄漏或二次释放。这通常通过 RAII 包装类（如我们示例中的 `AsyncTask`）的析构函数来保证。

#### Promise 访问 (Promise Access)
*   `_Promise& promise() const`
这个函数允许句柄的持有者**获取对协程内部 `promise` 对象的引用**。这是调用者与协程进行数据交换的关键。
*   **工作原理**：它再次调用 `__builtin_coro_promise`，传入协程状态的起始地址，编译器就能计算出 `promise` 对象在该内存块中的确切位置并返回其引用。
*   **使用场景**：调用者可以通过 `task.handle.promise()` 来获取协程计算的结果、抛出的异常，或者通过 `promise` 提供的接口向协程传递数据。

#### 私有成员 (Private Member)
*   `void* _M_fr_ptr = nullptr;`
这就是 `coroutine_handle` 的全部家当：一个指向协程状态（"frame pointer"）的裸指针。所有公开的成员函数最终都是在操作这个指针。

### **总结**
`coroutine_handle` 本身很简单，它只是一个接口。真正的复杂性隐藏在编译器生成的代码和这些 `__builtin_coro_*` 内置函数背后，它们共同构成了 C++ 协程高效运行的基石。