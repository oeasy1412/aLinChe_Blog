---
title: Linux Perf
published: 2025-11-20
description: Linux Perf 工具
tags: [perf, 教程]
category: perf
draft: false
---
在 Linux 系统性能调优的世界里，**Perf** (`Performance Counters for Linux`) 基于 Linux 内核的 perf_events 子系统，能够利用 CPU 的硬件计数器(`PMU`)和内核的监测点(`Tracepoints`)，以极低的损耗分析系统和应用程序的性能表现。

## 0. 知己知彼：查看支持的事件
在使用 Perf 之前，首先需要知道当前硬件和内核支持哪些性能事件。
```sh
# 列出所有支持的性能事件（Hardware, Software, Tracepoints 等）
perf list
```

## 1. Perf Stat
`perf stat` 是性能分析的**第一步**，它能给出程序的整体性能概览（如 IPC、缓存命中率等），而不产生大量的数据文件。
### 基础性能指标
分析 CPU 周期、指令数以及各级缓存（L1, LLC）的负载与未命中情况：
```sh
sudo perf stat -e cycles,instructions,L1-dcache-load-misses,L1-dcache-loads,LLC-load-misses,LLC-loads,cache-misses,cache-references ./a.out
```
### 操作系统层面指标
关注上下文切换（Context Switches）和缺页异常（Page Faults），这对排查系统调用开销过大或内存抖动非常有用：
```sh
sudo perf stat -e cycles,instructions,cache-misses,cache-references,page-faults,context-switches ./a.out
```
### 内核子系统追踪
利用 Tracepoints 监控内存分配（kmem）和调度器切换（sched）：
```sh
sudo perf stat -e kmem:mm_page_alloc,sched:sched_switch ./a.out
```
### 精准测量技巧：
为了获取更准确的数据，我们通常使用以下参数：
*   `-d` / `-vvv 2>&1 | tee perf.log`: 输出更详细的统计数据（Detailed）。
*   `-r 5`: 重复运行 5 次并计算标准差，排除波动干扰。
*   `-p <PID>`: 挂载到正在运行的进程上。
*   `taskset -c 0`: 绑定 CPU 核心，减少进程迁移带来的缓存失效干扰。

## 2. 寻找热点：Perf Record & Report
> 当 `stat` 告诉你“**性能有问题**”时，`record` 配合 `report` 能告诉你“**哪个函数有问题**”。
### 采样与记录
*   `-g`: 开启调用栈记录（Call Graph），这是分析函数调用关系的关键。
*   `-F 99`: 长期采样设置采样频率为 99Hz（避免与 100Hz 的时钟中断重叠，防止锁步效应），短期采样可以不手动设置采样频率。
*   `-c 10000`: 按事件周期采样（每发生10000次指定事件才采样一次）
```sh
sudo perf record -g ./a.out
```
### 过滤干扰 -e
有时候我们只关心某个特定问题：
```sh
# 只记录用户态周期 (:u)
sudo perf record -e cycles:u ./a.out
# 只记录内核态周期 (:k)
sudo perf record -e cycles:k ./a.out
# 只记录LLC-load-misses
sudo perf record -g -e LLC-load-misses ./a.out
# 显示源码行号
sudo perf report -g -F+period,srcline
```
### report 生成报告
```sh
# 生成文本格式报告并保存
sudo perf report -n --stdio > report.md
```

## 3. 代码级分析：Perf Annotate
如果说 `perf report` 告诉你“**哪个函数**”慢，那么 `perf annotate` 就能告诉你“**哪行代码**”慢，甚至能精确到“**哪条汇编指令**”是瓶颈。
### 指定数据源分析
默认情况下 `perf annotate` 会读取当前目录下的 `perf.data`。但在实际工作中，我们经常需要分析历史数据，或者在服务器录制数据后下载到本地分析。
```sh
# 读取指定的性能数据文件进行汇编级分析
sudo perf annotate -i perf.data
```
### TUI 交互模式
当你在终端运行上述命令时，会进入一个基于 TUI (Text User Interface) 的交互界面。这是专家最常用的模式。
*   **界面解读**：
    *   左侧列：**Percent**，显示该指令采样占总采样的百分比。
    *   中间列：**汇编指令**（如 `mov`, `add`, `cmp` 等）。
    *   右侧/混合：**源代码**（如果编译时带了 `-g` 且源码在路径下）。
