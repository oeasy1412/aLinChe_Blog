---
title: 虚拟地址空间
published: 2025-10-22
description: 实现进程的思想
tags: [OS, 内存管理, 线程]
category: OS
draft: false
---
### 引言：进程 = 戴上VR眼镜的线程
在操作系统的世界里，进程被视为资源分配的基本单位，而线程则是执行调度的基本单位。有一个绝妙的比喻可以帮助我们理解它们的关系：**进程 = 戴上VR眼镜的线程**。线程是真正在CPU上执行计算的实体，而它所戴的“VR眼镜”就是我们今天要探讨的核心——**虚拟地址空间（Virtual Address Space）**。这副眼镜让每个进程都以为自己独享整个计算机的内存，看到的是一个独立、连续且巨大的地址空间，而现实是，所有进程都共享着有限的物理内存。

这背后精妙的“VR系统”就是操作系统的虚拟内存管理机制。它不仅实现了进程间的安全隔离，还通过一系列高效的技术，在硬件效率、执行速度和空间利用率之间取得了完美的平衡。本文将带您深入探索这个系统的三大核心支柱：多级页表、写时复制（COW）与内核同页合并（KSM）。

---

### 一、虚拟内存的基石：多级页表（Radix Tree）
虚拟内存的核心任务是实现一个映射函数 `f`，它能将进程看到的“虚拟地址”翻译成物理内存的“真实地址”。
`f: (虚拟页号, 进程ID) -> 物理页帧号`
虚拟页号 Virtual Page Number(VPN), pid -> 物理页帧号 Physical Frame Number(PFN or PPN)

这个映射由硬件（MMU，内存管理单元）和操作系统共同完成，其“寻路图”——**​​页表**​​——存储在CPU的`CR3`寄存器指向的**页表**中。

#### 1. 地址的拆解艺术
在32位系统的经典设计中：**1024叉的二级Radix Tree​**（虚拟地址空间4GB）
*   **地址结构​​**：`32bits = 10 (页目录索引 PDI) + 10 (页表索引 PTI) + 12 (页内偏移 Pages Offset)`
*   **​页大小​**：`2^12 = 4096` 字节，即4KB。
*   **页表结构**​​：采用​​二级页表​​。
    1. **​页目录(Page Directory)​**：由CR3寄存器指向。它是一个4KB的页，包含1024个页目录项(PDE)（因为32位系统一个指针大小为4B）。虚拟地址的高10位(PDI)作为索引在此表中查找。
    2. **​页表(Page Table)**：如果PDE显示下一级页表存在，则用虚拟地址的中间10位(PTI)作为索引，在对应的页表中查找页表项PTE。每个页表也是一个4KB的页，包含1024个PTE。
    3. **​物理页帧(Page Frame)**：PTE中存储着最终的物理页帧号(PFN)，与12位的Pages Offset组合成物理地址。
    这形成了一个非常规整的​​1024叉树（1024-ary Tree）​​。每个页目录/页表正好占一页（4KB），每个项（PDE/PTE）占4字节（32位），完美利用一页空间（1024 * 4B = 4096B）。

在现代64位系统中，一个虚拟地址被这样拆解和使用：
*   **地址结构**：`64bits = 16 (符号扩展) + 36 (虚拟页号 VPN) + 12 (页内偏移 Pages Offset)`
*   **页大小**：`2^12 = 4096` 字节，即4KB。
*   **虚拟页号 (VPN)**：36位的VPN太大，无法用一张表直接映射，因此被进一步拆分。

#### 2. 64位的四级页表：硬件友好的`Radix Tree`
为了高效管理巨大的虚拟地址空间，x86-64架构采用了四级页表结构，它本质上是一种**基数树（Radix Tree）**。36位的VPN被分为4个9位的索引，分别对应四级页表：
*   **PML4** (Page Map Level 4)
*   **PDP** (Page Directory Pointer)
*   **PD** (Page Directory)
*   **PT** (Page Table)

翻译时，MMU从CR3寄存器开始，像查通讯录一样逐级索引，最终将VPN对应的PFN与Offset组合，得到物理地址。这个过程被称为​​页表遍历（Page Table Walk）​​。

> **为什么选择`Radix Tree`？**
> *   **空间效率高**：它只为实际使用的内存区域分配页表项，对于未使用的巨大地址空间，无需占用任何存储，实现了稀疏分配。
> *   **硬件友好**：查询过程是固定的、可预测的 O(k) 复杂度（k为页表级数），硬件实现简单高效。相比之下，哈希表（HashMap）的冲突处理和红黑树（Red-Black Tree）的平衡操作都过于复杂，不适合硬件实现。

