---
title: Barrier
published: 2025-04-16
description: Memory Barrier
tags: [OS, 并发, 内存管理, 编译原理, cpp]
category: 并发
draft: false
---

本文将深入探讨并发编程中的核心概念——内存屏障(Memory Barrier)，并系统性地阐述相关的 volatile 限定符、原子操作(Atomics) 以及内存顺序(Memory Order)。我们将从编译器和CPU体系结构两个层面，剖析这些工具如何协同工作，以构建正确且高效的多线程程序。

### **1. `volatile` 修饰符**
`volatile` 关键字的核心作用是**告知编译器，被修饰变量的值随时可能在编译器无法察觉的情况下发生改变**。例如通过硬件中断服务例程(ISR)、内存映射的硬件设备(MMIO)或信号处理函数。因此，它对编译器施加了两个关键约束：
*   **抑制编译器优化**：确保每次访问 `volatile` 变量时，都直接从其内存地址**读取**或**写入**，**而不是使用寄存器中缓存的旧值**。
*   **防止编译器重排**：编译器不会将对 `volatile` 变量的访问指令与其他 `volatile` 变量的访问指令进行重排序。

**一个经典的例子：（常见于嵌入式）**
```cpp
// done 变量可能被硬件、中断服务程序或其他线程修改
// volatile 确保编译器每次循环都会重新从内存加载 done 的值，而不是在寄存器中不断读取
volatile bool done = false; 
while (!done) {
    // 等待 done 变为 true
}
```

**`volatile` 的局限性：**
关键在于，`volatile` **仅对编译器**有效，它**无法限制 CPU 的乱序执行(Out-of-Order Execution)**。现代 CPU 为了最大化提升性能，可能会对内存操作进行重排序。因此，`volatile` **不能提供跨线程的内存可见性和顺序性的同步保证，也不能保证操作的原子性**。在多线程编程中，单独使用 `volatile` 来保证共享变量的可见性和顺序性是**错误且危险**的。
#### **澄清：volatile 无法解决数据竞争**
在 C++ 标准中，当多个线程访问同一个非原子变量，并且至少有一个是写操作(副作用)时，就会产生**数据竞争(Data Race)**，而数据竞争的后果是**未定义行为(UB, Undefined Behavior)**。
volatile其实无法解决 Racing 问题。它既不能保证操作的原子性（例如，一个线程正在写一个 `volatile` 变量时，另一个线程可能读到一半的值），也不能提供跨线程的内存顺序保证。因此，它无法消除数据竞争，也就无法避免未定义行为。


### **2. atomic = 原子操作 + 内存顺序: 现代并发编程的基石**
为了解决 `volatile` 的局限性，C++11 标准引入了原子操作(`std::atomic`)和内存顺序(`std::memory_order`)。
*   **原子性 (Atomicity)**：保证一个操作（如读、写、修改）在执行过程中不会被其他线程中断。它要么完全执行，要么完全不执行，不存在中间状态。这是通过特殊的 CPU 指令（如 x86 的 `LOCK` 前缀）实现的。
*   **内存顺序 (Memory Ordering)**：定义了原子操作如何影响其他内存操作的可见性顺序，即构造一个事件的“发生在…之前”(happens-before) 的关系，从而限制编译器和 CPU 的乱序行为。
std::atomic 有两个功能：原子 & 内存序
**原则：**
> 大多数变量无需是原子的。只有当一个变量可能被多个线程同时访问，且至少有一个访问是写操作时，才必须将其声明为原子类型。这能帮助编译器和 CPU 精确地知道哪些内存访问需要特别保护，从而在保证并发安全的同时，最大化对其他代码的优化。


### **3. 内存屏障(Memory Barrier)：显式控制乱序**
> 编译器优化、CPU、缓存 都有可能导致乱序执行