*   **常用快捷键**：
    *   `h`：显示帮助菜单。
    *   `H` / `Tab`：循环跳转到最热的指令。
    *   `k`：显示源码的行号。
    *   `Enter`：选中某条指令或函数，查看更详细的跳转来源或定义。
    *   `q` / `Esc`：退出当前视图或返回上一级。
    *   `/`：搜索特定的函数名或汇编指令。
### 过滤与精准定位
在一个庞大的项目中，直接运行 `annotate` 可能会列出所有函数，导致查找困难。我们可以配合过滤器使用：
```sh
# 1. 指定符号（函数名）
sudo perf annotate -i perf.data -s function_name
# 2. 指定动态库/内核模块 (DSO)
sudo perf annotate -i perf.data -d libc.so.6
# 3. 指定内核
sudo perf annotate -i perf.data --vmlinux /boot/vmlinux-$(uname -r)
```
### 导出为文本报告
如果你需要在 CI/CD 流水线中展示，或者习惯用文本编辑器查看，可以使用 `--stdio` 模式：
```sh
# 将汇编级分析结果输出到终端或文件
sudo perf annotate -i perf.data --stdio > annotation.log
# 配合 -n 显示样本数量，而不只是百分比
sudo perf annotate -i perf.data --stdio -n
```
### 技巧分享：
1.  **查找“最长”的条柱**：在 TUI 界面中，百分比最高的行通常是红色的。
2.  **分辨指令类型**：
    *   **高 `cmp` / `test`**：往往意味着分支预测失败或循环次数过多。
    *   **高 `mov`**：通常是 **Cache Miss** 的重灾区。如果一条简单的内存加载指令耗时极高，说明 CPU 在等待内存数据（L3 Miss 甚至内存访问）。
    *   **高 `div` / `sqrt`**：复杂的算术运算指令本身耗时较长。
3.  **源码对照**：
    *   为了获得最佳体验，**编译时务必加上 `-g` 选项**（`gcc -g -O2 ...`）。
    *   如果 `perf` 提示找不到源码，可以在 TUI 中按 `o` 设置源码路径，或者重新编译时使用相对/绝对路径对齐。

## 4. 高级诊断：内存与并发
对于多线程和高性能计算程序，内存访问模式和锁竞争往往是瓶颈所在。
### 内存访问分析
```sh
sudo perf mem record ./a.out
# 随后使用 perf report 查看内存访问详情
```
### 伪共享检测 (False Sharing)
这是多线程编程中的隐形杀手。`perf c2c` (Cache-to-Cache) 可以帮助识别多个核心争抢同一缓存行的情况。
```sh
# -a: 系统范围录制
# sleep 10: 采集 10 秒
sudo perf c2c record -a -- sleep 10
sudo perf c2c report
```
### 调度延迟分析
分析进程等待 CPU 的时间以及 CPU 迁移情况：
```sh
sudo perf sched latency
sudo perf sched migrate
```
### 实时监控：Perf Top
类似于 Linux 的 `top` 命令，但 `perf top` 显示的是消耗 CPU 周期最多的函数，适合实时排查生产环境飙高的问题。
```sh
# 实时显示消耗 cycles 最多的函数，按进程名(comm)和动态库(dso)分类
sudo perf top -e cycles -s comm,dso
# 仅监控特定进程组（如 gcc, clang）的 cache-misses
sudo perf top -e cache-misses --comms gcc,clang
```