#### 3. 加速翻译：`TLB`缓存 —— 硬件实现的地址缓存
页表存储在主内存中，这意味着每次地址翻译都需要多次访问内存（例如，四级页表最多需要4次内存读取），这个过程被称为页表遍历(`Page Table Walk`)，开销很大。为此，CPU芯片内部集成了一个专用于缓存地址映射的纯硬件高速缓存——**TLB(Translation-Lookaside Buffer)**。 它缓存了近期使用过的 `虚拟页号 -> 物理页帧号` 的映射关系。
*   **工作流程**：MMU收到虚拟地址，先查询TLB。若找到缓存（TLB命中），直接获得物理地址；若未命中，才MMU启动页表遍历，并将新映射存入TLB。
*   **进程隔离**：TLB是所有进程共享的硬件资源。为了区分不同进程的地址空间，TLB条目通常会包含一个 **`ASID`（Address Space ID）** 地址空间标识符，这样，即使不同进程使用相同的虚拟地址，硬件也能通过ASID精确匹配，确保进程A不会访问到进程B的缓存映射。上下文切换时，操作系统会更新ASID，从而使TLB能够区分新旧进程的地址映射。
##### TLB 硬件实现
- TLB的快速查询能力源于其核心组件——​​内容可寻址存储器`CAM`(Content-Addressable Memory)（跟Cache一样是一种`全相联/组相联`，比缓存更贵、面积更大、功耗更高）
CAM是根据存储的(key, value)。当输入一个VPN时，TLB内部的硬件会在一瞬间（通常一个时钟周期，依靠硬件）
SRAM存储实际数据
*   **分级**：
    *  L1 TLB​ （每个CPU核心私有，速度极快，与CPU核心同频工作，通常指令(ITLB)和数据(DTLB)分离）
    *  ​L2 TLB​ （大容量，指令和数据通常共享）

> 下面是 AMD Ryzen 5 3500X 6-Core Processor 的cpu相关信息
```sh
cat /proc/cpuinfo | grep -i tlb
    TLB size        : 3072 4K pages  # 所有核心输出相同 4KB页TLB容量​​为3072条目
dmesg | grep -i tlb
[    0.395181] Last level iTLB entries: 4KB 1024, 2MB 1024, 4MB 512
[    0.395183] Last level dTLB entries: 4KB 2048, 2MB 2048, 4MB 1024, 1GB 0
## L1 TLB​​：64项全相联，处理所有页尺寸（含 1GB 大页），专注最低延迟
## L2 TLB​​：1024/2048项组相联， 4KB 页优化更明显，专注高命中率
cpuid -1 &| vim -
## P.S. 现代CPU有从指令性能平衡到数据性能的重心转移趋势，即 dTLB 增强 而 iTLB 可能缩减(对比 AMD R7 8845HS)
## 因为​​数据密集型应用崛起（如视频处理）​，传统的指令密集型负载（如编译）相对减少
   brand = "AMD Ryzen 5 3500X 6-Core Processor             "
   L1 TLB/cache information: 2M/4M pages & L1 TLB (0x80000005/eax):
      instruction # entries     = 0x40 (64)  # iTLB 64项
      instruction associativity = 0xff (255) # 255表示全相联
      data # entries            = 0x40 (64)  # dTLB 64项
      data associativity        = 0xff (255)
   L1 TLB/cache information: 4K pages & L1 TLB (0x80000005/ebx):
      instruction # entries     = 0x40 (64)
      instruction associativity = 0xff (255)
      data # entries            = 0x40 (64)
      data associativity        = 0xff (255)
   L1 data cache information (0x80000005/ecx):
      line size (bytes) = 0x40 (64) # 缓存行大小 64 字节
      lines per tag     = 0x1 (1)
      associativity     = 0x8 (8)   # 8 路组相联
      size (KB)         = 0x20 (32) # 总容量 32KB
   L1 instruction cache information (0x80000005/edx):
      line size (bytes) = 0x40 (64)
      lines per tag     = 0x1 (1)
      associativity     = 0x8 (8)
      size (KB)         = 0x20 (32) # 总容量 32KB
      ## 可计算出CPU总L1容量 = (32KB + 32KB) * 核心数目 = 192KB + 192KB
      ## lscpu | grep "cache"
      ## L1d cache: 192 KiB (6 instances)
      ## L1i cache: 192 KiB (6 instances)
      ## L2 cache:  3 MiB   (6 instances)
      ## L3 cache:  32 MiB  (2 instances)
   L2 TLB/cache information: 2M/4M pages & L2 TLB (0x80000006/eax):
      instruction # entries     = 0x400 (1024) # iTLB 1024项
      instruction associativity = 8-way (6)    # 8 路组相联
      data # entries            = 0x800 (2048) # dTLB 2048项
      data associativity        = 4-way (4)    # 4 路组相联
   L2 TLB/cache information: 4K pages & L2 TLB (0x80000006/ebx):
      instruction # entries     = 0x400 (1024) 
      instruction associativity = 8-way (6)    # 8 路组相联
      data # entries            = 0x800 (2048)
      data associativity        = 8-way (6)    # 8 路组相联

NASID: number of address space identifiers = 0x8000 (32768):
   L1 TLB information: 1G pages (0x80000019/eax):
      instruction # entries     = 0x40 (64) # iTLB 64项
      instruction associativity = full (15) # 全相联
      data # entries            = 0x40 (64) # dTLB 64项
      data associativity        = full (15) # 全相联
   L2 TLB information: 1G pages (0x80000019/ebx):
      instruction # entries     = 0x0 (0)   # 不支持
      instruction associativity = L2 off (0)
      data # entries            = 0x0 (0)   # 不支持
      data associativity        = L2 off (0)
```

