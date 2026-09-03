---
title: ptrace
published: 2026-09-01
description: ptrace
tags: [OS, ptrace, gdb, signal, syscall]
category: OS
draft: false
---

地址空间隔离是操作系统的立身之本，一个进程拿不到另一个进程的页表，更不可能在对方 CPU 上插一脚。所以"调试"从第一天起就是一个内核问题。调试器是如何在进程隔离的墙上凿出洞，去读写另一个进程的寄存器与内存，并决定它下一步怎么走的？ptrace 就是内核对外开放的这条仲裁通道。

如果你已经熟悉进程与线程、信号、系统调用、虚拟内存和 x86 异常，那么理解 ptrace 只需要建立一个心智模型：**一切调试，本质上都是让被调试进程(`tracee`)在关键点停下来，等调试器(`tracer`)的指示，再恢复执行**。

```c
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```

---

## 一条主线：停止 → 等待 → 决断 → 恢复
tracer 和 tracee 是两个独立调度的执行流，它们之间没有锁，唯一的同步介质就是"停止态 + wait 状态字"。假设 tracer 已经和 tracee 建立了关系，一次完整的调试循环如下：

1. **停止**：tracee 在检查点(信号投递路径、系统调用边界、生命周期事件)被内核暂停，进入 ptrace-stop 状态。此时它的寄存器和栈被完整冻结，不再消耗 CPU。
2. **等待**：内核向 tracer 发送 SIGCHLD 并唤醒其 `waitpid` 调用，返回一份编码了停止原因的状态字。**铁律：SIGCHLD 是可缺席的提示，真相永远在 `waitpid` 里**。
3. **决断**：tracer 解码状态字，判断停止类别，然后安全地读写 tracee 的寄存器和内存。
4. **恢复**：tracer 发出恢复请求，可选地注入一个信号，tracee 从停止点醒来继续执行，直到撞上下一个检查点。

这个循环有三条不变式：
- **未恢复则不前进**：tracee 停下后，新到的信号只会排队，不会产生新报告(SIGKILL 除外，它会直接杀死停止态中的 tracee)。
- **一次停止只报告一次**：`waitpid` 是消费性的，想"看一眼再决定"需要用 `WNOWAIT` 选项。
- **事件串行到达**：多 tracee 的事件通过等待队列逐个送出，tracer 天然是一个事件循环程序。

---

## 一切的起点：ptrace 追踪关系的建立。正交的父子，强耦合的wait
建立追踪关系时，内核做了一件表面平静、影响深远的事：把 tracee 的"父进程"字段换成 tracer，而把它的真实父进程(real parent)另存一份保存下来。此后在 wait 体系的眼睛里，这个 tracee 就是 tracer 的孩子；断开追踪时再把父进程字段还原为真实父进程。注意这不是文件系统或信号意义上的收养，只是 wait 体系内部的记账——但它派生出整整一串后果。

### 权限：谁能追踪谁
- 目标不是内核线程
- 目标不能处于正在退出的状态，且不能已经被其他 tracer 附着
- 同线程组直接放行
- __ptrace_may_access()
    - 同线程组直接放行
    - tracer 的 UID/GID 必须同时等于 目标的三UID/三GID（真实Real、有效Effective、已保存Saved） 或 tracer 在目标的 user_ns 中有 CAP_SYS_PTRACE 最高特权；
    - dumpable 检查：若目标不可 dumpable（如执行过 setuid）且​ tracer 无 CAP_SYS_PTRACE → 拒绝
    - 其他LSM安全模块限制；
- 目标不能正在退出；
- 目标不能被其他 tracer 附着；

### 建立关系的四种方式
| 方式 | 发起者 | 时机 | 需要权限检查 | 建立后的立即效果 |
|---|---|---|---|---|
| PTRACE_TRACEME | 被追踪者自己 | 出生后、被调试前 | 无凭证检查(自愿) | 继续运行，不停止 |
| PTRACE_ATTACH | tracer | 事后强制 | 完整权限链 | 目标被发 SIGSTOP，异步停止 |
| PTRACE_SEIZE | tracer | 事后强制 | 完整权限链 | 不停止，静默挂上关系 |
| fork 联动 | 内核自动 | 子线程出生时 | 免检(继承父tracee的关系) | 子线程带初始停止出生 |

