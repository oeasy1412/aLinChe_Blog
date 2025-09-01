---
title: Linux Signal
published: 2025-08-31
description: Signal_handelr
tags: [OS, signal]
category: OS
draft: false
---

# **深入Linux信号(Signal)机制：从内核到用户态**

信号是Linux/Unix系统中一种历史悠久且至关重要的通信机制。它用于在进程间异步地传递通知或事件。无论是用户按下`Ctrl+C`来中断一个程序，还是内核通知进程发生了内存访问等错误，背后都是信号在发挥作用。理解信号处理的机制与实现，对于编写健壮、可靠的系统软件至关重要。

## **一、 信号的基本概念**

想象一下，当你的手机收到一条短信时，它会中断你正在做的事情（比如看视频）,你看完通知后，可以选择忽略它，或者点击它去处理，然后回到刚才的视频。

信号(Signal)，就是操作系统层面的“短信通知”。它是一种“软件中断”，是发送给进程的异步通知，用于通知一个进程发生了某个特定的事件。

- 进程收到信号后，有三种处理方式：
  1. **执行默认操作（Default）**：每个信号都有一个系统预设的默认行为。最常见的默认行为是终止进程，其他行为还包括忽略信号或停止进程等。
  2. **忽略信号（Ignore）**：主动告诉内核丢弃该信号，不对其做任何处理。但有两个特殊的信号`SIGKILL` 和 `SIGSTOP`，它们不能被忽略，也无法被捕获，以确保系统管理员总有办法终止或停止任何进程。
  3. **捕获信号（Catch）**：提供一个用户自定义的函数（称为“`信号处理函数`”），当信号到达时，内核会中断进程的正常执行流程，转而执行这个自定义函数。

## **二、 信号的生命周期：从产生到处理**

一个信号从诞生到消亡会经历几个关键阶段，理解这些阶段是掌握信号机制的核心。

