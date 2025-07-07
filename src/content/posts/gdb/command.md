---
title: gdb教程
published: 2025-02-10
description: gdb 命令
# image: ./cover.jpg
tags: [gdb, 教程]
category: 调试工具
draft: true
---

## breakpiont
```sh
    b filename.cpp:10
    info b
    d 1
    disable 1
    enable 1
```
## watch
```sh
    info wat
    watch variable_name
    wa # (监视Race共享变量)
    wa buf if buf == nullptr
    wa -location &shell.edit_pos
    (gdb) commands
    > printf "x=%d at %s:%d\n", x, $_sargv[1], $bpnum # 用户态$_sargv[1]
    > continue
    > end
```
## run 
```sh
    start
    r
    n
    s
    si

    finish
    c
    q
```
## info 
```sh
    info b
    info rec

    info v
    info locals

    p variable_name
    p &rax
    display
    list

    info registers
    info threads
    thread 2

    (gdb) commands 2
    > printf "x=%d at %s:%d\n", x, $_sargv[1], $bpnum
    > continue
    > end
```
## layout 
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
## record
```sh
    rec full
    rec stop
    info rec
    rs
    rsi

    rec save filename
    rec restore filename

    rec goto begin
    rec goto end
    rec goto 1
    rec delete
```
# env
```sh
    set $source_file = "mysh.cpp"
```
# 并发
```sh
    thread id
    info threads
    set follow-fork-mode child #（自动追踪fork出的子进程）
    set detach-on-fork off + inferior 

    set follow-exec-mode new

    info fds

    handle SIGINT print stop
    catch syscall setpgid
    catch syscall tcsetpgrp
    call (int)tcgetpgrp(0)
    signal SIGINT

    break close
    bt # (程序崩溃/未预期行为/定位线程状况/)
```