#### PTRACE_TRACEME
tracee 主动请求父进程成为自己的tracer。
- `TRACEME`的教科书用法（strace式的启动）：
1. 调试器 fork 出子进程，然后立刻调用 waitpid 阻塞等待；
2. 子进程(tracee)立刻调用 TRACEME，挂上 PT_PTRACED 标记，然后 exec 目标程序。在内核执行 execve 时（清空子进程旧的内存空间，加载 target_program 的 ELF 文件，解析动态链接库，分配新的栈等），修改 RIP 前检查发现子进程拥有 PT_PTRACED 标记，内核会给子进程强行发送一个 SIGTRAP 信号，使其进入 ptrace-stop 状态。
3. 父进程(tracer) waitpid 收到子进程的信号投递停止而被唤醒，此时父进程知道子进程已经停在目标程序的第一条指令前。在这个完美的停顿点，通过 ptrace(PTRACE_SETOPTIONS, ...) 完成各种配置（如设置选项、是否跟踪系统调用等），再恢复子进程。

#### PTRACE_ATTACH
ATTACH 允许一个进程在目标已经运行之后强行建立追踪关系。这是 gdb attach pid 和 strace -p pid 的底层机制。
- `ATTACH`是异步的。ATTACH 的返回值只承诺一件事：**关系已建立**。（目标不一定已经停下）
内核在关系建立后向目标线程投递一个内核生成的 SIGSTOP (si_code 为 SI_KERNEL)，然后立刻返回。此时目标线程可能在另一个CPU上跑着、或可能在不可中断的睡眠中，需要过一会儿才能消化这个SIGSTOP信号。

所以正确的 ATTACH 操作永远是：
1. 调用 ATTACH，检查返回值为0；
2. 直到 waitpid 收到目标以 SIGSTOP 为停止信号(WSTOPSIG)的停止报告；
3. 从这一刻起才能安全地发其他 ptrace 请求（读寄存器、设置选项、恢复执行等）。

#### PTRACE_SEIZE
- SEIZE 是 ATTACH 的"现代优雅附着版本"，其设计目标是不干扰目标进程的运行状态：**静默挂载，按需停止，挂载瞬间原子地配置好 Ptrace Options**。

SEIZE 有其配套的 INTERRUPT、LISTEN 请求和专用的 PTRACE_EVENT_STOP (事件号 128)：
1. INTERRUPT：纯净的按需停止。在内核的作业控制结构里挂起一个“待决陷阱”标志位，强迫进程在返回用户态前停下。
2. 在 SEIZE 模式下，所有的 INTERRUPT 打断、组停止、以及新生线程的首次停止，都不再伪装成“信号投递”，而是统一上报为一个全新的事件：PTRACE_EVENT_STOP ，其低8位依旧携带信号语义。
3. LISTEN：处理好作业控制的透明传递。在 SEIZE 模式下，当 tracee 因组停止信号（如 Ctrl+Z 产生的 SIGTSTP）而停止时，tracer 调用 PTRACE_LISTEN 让 tracee 从 ptrace-stop 平滑过渡到 group-stop——tracee 真正休眠、不消耗 CPU，但在 wait 体系中仍可被 tracer 通过 PTRACE_INTERRUPT 重新接管。它解除了"tracer必须注入信号才能让进程停下"的困境，让操作系统原生作业控制正常工作，同时保留 tracer 通过 INTERRUPT 随时重新介入的能力。

---

## 四种停止，一套编码
### checkpoints 检查点
tracee 在**检查点**(`信号投递路径`、`系统调用边界`、`生命周期事件`)被内核暂停。它们都是执行流从用户态到内核态的切换边界，寄存器含义明确、悬停安全、且天然是"世界即将改变"的门槛。

ptrace 停止指"tracee 愿意接受 ptrace 命令的停止态"。从调度器的角度看，ptrace 停止就是"这个线程不再被挑中执行"：它不退出、不释放资源、页表原封不动，只是被摘出了运行调度队列。它与阻塞休眠的区别在于唤醒条件——阻塞休眠等的是数据或事件，ptrace 停止等的是一个明确的恢复指令；能不等指令就结束这个状态的，只有SIGKILL死亡与ptarce关系解除(tracer主动断开 或 tracer死亡)两条路。