1.  **产生（Generation）**：信号由某个事件源创建。常见的事件源包括：
    *   **用户终端**：用户在终端按下特殊组合键，如 `Ctrl+C` 产生 `SIGINT`（中断信号），`Ctrl+\` 产生 `SIGQUIT`（退出信号）。
    *   **硬件异常**：当CPU检测到异常，如除零错误或非法内存访问，会通知内核，内核再向当前进程发送相应的信号（如 `SIGFPE`、`SIGSEGV`）。
    *   **内核**：内核可以主动发送信号来通知进程发生了某个事件，例如子进程退出时向父进程发送 `SIGCHLD`，或由`alarm()`系统调用设置的定时器到期时发送 `SIGALRM`。
    *   **其他进程**：一个进程可以通过 `kill()` 或 `sigqueue()` 等系统调用向另一个进程（需要有相应权限）发送信号，这是最常见的进程间通信(IPC)方式之一。
    *   **进程自身**：进程可以通过 `raise()` 或 `abort()` 函数给自己发送信号，常用于测试或异常处理。

2.  **未决（Pending）**：从信号产生到它被进程处理之前的这段时间，该信号的状态被称为“未决”，目标进程尚未对其做出反应。

3.  **阻塞（Blocked）**：每个进程都有一个“信号掩码”(`Signal Mask`)，它规定了哪些信号暂时不希望被接收。如果一个处于未决状态的信号恰好在信号掩码中，那么它将被阻塞，无法被递送，并会一直保持未决状态，直到进程解除对该信号的阻塞，它才会被递送。**阻塞和忽略是完全不同的概念**：忽略是“丢弃”信号，而阻塞是“推迟”信号的递送。

4.  **递送（Delivery）**：内核会在某个特定时机 **（通常是从内核态返回用户态前）** 将一个未决且未被阻塞的信号“递送”给进程，让进程处理它。

5.  **处理（Handling）**：进程在接收到递送的信号后，会根据预先设置的方式（默认、忽略或捕获）执行相应操作。一旦信号被处理，它就不再处于未决状态。

## **三、 信号处理的完整流程**

### 内核视角

- 用户空间 (User Space)：这是进程执行非特权的应用程序代码的地方，CPU处于低权限的用户模式（例如x86-64的Ring 3）下。每个进程拥有独立的虚拟地址空间和用户栈 (User Stack)，用于函数调用、参数传递和局部变量存储等。
- 内核空间 (Kernel Space)：这是操作系统内核执行的地方。当进程需要内核服务（通过系统调用）或被外部事件（中断）打断时，它会进入内核空间，CPU切换到高权限的内核模式（Ring 0）。每个线程都关联一个独立的、位于内核地址空间的内核栈 (Kernel Stack)。

> 信号的递送需要一个契机，这个契机就是一次从用户模式到内核模式的转换。我们以最常见的**异步硬件中断**（如时钟中断）为例。
- 中断发生: 时钟硬件向CPU发送中断信号，CPU立即停止执行代码。
  - 硬件完成：
    - CPU的特权级别从用户模式提升至内核模式。
    - 最小化上下文保存，用户态的 rip,rsp,rflags,cs等寄存器压入该进程的内核栈，用于后续返回用户空间的正确位置。
    - CPU​​根据​​中断号​​查询中断描述符表 (IDT)，找到对应的内核中断处理程序的入口地址。
    - rip 指向内核中断处理程序的地址，rsp 指向内核栈的栈顶。
  - 跳转到内核，内核接管控制权，开始在内核栈上运行。
- 内核处理完时钟中断的既定任务后，在通过 `iret` 指令返回用户空间之前，会执行信号分发的核心逻辑 （ [Linux do_signal()](https://code.dragonos.org.cn/xref/linux-6.1.9/arch/arm64/kernel/signal.c#1037) ），检查当前进程是否有待处理的信号：
  - 若发现没有未被阻塞的待处理信号，内核将直接使用内核栈上保存的最小上下文返回用户空间，继续执行被中断前的指令。
  - 反之，递送信号到进程：
    - 修改`用户栈`的内容，构建 **`信号帧`** `rt_sigframe` 结构体。从高地址向低地址依次压入：
      - ucontext.`sigcontext`: 给完整的用户态寄存器的一份“快照”。
      - siginfo: ​​信号信息
      - 信号恢复器 (`Restorer`) 的地址: 在sigcontext之上，内核压入一个返回地址。此地址指向一段特殊的、位于用户空间的代码（通常由VDSO或C库提供）。这段代码的唯一功能是触发sigreturn系统调用。
    - 修改`内核栈`的内容：
      - 修改rip: 将原本指向用户程序的下一条指令的rip值覆盖为用户自定义的信号处理函数(signal_handler)的入口地址。
      - 修改rsp: 将用户栈指针rsp的值，指向刚刚在用户栈上构建的信号帧的栈顶。
      - 设置函数参数: 将signo放入rdi寄存器，这是x86_64架构下函数调用的第一个参数。
- 内核执行 iret 中断返回指令，从内核栈弹出被修改过的上下文
  - 由于 rip 已被修改，CPU不会跳回原本用户程序的下一条指令，而是直接跳转到 signal_handler 函数并开始执行。
  - 由于 rsp 已被修改，signal_handler将在内核为其准备的、位于用户栈的新栈帧上运行，就如同一个普通的函数调用。
- 当 signal_handler 执行完毕并发出 ret 指令时：
  - ret 指令会从用户栈顶弹出返回地址，即内核安插的信号恢复器(Restorer)的地址。
  - Restorer 代码执行，发起 sigreturn 系统调用，使进程再次陷入内核，内核的 sigreturn 处理程序被调用
  - sigreturn 根据当前的用户栈指针，在用户栈上定位到sigcontext结构体。随后，它使用sigcontext中的内容，原子性地恢复所有用户态寄存器至被中断前的原始状态。
- 最终内核再次执行iret，这一次由于上下文是原始的，程序可以无缝地返回到最初被中断的指令，继续执行。


> 下面是一个 cpp 的示例代码
```cpp
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

// 使用 volatile sig_atomic_t 来确保原子访问和防止编译器过度优化
volatile sig_atomic_t shutdown_flag = 0;

void signal_handler(int signum) {
    printf("target received %d (%s)\n", signum, strsignal(signum));
    if (signum == SIGINT || signum == SIGTERM) {
        shutdown_flag = 1;
    }
}

int main() {
    if (signal(SIGINT, signal_handler) == SIG_ERR) {
        perror("Failed to register SIGINT handler");
        exit(EXIT_FAILURE);
    }
    if (signal(SIGTERM, signal_handler) == SIG_ERR) {
        perror("Failed to register SIGTERM handler");
        exit(EXIT_FAILURE);
    }
    int i = 0;
    while (!shutdown_flag) {
        // printf("Working...\n");
        ++i;
        // sleep(1);
    }
    // printf("i=%d\n", i);
    printf("Shutdown signal received. exiting...\n");
    return 0;
}
```
```sh
# 我们使用 g++ 编译 然后反汇编用 vim 查看
g++ -g -O2 ./sig.cpp
objdump -D ./a.out &| vim -
```

```nasm
Disassembly of section .text:

