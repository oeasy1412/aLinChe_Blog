---
title: concurrent
published: 2025-04-16
description: 并发同步原语
# image: ./images/cover1.png
tags: [OS, 并发，线程]
category: 并发
draft: false
---

# 并发编程中的同步问题，硬件和软件在同步中的作用

## 0. thread.h
```cpp
#include <atomic>
#include <cassert>
#include <chrono>
#include <condition_variable>
#include <cstdlib>
#include <functional>
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

#define NTHREAD 64
enum ThreadStatus { T_FREE = 0, T_LIVE, T_DEAD };

struct Thread {
    int id;
    std::atomic<int> status;
    std::thread thread;
    std::function<void(int)> entry;
};

class ThreadPool {
  private:
    std::vector<Thread> threads;
    std::vector<std::function<void(int)>> task_queue;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop = false;

  public:
    ThreadPool() : threads(NTHREAD) {
        // 预先启动所有线程，等待任务
        for (int i = 0; i < NTHREAD; ++i) {
            threads[i].id = i + 1;
            threads[i].status.store(T_LIVE);
            threads[i].thread = std::thread([this, i]() {
                while (true) {
                    std::unique_lock<std::mutex> lock(this->mtx);
                    this->cv.wait(lock, [this]() { return !this->task_queue.empty() || this->stop; });
                    if (this->stop && this->task_queue.empty()) {
                        break;
                    }
                    if (!this->task_queue.empty()) {
                        auto task = this->task_queue.front();
                        this->task_queue.erase(this->task_queue.begin());
                        lock.unlock();
                        task(threads[i].id);
                        threads[i].status.store(T_FREE);
                    }
                }
            });
        }
    }

    void create(const std::function<void(int)>& fn) {
        std::lock_guard<std::mutex> lock(mtx);
        task_queue.push_back(fn);
        cv.notify_one();
    }

    void join() {
        stop = true;
        cv.notify_all();
        for (auto& thread : threads) {
            if (thread.thread.joinable()) {
                thread.thread.join();
            }
        }
    }

    ~ThreadPool() { join(); }
};
```

## 1. 原子操作 (Atomic Operations)
- 不可中断的指令序列，保证内存操作的完整性
  - 现代CPU通过硬件支持（如缓存一致性协议MESI）实现原子性
```cpp
#include "thread.h"
#include <atomic>

using namespace std;

constexpr int N = 100000;
// int sum = 0; // Racing (<100000)
atomic<int> sum(0);
atomic_flag flag = ATOMIC_FLAG_INIT;

void Tsum(int id) {
    for (int i = 0; i < N; ++i) {
        // sum++;
        sum.fetch_add(1, std::memory_order_relaxed); // atomic_fetch_add(&sum, 1);
    }
}
int main() {
    ThreadPool tpool;
    tpool.create(Tsum);
    tpool.create(Tsum);
    this_thread::sleep_for(chrono::seconds(1));
    cout << "sum = " << sum << "\n";
    cout << "end\n";
}
```
### std::atomic 内存序 
> memory_order_relaxed ：最弱的内存序，仅保证基本的原子性，没有额外的内存同步语义。

> memory_order_consume ：用于消费操作(consume operation)，确保后续的操作不会被重排序到该操作之前。用于后续操作依赖于该操作的结果。

> memory_order_acquire ：用于获取操作(acquire operation)，通常用于读取共享变量，确保后续的操作不会被重排序到该获取操作之前，提供了一种从发布-获取同步模型中的获取操作的语义。

> memory_order_release ：用于释放操作(release operation)，通常用于写入共享变量，确保之前的操作不会被重排序到该释放操作之后，为后续的获取操作提供一种同步点。

> memory_order_acq_rel ：结合了 acq 和 rel 的语义，用于读写操作，既确保之前的写操作不会被重排序到该操作之后，也确保后续的读操作不会被重排序到该操作之前。

> memory_order_seq_cst ：最强的内存序，提供顺序一致性(sequentially consistent)的保证，确保所有线程中的操作都按照全局的顺序执行。
### CAS 比较并交换 (Compare-And-Swap)
```cpp
if (!global_var.compare_exchange_strong(expected_val, new_val, std::memory_order_acq_rel)) {}
```
### LR/SC 加载保留 (Load-Reserve) / 条件存储 (Store-Conditional)
```cpp
atomic<int> global_var(0);
atomic<bool> sc_failed(false);
void LR_SC() { // Load-Reserved && Store-Conditional
    int expected_val;
    while (true) {
        expected_val = global_var.load(std::memory_order_relaxed); // Load-Linked
        int new_val = expected_val + 1;
        // Store-Conditional
        if (!global_var.compare_exchange_strong(expected_val, new_val, std::memory_order_acq_rel)) {
            sc_failed.store(true, std::memory_order_relaxed);
            break;
        }
        // 如果 Store-Conditional 成功，继续循环
    }
}
```

