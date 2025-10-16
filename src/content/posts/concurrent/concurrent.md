---
title: concurrent
published: 2025-04-16
description: 并发同步原语
# image: ./images/cover1.png
tags: [OS, 并发, 线程]
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
    atomic_int  __lock;         // 内部状态锁（自旋锁）
    int         __total_seq;    // 总等待序列号​​计数器 “挂号”
    int         __woken_seq;    // 已唤醒线程数计数器 “叫号”
    struct list_head __wait_queue; // 等待线程链表
    // 其他字段...
};
```
- 关键函数
pthread_cond_signal(): 锁住内部__lock, 选择__wait_queue等待线程的一个节点, 更新__woken_seq, 解锁内部__lock, futex_wake(__woken_seq), Kernel: 唤醒目标线程
pthread_cond_wait():   锁住内部__lock, 挂载到__wait_queue, 原子增加__total_seq, 进行唤醒检查, 如果需要等待则释放mutex, 解锁内部__lock, futex_wait(__woken_seq)
Fast-paths​：无竞争时完全在用户态操作
Slow-paths​​：竞争时触发系统调用进入内核
-   **`__lock`**: 这是一个用户态的自旋锁，用于保护 `pthread_cond_t` 结构内部其他字段的并发访问。当多线程同时调用 `pthread_cond_wait` 或 `pthread_cond_signal` 时，这个锁能确保操作的原子性。又由于对 pthread_cond_t 的操作通常很快，所以使用自旋锁可以避免陷入内核的开销。

- 通过无锁链表设计 CAS操作管理等待队列
    *   **`__wait_queue`**: 这是一个用户态的等待队列，通常实现为一个无锁链表，通过 CAS (Compare-and-Swap) 原子操作来管理节点的增删，以提高并发性能。当一个线程/进程需要等待条件变量时，它会创建一个节点并挂载到这个链表上（进入等待队列）。

- 双计数器设计(__total_seq & __woken_seq) 解决信号丢失问题
    *   **`__total_seq`**: 每当一个线程准备进入等待状态，它就会原子地增加 __total_seq 的值，记录总共有多少个线程曾经或正在等待（挂号）。
    *   **`__woken_seq`**: 每当 `pthread_cond_signal` 决定唤醒一个线程时，它会增加 __woken_seq 的值，记录已经发出了多少个唤醒信号（叫号）。
    通过比较这两个计数器的值，一个线程可以判断自己是否错过了唤醒信号，如果__woken_seq值变化，说明有叫号。P.S. 如果 `__woken_seq` 的值追上了 `__total_seq`，就意味着所有等待的线程都已经被唤醒了。

```c
int pthread_cond_signal(pthread_cond_t *__cond);
int pthread_cond_wait(pthread_cond_t *__restrict __cond, pthread_mutex_t *__restrict __mutex);