0000000000001100 <main>:
    1100:	f3 0f 1e fa          	endbr64                  ; 启用 CET 安全特性
    1104:	55                   	push   %rbp
    1105:	48 8d 2d 64 01 00 00 	lea    0x164(%rip),%rbp        # 1270 <_Z14signal_handleri>
    110c:	bf 02 00 00 00       	mov    $0x2,%edi         ; $edi=2(SIGINT)
    1111:	48 89 ee             	mov    %rbp,%rsi         ; $rsi=signal_handler 地址
    1114:	e8 97 ff ff ff       	call   10b0 <signal@plt> ; 注册 SIGINT handler
    1119:	48 83 f8 ff          	cmp    $0xffffffffffffffff,%rax  ; 检查返回值
    111d:	74 33                	je     1152 <main+0x52>  ; 失败则跳转到错误处理
    ...
    1132:	66 0f 1f 44 00 00    	nopw   0x0(%rax,%rax,1)  ; 对齐填充
    ; 这里是 while (!shutdown_flag) 循环
    1138:	8b 05 d6 2e 00 00    	mov    0x2ed6(%rip),%eax        # 4014 <shutdown_flag>
    113e:	85 c0                	test   %eax,%eax
    1140:	74 f6                	je     1138 <main+0x38>
    ; 
    1142:	48 8d 3d 27 0f 00 00 	lea    0xf27(%rip),%rdi        # 2070 <_IO_stdin_used+0x70>
    1149:	e8 52 ff ff ff       	call   10a0 <puts@plt>
    114e:	31 c0                	xor    %eax,%eax         ; return 0 的三步走
    1150:	5d                   	pop    %rbp              ; 恢复基指针 rbp
    1151:	c3                   	ret                      ; 从函数调用中返回
    ; 下面是​错误处理部分 `1152 <main+0x52>` (省略)
    1152:	48 8d 3d c7 0e 00 00 	lea    0xec7(%rip),%rdi        # 2020 <_IO_stdin_used+0x20>
    1159: e8 82 ff ff ff        call   10e0 <perror@plt> ; 打印错误
    115e: bf 01 00 00 00        mov    $0x1,%edi       ; 退出码 1
    1163: e8 88 ff ff ff        call   10f0 <exit@plt> ; 退出程序
    ...

0000000000001270 <_Z14signal_handleri>:
    1270: f3 0f 1e fa           endbr64          ; 启用 CET
    1274: 53                    push   %rbx      ; 保存 rbx
    1275: 89 fb                 mov    %edi,%ebx ; 保存信号编号到 ebx
    1277: e8 44 fe ff ff        call   10c0 <strsignal@plt> ; 获取信号描述的字符串
    127c: 89 da                 mov    %ebx,%edx ; edx=signo
    127e: bf 01 00 00 00        mov    $0x1,%edi ; edi=1 (stdout)
    1283: 48 8d 35 7a 0d 00 00  lea    0xd7a(%rip),%rsi        # 2004 <_IO_stdin_used+0x4>
    128a: 48 89 c1              mov    %rax,%rcx ; rcx=信号描述的字符串指针
    128d: 31 c0                 xor    %eax,%eax ; 清空 eax (用于浮点参数)
    128f: e8 3c fe ff ff        call   10d0 <__printf_chk@plt> ; 打印消息
    1294: 83 fb 02              cmp    $0x2,%ebx  ; 检查是否为 SIGINT(2)
    1297: 74 07                 je     12a0       ; 是则跳转
    1299: 83 fb 0f              cmp    $0xf,%ebx  ; 检查是否为 SIGTERM(15)
    129c: 74 02                 je     12a0       ; 是则跳转
    129e: 5b                    pop    %rbx       ; 恢复 rbx
    129f: c3                    ret               ; 从函数调用中返回
    12a0:	c7 05 6a 2d 00 00 01 	movl   $0x1,0x2d6a(%rip)        # 4014 <shutdown_flag>
```

## **四、 核心数据结构与系统调用**

```cpp
typedef struct {
    unsigned long sig[1]; // 64位掩码（x86_64）
} sigset_t;
typedef struct {
    void  *ss_sp;     // 栈指针（RSP）
    int    ss_flags;  // 标志（SS_ONSTACK等）
    size_t ss_size;   // 栈大小
} stack_t;
// sigcontext 通用寄存器+浮点寄存器状态
struct ucontext {
	unsigned long	  uc_flags;    // 上下文标志位（保留字段）
	struct ucontext  *uc_link;     // 指向下一级上下文
	stack_t		  uc_stack;        // 栈信息
	struct sigcontext uc_mcontext; // 寄存器状态
	sigset_t	  uc_sigmask;	   // 信号屏蔽字 /* mask last for extensibility */
};

#define SI_MAX_SIZE	128
union __sifields {
    // 多种 struct
    /* kill(), timers, signals, SIGCHLD, SIGFAULT, SIGPOLL, SIGSYS */
}
typedef struct siginfo {
	union {
		struct {
            int si_signo;
            int si_errno;
            int si_code;   // 信号来源代码
            union __sifields _sifields;  // 信号来源特定数据
        }
		int _si_pad[SI_MAX_SIZE/sizeof(int)];  // 保证结构体大小128字节
	};
} siginfo_t;