内存屏障是用于在代码中创建同步点的指令，它可以阻止编译器和 CPU 跨越屏障进行指令重排。
#### **编译器屏障 (Compiler Barrier)**
```cpp
// 告诉编译器，内存中的所有内容都可能已改变
// 禁止编译器将屏障前的内存读写指令重排到屏障后，反之亦然
asm volatile("" ::: "memory");
```
*   **作用**：仅对编译器有效，禁止编译器跨屏障重排内存访问指令，强制编译器将屏障前的所有内存读写操作完成，并让屏障后的内存读取操作**重新从内存加载**。
*   **局限**：与 `volatile` 类似，它无法阻止 CPU 级别的乱序执行。

#### **硬件内存屏障 (Hardware Memory Barrier)**
硬件内存屏障通过特定的 CPU 指令，直接作用于 CPU，以确保内存操作的顺序性。
```c
// GCC/Clang 内置函数，生成一个全功能的内存屏障
__sync_synchronize(); 
```
*   **作用**：这是一条“重量级”指令，通常会生成如 x86 的 `mfence` 或 ARM 的 `dmb ish` 指令。它确保**屏障之前的所有内存读写操作，必须在屏障之后的任何内存读写操作开始之前，对其他核心全局可见**。
*   **对应关系**：在 C++ 原子操作中，`std::memory_order_seq_cst` 通常会产生一个完整的硬件内存屏障。


### **4. 内存序 `std::memory_order`** (重点)
内存顺序是 C++ 并发编程中最精细也最复杂的工具，它允许程序员根据具体场景选择不同强度的同步保证，以实现性能和正确性之间的最佳平衡。
*   `memory_order_relaxed`：**宽松序**。 这是最弱的内存序，仅保证操作的原子性，不提供任何跨线程的顺序保证。没有额外的内存同步语义，允许指令自由重排（与架构有关）。
*   `memory_order_acquire`：**获取序**。 通常用在读操作上。它建立了一个“获取屏障”，**禁止该操作之后的所有内存操作被重排到该操作之前**。它必须与一个 `release` 操作配对，以观察其写入的数据。
*   `memory_order_release`：**释放序**。 通常用在写操作上。它建立了一个“释放屏障”，**禁止该操作之前的所有内存操作被重排到该操作之后**。所有在 `release` 操作之前发生的写操作，对于之后执行相应 `acquire` 操作的线程都是可见的。
*   `memory_order_acq_rel`：**获取-释放序**。 同时具备 `acquire` 和 `release` 的特性，通常用于“读-修改-写”类型的操作。它既能“获取”其他线程 `release` 的数据，又能向其他线程 `release` 自己的写入。
*   `memory_order_consume`：**消费序**。这是一个与 `acquire` 类似的较弱版本，仅对存在依赖关系的操作施加顺序限制。由于其复杂性和实现难度，目前主流编译器通常会将其提升为 `memory_order_acquire` 对待，因此在实践中较少使用。
*   `memory_order_seq_cst`：**顺序一致性**。 这是最强的内存序，也是默认的内存序。它不仅提供 `acquire` 和 `release` 的保证，还确保所有线程看到的 `seq_cst` 操作都遵循一个单一的、全局的总顺序。 这意味着不会出现“凭空加载”(out-of-thin-air reads) 等现象，但性能开销也最大。

显然，memory_order是有性能开销的：
#### 深度解析：C++内存序 与 CPU缓存架构 的性能博弈
CPU核心的缓存子系统: 从硬件层面剖析`std::memory_order`不同选项如何与**存储缓冲区**(`Store Buffer`)**、无效化队列**(`Invalidate Queue`)等机制交互，从而揭示其性能开销的根源。
##### 核心前提：现代CPU的性能优化与乱序执行
为隐藏内存访问延迟，现代CPU普遍采用**多级缓存**、**乱序执行(Out-of-Order Execution)** 和**存储转发(Store-to-load forwarding)** 等优化手段。其中两个关键组件是：
*   **存储缓冲区(Store Buffer)**：当一个CPU核心执行写操作时，数据被临时置于此缓冲区，核心可立即执行后续指令，无需等待数据写入L1缓存。这极大地隐藏了写延迟。
*   **无效化队列(Invalidate Queue)**：当一个核心修改了其缓存中的数据后，它会通过缓存一致性协议（如MESI）向其他核心广播“无效化”消息。其他核心接收到消息后，会将其放入无效化队列，并在适当时机处理，以避免阻塞当前正在执行的指令。

