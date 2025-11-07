---
title: 基于协程的线程池
published: 2025-10-16
description: co_ThreadPool
tags: [OS, 并发, 线程, cpp]
category: OS
draft: false
---
```cpp
#include <atomic>
#include <cassert>
#include <coroutine>
#include <deque>
#include <functional>
#include <iostream>
#include <mutex>
#include <thread>
#include <utility>
#include <vector>

#define NTHREAD 32

// 前向声明
class CoroutineThreadPool;

// 协程任务类型
// [[nodiscard]] 属性提示编译器，如果返回值未被使用，则发出警告
struct [[nodiscard]] CoroutineTask {
  public:
    struct promise_type {
        CoroutineTask get_return_object() {
            return CoroutineTask{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; } // 延迟执行控制+异步任务调度
        std::suspend_always final_suspend() noexcept { return {}; }   // 协程结束后能安全结果获取或异常
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };

  private:
    std::coroutine_handle<promise_type> handle;
    // 私有构造函数，只能由 promise_type 创建
    explicit CoroutineTask(std::coroutine_handle<promise_type> h) : handle(h) {}

  public:
    friend class CoroutineThreadPool;
    friend struct ThreadPoolScheduler;

    bool done() const { return handle.done(); }
    void resume() const {
        if (handle && !handle.done()) {
            handle.resume();
        }
    }
    // 禁用拷贝
    CoroutineTask(const CoroutineTask&) = delete;
    CoroutineTask& operator=(const CoroutineTask&) = delete;
    // 移动语义
    CoroutineTask(CoroutineTask&& other) noexcept : handle(std::exchange(other.handle, nullptr)) {}
    CoroutineTask& operator=(CoroutineTask&& other) noexcept {
        if (this != &other) {
            if (handle)
                handle.destroy();
            handle = std::exchange(other.handle, nullptr);
        }
        return *this;
    }
    // RAII
    ~CoroutineTask() {
        if (handle) {
            handle.destroy();
        }
    }
};

// Awaitable 对象，用于将协程调度回线程池
struct ThreadPoolScheduler {
    CoroutineThreadPool* pool;

    explicit ThreadPoolScheduler(CoroutineThreadPool* p) noexcept : pool(p) {}

    bool await_ready() const noexcept { return false; }       // 总是挂起以进行重新调度
    void await_suspend(std::coroutine_handle<> handle) const; // 实现放在线程池定义之后
    void await_resume() const noexcept {}
};

// 一个简单的线程安全队列 
template <class T>
class ConcurrentQueue {
    mutable std::mutex mtx;
    std::deque<T> queue;

  public:
    void push(T&& task) {
        std::lock_guard lock(mtx);
        queue.push_back(std::move(task));
    }

    bool pop(T& task) {
        std::lock_guard lock(mtx);
        if (queue.empty())
            return false;
        task = std::move(queue.front());
        queue.pop_front();
        return true;
    }

    bool empty() const {
        std::lock_guard lock(mtx);
        return queue.empty();
    }
};

// 线程池
class CoroutineThreadPool {
  private:
    struct Worker {
        std::thread thread;
    };

    std::vector<Worker> workers;
    ConcurrentQueue<std::function<void()>> task_queue;
    std::atomic<bool> stop_flag{false};

  public:
    CoroutineThreadPool(size_t n = NTHREAD) : workers(n) {
        // 预先启动所有线程，等待任务
        for (auto& worker : workers) {
            worker.thread = std::thread([this] {
                while (!stop_flag) {
                    std::function<void()> task;
                    if (task_queue.pop(task)) {
                        task();
                    } else {
                        // 队列为空时让出 CPU
                        std::this_thread::yield();
                    }
                }
            });
        }
    }

    // 调度一个普通的函数或 Lambda
    template <class Func>
    void schedule(Func&& func) {
        task_queue.push([func = std::forward<Func>(func)] { func(); });
    }

    // 调度一个协程任务（这是协程的入口点）
    void schedule(CoroutineTask&& task) {
        if (task.handle) {
            // 从 task 对象中"窃取"句柄，并将其包装成一个启动任务
            auto handle = std::exchange(task.handle, nullptr);
            task_queue.push([handle] { handle.resume(); });
        }
    }

    // 用于被协程 co_await 重新调度
    void schedule_handle(std::coroutine_handle<> handle) {
        task_queue.push([handle] {
            if (!handle.done()) {
                handle.resume();
            }
            if (handle && handle.done()) {
                handle.destroy();
            }
        });
    }

    // 返回一个 Awaitable，用于在协程内部通过 `co_await` 进行调度
    ThreadPoolScheduler schedule_on_pool() noexcept { return ThreadPoolScheduler{this}; }

    void shutdown() {
        stop_flag.store(true);
        for (auto& worker : workers) {
            if (worker.thread.joinable()) {
                worker.thread.join();
            }
        }
    }

    ~CoroutineThreadPool() { shutdown(); }
};

// await_suspend 将当前挂起的协程句柄交由线程池重新调度
inline void ThreadPoolScheduler::await_suspend(std::coroutine_handle<> handle) const { pool->schedule_handle(handle); }

// 示例使用
CoroutineTask example_coroutine(CoroutineThreadPool& pool, int id,std::atomic<int>& counter) {
    std::cout << "Task " << id << " executed 1 on thread " << std::this_thread::get_id() << std::endl;
    co_await pool.schedule_on_pool();
    std::cout << "Task " << id << " executed 2 on thread " << std::this_thread::get_id() << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(2));
    counter--;
    co_return;
}

int main() {
    CoroutineThreadPool pool(32);

    std::cout << "Main thread ID: " << std::this_thread::get_id() << std::endl;
    // 调度普通函数
    pool.schedule([] { std::cout << "Lambda task executed on thread " << std::this_thread::get_id() << std::endl; });

    // 调度协程
    const int task_count = 10;
    std::atomic<int> task_counter = task_count;
    for (int i = 0; i < task_count; ++i) {
        std::this_thread::sleep_for(std::chrono::milliseconds(5));
        pool.schedule(example_coroutine(pool, i, task_counter));
    }
    std::cout << "All tasks scheduled. Main thread will now wait for pool to shutdown.\n";
    while (task_counter > 0) {
        std::this_thread::yield();
    }
    std::cout << "\nAll coroutine tasks have completed.\n";
    pool.shutdown();
    std::cout << "Pool has been shut down.\n";
    return 0;
}
```