理解了检查点，再看并发与竞态在哪里潜伏。tracer 与 tracee 是两个独立调度的执行流，之间没有锁，唯一的同步介质是"停止态 + wait 状态字"这一对。设计者刻意让每条 ptrace 请求的处理都要求 tracee 已处于停止态，且请求执行期间 tracee 不得被唤醒。剩余的竞态窗口因此都收缩到 wait 语义附近。

### 四种停止的关系与权限
| 类别 | 触发方式 | 状态字特征 | tracer 的权限 |
| :--- | :--- | :--- | :--- |
| **信号投递停止** | 信号被选中投递给被追踪线程 | 信号号 = 被拦信号，事件号 = 0 | **完全决断权**：抑制(吞掉)、转发、或改号注入 |
| **组停止** | 作业控制停止信号(SIGSTOP/SIGTSTP等) | 非 SEIZE 下与信号投递停止同形 | **观测**：恢复即吞掉停止信号，tracee 直接跑 |
| **系统调用停止** | tracer 以 syscall 模式恢复 | 信号号 = SIGTRAP(可选带 0x80 位) | **拦截篡改模拟**：入口改参数，出口读返回值 |
| **事件停止** | 生命周期事件锚点(fork/exec/exit) | 事件号 = 1~7，信号号 = SIGTRAP | **观测**：不能阻止事件发生 |

#### 撞见式 与 预约式
还可以换一个角度给四类停止分类：
- 信号投递停止 和 组停止是"世界找上门"的**撞见式**，tracer 被动地被叫醒，停止原因在 tracee 与 世界 那边；
- 系统调用停止 和 事件停止是 tracer 的**预约式**，只有 tracer 上次恢复时打开了对应开关才会发生。

#### P.S. 编码的历史原因
在最早的 Unix 里，wait 只需要报告两件事：孩子是怎么死的；后来作业控制(Job Control)引入了"停止"的概念；再后来 ptrace 引入了"事件"的概念。为了不破坏古老的 wait 接口 ABI，内核把全部停止信息压缩进一个 int 的位域里：**低 8 位表示停止/退出，次 8 位表示信号号，高 16 位表示事件号**。又因为信号号不会超过127个，所以第8位可以被用来做标记(0x80)。

- 低7位==0 → 正常退出，退出码在高8位。
- 低7位!=0且!=0x7f → 被信号杀死，信号号在低7位，第8位为1(0x80)表示核心转储。
- 低8位==0x7f → 停止，信号号在次8位的低7位，次8位的第8位为1(0x80)表示系统调用停止，高16位表示事件号。

#### 信号投递停止(signal-delivery-stop)
多线程进程收到除SIGKILL外的任何信号时，内核先随机挑一个线程来处理。若被选中的线程正被追踪，内核就在"信号即将投递"的瞬间把它按停。此时信号已经出队，但还没有真正生效，处置表没查处理函数更没跑，类似"拦截"了，tracer 有完全决断权。

- 状态字特征：WSTOPSIG(status) 解出信号号本身，事件号(状态字高16位)为零，无0x80标记。
- 恢复语义：可以抑制(吞掉)、放行转发(原样投递)、或改号注入(投递一个siginfo为"来自tracer"的新信号)。

#### 组停止(group-stop)
组停止不是 ptrace 的发明，它是作业控制的概念：SIGSTOP、SIGTSTP、SIGTTIN、SIGTTOU 这四个停止信号的默认效果是让**整个线程组全体停止**，由父进程用 wait 观测、由 SIGCONT 唤醒。

例如：被追踪的多线程进程收到 SIGSTOP。内核先在信号投递处按停它（即信号投递停止），tracer若转发它默认处置生效，则组停止在整个线程组发生，组内**每个**被追踪的线程（包括刚才触发信号投递停止的那个线程）会各自报告一次自己的组停止。也就是说，tracer 通常会先后看到(1+n)次与 SIGSTOP 有关的停止，一次是信号投递停止的决断机会，n次是n个被追踪线程的既成事实。

- 状态字特征：这是四类停止中最阴险的一类。非 SEIZE 模式下，它的状态字与信号投递停止**完全同形**。只能通过 PTRACE_GETSIGINFO siginfo读取失败返回EINVAL的方式来区分（因为组停止没有正在投递的信号没有siginfo）。
- 恢复语义：但因为ptrace是作用于单线程的，所以PTRACE_CONT恢复时只会解除当前线程的组停止状态，其他线程仍然停着。若tracer只是想让该线程维持停止并继续监听其他事件，应使用PTRACE_LISTEN。