这些优化虽然提升了单核性能，却打破了程序代码顺序与实际执行顺序的一致性，为多线程编程带来了挑战。内存序正是用于在编译器和硬件层面施加约束，以重建跨线程的可见性和顺序性。
#### 不同内存序的硬件级成本分析
| 内存序 | 核心同步属性 | 编译器与硬件重排约束 | 典型硬件实现（指令/屏障） | 性能影响 |
| :--- | :--- | :--- | :--- | :--- |
| **`memory_order_relaxed`** | 仅保证单次操作的原子性 | 几乎无约束，允许指令自由重排 | 普通的`MOV`或`ADD`等原子指令 | **极低**。几乎不干扰CPU的优化流水线 |
| **`memory_order_acquire/release`** | 建立成对的线程间同步关系(Happens-Before) | 阻止特定方向的局部重排 | **Release**: 可能需要写屏障(Store Barrier) <br> **Acquire**: 可能需要读屏障(Load Barrier) | **中等**。引入定向屏障，产生可控的同步开销 |
| **`memory_order_seq_cst`** | 建立所有`seq_cst`操作的全局单一总顺序 | 禁止几乎所有指令重排 | 可能需要全功能内存屏障(Full Memory Fence)，如x86的`MFENCE`或`LOCK`前缀指令 | **高**。可能导致流水线停顿，严重影响性能 |

##### 0. 普通变量：完全无约束非原子变量
普通变量的读写，编译器可能会进行非常激进的优化，比如将变量长时间保留在寄存器中，根本不写回缓存，从而导致其他线程完全无法观察到其变化。
##### 1. `memory_order_relaxed`：无约束的执行
`relaxed`内存序仅确保操作的原子性，即不会发生指令撕裂。它对编译器和CPU的乱序执行不施加任何额外的限制。
*   **硬件交互**：
    *   **写操作**(`store`): 一个`relaxed`写操作会将其值放入当前核心的**存储缓冲区**，CPU流水线可以无缝地继续执行后续指令，直到缓存系统准备好（例如，获得了对应Cache Line的独占权），再异步地被刷新到L1缓存。该值何时被刷新到L1缓存并对其他核心可见，是不确定的。（但这也正是relaxed操作几乎无额外开销的原因）
    *   **读操作**(`load`): 一个`relaxed`读操作可以从本地缓存、甚至直接从存储缓冲区（若发生Store-to-load forwarding）获取数据。它不会等待其他核心的更新。
    *   **结论**：此模型完全拥抱CPU的缓存和乱序优化，只保证自身原子性，几乎没有额外开销，但无法用于线程间的状态同步。
##### 2. `memory_order_acquire` / `memory_order_release`：定向的同步信道
这对内存序是实现高效线程间同步的关键。它们必须配对使用，共同在生产者和消费者线程之间建立明确的“先行发生（Happens-Before）”关系。
*   **硬件交互**：
    *   **`acquire` 读操作**: 它扮演一个“获取屏障”的角色。该指令会确保所有在`acquire`操作之后的内存读写，必须在该`acquire`操作完成**之后**才能开始执行。硬件层面，这可能要求CPU**处理其无效化队列中所有待处理的条目**，确保本地缓存状态是“最新”的，然后再执行`acquire`读操作。这保证了当前线程能够正确地“看到”由其他线程通过`release`操作发布的所有数据。
    *   **`release` 写操作**: 它扮演一个“释放屏障”的角色。该指令会确保所有在`release`操作之前的内存读写，其结果必须在`release`操作本身对其他核心可见**之前**完成。在硬件层面，这通常意味着**强制清空(drain)存储缓冲区**，确保所有缓冲区的写操作都已提交到L1缓存。只有这样，`release`的写操作才能被提交，从而确保数据对其他核心是“可发布的”。
    *   **结论**：Acquire-Release通过定向的内存屏障，在特定线程间构建了高效的同步通道，其开销仅限于清空存储缓冲区和处理无效化队列，相比全局屏障更为精准和低廉。
