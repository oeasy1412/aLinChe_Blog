---
title: gdb教程
published: 2025-02-24
description: gdb 命令
image: ./images/cover1.png
tags: [gdb, 教程]
category: 调试工具
draft: false
---

## ⚡ 断点调试（Breakpoints）
```sh
    gdb -x in ./a.out

    b filename.cpp:10 # 在filename.cpp的第10行设置断点
    info b            # 查看所有断点信息
    d 1               # 删除编号为1的断点
    disable 1         # 临时禁用1号断点
    enable 1          # 重新启用1号断点
```
## 🕵️‍♂️ 观察点（Watchpoints）
```sh
    info wat
    watch variable_name
    wa # (监视Race共享变量)
    wa buf if buf == nullptr     # 条件监视：当buf为null时触发
    wa -location &shell.edit_pos
    (gdb) commands # 自定义观察点行为
    > printf "x=%d at %s:%d\n", x, $_sargv[1], $bpnum # 用户态$_sargv[1]
    > continue
    > end
```
## 🚀 程序运行控制（run） 
```sh
    start  # start 开始调试，在main函数处暂停
    r      # run 开始/继续运行程序
    n      # next 执行下一行（不进入函数）
    s      # step 执行下一行（进入函数）
    si     # step i 执行下一行指令（asm）

    finish # finish 执行完当前函数并暂停
    c      # continue 继续运行直到下一个断点
    q      # quit 退出GDB
```
## 📊 信息查看与变量检查（info / print）
```sh
    info b          # 查看断点
    info rec        # 查看 record

    info v
    info locals     # 查看当前栈帧的局部变量

    p variable_name # 打印变量值
    p &rax
    display var     # 每次暂停自动显示变量
    list            # 显示当前位置的源代码

    info reg        # 查看寄存器值
    info threads
    thread 2

    (gdb) commands 2
    > printf "x=%d at %s:%d\n", x, $_sargv[1], $bpnum
    > continue
    > end

    x/4i $rip
    x/8 $rsp-8
    disp/x *(long*)($rsp)
    disp/xi $rip
    x/16xb 0x7fffffffdb28 # byte halr word g i char string
```
## 🖥️ 界面布局控制（layout） 
```sh
    la src
    la asm
    la split
    la regs

    tui enable
    tui disable

    Ctrl X + 1
    Ctrl X + 2
    Ctrl X + A
```
## ⏪ 执行记录与回放（record）
```sh
    rec full # 开始记录完整执行历史
    rec stop # 停止记录
    info rec # 显示记录信息
    info fds # 显示当前进程打开的文件描述符

    rec save trace.gdb    # 保存执行记录
    rec restore trace.gdb # 载入执行记录

    rs             # 反向单步执行
    rsi            # 反向单条指令执行
    rec goto begin
    rec goto end
    rec goto 1     # 跳转到执行历史中的第15步
    rec delete
```
## 🔧 环境变量设置（env）
```sh
    set $source_file = "mysh.cpp"       # 定义自定义变量
    set environment LD_PRELOAD=mylib.so # 设置环境变量
```
## 🧵 并发调试技巧
```sh
    thread id                  # 切换到线程id
    info threads               # 查看所有线程
    b worker.cpp:20 thread 2   # 断点调试
    # 双进程调试模式
    set detach-on-fork off     # 同时保留父子进程的调试控制
    inferior id                # 切换到进程id
    info inferiors             # 查看所有进程

    # fork()处理策略
    set detach-on-fork off + inferior # 保留所有fork进程
    set follow-fork-mode child        # 自动追踪fork出的子进程，（默认parent）
    # execve()处理
    set follow-exec-mode new   # exec函数族创建一个新进程，（默认same）

    # 信号处理
    handle SIGINT print stop # 捕获SIGINT信号
    signal SIGINT            # 手动发送信号（调试信号处理器）
    # 捕获特定系统调用
    catch syscall setpgid    # 拦截进程组设置调用
    catch syscall tcsetpgrp  # 拦截终端控制权转移
    call (int)tcgetpgrp(0)   # 检查当前终端前台进程组
    p setsid()                 # 创建新会话（验证会话行为）

    # 自定义
    catch syscall setpgid
    commands
    printf "setpgid(%d, %d) by pid=%d\n", $rdi, $rsi, $rax
    continue
    end

    break close
    commands
    printf "Closing fd=%d\n", (int)$rdi
    backtrace
    continue
    end
    bt # 打印当前线程的调用栈 (程序崩溃/对未预期行为/定位线程状况/...)
```