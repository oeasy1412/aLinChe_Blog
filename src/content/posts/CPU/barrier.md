---
title: Barrier
published: 2025-04-16
description: Barrier
# image: ./images/cover1.png
tags: [OS, 编译原理]
category: 编译原理
draft: false
---

## volatile 修饰符​
- 确保编译器不会优化对变量的读写操作
  - 每次读取变量时强制从内存加载新值（非 CPU 寄存器缓存）
  - 每次写入变量时立即同步到内存
    - 仅作用于编译器优化，对 CPU 乱序执行无效，不提供内存顺序保证（如读/写重排序）
```cpp
// 防止编译器将 !done 优化为只检查一次（如循环外）
bool volatile done = false;
while (!done) {}
```

## 编译器 Barrier​
```cpp
asm volatile("" ::: "memory");
```
- 禁止编译器跨屏障重排内存访问指令
- 强制编译器在屏障前完成所有待定内存访问，屏障后重新加载所需变量
  - 适用于防止编译器优化导致的逻辑错误

## 硬件内存 Barrier​
```c
// Full Barrier
__sync_synchronize(); // GCC 内置函数
```
- 强制执行 CPU 级别的全局内存顺序一致性
  - 生成特定 CPU 指令（如 x86 的 mfence，ARM 的 dmb），确保屏障前的所有内存操作（Load/Store）​​全局可见​​后，才执行屏障后的操作。
  - 

## memory_order_acquire
> relaxed, consume, acquire, release, acq_rel, seq_cst

## 原子操作
```cpp
#include <stdatomic.h>
atomic_bool done = ATOMIC_VAR_INIT(false);
void wait() {
    while (!atomic_load_explicit(&done, memory_order_acquire));
}
```