##### 3. `memory_order_seq_cst`：全局的执行仲裁者
这是最强、最直观，同时也是开销最高的内存序。它不仅具备`acquire`和`release`的所有特性，还额外保证**所有线程看到的全部`seq_cst`操作都遵循一个唯一的、全局一致的顺序**。
*   **硬件交互**：
    *   要实现全局单一顺序，`seq_cst`操作需要插入**全功能内存屏障(Full Memory Fence)**。
    *   在x86这类强内存模型架构上，一个`seq_cst`写操作通常会编译成带有`LOCK`前缀的指令（如`XCHG`），这本身就隐含了一个全功能的内存屏障。而在ARM等弱内存模型架构上，则可能需要显式的屏障指令（如`DMB SY`）。
    *   这个屏障是一个极其“昂贵”的操作。它会：
        1.  **暂停指令派发**，等待流水线中所有已执行的内存操作完成。
        2.  **完全清空存储缓冲区**，并等待所有写操作被系统确认。
        3.  **处理完整个无效化队列**。
    *   这种行为**严重破坏了CPU的乱序执行和缓存优化**，导致显著的流水线停顿（Pipeline Stall），是其高昂性能成本的直接原因。

#### fetch_add()
```cpp
std::atomic<int> count = 0;
count.fetch_add(1, std::memory_order_relaxed); // (x86)lock add ✅
// 类似于（但其实是一个不可分的 lock add 原子指令操作）
// r0 = load // relaxed
// add r0, r0, #1
// store r0  // relaxed
count.fetch_add(1, std::memory_order_acq_rel); // (x86)lock add ✅
// r0 = load // acquire
// add r0, r0, #1
// store r0  // release

// 等价于 .fetch_add(1, std::memory_order_seq_cst); ✅
count += 1; // count++; ++count;    

// 等价于 count.store(count.load() + 1); // 是3条可分的指令，不保证原子性，线程不安全 ❌
count = count + 1;

// Q: 现在我知道了count = count + 1;线程不安全，但是我怎么处理自定义运算呢？
// A: CAS
// P.S 如果 count 是会被其他线程读取的，需改为release的CAS，反之可以直接全部使用relaxed
int old_count = count.load(std::memory_order_relaxed);
int new_count;
do {
    new_count = f(old_count) // 自定义运算(不依赖于其他共享数据)
} while (!count.compare_exchange_weak(old_count, new_count, std::memory_order_release, std::memory_order_relaxed));

```