#### 系统调用停止 (syscall-entry-stop / syscall-exit-stop)
当 tracer 使用 `PTRACE_SYSCALL` 或 `PTRACE_SYSEMU` 唤醒 tracee 时，内核会在被追踪线程跨越用户态与内核态边界的瞬间将其拦截。一个完整的系统调用执行周期会产生成对的两次停止：
1. 进入停止(syscall-entry-stop)：tracee 刚发出系统调用指令（如 syscall / sysenter / int 0x80），内核已将系统调用号与参数载入寄存器，但**尚未执行任何内核层面的系统调用实现**。
2. 退出停止(syscall-exit-stop)：内核已执行完系统调用的具体实现，并将返回值写入返回值寄存器（如 x86_64 的 rax），但**尚未返回到用户态**。

- 状态字特征：
未开启 PTRACE_O_TRACESYSGOOD 时：WSTOPSIG(status) == SIGTRAP，高16位为零；开启 PTRACE_O_TRACESYSGOOD 时**：内核将信号标记为 (SIGTRAP | 0x80)是系统调用边界。Entry 与 Exit 的区分：传统接口下，状态字本身**不包含**是 entry 还是 exit 的标识，调用 ptrace(PTRACE_GET_SYSCALL_INFO)，直接返回 PTRACE_SYSCALL_INFO_ENTRY、EXIT 或 SECCOMP。

- 恢复与篡改语义：
在 syscall-entry-stop 时，tracer 拥有最高裁判权：修改寄存器中的参数即可改变系统调用行为；如果将系统调用号修改为 -1，内核会直接跳过该系统调用的实际执行并置 errno = ENOSYS。随后的 syscall-exit-stop 阶段，tracer 可以篡改 rax 寄存器写入伪造的返回值（这正是 strace 故障注入与系统调用模拟器的底层原理）。继续调用 PTRACE_SYSCALL 驱动其前进至下一个系统调用边界。

#### 事件停止 (ptrace-event-stop)
当 tracer 通过 `PTRACE_SETOPTIONS` 启用了特定选项（`PTRACE_O_TRACE*`）后，tracee 发生关键的**进程/线程生命周期状态跃迁**时，内核会在特定代码锚点主动挂起 tracee 并报告专属事件。事件停止往往伴随着丰富的内核上下文，tracer 可在此时调用 `ptrace(PTRACE_GETEVENTMSG, pid, 0, &msg)` 提取关键数据

- 状态字特征：WSTOPSIG(status) == SIGTRAP。状态字高16位对应事件类型。

---

## 恢复执行：先决断，后放行
- 所有恢复请求有一条**共同的前置约束**：发出请求时，**目标 tracee 必须正处于 ptrace 停止中**。

无论你调用哪个恢复请求，内核底层的动作都由同样的三步构成：
1. **布置下次陷阱**。
PTRACE_CONT 清除所有 ptrace 工作位；PTRACE_SYSCALL 设置"系统调用追踪"PT_TRACESYSGOOD 工作位；PTRACE_SINGLESTEP 在保存的寄存器现场置起 TF；PTRACE_SYSEMU 设置 PT_SYSCALL_EMU 位。
2. **写下信号决断**。
恢复请求的 data 参数（也就是"信号参数"）被原样写入 tracee 的出口码字段 `exit_code`。exit_code 是 tracer 与 tracee 之间唯一的决议通道：tracee 醒来后读走它，读到的数字就决定了那个待裁决信号的命运。
3. **唤醒**。
清除 ptrace-stop 状态位、调度状态迁移回 `TASK_RUNNING` 并将 tracee 放入运行队列。**从这一刻起，tracer 与 tracee 重新并发**。

#### PTRACE_DETACH
DETACH 是「恢复」操作的一员，断开也是一种恢复。这决定了它的**前置条件：tracee 必须正处于 ptrace 停止**。（这也是为什么对一个正在运行的 tracee 直接 DETACH 会得到 ESRCH 错误）。

#### PTRACE_CONT
CONT 是语义最简单的恢复：不布置陷阱，清除所有 ptrace 工作位，写下信号决断，唤醒。