## 5. 可视化：火焰图 (Flame Graph)
文本报告虽然详细，但不够直观。Brendan Gregg 发明的火焰图能够将调用栈可视化，快速识别“平顶”的性能瓶颈。
### 安装工具
```sh
git clone https://github.com/brendangregg/FlameGraph.git --depth 1
```
### 绘制流程
1.  **录制**：使用 `dwarf` 格式获取更完整的调用栈（需要 debug info）。
2.  **解析**：将二进制数据转换为文本。
3.  **折叠**：整合相同的调用栈。
4.  **绘制**：生成 SVG 图片。
```sh
# 1. 录制
sudo perf record -F 99 -g --call-graph dwarf ./a.out
# 2-4. 一条龙生成 
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg
# 或者分步执行
# sudo perf script > out.perf
# ./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
# ./FlameGraph/flamegraph.pl out.folded > flame.svg
```
生成后，用浏览器打开 `flame.svg`，横轴越长代表占用 CPU 时间越久，颜色深浅通常无特定含义（或者是随机）。

## 6. 经典补充：Gprof
虽然 `perf` 是非侵入式的系统级分析工具，但老牌的 `gprof` (GNU Profiler) 在源码级分析上依然有一席之地。
### 编译与运行
必须加上 `-pg` 选项：
```sh
g++ -pg -g -O0 test.cpp
time ./a.out
# 运行结束后会生成 gmon.out 文件
```
### 生成报告与可视化
Gprof 的文本报告通常很长，配合 `gprof2dot` 和 `Graphviz` 可以生成直观的调用关系图。
```sh
# 生成文本报告
gprof -q ./a.out > ganalysis.md
# 生成调用关系图 (SVG)
# 简单视图
gprof ./a.out | gprof2dot -s -n 1.0 --skew=1 | dot -Tsvg -o callgraph.svg
# 深度视图 (控制层级和节点间距)
gprof ./a.out | gprof2dot -s -n 1.0 --skew=1 --depth=4 | dot -Tsvg -Granksep=1 -Gnodesep=0.1 -o callgraph_detailed.svg
```

## 一次 perf 示例：
```sh
g++ -g -std=c++20 -mavx2 -mfma Gaussian_Blur.cpp && ./a.out 
sudo perf stat -e cycles,instructions,L1-dcache-load-misses,L1-dcache-loads,LLC-load-misses,LLC-loads,cache-references,cache-misses ./a.out
# 找出代码级问题点
sudo perf report -g -F+period,srcline
# 优化问题
```

> **一些建议与经验**：
> *   `Lambda`应该使用[=]按值捕获__m256 而不是[&]：因为**按值捕获**允许编译器将常量数据永久保留在 `YMM` 寄存器中，从而减少对内存的访问。也防止了潜在的`指针别名`问题，导致每次寄存器必须写回栈内存、每次都从栈内存重新读取，无法高效使用寄存器。
> *   `cache miss` 不能只孤立地看待缓存未命中率，还要考虑绝对数值大小。比如，不同优化下的-O0强制把所有变量都存放在栈内存，而-O3会尽可能的使用寄存器，导致-O0访问L1的次数（分母）明显变大，使未命中率看起来很低，给人一种“-O0的缓存命中概率更高”的假象。即`内存乒乓` (Memory Ping-Pong) 
> *   `高 IPC 不一定代表高性能`！-O0 的 IPC 高，那是因为它在疯狂执行 MOV 和 整数运算 等简单的指令，这些简单指令易于被 CPU 流水线填满，但其实它是在全速运行“垃圾代码”。
> *   不要试图去优化每一行代码。根据帕累托原则， **90% 的性能问题集中在 10% 的代码中**。熟练使用 `perf annotate` 等工具精准定位性能热点，再结合 `perf c2c` 排除并发陷阱，你就能用最小的力气获得最大的性能提升。
> *  ` 火焰图`是向团队和老板展示优化成果的最佳工具。