// 让调用线程进入睡眠，等待在用户态地址(uaddr)的值发生变化
int futex_wait(u32 __user *uaddr, unsigned int flags, u32 val, ktime_t *abs_time, u32 bitset);
// 唤醒最多nr_wake个在指定用户态地址(uaddr)睡眠的线程
int futex_wake(u32 __user *uaddr, unsigned int flags, int nr_wake, u32 bitset);
```
-   **`futex_wait` 系统调用**: 
    *   **原子检查**: 内核会原子地检查 `pthread_cond_t` 中一个与 `futex` 关联的整数（通常就是 `__woken_seq` 或者一个专门的 `futex` 变量）的值是否与 Waiter 在用户态读取的值相匹配。
    *   **进入休眠**: 如果值匹配（意味着在 Waiter 检查和调用 `futex_wait(__woken_seq)` 之间，没有新的唤醒发生），内核就会将 Waiter 线程的状态设置为休眠，并将其放入一个位于内核空间的、与该 `futex` 地址相关联的`等待队列`中。
    *   **避免竞争**: 如果值不匹配，说明在 Waiter 进入内核前，`pthread_cond_signal` 已经执行并修改了该 `futex` 变量的值。`futex_wait()` 会立即返回，Waiter 线程不会休眠，而是回到用户态重新检查条件并尝试获取锁。

#### 对于cv条件不满足进入等待队列到被唤醒的逻辑详解
`pthread_cond_wait` 的核心任务是：**原子地释放互斥锁并开始等待条件变量，当被唤醒后，重新获取互斥锁**。这其中涉及到用户态和内核态的紧密协作。
- 用户态操作 (Fast-Path and Slow-Path a)
    当一个线程（我们称之为 Waiter）调用 `pthread_cond_wait(&cv, &mutex)` 时，它必须已经持有了 `mutex`。
    *  **锁定内部状态**: Waiter首先会尝试获取 cv 内部的 `__lock`，以保护对 cv 结构的访问。
    *  **加入等待队列**: 在用户态将自己封装成一个节点，通过 CAS 操作加入到 `__wait_queue` 链表中。
    *  **修改与记录计数器**: 原子地增加 `__total_seq` 的值。此操作向潜在的 pthread_cond_signal调用者宣告其即将进入等待状态。读取并记录当前的 __woken_seq值，这是后续检查是否错过信号的关键。
    *  **释放互斥锁和内部锁**: Waiter会释放cv关联的 `mutex`（这一步允许其他线程（特别是发信号的线程）进入临界区修改共享条件），然后释放 cv 的内部锁 `__lock`。
    *  **检查是否需要等待（是否到号）**: 在真正进入内核等待之前，Waiter 会再次检查 `__woken_seq` 和它之前读取的值是否一致。如果在这期间，`pthread_cond_signal` 已经被调用，且 `__woken_seq` 的值“叫号”已经“追上”了它，那么 Waiter 就无需进入内核等待，可以直接尝试重新获取 `mutex`。即 **先挂号 (`__total_seq++`)，再检查是否已经叫到自己的号 (`__woken_seq`)**。
       1. Fast-Path 信号已到无需等待: 无需调用futex_wait，尝试重新获取mutex
       2. Slow-Path a 信号未到要等待: `futex_wait(&cv->__woken_seq, old_woken_seq)`, 内核检查cv->__woken_seq发现已经发生变化（叫到号了）futex_wait立即返回错误，线程再次尝试获取mutex
- 内核态操作 (Slow-Path b)：Futex 等待
    如果在上一步的检查中，Waiter 发现自己仍然需要等待（即没有叫到号），它就会**进入与该 `futex` 关联的内核等待队列中休眠**。

`pthread_cond_signal` 唤醒逻辑，当另一个线程（我们称之为 Signaler）
- 调用 `pthread_cond_signal(&cv)` 时：
    *   Signaler 同样会先获取cv的内部锁 `__lock`，检查 `__wait_queue` 是否为空 或者 比较 `__total_seq` 和 `__woken_seq` 来判断是否有等待的线程。如果没有等待者，直接释放 `__lock` 并返回，几乎没有开销。
    *   如果有等待者，Signaler 会增加 `__woken_seq` 的值。 这个增加操作是给即将休眠或正在休眠的 Waiter 线程看的。
    *   **`futex_wake` 系统调用**: Signaler 随后调用 `futex(FUTEX_WAKE, ...)`，并通常将唤醒数量设置为 1。
        *   内核接收到这个请求后，会找到与该 `futex` 地址关联的内核等待队列。
        *   它会从队列中取出一个（或指定的数量）Waiter 线程，将其状态从休眠改回就绪（Runnable）。
        *   被唤醒的 Waiter 线程会被放入调度器的就绪队列中，等待 CPU 时间片。
    *   Signaler 释放 `__lock`。

#### CV 唤醒后但是获取锁失败的逻辑详解
当 Waiter 线程从 `futex_wait()` 调用中返回（即被唤醒）后，它知道条件可能已经满足，但它**必须重新获取 `mutex`**才能安全地访问共享资源。
*   **唤醒后的第一件事**: Waiter 从内核态返回到用户态。此时，它知道自己被唤醒了但并不持有 `mutex`，调用 `pthread_mutex_lock(&mutex)` 来尝试获取互斥锁。
*   **获取锁失败的逻辑**:
    *   此时，`mutex` 可能仍然被 Signaler 线程持有（如果 Signaler 在调用 `pthread_cond_signal` 后还有其他操作），或者已经被另一个刚刚被唤醒的 Waiter 线程获取。
    *   如果 Waiter 尝试获取 `mutex` 失败，它会进入 `pthread_mutex_lock` 的慢速路径（contended case）。
    *   `pthread_mutex_lock` 的实现同样基于 `futex`。Waiter 会在一个循环中尝试通过原子操作（如 `cmpxchg`）获取锁。
    *   如果多次尝试失败，Waiter 就会调用 `futex_wait(&mutex, 1)`，**进入与该 `mutex` 关联的内核等待队列中休眠**。
    **关键点**: **一个从条件变量等待中被唤醒的线程，如果未能立即获得互斥锁，它会转而进入该互斥锁的等待队列，而不是条件变量的等待队列。**
*   **互斥锁的唤醒**: 当持有 `mutex` 的线程（无论是 Signaler 还是另一个 Waiter）调用 `pthread_mutex_unlock(&mutex)` 时，它会修改 `mutex` 的状态，并通过 `futex_wait(&mutex, 1)` 唤醒一个正在等待该 `mutex` 的线程。

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