#### PTRACE_SYSCALL
SYSCALL 会在 tracee 跨越系统调用的入口和出口时，强行将其拦下。

我们知道，处于系统调用停止(syscall-stop)时，rax 寄存器会被内核覆盖掉，用来存放系统调用的返回值。为了不丢失最初始的系统调用号，内核把它备份在了 `orig_rax` 里。

入口停止处可以改寄存器换参数（如果把调用号 orig_rax 改成 -1，内核会直接跳过该调用）；出口停止处可以改返回值寄存器 rax，伪造系统调用的返回值欺骗用户态。

> 具体为：syscall 指令 → 硬件切换到内核态(Ring 0) → entry_SYSCALL_64() 汇编入口，保存寄存器到 pt_regs → do_syscall_64() C 语言入口，拿到 syscall 号，里面的 syscall_trace_enter() 是 ptrace 入口检查点。处理完成后，syscall_trace_exit() 是出口检查点。

#### PTRACE_SINGLESTEP
SINGLESTEP 要求 tracee 在执行**恰好一条指令**后再次停止。在 x86 上，这是通过 EFLAGS 寄存器里的陷阱标志（TF）实现的。

#### PTRACE_SYSEMU
SYSEMU (System Call Emulation) 是为 User-Mode Linux (UML) 等这类"把整个内核编译成用户态程序"沙箱/仿真器量身定制的。

它只在系统调用入口处停止，但无论你之后怎么恢复，这个系统调用都不会被执行，唤醒后直接回到用户态。

> 具体为：当 Guest主机(tracee) 执行 syscall 指令陷入内核时，检查点发现 PT_SYSCALL_EMU 标志，停住 tracee 并投递 SIGTRAP 等待 宿主机(tracer) 决策。宿主机(tracer)内核直接跳过系统调用的执行，恢复 tracee 回到用户态。

---

## 寄存器与内存访问
前几节解决的是「怎么把一个线程拦下来」：追踪关系的建立、停止和恢复执行。对一个调试器来说，观察和修改tracee的状态也是核心需求。

tracer 能触到的 tracee 状态有三类：
1. 临时保存到内核栈 Trap Frame 中的**通用寄存器**；
2. 保存在内核为每个线程准备的专用的线程扩展缓冲区中的**浮点与扩展寄存器**；‘
3. tracee自己页表(Page Tables)中的**进程虚拟地址空间**。

### 第一代接口：逐机器字访问的 PEEK/POKE 家族
1. PEEKTEXT / PEEKDATA：读内存
PEEK 类请求的语义是：读出 tracee 某地址上的一个机器字。（里面有小坑，比如返回值歧义，data参数是传出指针而不是入参。这里就不展开了）
2. POKETEXT / POKEDATA：写内存
POKE 常用来植入软件断点（将原本的机器指令替换为 `0xCC` 即 `int 3`）。
POKE 能改写只读的代码段，这是因为内核在跨进程写入时，携带了一个特殊的“无视权限”旗标。当底层调页机制遇到只读的私有映射（如代码段）时，会强行触发写时复制(Copy-On-Write, COW)。内核会为该页面造一份私有的匿名副本，把修改落在副本上。（这也是为什么平时gdb调试时，修改代码段不会影响原ELF文件，以及每一次重新调试都要重新打断点而不会自动保存断点）

> gdb 的 `watch` 是怎么实现的：软件断点是靠改写内存代码实现的。但如果是监控某个数据的变化(Data Watchpoint)，改代码是做不到的，这就需要用到**硬件断点**。
> watch 依赖 CPU 硬件调试寄存器（x86-64 上的 DR0 ~ DR7）：
> - DR0 ~ DR3：存放 4 个需要监控的目标线性虚拟地址。
> - DR4~DR5：保留(Reserved)。
> - DR6 调试状态寄存器：当 CPU 执行指令触发了硬件断点条件时，硬件会自动将原因写入 DR6，并引发 #DB (1号调试异常)。内核捕获该异常后向 tracee 发送 SIGTRAP，tracee 随之停止，GDB 即可获知是哪个 watchpoint 被击中。
> - DR7 调试控制寄存器：配置触发条件（执行/写入/读写）和监控长度（1/2/4/8字节）。

