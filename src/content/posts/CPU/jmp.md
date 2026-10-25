---
title: jmp
published: 2026-10-24
description: 跳转指令
tags: [cpp, CPU, 编译原理, perf]
category: CPU
draft: false
---

## 控制流指令的“五大类”：硬件视角的真实面貌
在高级语言中，我们有 `if`、`for`、`while`、`switch`、`函数调用` 等丰富的控制流语法；但在 x86-64 等现代 CPU 架构眼中，控制流（改变 RIP/EIP 指令指针）在硬件层面基本上只分为 五大类 “跳转”。

| 类别 | 典型汇编指令 | 目标地址从哪来？ | 是否压栈？ | 常见的 C++ 源码映射 |
| :--- | :--- | :--- | :--- | :--- |
| **直接条件跳转** | `je`, `jne`, `jg` | 指令里的立即数（编译期固定） | 否 | `if-else`链；for/while循环判定等条件回边 |
| **直接无条件跳转** | `jmp .label` | 指令里的立即数（编译期固定） | 否 | 普通 `goto`；循环中的`continue`和for(;;)等无条件回边、已知目标的尾调用(TCO)`jmp func` |
| **间接跳转** | `jmp reg` 或 `jmp [mem]` | **寄存器或内存（运行期读取）** | 否 | **switch跳转表**、计算goto、间接TCO |
| **间接调用** | `call reg` 或 `call [mem]` | **寄存器或内存（运行期读取）** | **是** | 虚函数调用`vtable`、函数指针调用、`std::function`类型擦除类模板 |
| **返回指令** | `ret` | 栈顶弹出（硬件返回栈RSB） | 弹出 | `return`语句 |

注：x86-64 的 call symbol 在硬件上等价于一条原子指令一次性完成三件事（仅用于理解硬件行为，不是实际情况）：
```asm
lea  rax, [rip + 0]  ; 算出返回地址(call的下一条指令的地址，不是这里演示的push rax)
push rax             ; 返回地址压栈
jmp  symbol          ; 跳转到目标地址
```

---

## 跳转表：O(1) 的秘密
- 因为switch-case中 case 只接受**整型常量表达式**，在开启编译优化（如 `-O2`）的情况下，编译器可能不会生成 多次 `if-else` 的比较，而是在程序的 `.rodata` (**只读数据段**) 中生成一块特殊的数组，**跳转表(Jump Table)**。跳转表的本质是**数组索引寻址**
```cpp
// case 只接受 编译后整型常量表达式
constexpr int f(int x) { return x; }
switch (x) {
    case 0:     puts("A"); break;
    case f(1):  puts("B"); break;
    case 2:     puts("C"); break;
    case 333:   puts("D"); break;
    default:    puts("?");
}
```
如果生成跳转表，其数据段内存布局示意图如下：
```text
┌────────────────────────────────────────┐
│  jump_table (只读数据段，占 2672 字节):  │
│  [0]   → .Lcase0   的内存地址           │  ← puts("A")
│  [1]   → .Lcase1   的内存地址           │  ← puts("B")
│  [2]   → .Lcase2   的内存地址           │  ← puts("C")
│  [3]   → .Ldefault 的内存地址           │  ← 填充 default（case 3 不存在）
│  [4]   → .Ldefault 的内存地址           │  ← 填充 default
│  ...                                   │
│  [332] → .Ldefault 的内存地址           │  ← 填充 default
│  [333] → .Lcase333 的内存地址           │  ← puts("D")
└────────────────────────────────────────┘
```

如果生成跳转表，其核心汇编逻辑（伪代码）：**
```asm
cmp     edi, 333               ; 1. 边界检查：x 是否超过最大 case 值？
ja      .default               ;    如果越界（ja无符号比较下负数为很大的正数），无条件跳转 default
jmp     [jump_table + rdi * 8] ; 2. 间接跳转：以 x 为索引查表，O(1)一步到位
```