### **5. 示例代码**
这个 `release-acquire` 模型是并发编程中最常用、最高效的同步范式之一。
```cpp
#include <atomic>
#include <thread>
std::atomic<bool> done {false};
void worker() {
    // 生产者线程完成一些工作后...
    // 使用 memory_order_release，确保在 done = true 之前的所有写操作
    // 对于消费线程都是可见的。
    done.store(true, std::memory_order_release);
}
void waiter() {
    // 消费者线程等待工作完成
    // 使用 memory_order_acquire，确保在读取到 done == true 后
    // 能看到生产者在 release 之前的所有写操作。
    while (!done.load(std::memory_order_acquire)) {
        // 等待
    }
    // 在此之后，可以安全地访问生产者写入的数据
}
```
```cpp
// 自旋锁
struct SpinMutex {
    std::atomic<bool> flag{false};
    bool try_lock() {
        bool expected = false;
        if (flag.compare_exchange_strong(expected, true, std::memory_order_acquire, std::memory_order_relaxed))
            // load barrier
            return true;
        return false;
    }
    void lock() {
        bool expected = false;
        while (!flag.compare_exchange_weak(expected, true, std::memory_order_acquire, std::memory_order_relaxed))
            expected = false; // 因为STD的CAS中 weakCAS失败后会把传入的引用(expected)修改，只能手动赋值为false重置。。
        // load barrier
    }
    void unlock() {
        // data change and then unlock.
        // store barrier
        flag.store(false, std::memory_order_release);
    }
    // 使用 futex 等待队列 优化自旋
    void lock_futex() {
        bool expected;
#if __cpp_lib_atomic_wait
        int retries = 1000;
        do {
            expected = false;
            if (flag.compare_exchange_weak(expected, true, std::memory_order_acquire, std::memory_order_relaxed))
                // load barrier
                return;
        } while (--retries);
#endif
        do {
#if __cpp_lib_atomic_wait
            // fast-user-space mutex = futex (linux) SYS_futex
            flag.wait(true, std::memory_order_relaxed); // wait until not true
#endif
            expected = false;
        } while (!flag.compare_exchange_weak(expected, true, std::memory_order_acquire, std::memory_order_relaxed));
        // load barrier
    }
    void unlock_futex() {
        // data change and then unlock
        // store barrier
        flag.store(false, std::memory_order_release);
#if __cpp_lib_atomic_wait
        flag.notify_one();
#endif
    }
};
```

### **6. 不同架构的内存模型的区别：**
`x86` 架构：强内存模型(TSO, Total Store Order 完全存储定序)，除 `StoreLoad` 外，基本不允许其他重排。
`ARM` 架构：弱内存模型，允许 LoadLoad, LoadStore, StoreStore等多种重排。
- 对 memory_order_relaxed 来说：
  - x86因架构本身内存限制已很强，跟seq_cst只有禁止StoreLoad重排、保证全局顺序一致性的区别
  - ARM则会允许各种重排，内存序relax只保证该原子操作本身是原子的，别的不管。
- 对 memory_order_seq_cst ：
  - ARM 使用指令集 stlr (=dmb ish + str + dmb ish), ldar
> P.S. 没有 ARM 架构的PC完全可以考虑手机下载 `Termux`，然后电脑ssh上去编译cpp代码来观察其内存序行为(pkg install clang)

- Q: 性能动机: 为什么x86“独爱”StoreLoad重排，只允许了它的存在
  - A: 核心在于写操作Store的延迟远高于读操作Load。当一个CPU核心要写入一个内存地址时，它必须首先通过缓存一致性协议（如MESI）获得该地址所在缓存行(Cache Line)的独占所有权(Exclusive)。这个过程可能非常耗时：对非独占的缓存行，此核心必须发送“读取并无效化”(**Read For Ownership**)的请求，等待其他核心将缓存行失效并把最新数据发送过来；甚至cache miss导致要从主存中加载，延迟更高。导致整个执行流水线都必须停顿！
  - `Store-Load`重排的实际流程：不等待直接将“STORE x, 数据”这个写操作放入**存储缓冲区**，然后**立即**继续执行下一条指令。执行Load指令。此时，内存控制器并行地在后台缓慢地处理存储缓冲区中的写请求，以及Load指令的读请求，实现对高延迟的写操作“异步化”。

### **总结**
*   **`volatile`**：用于与内存映射的硬件交互或在信号处理程序中使用（如：嵌入式），**不用于线程间同步**。
*   **编译器屏障**：仅阻止编译器重排，无法阻止硬件乱序。
*   **硬件屏障**：功能强大但开销高，通常通过原子操作间接使用。
*   **原子操作 + 内存顺序**：是现代 C++ 中进行多线程编程的**正确且唯一可靠**的方式。它通过提供精确的原子性和内存顺序保证，从根本上解决了数据竞争和未定义行为的问题。
*   **认识到架构差异**: 为编写可移植的正确代码，必须依据C++标准，而非某种特定架构的行为。所以x86开发者（强内存模型）也需要学习完整的内存序知识并在CI中添加ARM架构测试（