### 第二代接口：整组寄存器读写的 GETREGS / SETREGS
逐字读取 20 多个寄存器需要发起 20 多次系统调用，这太慢了。于是我们需要内核提供了整组寄存器的读写接口 GETREGS / SETREGS / GETFPREGS / SETFPREGS。

### 第三代接口：面向未来的 GETREGSET / SETREGSET
我们不希望 CPU 每出一个新扩展（如 SSE, AVX, AVX-512）就要加一对新请求号。于是我们借用了 ELF **核心转储文件**(`Core Dump`) 的 `Note` 机制，自适应长度协商。

### P.S. 为什么访问内存不需要停止，而读寄存器必须停稳才行？
- 内存独立于执行态：内存的内容由页表描述。即使 tracee 正在另一个 CPU 核心上狂奔，它的**页表也是稳定可解析的**。因此，我们可以通过 /proc/pid/mem 或 process_vm_readv 等不依赖 ptrace 停止的接口，在运行时强行读取内存。
- 寄存器瞬息万变：通用寄存器只在一个时刻拥有权威的静态副本——那就是进程发生中断/异常、陷入内核，将通用寄存器现场保存到内核栈的**Trap Frame(陷阱帧)**。当 tracee 在用户态奔跑时，物理 CPU 寄存器里的值每纳秒都在变，内核根本没有有效的快照给你读。

---

## 选项与事件：PTRACE_SETOPTIONS 与 PTRACE_EVENT_* （简述版）

### Fork 家族：TRACEFORK、TRACEVFORK 与 TRACECLONE
- 防止恶意程序或多线程应用通过分裂逃出沙箱
让新出生的子任务，在执行第一条用户态汇编指令之前，就直接被内核打上追踪标记，绑定到同一个 tracer 账下。

### TRACEEXEC
execve() 是进程生命中最决绝的一跃：旧程序的虚拟地址空间被整平摧毁，全部代码、堆、栈全部抹去，所有信号处理句柄重置回默认，换上一具全新的 ELF 文件。

开启 PTRACE_O_TRACEEXEC 后，内核在全新的 ELF 镜像完全加载就绪、但新程序入口点未执行第一条汇编的绝对安全时刻，将进程拦截在 PTRACE_EVENT_EXEC 停止点上。

### TRACEEXIT 与 EXITKILL
TRACEEXIT：tracee死亡前踩下刹车取样。进程在宣称走向毁灭的第一阶段，内核会插手踩下刹车，tracee 还没有进入僵尸状态(Zombie)，它的内存、栈、寄存器全都原封不动地保留着！tracer 可以从容读取导致崩溃的完整上下文、抓取 Core Dump。tracer 放行后，tracee 才会Zombie，最后 tracer 会通过 waitpid 收到常规的死亡通知，完成收尸(Reap).

EXITKILL：拉着tracee一起陪葬。一旦 tracer 死亡，内核在清理追踪关系前，会自发向该 tracer 旗下的所有受控 tracee 逐一投递致命的SIGKILL。
调试器或监控沙箱在分析未知的恶意样本时，最怕的一幕就是：分析器自身因为 Bug 意外崩溃或被强行杀死。在默认情况下，tracer 死亡后，内核会将 tracee 解除追踪，将其释放并恢复自由运行！一个被分析到一半、可能携带着攻击荷包的恶意进程，瞬间脱缰回归真实世界，甚至配合未被清理的子孙树，造成极其危险的安全事故。

---

## 两套机制(异常驱动 vs 检查点驱动)
- 自下而上的**异常驱动**：比如你设了一个断点，gdb 在目标地址写入 `int3` 指令。tracee 执行到此处，物理 CPU 硬件抛出陷阱异常，内核将其转化为 `SIGTRAP` 信号，送入信号投递路径，在"信号即将投递"的瞬间被 ptrace 拦截——变成一次**信号投递停止**。（例如：#BP / #DB）
- 自上而下的**检查点驱动**：比如你开启了系统调用追踪。这不需要任何 CPU 异常，内核只需要在 tracee 进入和退出内核的两个边界上设置检查点，直接将其按停——变成一次**系统调用停止**。（例如：Syscall / Event）

---

## 其他
- /proc/pid/mem 伪文件接口，把目标进程的整个虚拟地址空间投影成了一个"无始无终:的字节流视图。