## 2. lock 锁
### 自旋锁 (Spinlock)
```cpp
inline void spin_lock(std::atomic_flag& lock_flag) {
    while (lock_flag.test_and_set(std::memory_order_acquire)) { // CAS
        // 自旋等待，直到锁被释放
    }
}
inline void spin_unlock(std::atomic_flag& lock_flag) { lock_flag.clear(std::memory_order_release); }
```
### 互斥锁 (mutex)
```cpp
inline void mutex_lock(std::mutex* lk) { lk->lock(); }
inline void mutex_unlock(std::mutex* lk) { lk->unlock(); }
```
### TODO futex

## 3. cv 条件变量 (Condition Variable)
### 1. 用户态cv等待队列：
```cpp
struct pthread_cond {
    atomic_int  __lock;         // 内部状态锁
    int         __total_seq;    // 总唤醒信号计数器
    int         __woken_seq;    // 已唤醒线程计数器
    struct list_head __wait_queue; // 等待线程链表
    // 其他字段...
};
```
- pthread_cond_signal(): 锁住内部__lock, 若__wait_queue非空取头节点, 更新__woken_seq, 解锁内部__lock, futex_wake(__woken_seq), Kernel: 唤醒目标线程
- pthread_cond_wait():   锁住内部__lock, 挂载到__wait_queue, 原子增加__total_seq, 解锁内部__lock
- Fast-paths​：无竞争时完全在用户态操作
- Slow-paths​​：竞争时触发系统调用进入内核
- 无锁链表设计 CAS 操作管理队列
- 双计数器设计（__total_seq/__woken_seq）解决信号丢失问题

### 2. Linux内核态cv等待队列：(wait_queue_head_t)
```cpp
// TODO
void wake_up(wait_queue_head_t *q) {
    struct task_struct *task;
    list_for_each_entry(wait, &q->head, entry) {
        task = wait->private;
        try_to_wake_up(task); // 调用核心调度器
    }
}
```

## 3. 经典 pc 生产者消费者问题
```cpp
queue<int> q;
mutex mtx;
condition_variable cv;
bool done = false;
void Tproducer(int id) {
    this_thread::sleep_for(chrono::seconds(1));
    for (int i = 0; i < 20; ++i) {
        lock_guard<mutex> lock(mtx);
        q.push(i);
        cout << "Producer " << id << " produced the " << i << "\n";
        cv.notify_one(); // 通知消费者
    }
}
void Tconsumer(int id) {
    while (true) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [] { return !q.empty() || done; }); // 等待条件成立
        if (done && q.empty()) {
            break;
        }
        int value = q.front();
        q.pop();
        cout << "Consumer " << id << " consumed " << value << "\n";
    }
}

int np = 2, nc = 3;
thread producers[np];
thread consumers[nc];
```

## 4. 信号量 Semaphore
- 优势
  - 信号量可用于​​跨进程间的同步
  - 轻量级事件通知，​​避免惊群效应
```c
// Linux
struct semaphore {
    raw_spinlock_t lock;    // 自旋锁保护计数器
    unsigned int count;     // 可用资源数
    struct list_head wait_list; // 等待队列
};
struct semaphore_waiter {
    struct list_head list;
    struct task_struct *task;
    bool up;
};
void __sched down(struct semaphore *sem) {
	unsigned long flags;
	raw_spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		__down(sem); // 加入等待队列
	raw_spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(down);
void __sched up(struct semaphore *sem) { // V操作
    unsigned long flags;
    raw_spin_lock_irqsave(&sem->lock, flags);
    if (likely(list_empty(&sem->wait_list)))
        sem->count++;
    else
        __up(sem); // 唤醒队列头
    raw_spin_unlock_irqrestore(&sem->lock, flags);
}
static noinline void __sched __down(struct semaphore *sem) {
	__down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}
static noinline void __sched __up(struct semaphore *sem) {
	struct semaphore_waiter *waiter = list_first_entry(&sem->wait_list,
						struct semaphore_waiter, list);
	list_del(&waiter->list);
	waiter->up = true;
	wake_up_process(waiter->task);
}
```
```cpp
// cpp 示例
class Semaphore {
  private:
    mutex mtx;
    condition_variable cv;
    unsigned int count;
  public:
    Semaphore(unsigned int initial_count) : count(initial_count) {}
    void acquire() { // P操作
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [this] { return count > 0; });
        --count;
    }
    void release() { // V操作
        unique_lock<mutex> lock(mtx);
        ++count;
        cv.notify_one();
    }
};
```