- 所以你会发现：跳转表本质上是用 1 次缓存读取（如果触发间接跳转），换掉了 $N-1$ 次的分支比较，这是经典的“空间换时间”。
- 如果编译器优化的时候认为生成跳转表划不过来，就会考虑别的优化方式

比如前面说的例子中，显然只读数据段开2672字节换3个分支跳转是不合适的，只有case值数量多且密集才应该开跳转表，所以编译器实际上会考虑改用二分查找决策树，belike：
```asm
cmp     edi, 2
je      .Lcase2
jbe     .Lcase0_or_1
cmp     edi, 333
je      .Lcase333
jmp     .Ldefault
```

---

## CPU 硬件 分支预测 与 性能陷阱
到了这一步，你可能觉得“开启 -O2 且 case 密集的 switch 一定是最快的”。**但真实的物理世界并非如此。** 这里我们需要补充一个高阶知识点：**分支预测硬件的差异**。

前面提到，if是直接条件跳转，而跳转表是间接跳转。这两者在 CPU 内部使用的预测硬件完全不同：
*   **条件跳转(if)**: 分支预测 使用的是 **`PHT`(Pattern History Table)**，模式历史表。它只记录“跳/不跳”，现代 CPU 采用局部/全局历史关联（如感知器算法），对有规律的逻辑预测准确率高达 99%。（老一点的会用 `BHB` 本质是 PC + “11强跳 / 10弱跳 / 01弱不跳 / 00强不跳” 的 2-bit计数器）
*   **间接跳转(跳转表)**: 目标地址预测 使用的是 **`BTB`(Branch Target Buffer)** 以 PC 为索引的缓存表 记录最近一次该PC跳到的地址 + **`IBP`(Indirect Branch Predictor)** 以 PC+全局历史BHB 的哈希 为索引的缓存表。它不仅要预测跳不跳，还要预测“具体跳到哪个内存地址”。

**性能陷阱：**

如果你写了一个 字节码解释器（Virtual Machine），打算用一个巨大且密集的 `switch` 来分发 Opcode。由于 Opcode 的执行序列几乎是**完全随机**的，CPU 的间接分支预测器会频繁预测失败(Miss)。

每次预测失败，CPU 需要清空流水线(Pipeline Flush)，代价高达 15~20 个时钟周期。这种情况下，看似 $O(1)$ 的跳转表，其性能可能被预测失败的清空流水线惩罚而拖垮。有时候，手动把解释器改成一棵可预测性稍强的 $O(\log N)$ 二分查找树，甚至可能获得反直觉的性能提升。

### 怎么让 CPU 猜得到，甚至不用猜
1. **条件分支jcc猜错**：
- 让规律显形：在热路径前对数据排序或重组；
- 无分支写法：没有分支，就没有伤害。
算术与位运算替代分支：
```cpp
if (x > 0 && x < 1) [[likely]] {
    sum += val;
}
// 修改为
int mask = - (x > 0 && x < 1); // -1(0xFFFFFFFF) 或 0
sum += val & mask;
```
编译器 if-conversion 生成 `cmov`（-O2 会自动做；想看清真实分支可用 `-fno-if-conversion -fno-if-conversion2` 关掉对比）。这笔账可以量化：`CPI ≈ 1 + miss率 × 冲刷代价`，流水线越深越亏：
```cpp
int max(int x, int y) {
    return x > y ? x : y;
}
// max:
//     cmp   edi, esi  ; 比较 x 和 y
//     cmovg esi, edi  ; 若x>y，esi = edi（即 y = x）
//     mov   eax, esi  ; 返回值
//     ret
```

2. **间接分支猜错**：
- 去虚化(Devirtualization)：在热路径上加 `final`关键字、通过架构设计缓存具体的类型指针。这能让编译器把 `call *%rax` 变回静态的 `call 具体函数` 甚至内联(Inline)；
- 分支打散：把一个不可预测的大 switch 拆成多级可预测的小判断。前面跳转表一节中编译器在“开表划不来”时改用二分决策树，也就是有这个思路；
- 改输入序列：重排输入，按类分桶打包；