#### 4. 页表项控制位详解
无论是32位的二级页表还是64位的四级页表，其核心组成部分——​​​​页目录项PDE和页表项PTE​​​​——不仅仅只存储物理页帧号PFN。它们还包含一组至关重要的​​控制位（`Control Bits`）​​，用于管理内存访问权限、跟踪页面状态以及辅助硬件和操作系统进行高效的内存管理。这些位通常占据PDE/PTE的低位部分（Least Significant Bits, LSBs）

*   一个页的大小是 4KB，也就是 `2^12` = 4096 字节。
*   这意味着任何一个物理页帧(Page Frame)的起始地址，其**低12位必然全为0**。
既然物理页帧地址的低12位永远是0，那么在PTE里存储这12个0就是一种浪费。硬件设计师们利用了这一点让PTE剩余的12位作为控制位。

0: Present (P) - **有效位**

1: Write Enable (W) / Read-Only (R/W) - **写控制位**

2: User/Supervisor (U/S) - **访问权限位**

3: Page-Level Write-Through (PWT) - **缓存写策略**

4: Page-Level Cache Disable (PCD) - **缓存策略**

5: Accessed (A) - **访问位**

6: Dirty (D) - **脏位** (PTE特有)

7: Page Size (PS) - **页大小** (PDE特有，0=4KB, 1=4MB/2MB)

8: Global (G) - **全局页** (TLB相关)

9-11: Available for OS use (Avail) - **操作系统专用位** (如是否锁定内存、页面类型、自定义内存管理策略的信息)

##### 在哪个硬件上实现Control Bits的
实现和使用这些控制位的核心硬件是 **CPU内部的内存管理单元（MMU）**。

MMU的工作流程是这样的：
1.  **接收虚拟地址**：CPU要执行一条指令，比如 `MOV EAX, [0x12345678]`，MMU接收到虚拟地址 `0x12345678`。
2.  **查询TLB**：如果TLB中存在该虚拟页的映射，硬件会立即返回物理页帧号，整个翻译过程在单个时钟周期内完成，无需访问主内存。如果TLB中没有缓存，从CR3寄存器开始逐级遍历页表
3.  **页表遍历 (Page Table Walk)**：MMU从`CR3`寄存器开始，逐级查找页目录和页表。
4.  **读取并解析PTE**：当MMU最终找到对应的PTE时，它不会把它当作一个普通的数字。它会：
    *   **检查控制位**：
        *   检查**Present位**。如果是0，说明页面不在内存中，MMU会立即停止翻译，并触发一个**缺页异常（Page Fault）**，通知操作系统介入。
        *   检查**权限位 (R/W, U/S)**。如果当前是用户态程序试图写入一个只读页面（R/W=0），或者试图访问一个内核页面（U/S=0），MMU会触发一个**保护异常（Protection Fault）**。
        *   ...等等，它会检查所有相关的控制位。
    *   **更新状态位**：如果访问被允许，MMU会自动在硬件层面**设置**某些位。比如，只要对这个页面进行了读或写访问，MMU就会将**Accessed位**设置为1。如果进行了写操作，MMU会将**Dirty位**设置为1。
    *   **计算物理地址**：如果所有检查都通过，MMU会提取PTE中的**PFN（高20位）**，与虚拟地址中的**页内偏移（低12位）** 组合起来，形成最终的物理地址，然后发送到内存总线。