struct rt_sigframe {
	char *pretcode;      // 指向恢复代码 (__restore_rt)
	struct ucontext uc;  // 用户上下文
	struct siginfo info; // 信号信息
	/* fp state follows here */
};
```

## More detail
### [man 7 signal](https://www.man7.org/linux/man-pages/man7/signal.7.html)

### sigaction Rust 示例
```rust
#[derive(Debug, Copy, Clone)]
pub struct Sigaction {
    action: SigactionType,
    flags: SigFlags,
    mask: SigSet,
    /// 信号处理函数执行结束后，将会跳转到这个函数内进行执行，然后执行 sigreturn 系统调用
    restorer: Option<VirtAddr>,
}
```


P.S.  **多线程中的信号处理**
*   信号是发送给**整个进程**的，但最终会由**单个线程**来处理。
*   每个线程都有自己**独立的信号掩码**（阻塞集），但所有线程共享同一个**信号处理函数表**。
*   当一个信号被发送到进程时，内核会选择第一个它找到的**没有阻塞该信号的线程**来递送和处理。如果所有线程都阻塞了该信号，它将保持在进程的未决队列中。
*   内核选择的线程是不确定的 (indeterminate)，是多线程编程中常见的bug来源：
    *   库函数不安全 (Unsafe Library Functions)：信号处理函数可以中断任何线程的任何代码。如果被中断的线程正在执行一个不可重入 (non-reentrant) 的函数（如 malloc, printf，或任何持有锁的函数），就会导致死锁或数据损坏。（e.g 线程A调用malloc()获取了一个全局锁，内核选择线程A处理信号，信号处理函数被执行，它也尝试调用 malloc() => **线程A死锁在自己的信号处理器里**）
    *   上下文混乱 (Context Confusion)：你无法预知哪个线程会执行信号处理函数，因此也就无法预知该线程当时的上下文
    *   竞争条件 (Race Conditions)：如果多个信号快速到达，它们可能被不同的线程处理，从而对共享数据产生竞争。
*   可以使用 `pthread_kill()` 向特定的线程发送信号。

> Q: 为什么不单独开一个线程来处理信号？ A: POSIX信号模型的原始设计早于现代多线程库的普及，它的设计初衷是针对单进程模型的。单独开一个线程来处理信号是现代多线程编程中处理信号的推荐方法，这种模式被称为 “信号处理线程 (`Signal Handling Thread`)” 或 “专用信号线程 (`Dedicated Signal Thread`)”。

> 当然，你可以亲手hack这个问题：（笑）
```cpp
#include <chrono>
#include <csignal>
#include <iostream>
#include <mutex>
#include <thread>

using namespace std;

// 信号处理器函数
void signal_handler(int sig) {
    cerr << "Signal handler: attempting malloc\n";
    void* ptr = malloc(1 << 30); // 尝试获取 malloc 内部锁
    if (ptr)
        free(ptr);
    cerr << "Signal handler: malloc succeeded (never reached)\n";
}

// 工作线程函数
void worker_thread() {
    cout << "Worker thread started\n";
    // 调用 malloc 获取其内部锁
    cout << "Worker thread start malloc\n";
    void* ptr = malloc(1 << 30);
    ptr = malloc(1 << 30);
    ptr = malloc(1 << 30);
    ptr = malloc(1 << 30);
    ptr = malloc(1 << 30);
    cout << "Worker thread malloc end\n";
    // 如果出现 Working ，根据输出调整睡眠的时间，直到有机会在malloc触发的中断时，处理信号（笑）
    for (int i = 0; i < 2; ++i) {
        cout << "Working " << i + 1 << "/2...\n";
        this_thread::sleep_for(chrono::seconds(1));
    }
    free(ptr);
}

int main() {
    // 设置信号处理器
    struct sigaction sa {};
    sa.sa_handler = signal_handler;
    sigaction(SIGUSR1, &sa, nullptr);

    thread worker(worker_thread);
    // 等待线程进入临界区
    this_thread::sleep_for(chrono::nanoseconds(90)); // 需要根据你的实际情况调整sleep的时间
    pthread_kill(worker.native_handle(), SIGUSR1);
    cout << "\nMain: signal sent to worker thread\n";

    worker.join();
    cout << "Main: thread finished (never reached)\n";
    return 0;
}
```

> 当然，你在 RTFM 的时候，[signal-safety(7) Linux](https://www.man7.org/linux/man-pages/man7/signal-safety.7.html) 可以找到这方面的解释：
>  手册页的表格中展示了POSIX.1 要求的 **异步信号安全函数** 的函数集。一个函数是异步信号安全的，要么因为它是可重入的，要么因为它在信号方面是原子的（即，它的执行不能被信号处理程序中断）。