---

### 二、效率的魔法：写时复制（Copy-on-Write）
`fork()` 系统调用是Unix/Linux的基石，它能快速创建新进程。如果每次`fork()`都完整复制父进程的全部内存，开销将无法承受。**写时复制(COW)**技术巧妙地解决了这个问题。

#### 1. COW核心思想
1.  **共享读取**：`fork()`时，子进程并不复制父进程的物理内存，而是复制父进程页表的副本，父子进程拥有各自独立的页表结构(PML4, PDP, PD等)，但页表项(PTE)指向的是与父进程相同的物理页帧，并将这些共享页面的PTE标记为**只读**。(​​“虚假”复制​​)
2.  **按需复制**：当父进程或子进程尝试**写入**某个共享页面时，会触发一个**缺页异常（Page Fault）**。
3.  **触发复制**：内核的缺页异常处理程序（如`do_wp_page`）被调用，此时才真正为写入方分配一个新的物理页面，复制原页面的内容，并更新其页表映射为可写。(“真实”复制​​Pages)

COW将可能不必要的内存复制推迟到最后一刻（真正需要写入时），这种延迟复制的策略，极大地提升了进程创建效率fork()的速度并减少了内存占用。

#### 2. 代码揭秘：`do_wp_page` 函数解析
在Linux内核中，处理写保护缺页异常的核心逻辑位于 `do_wp_page` 等函数中。让我们来看看其简化逻辑：
```c
// 简化逻辑
vm_fault_t do_wp_page(struct vm_fault* vmf) {
    struct page* old_page = vmf->page;
    // 关键决策点：这个页面是否被多个进程或映射共享？
    // page_count() 返回页面的总引用计数
    if (page_count(old_page) == 1) {
        // 如果页面是唯一的（只有当前进程在用）则无需复制，直接将页面标记为可写即可
        wp_page_reuse(vmf);    // 直接重用页面
        return VM_FAULT_WRITE; // 返回写错误处理结果
    }
    // 如果页面被共享 (page_count() > 1)
    // 则必须执行复制操作
    return wp_page_copy(vmf);
}

// 完整逻辑
static vm_fault_t do_wp_page(struct vm_fault* vmf) __releases(vmf->ptl) {
    // 获取触发页错误的虚拟内存区域(VMA)
    struct vm_area_struct* vma = vmf->vma;

    // 检查是否是userfaultfd的写保护页错误
    if (userfaultfd_pte_wp(vma, *vmf->pte)) {
        pte_unmap_unlock(vmf->pte, vmf->ptl);
        return handle_userfault(vmf, VM_UFFD_WP);
    }
    if (unlikely(userfaultfd_wp(vmf->vma) && mm_tlb_flush_pending(vmf->vma->vm_mm)))
        flush_tlb_page(vmf->vma, vmf->address); // 处理userfaultfd延迟的TLB刷新，刷新特定页面的TLB条目

    // 获取与虚拟地址对应的物理页描述符
    vmf->page = vm_normal_page(vma, vmf->address, vmf->orig_pte);
    if (!vmf->page) {
        // 如果是共享可写映射的特殊页，直接标记为可写
        if ((vma->vm_flags & (VM_WRITE | VM_SHARED)) == (VM_WRITE | VM_SHARED))
            return wp_pfn_shared(vmf);
        // 否则进行正常的写时复制
        pte_unmap_unlock(vmf->pte, vmf->ptl);
        return wp_page_copy(vmf);
    }

    // 优先处理匿名页（Anonymous Pages）
    if (PageAnon(vmf->page)) {
        struct page* page = vmf->page;
        // 检查是否KSM页面或存在多个引用
        if (PageKsm(page) || page_count(page) != 1)
            goto copy; // 需要复制
        if (!trylock_page(page))
            goto copy; // 锁定失败则复制
        // 再次检查页面状态
        if (PageKsm(page) || page_mapcount(page) != 1 || page_count(page) != 1) {
            unlock_page(page);
            goto copy; // 状态变化需要复制
        }
        // 满足重用条件：唯一映射+唯一引用+已锁定
        unlock_page(page);
        wp_page_reuse(vmf);    // 直接重用页面
        return VM_FAULT_WRITE; // 返回写错误处理结果
    }
    // 处理共享可写的文件映射页
    else if (unlikely((vma->vm_flags & (VM_WRITE | VM_SHARED)) == (VM_WRITE | VM_SHARED))) {
        return wp_page_shared(vmf); // 特殊处理共享文件映射
    }

copy:
    get_page(vmf->page); // 增加页面引用计数
    pte_unmap_unlock(vmf->pte, vmf->ptl); // 释放页表锁
    return wp_page_copy(vmf); // 执行实际的页面复制
}
```

从代码中可以看出，内核会检查页面的引用计数。如果发现这个只读页面实际上只有一个使用者，就没必要大费周章地复制它，直接把它变成可写状态即可。这是一种极致的优化，只有在页面确实被共享且现在需要写入（例如，`fork`后父子进程都还存在）的情况下，才会执行真正的复制。

#### 3. 零页优化：极致的空间节省艺术
​​问题背景​​：进程的`BSS段(Block Started by Symbol)`（未初始化的全局/静态变量，在磁盘上​​不占用空间）和匿名映射的只读页，其初始内容通常全为0。如果为每个这样的页面都分配一个物理帧并用0填充，将造成巨大的内存浪费，尤其是对于大量fork()出的子进程。
*   **零页优化**：内核预留一个特殊的、内容全为0的物理页面，称为​​零页​​。当需要建立一个映射，且要求页面内容初始化为0时，页表项PTE并不指向一个新分配的物理页，而是指向这个​​全局唯一的零页​​，并标记为​​只读​​。
*   写入时：分配新页->清零->更新PTE，(清零由页分配器高效完成，比memcpy内存复制O(n)高效，仅内存分配​)​。这也是为什么有人常说全局的int vec[1e7];默认分配为全0而不用手动初始化(doge)

---

### 三、空间的艺术：内核同页合并（KSM）
在虚拟化和容器化场景中，多个虚拟机或容器可能运行着相同的操作系统和应用程序，导致内存中存在大量内容完全相同的页面。**KSM(Kernel Samepage Merging)**正是为此而生的内存去重技术。

#### 1. KSM工作原理

KSM的核心是一个名为 `ksmd` 的内核守护进程，它会周期性地扫描进程声明可以合并的内存区域，寻找内容相同的页面，并将它们合并为一个物理页面。这个共享页面会被标记为**写时复制（COW）**。当任何一个进程尝试写入该共享页时，会触发缺页异常，系统会为其分配一个新页面，从而实现“透明”的去重。

#### 2. 高效管理：稳定树与不稳定树

为了快速查找和管理这些页面，KSM在内部维护了两棵**红黑树**：

*   **稳定树 (Stable Tree)**：存储已经被合并且内容稳定的共享页面。这些页面是写保护的，内容可靠。
*   **不稳定树 (Unstable Tree)**：用作一个临时区域，存放那些内容可能随时变化的候选页面。每次扫描时，新页面会先在这里进行比较和配对，成功合并后，其信息会被移入稳定树。

这种双树结构，既保证了已合并页面的稳定性，又高效地处理了不断变化的候选页面。

#### 3. 监控与调优

管理员可以通过`sysfs`文件系统对KSM进行精细的控制和监控。

```sh
# 查看KSM状态 (默认是不开启的)
# /bin/bash
for f in /sys/kernel/mm/ksm/*; do
    printf "%-25s: %s\n" "$(basename $f)" "$(cat $f)";
done
```
通过读写 `/sys/kernel/mm/ksm/` 目录下的文件，可以启动/停止`ksmd`（`run`文件）、调整扫描速度（`pages_to_scan`）和扫描周期（`sleep_millisecs`），并查看合并成果（`pages_shared`, `pages_sharing`等）。

---

### 结论：设计哲学的胜利
现代操作系统的虚拟内存管理，是一套在**硬件效率（Radix Tree）、速度（TLB）和空间利用率（稀疏分配、COW、KSM）**之间取得精妙平衡的系统。它遵循着UNIX“少即是多”的哲学，通过分层和优化的设计，为上层应用提供了一个看似简单、实则功能强大的内存环境。

从为每个进程戴上隔离的“VR眼镜”，到通过写时复制和内存合并等技术在幕后精打细算，虚拟内存机制无疑是现代计算体系中最伟大的工程奇迹之一。